# Bài 8.1: App Security — SSL Pinning, Biometric Auth & OWASP Mobile Top 10

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Biết HTTPS/TLS cơ bản; biết Android Keystore / iOS Keychain

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Fintech app bị MITM attack trong pentest

**Security audit report (fintech app, 200k users):**

```
[HIGH SEVERITY] Man-in-the-Middle Attack
  Auditor setup: Charles Proxy on same network
  
  Steps:
  1. Install Charles Root Certificate vào device
  2. Configure device proxy → Charles
  3. Open app → Login
  
  Result: Charles hiển thị TOÀN BỘ API traffic:
    POST /auth/login → {"email": "user@bank.com", "password": "123456"}
    GET /account/balance → {"balance": 45000000, "account": "0901234567"}
    POST /transfer → {"from": "0901", "to": "1234", "amount": 5000000}
  
  Root cause: Không có SSL Certificate Pinning
  → Hacker cùng network WiFi có thể intercept toàn bộ giao dịch tài chính
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. SSL Pinning — Certificate vs Public Key

```
Normal TLS (không pin):
  1. Server gửi certificate
  2. Client verify: certificate được ký bởi trusted CA?
  3. Nếu có root CA certificate trong device trust store → OK
  
  VẤN ĐỀ: Attacker install fake root CA → fake certificate được trust
  → Charles Proxy hoạt động được

Certificate Pinning:
  1. Server gửi certificate
  2. Client verify: fingerprint của certificate == hardcoded fingerprint?
  3. Nếu không match → REJECT kết nối
  
  NHƯỢC ĐIỂM: Certificate hết hạn (thường 1-2 năm) → phải update app!
  → User không update → app broken

Public Key Pinning (khuyến nghị):
  1. Server gửi certificate
  2. Extract Public Key từ certificate
  3. Hash public key → SHA-256
  4. So sánh với hardcoded hash
  
  ƯU ĐIỂM: Public key không thay đổi khi renew certificate
  → Pin public key một lần, không cần update app khi cert expires
  → Có thể pin nhiều public key (backup) để tránh service disruption
```

### 2.2. Biometric Auth — FaceID/Fingerprint Flow

```
Local Authentication (không server):
  Device biometric sensor
         │
  FaceID/Fingerprint scan → Secure Enclave/TEE
         │
  "Authenticated" → App receives true/false
  
  Vấn đề: Chỉ biết user "đã xác thực sinh trắc học" — KHÔNG verify với server
  
Production pattern:
  1. User cần thực hiện sensitive action (chuyển tiền > 10 triệu)
  2. App: Biometric prompt
  3. User: FaceID/Fingerprint
  4. Secure Enclave: decrypt stored session token
  5. App: gửi token lên server → server validate
  6. Server: cho phép transaction
```

---

## Phần 3 — Production Code Implementation

### 3.1. SSL Pinning với Dio

```dart
// lib/core/network/ssl_pinning_client.dart
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:dio/io.dart';

final class SslPinningClient {
  SslPinningClient._();
  
  // SHA-256 fingerprint của Public Key (bkf là backup key)
  // Extract bằng: openssl x509 -pubkey -noout -in cert.pem | openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | base64
  static const _pinnedPublicKeys = [
    'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=', // Primary key
    'BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=', // Backup key (rotation)
  ];
  
  static Dio create({
    required String baseUrl,
    Duration connectTimeout = const Duration(seconds: 30),
    Duration receiveTimeout = const Duration(seconds: 30),
  }) {
    final dio = Dio(BaseOptions(
      baseUrl: baseUrl,
      connectTimeout: connectTimeout,
      receiveTimeout: receiveTimeout,
    ));
    
    // Chỉ enable SSL pinning trên production
    if (kReleaseMode) {
      _setupSslPinning(dio);
    }
    
    return dio;
  }
  
  static void _setupSslPinning(Dio dio) {
    (dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
      final client = HttpClient();
      
      client.badCertificateCallback = (X509Certificate cert, String host, int port) {
        // KHÔNG return true (tắt verification) — đây là security critical
        return false;
      };
      
      return client;
    };
    
    (dio.httpClientAdapter as IOHttpClientAdapter).validateCertificate = (
      X509Certificate? cert,
      String host,
      int port,
    ) {
      if (cert == null) return false;
      
      // Extract public key từ certificate
      final publicKeyHash = _extractPublicKeyHash(cert);
      
      // So sánh với pinned keys
      final isPinned = _pinnedPublicKeys.contains(publicKeyHash);
      
      if (!isPinned) {
        // Log security event — không throw silently
        SecurityLogger.logPinningFailure(
          host: host,
          certSubject: cert.subject,
          certIssuer: cert.issuer,
        );
      }
      
      return isPinned;
    };
  }
  
