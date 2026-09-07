# Bài 5.3: Dart FFI — Zero-overhead C/Rust Integration

> **Cấp độ**: Staff Engineer  
> **Thời gian đọc**: ~35 phút  
> **Yêu cầu**: Biết cơ bản C/Rust; đã đọc Bài 5.1

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Mã hóa AES cần native speed

**Fintech app, mã hóa dữ liệu nhạy cảm:**

```
Yêu cầu: Mã hóa 10MB file trước khi upload (AES-256-GCM)

Pure Dart (pointycastle):
  Thời gian: 2,800ms — QUÁ CHẬM (user thấy rõ)

MethodChannel + Android Crypto API:
  Thời gian: 380ms — OK
  Nhưng: 2 code path khác nhau (Android Keystore vs iOS SecKey)
  → Double maintenance cost

Dart FFI + libsodium (cross-platform C library):
  Thời gian: 45ms — NATIVE SPEED
  Một codebase: Dart FFI binding → dùng được trên cả Android và iOS
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. Dart FFI — Cách hoạt động

```
Dart Code:
  final result = nativeEncrypt(dataPtr, dataLen, keyPtr);
          │
          │ Direct function call (không qua JNI, không serialize)
          ▼
C Library (libsodium.so / libsodium.dylib):
  crypto_aead_xchacha20poly1305_ietf_encrypt(...)
          │
          │ return value (int, pointer)
          ▼
  result in Dart

KHÁC BIỆT với MethodChannel:
  MethodChannel: Dart → serialize → JNI → Platform Thread → JNI → deserialize → Dart
                 Latency: 12-28ms
                 
  FFI:           Dart → C function call (trực tiếp trong process)
                 Latency: 0.1-0.5ms
                 
  Nhưng FFI yêu cầu:
  - Quản lý memory thủ công (malloc/free)
  - Không tự động handle platform thread
  - Phải ship native library cho mỗi ABI
```

### 2.2. Dart FFI Type System

```
Dart Type          → C Type (dart:ffi)
──────────────────────────────────────────────────
int                → Int8/Int16/Int32/Int64
                     Uint8/Uint16/Uint32/Uint64
double             → Float/Double
bool               → Bool
Pointer<T>         → T* (pointer to T)
Pointer<Utf8>      → char* (null-terminated string)
Pointer<Void>      → void*
Array<T>           → T[] (fixed-size array)
Struct             → struct (C struct, value type)
Union              → union (C union)
NativeFunction<F>  → C function pointer

Ví dụ mapping:
  C:    int crypto_hash(uint8_t* out, const uint8_t* in, uint64_t inlen);
  Dart: int Function(Pointer<Uint8>, Pointer<Uint8>, int)
```

### 2.3. Memory Management với Arena Allocator

```
Dart GC quản lý Dart objects
C malloc quản lý native heap

Vấn đề: Pointer<Uint8> được cấp phát bằng calloc.allocate()
         Nếu không free → Native memory leak (không được Dart GC thu hồi)

Giải pháp: Arena allocator — tự động free khi scope kết thúc

using((Arena arena) {
  final ptr = arena.allocate<Uint8>(1024); // Cấp phát 1KB
  // ... sử dụng ptr ...
  // Arena tự động free ptr khi function này kết thúc
});
// ptr đã bị free — không memory leak
```

---

## Phần 3 — Production Code Implementation

### 3.1. Tích hợp libsodium qua FFI

```bash
# 1. Compile libsodium cho Android (arm64-v8a, armeabi-v7a, x86_64)
# Script: scripts/build_libsodium.sh
NDK_PATH=$ANDROID_NDK_HOME

for ABI in arm64-v8a armeabi-v7a x86_64; do
  ./configure --host=aarch64-linux-android \
    --prefix=$PWD/build/$ABI \
    --disable-shared \
    --enable-static
  make -j4 install
done

# 2. Copy vào Flutter project
cp build/arm64-v8a/lib/libsodium.a \
   android/app/src/main/jniLibs/arm64-v8a/libsodium.a
```

```dart
// lib/core/ffi/sodium_bindings.dart
// FFI bindings cho libsodium

import 'dart:ffi';
import 'dart:io';
import 'package:ffi/ffi.dart';