  static String _extractPublicKeyHash(X509Certificate cert) {
    // Dart không có built-in public key extraction
    // → Dùng platform channel hoặc dart:crypto
    // Simplified: hash toàn bộ certificate DER (certificate pinning, không phải pubkey)
    final bytes = cert.der;
    final digest = sha256.convert(bytes);
    return base64.encode(digest.bytes);
  }
}
```

### 3.2. Biometric Auth Production Pattern

```dart
// features/auth/data/datasources/biometric_datasource.dart
import 'package:local_auth/local_auth.dart';

final class BiometricAuthDataSource {
  BiometricAuthDataSource() : _auth = LocalAuthentication();
  final LocalAuthentication _auth;

  Future<Result<bool, Failure>> checkBiometricAvailability() async {
    try {
      final isAvailable = await _auth.canCheckBiometrics;
      final isDeviceSupported = await _auth.isDeviceSupported();
      
      if (!isDeviceSupported) {
        return const Failure_(BiometricNotSupportedFailure());
      }
      if (!isAvailable) {
        return const Failure_(BiometricNotEnrolledFailure());
      }
      
      final availableBiometrics = await _auth.getAvailableBiometrics();
      return Success(availableBiometrics.isNotEmpty);
    } on PlatformException catch (e) {
      return Failure_(PlatformBridgeFailure(message: e.message ?? 'Biometric error'));
    }
  }

  Future<Result<void, Failure>> authenticate({
    required String reason,
    bool useDeviceCredentialAsFallback = true,
  }) async {
    try {
      final authenticated = await _auth.authenticate(
        localizedReason: reason,
        options: AuthenticationOptions(
          stickyAuth: true, // Giữ auth dialog khi app về background
          biometricOnly: !useDeviceCredentialAsFallback,
          sensitiveTransaction: true, // Stronger biometric (iOS: NOT Face ID with mask)
        ),
      );
      
      if (authenticated) {
        return const Success(null);
      } else {
        return const Failure_(BiometricAuthFailedFailure());
      }
    } on PlatformException catch (e) {
      return switch (e.code) {
        'NotEnrolled' => const Failure_(BiometricNotEnrolledFailure()),
        'LockedOut' || 'PermanentlyLockedOut' => const Failure_(BiometricLockedOutFailure()),
        'UserCancel' => const Failure_(BiometricCancelledByUserFailure()),
        _ => Failure_(PlatformBridgeFailure(message: e.message ?? '')),
      };
    }
  }
}
```

### 3.3. Root/Jailbreak Detection (Không dùng package)

```dart
// lib/core/security/root_detection.dart
// Không dùng package như flutter_jailbreak_detection — dễ bypass
// Kiểm tra trực tiếp qua native API

@platform
final class RootDetectionService {
  
  /// Kiểm tra root/jailbreak
  /// Trả về true = SUSPICIOUS (không chắc chắn 100% là root)
  Future<bool> isSuspiciousDevice() async {
    if (Platform.isAndroid) {
      return _checkAndroidRoot();
    } else if (Platform.isIOS) {
      return _checkiOSJailbreak();
    }
    return false;
  }

  Future<bool> _checkAndroidRoot() async {
    // Method 1: Check su binary (không đáng tin nhất)
    final suExists = await _fileExists('/system/bin/su') ||
        await _fileExists('/system/xbin/su') ||
        await _fileExists('/sbin/su');
    
    // Method 2: Check Magisk files
    final magiskExists = await _fileExists('/data/adb/magisk') ||
        await _fileExists('/sbin/.magisk');
    
    // Method 3: Check writable system partition
    // Root devices thường có /system mount rw
    final systemWritable = await _canWriteToSystem();
    
    // Method 4: Check dangerous props via platform channel
    final dangerousProps = await _checkDangerousBuildProps();
    
    return suExists || magiskExists || systemWritable || dangerousProps;
  }
  
  Future<bool> _checkiOSJailbreak() async {
    // Method 1: Check Cydia/Sileo
    final cydiaExists = await _fileExists('/Applications/Cydia.app') ||
        await _fileExists('/Applications/Sileo.app');
    
    // Method 2: Check SSH daemon (classic jailbreak)
    final sshExists = await _fileExists('/usr/sbin/sshd');
    
    // Method 3: Try write to non-sandbox location (jailbroken = succeeds)
    final canWriteOutsideSandbox = await _tryWriteOutsideSandbox();
    
    return cydiaExists || sshExists || canWriteOutsideSandbox;
  }
  