// Khai báo C struct (nếu API cần)
final class SodiumInitState extends Struct {
  @Int32()
  external int version;
  
  @Int64()
  external int nonce;
}

// Khai báo function signatures
typedef _SodiumInitC = Int32 Function();
typedef _SodiumInitDart = int Function();

typedef _EncryptC = Int32 Function(
  Pointer<Uint8> ciphertext,
  Pointer<Uint64> ciphertextLen,
  Pointer<Uint8> message,
  Uint64 messageLen,
  Pointer<Uint8> additionalData,
  Uint64 additionalDataLen,
  Pointer<Uint8> nsec,
  Pointer<Uint8> npub,   // nonce
  Pointer<Uint8> k,      // key
);

typedef _EncryptDart = int Function(
  Pointer<Uint8> ciphertext,
  Pointer<Uint64> ciphertextLen,
  Pointer<Uint8> message,
  int messageLen,
  Pointer<Uint8> additionalData,
  int additionalDataLen,
  Pointer<Uint8> nsec,
  Pointer<Uint8> npub,
  Pointer<Uint8> k,
);

/// Bindings singleton — load library 1 lần
final class SodiumBindings {
  SodiumBindings._() {
    // Load library theo platform
    _lib = _loadLibrary();
    _init = _lib.lookupFunction<_SodiumInitC, _SodiumInitDart>(
      'sodium_init',
    );
    _encrypt = _lib.lookupFunction<_EncryptC, _EncryptDart>(
      'crypto_aead_xchacha20poly1305_ietf_encrypt',
    );
  }

  static SodiumBindings? _instance;
  static SodiumBindings get instance => _instance ??= SodiumBindings._();
  
  late final DynamicLibrary _lib;
  late final _SodiumInitDart _init;
  late final _EncryptDart _encrypt;
  
  bool _initialized = false;
  
  void initialize() {
    if (_initialized) return;
    final result = _init();
    if (result == -1) throw StateError('sodium_init() thất bại');
    _initialized = true;
  }

  static DynamicLibrary _loadLibrary() {
    if (Platform.isAndroid) {
      return DynamicLibrary.open('libsodium.so');
    } else if (Platform.isIOS) {
      // iOS: static link vào app binary
      return DynamicLibrary.process();
    } else if (Platform.isMacOS) {
      return DynamicLibrary.open('/usr/local/lib/libsodium.dylib');
    }
    throw UnsupportedError('Platform không được hỗ trợ');
  }
  
  // Constants
  static const keyBytes = 32; // 256 bits
  static const nonceBytes = 24;
  static const macBytes = 16; // Authentication tag

  /// Encrypt data — returns ciphertext (authenticated)
  Uint8List encrypt({
    required Uint8List message,
    required Uint8List key,
    required Uint8List nonce,
    Uint8List? additionalData,
  }) {
    if (key.length != keyBytes) {
      throw ArgumentError('Key phải đúng $keyBytes bytes');
    }
    if (nonce.length != nonceBytes) {
      throw ArgumentError('Nonce phải đúng $nonceBytes bytes');
    }
    
    // Cấp phát buffer cho ciphertext (message + MAC)
    final ciphertextLength = message.length + macBytes;
    
    // Arena = auto-free native memory khi block kết thúc
    return using((Arena arena) {
      final ciphertextPtr = arena.allocate<Uint8>(ciphertextLength);
      final ciphertextLenPtr = arena.allocate<Uint64>(1);
      final messagePtr = arena.allocate<Uint8>(message.length);
      final keyPtr = arena.allocate<Uint8>(keyBytes);
      final noncePtr = arena.allocate<Uint8>(nonceBytes);
      
      // Copy Dart data vào native heap
      messagePtr.asTypedList(message.length).setAll(0, message);
      keyPtr.asTypedList(keyBytes).setAll(0, key);
      noncePtr.asTypedList(nonceBytes).setAll(0, nonce);
      
      Pointer<Uint8> adPtr = nullptr;
      int adLen = 0;
      if (additionalData != null && additionalData.isNotEmpty) {
        final ad = arena.allocate<Uint8>(additionalData.length);
        ad.asTypedList(additionalData.length).setAll(0, additionalData);
        adPtr = ad;
        adLen = additionalData.length;
      }
      
      // Gọi C function trực tiếp
      final result = _encrypt(
        ciphertextPtr,
        ciphertextLenPtr,
        messagePtr,
        message.length,
        adPtr,
        adLen,
        nullptr,  // nsec = null cho IETF variant
        noncePtr,
        keyPtr,
      );
      
      if (result != 0) throw StateError('Encryption thất bại: $result');
      
      // Copy kết quả từ native heap về Dart — sau đây native memory được free
      final actualLength = ciphertextLenPtr.value;
      return Uint8List.fromList(
        ciphertextPtr.asTypedList(actualLength),
      );
    });
  }
}
```

### 3.2. Tích hợp Rust library qua cbindgen

```rust
// crypto/src/lib.rs — Rust library
use std::slice;
use ring::aead::{Aad, LessSafeKey, Nonce, UnboundKey, AES_256_GCM};