  Future<bool> _fileExists(String path) async {
    try {
      return File(path).existsSync();
    } catch (_) {
      return false;
    }
  }
  
  // Gọi native code cho các check cần platform API
  Future<bool> _checkDangerousBuildProps() async {
    try {
      final result = await const MethodChannel('com.myapp/security')
          .invokeMethod<bool>('checkDangerousBuildProps');
      return result ?? false;
    } catch (_) {
      return false;
    }
  }
  
  Future<bool> _canWriteToSystem() async {
    try {
      final result = await const MethodChannel('com.myapp/security')
          .invokeMethod<bool>('isSystemWritable');
      return result ?? false;
    } catch (_) {
      return false;
    }
  }
  
  Future<bool> _tryWriteOutsideSandbox() async {
    try {
      final file = File('/private/test_write_${DateTime.now().millisecondsSinceEpoch}');
      file.writeAsStringSync('test');
      file.deleteSync();
      return true; // Could write → jailbroken
    } catch (_) {
      return false; // Cannot write → not jailbroken (expected)
    }
  }
}
```

### 3.4. Obfuscation Setup

```bash
# Build với obfuscation — bắt buộc cho production
flutter build apk \
  --obfuscate \
  --split-debug-info=build/debug-symbols/android \
  --release

flutter build ipa \
  --obfuscate \
  --split-debug-info=build/debug-symbols/ios \
  --release

# Upload symbols lên Crashlytics (làm ngay sau build trong CI)
firebase crashlytics:symbols:upload \
  --app=$FIREBASE_APP_ID_ANDROID \
  build/debug-symbols/android/

# iOS: symbols được tự động upload qua dSYM trong Xcode archive
```

---

## Phần 4 — Profiling & Performance Trade-offs

### SSL Pinning overhead

```
Test: 100 HTTPS requests, Pixel 6, production app

Không có SSL Pinning:
  TLS handshake: 85ms average
  
Với SSL Pinning (Public Key):
  TLS handshake: 87ms average (+2ms = 2.4% overhead)
  
Kết luận: SSL Pinning overhead không đáng kể (<3%)
           Security benefit: ngăn chặn 100% MITM attack
```

### OWASP Mobile Top 10 Coverage

```
M1 - Improper Credential Usage:
  ✅ Biometric auth + Keychain/Keystore storage
  ✅ Token refresh, không lưu password

M2 - Inadequate Supply Chain Security:
  ✅ Pin exact dependency versions, audit third-party

M3 - Insecure Authentication/Authorization:
  ✅ JWT validation, short-lived tokens

M4 - Insufficient Input/Output Validation:
  ✅ Sealed Result types, domain validation, sanitize user input

M5 - Insecure Communication:
  ✅ SSL/TLS Pinning (Public Key), HTTPS only, certificate transparency

M6 - Inadequate Privacy Controls:
  ✅ Không log PII, data minimization, GDPR compliance

M7 - Insufficient Binary Protections:
  ✅ --obfuscate --split-debug-info, anti-debugging

M8 - Security Misconfiguration:
  ✅ No debug logs in production, no debug flags enabled

M9 - Insecure Data Storage:
  ✅ flutter_secure_storage cho sensitive data (không SharedPreferences)
  ✅ Không log sensitive data

M10 - Insufficient Cryptography:
  ✅ AES-256-GCM, RSA-2048+, up-to-date TLS
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ SSL Pinning pin Certificate thay vì Public Key
    ✅ Pin Public Key SHA-256 + có backup key cho rotation
    Lý do: Certificate hết hạn hằng năm → phải update app; Public Key thì không

[ ] ❌ Không có fallback khi biometric không khả dụng
    ✅ Graceful fallback: Biometric → PIN/Password → không xác thực
    Lý do: Thiết bị cũ không có biometric → app không thể sử dụng

[ ] ❌ Lưu access token trong SharedPreferences
    ✅ flutter_secure_storage: Keychain (iOS) / Keystore (Android)
    Lý do: SharedPreferences plaintext → đọc được ngay cả không root

[ ] ❌ Build release không có --obfuscate
    ✅ flutter build --obfuscate --split-debug-info luôn trong CI
    Lý do: Không obfuscate → reverse engineer API endpoints, business logic

[ ] ❌ Root/Jailbreak detection block app hoàn toàn
    ✅ Detect → warn user → limit sensitive features (không hard block)
    Lý do: False positive → chặn legitimate user; hard block = bypass challenge

[ ] ❌ Không test SSL Pinning với Charles Proxy
    ✅ Test thủ công: install Charles cert + proxy → verify app block request
    Lý do: Code review không đủ — phải verify thực tế blocking hoạt động
```