/// C-compatible API cho Dart FFI
/// cbindgen sẽ tạo header file từ các function này
#[no_mangle]
pub extern "C" fn encrypt_aes256gcm(
    plaintext: *const u8,
    plaintext_len: usize,
    key: *const u8,   // 32 bytes
    nonce: *const u8, // 12 bytes
    output: *mut u8,  // Must be plaintext_len + 16 bytes
) -> i32 {
    // Safety: Dart đảm bảo pointer valid và length đúng
    let plaintext_slice = unsafe { slice::from_raw_parts(plaintext, plaintext_len) };
    let key_slice = unsafe { slice::from_raw_parts(key, 32) };
    let nonce_slice = unsafe { slice::from_raw_parts(nonce, 12) };
    let output_slice = unsafe { slice::from_raw_parts_mut(output, plaintext_len + 16) };
    
    let unbound_key = match UnboundKey::new(&AES_256_GCM, key_slice) {
        Ok(k) => k,
        Err(_) => return -1,
    };
    
    let key = LessSafeKey::new(unbound_key);
    let nonce = match Nonce::try_assume_unique_for_key(nonce_slice) {
        Ok(n) => n,
        Err(_) => return -2,
    };
    
    let mut in_out = plaintext_slice.to_vec();
    match key.seal_in_place_append_tag(nonce, Aad::empty(), &mut in_out) {
        Ok(()) => {
            output_slice[..in_out.len()].copy_from_slice(&in_out);
            in_out.len() as i32
        }
        Err(_) => -3,
    }
}

#[no_mangle]
pub extern "C" fn rust_free(ptr: *mut u8, len: usize) {
    // Cho phép Dart free memory được cấp phát bởi Rust
    if !ptr.is_null() {
        unsafe {
            Vec::from_raw_parts(ptr, len, len);
        }
    }
}
```

```bash
# Compile Rust → .so/.a cho iOS và Android
cargo build --release --target aarch64-apple-ios     # iOS
cargo build --release --target aarch64-linux-android # Android arm64
cargo build --release --target x86_64-linux-android  # Android x86_64

# Generate C header với cbindgen
cbindgen --config cbindgen.toml --crate crypto --output include/crypto.h
```

### 3.3. Dart FFI binding cho Rust library

```dart
// lib/core/ffi/rust_crypto_bindings.dart
typedef _EncryptRustC = Int32 Function(
  Pointer<Uint8> plaintext, IntPtr plaintextLen,
  Pointer<Uint8> key, Pointer<Uint8> nonce,
  Pointer<Uint8> output,
);

typedef _EncryptRustDart = int Function(
  Pointer<Uint8> plaintext, int plaintextLen,
  Pointer<Uint8> key, Pointer<Uint8> nonce,
  Pointer<Uint8> output,
);

final class RustCryptoBindings {
  RustCryptoBindings._();
  static final instance = RustCryptoBindings._();
  
  late final _EncryptRustDart _encryptAes256gcm;
  
  void initialize() {
    final lib = Platform.isIOS
        ? DynamicLibrary.process()
        : DynamicLibrary.open('libcrypto_rust.so');
    
    _encryptAes256gcm = lib.lookupFunction<_EncryptRustC, _EncryptRustDart>(
      'encrypt_aes256gcm',
    );
  }
  
  Future<Uint8List> encryptInIsolate(
    Uint8List plaintext,
    Uint8List key,
    Uint8List nonce,
  ) async {
    // FFI calls PHẢI chạy trên isolate vì:
    // 1. CPU-intensive → block main thread nếu lớn
    // 2. Native code không có isolate constraint
    return Isolate.run(() => _encryptSync(plaintext, key, nonce));
  }
  
  Uint8List _encryptSync(Uint8List plaintext, Uint8List key, Uint8List nonce) {
    final outputLen = plaintext.length + 16; // + GCM tag
    
    return using((arena) {
      final plaintextPtr = arena.allocate<Uint8>(plaintext.length)
        ..asTypedList(plaintext.length).setAll(0, plaintext);
      final keyPtr = arena.allocate<Uint8>(32)
        ..asTypedList(32).setAll(0, key);
      final noncePtr = arena.allocate<Uint8>(12)
        ..asTypedList(12).setAll(0, nonce);
      final outputPtr = arena.allocate<Uint8>(outputLen);
      
      final resultLen = _encryptAes256gcm(
        plaintextPtr, plaintext.length,
        keyPtr, noncePtr,
        outputPtr,
      );
      
      if (resultLen < 0) throw StateError('Rust encryption failed: $resultLen');
      
      return Uint8List.fromList(outputPtr.asTypedList(resultLen));
    });
  }
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Benchmark: Pure Dart vs MethodChannel vs FFI

```
Test: AES-256-GCM encrypt 10MB file
Device: iPhone 14 Pro (A16 Bionic), Profile mode

Pure Dart (pointycastle):     2,840ms  (baseline, slow)
MethodChannel + iOS SecKey:     385ms  (10x faster)
Dart FFI + libsodium:            48ms  (59x faster vs Dart, 8x vs MethodChannel)
Dart FFI + Rust ring:            42ms  (similar to libsodium)

Dart FFI overhead vs raw C call:
  Raw C (measured via benchmark): 41ms
  Dart FFI call overhead:          1ms  (<2.5%)
  → "Zero-overhead" trong thực tế: overhead < 3%

Memory overhead (FFI vs MethodChannel):
  MethodChannel: serialize (copy 10MB) + native process + deserialize (copy 10MB)
                 Peak: 30MB extra
  FFI:           arena.allocate (1 copy, 10MB) + output (10MB)
                 Peak: 20MB extra
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ FFI pointer không được free sau khi dùng
    ✅ Luôn dùng Arena (using block) hoặc malloc/calloc.free()
    Lý do: Native heap không có GC — mỗi allocate không free = native memory leak

[ ] ❌ Gọi CPU-intensive FFI trực tiếp trên main isolate
    ✅ Wrap trong Isolate.run(() => nativeCall())
    Lý do: FFI không block Dart thread nhưng chiếm CPU → frame budget ảnh hưởng

[ ] ❌ Không validate pointer null trước khi dereference
    ✅ Kiểm tra pointer != nullptr; xử lý return code lỗi từ C function
    Lý do: Null pointer dereference trong C → SIGFAULT → app crash (không có stack trace)

[ ] ❌ Ship libsodium.so không được stripped (có debug symbols)
    ✅ Compile với --release flag, strip debug symbols: strip -s libsodium.so
    Lý do: Debug symbols thêm 5-15MB vào app size

[ ] ❌ Không kiểm tra FFI trên thiết bị 32-bit (armeabi-v7a)
    ✅ Test trên cả arm64 và armeabi-v7a
    Lý do: Pointer size khác nhau (4 vs 8 bytes) → bug tinh vi trên 32-bit

[ ] ❌ Dùng FFI cho API có callback phức tạp (async event loop)
    ✅ FFI cho synchronous C functions; Platform Channel cho event-driven API
    Lý do: C callbacks chạy ngoài Dart isolate → không thể gọi Dart code trực tiếp

[ ] ❌ Không isolate C library initialization (sodium_init) khỏi hot path
    ✅ Initialize 1 lần trong app startup, cache singleton
    Lý do: sodium_init() mất 10-50ms (setup random number generator)
```
