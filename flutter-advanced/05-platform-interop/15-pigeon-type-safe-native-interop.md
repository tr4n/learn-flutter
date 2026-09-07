# Bài 5.2: Pigeon — Type-safe Native Interop với Code Generation

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Đã đọc Bài 5.1 (Platform Channels)

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Runtime crash từ type mismatch

**Thực tế dự án (production crash, 0.4% users):**

```dart
// ❌ MethodChannel thủ công — không type-safe
Future<Map<String, dynamic>> getUserInfo() async {
  final result = await _channel.invokeMethod<Map>('getUserInfo');
  return {
    'name': result!['name'] as String,   // Crash nếu 'name' là null hoặc int
    'age': result['age'] as int,          // Crash nếu Android trả về String
    'score': result['score'] as double,   // Crash nếu iOS trả về int (type mismatch)
  };
}

// Android dev viết: result.put("score", 100)   // int
// iOS dev viết:     result["score"] = 100.0    // double
// → Runtime crash: type 'int' is not a subtype of type 'double'
```

**Pigeon giải quyết bài toán này**: API contract được định nghĩa **một lần** trong Dart → codegen tự động sinh ra Kotlin và Swift wrapper type-safe.

---

## Phần 2 — Low-Level Mechanics

### 2.1. Pigeon Architecture

```
Bạn viết:         pigeons/user_api.dart
                         │
                 pigeon code generator
                 /                    \
           Android                   iOS
     UserApi.g.kt               UserApi.g.swift
     (Kotlin)                   (Swift)
           \                         /
            └───────────────────────┘
                      │
               Platform Channel
                      │
             pigeons/user_api.dart
         (Dart wrapper đã generated)
         
KHÔNG CÒN:
- Manual Map<String, dynamic> parsing
- Manual type casting với 'as'
- Inconsistency giữa Android và iOS impl
- Runtime type errors
```

### 2.2. Codegen Flow

```
1. Define API contract (Dart):
   class UserInfo { String name; int age; double score; }
   abstract class UserHostApi {
     UserInfo getUserInfo();
     void updateProfile(UpdateRequest request);
   }

2. Run generator:
   dart run pigeon --input pigeons/user_api.dart
   
3. Output files:
   lib/src/generated/user_api.dart      (Dart)
   android/.../UserApi.kt              (Kotlin)
   ios/.../UserApi.swift               (Swift)

4. Implement native side:
   class UserApiImpl : UserApi {
       override fun getUserInfo(): UserInfo { ... }
   }
   
5. Use from Dart:
   final api = UserHostApi();
   final user = await api.getUserInfo(); // UserInfo type-safe!
```

---

## Phần 3 — Production Code Implementation

### 3.1. Setup Pigeon

```yaml
# pubspec.yaml
dev_dependencies:
  pigeon: ^22.0.0
```

```bash
# Chạy codegen:
dart run pigeon \
  --input pigeons/camera_api.dart \
  --dart_out lib/src/generated/camera_api.dart \
  --kotlin_out android/app/src/main/kotlin/com/myapp/CameraApi.kt \
  --kotlin_package com.myapp \
  --swift_out ios/Runner/CameraApi.swift
```

### 3.2. Định nghĩa API Contract

```dart
// pigeons/camera_api.dart
// File này là NGUỒN SỰ THẬT DUY NHẤT cho Camera API contract

import 'package:pigeon/pigeon.dart';

@ConfigurePigeon(PigeonOptions(
  dartOut: 'lib/src/generated/camera_api.dart',
  dartOptions: DartOptions(),
  kotlinOut: 'android/app/src/main/kotlin/com/myapp/CameraApi.kt',
  kotlinOptions: KotlinOptions(package: 'com.myapp'),
  swiftOut: 'ios/Runner/CameraApi.swift',
  swiftOptions: SwiftOptions(),
))

// Enums được generate cho cả Dart, Kotlin, Swift
enum CameraFacing { front, back }
enum FlashMode { off, on, auto, torch }
enum ImageQuality { low, medium, high }

// Data class — generate Kotlin data class, Swift struct
class CameraConfig {
  CameraConfig({
    required this.facing,
    required this.flashMode,
    required this.quality,
    this.maxZoom,
    this.enableHDR = false,
  });
  
  final CameraFacing facing;
  final FlashMode flashMode;
  final ImageQuality quality;
  final double? maxZoom;      // nullable → Optional<Double> trong Swift, Double? trong Kotlin
  final bool enableHDR;
}

class CapturedPhoto {
  CapturedPhoto({
    required this.filePath,
    required this.width,
    required this.height,
    required this.sizeBytes,
    this.exifData,
  });
  
  final String filePath;
  final int width;
  final int height;
  final int sizeBytes;
  final Map<String?, Object?>? exifData; // Pigeon support nested nullable Map
}

// Custom error type — không dùng PlatformException raw
class CameraError {
  CameraError({required this.code, required this.message});
  final String code;
  final String message;
}

// HostApi: Dart gọi → Native thực hiện
@HostApi()
abstract class CameraHostApi {
  // void = fire-and-forget, không có return
  void initialize(CameraConfig config);
  
  // @async = native xử lý async, Dart nhận Future
  @async
  CapturedPhoto capturePhoto();
  
  @async
  void startVideoRecording(String outputPath);
  
  @async
  String stopVideoRecording(); // return path của video file
  
  List<double> getAvailableZoomLevels();
  
  @async
  void setZoom(double zoom);
  
  void dispose();
}

// FlutterApi: Native gọi → Dart
// Dùng khi native cần push events về Dart (không phải request-response)
@FlutterApi()
abstract class CameraFlutterApi {
  void onPhotoTaken(CapturedPhoto photo);
  void onError(CameraError error);
  void onZoomChanged(double zoom);
}
```

### 3.3. Implement Native Side (Kotlin)

```kotlin
// android/app/src/main/kotlin/com/myapp/CameraApiImpl.kt
import com.myapp.CameraHostApi
import com.myapp.CameraConfig
import com.myapp.CapturedPhoto

class CameraApiImpl(
    private val context: Context,
    private val activity: Activity,
) : CameraHostApi {
    
    private var camera: Camera? = null
    private var imageCapture: ImageCapture? = null
    
    override fun initialize(config: CameraConfig) {
        // Type-safe: config.facing là CameraFacing enum, không phải String
        val lensFacing = when (config.facing) {
            CameraFacing.FRONT -> CameraSelector.LENS_FACING_FRONT
            CameraFacing.BACK -> CameraSelector.LENS_FACING_BACK
        }
        
        val flashMode = when (config.flashMode) {
            FlashMode.OFF -> ImageCapture.FLASH_MODE_OFF
            FlashMode.ON -> ImageCapture.FLASH_MODE_ON
            FlashMode.AUTO -> ImageCapture.FLASH_MODE_AUTO
            FlashMode.TORCH -> ImageCapture.FLASH_MODE_ON // Map torch → on
        }
        
        // Setup CameraX...
        imageCapture = ImageCapture.Builder()
            .setFlashMode(flashMode)
            .build()
    }
    
    // @async trong pigeon → callback: (Result<CapturedPhoto>) -> Unit
    override fun capturePhoto(callback: (Result<CapturedPhoto>) -> Unit) {
        val imageCapture = this.imageCapture
            ?: return callback(Result.failure(Exception("Camera chưa được initialize")))
        
        val outputFile = File(context.cacheDir, "photo_${System.currentTimeMillis()}.jpg")
        val outputOptions = ImageCapture.OutputFileOptions.Builder(outputFile).build()
        
        imageCapture.takePicture(
            outputOptions,
            ContextCompat.getMainExecutor(context),
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    val exif = ExifInterface(outputFile.absolutePath)
                    callback(Result.success(CapturedPhoto(
                        filePath = outputFile.absolutePath,
                        width = exif.getAttributeInt(ExifInterface.TAG_IMAGE_WIDTH, 0).toLong(),
                        height = exif.getAttributeInt(ExifInterface.TAG_IMAGE_LENGTH, 0).toLong(),
                        sizeBytes = outputFile.length(),
                        exifData = null,
                    )))
                }
                
                override fun onError(exception: ImageCaptureException) {
                    callback(Result.failure(exception))
                }
            }
        )
    }
    
    override fun dispose() {
        camera?.let { ProcessCameraProvider.getInstance(context).get().unbindAll() }
        camera = null
        imageCapture = null
    }
}

// Đăng ký trong MainActivity.kt
class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        CameraHostApi.setUp(
            flutterEngine.dartExecutor.binaryMessenger,
            CameraApiImpl(this, this),
        )
    }
}
```

### 3.4. Sử dụng từ Dart (Generated code)

```dart
// features/camera/data/datasources/camera_platform_datasource.dart
// Generated code đã handle serialization — không cần cast manual

final class CameraPlatformDataSource {
  CameraPlatformDataSource() : _api = CameraHostApi();
  final CameraHostApi _api;

  Future<void> initialize({
    required bool useFrontCamera,
    required bool enableFlash,
  }) async {
    // Type-safe: truyền enum, không phải string 'front'/'back'
    await _api.initialize(CameraConfig(
      facing: useFrontCamera ? CameraFacing.front : CameraFacing.back,
      flashMode: enableFlash ? FlashMode.on : FlashMode.off,
      quality: ImageQuality.high,
      enableHDR: true,
    ));
  }

  Future<CapturedPhoto> capturePhoto() async {
    // Return type CapturedPhoto — không phải Map<String, dynamic>
    // Không cần cast, không thể có type mismatch
    return _api.capturePhoto();
  }
  
  Future<void> dispose() => _api.dispose();
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### So sánh: Manual MethodChannel vs Pigeon

```
DEVELOPMENT COST:
  Manual MethodChannel:
    - Dart: 30 lines (parsing, casting, error handling)
    - Android Kotlin: 40 lines
    - iOS Swift: 40 lines
    = 110 lines, 3 files, high duplication risk
    
  Pigeon:
    - API contract (pigeons/): 40 lines
    - Generated: auto (không tính)
    - Implementation: 25 lines Kotlin + 25 lines Swift
    = 90 lines, nhưng ZERO manual serialization code

RUNTIME PERFORMANCE:
  Manual: Map parsing + type cast ≈ 0.3-0.5ms extra overhead
  Pigeon: Generated codec tương đương StandardMessageCodec ≈ 0.3ms
  → Performance tương đương — Pigeon không có overhead thêm

SAFETY:
  Manual: Runtime crash 0.4% (type mismatch, null pointer)
  Pigeon: Compile-time error nếu type sai → 0% runtime type crash
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Sử dụng Map<String, dynamic> làm API contract giữa Dart và Native
    ✅ Định nghĩa typed class trong pigeons/ và generate code
    Lý do: Type mismatch chỉ phát hiện ở runtime → production crash

[ ] ❌ Chạy pigeon codegen thủ công, không integrate CI
    ✅ Thêm vào CI: dart run pigeon --input pigeons/xxx.dart && check git diff
    Lý do: Generated files out-of-date → compile error khi build

[ ] ❌ Implement native code chạy IO trực tiếp trong @HostApi handler
    ✅ Dùng @async cho các method cần IO, implement async trong native
    Lý do: Blocking native thread → ANR (Android) hoặc UI freeze (iOS)

[ ] ❌ Không xử lý FlutterApi init order (Dart gọi Native trước khi Flutter ready)
    ✅ Initialize native API trong configureFlutterEngine(), không phải onCreate()
    Lý do: configureFlutterEngine đảm bảo Flutter binary messenger sẵn sàng

[ ] ❌ pigeons/ file không được commit vào git
    ✅ Commit pigeons/ (source) VÀ generated files → dễ review diff khi thay đổi API
    Lý do: Generated files phải đồng bộ giữa tất cả developer trong team

[ ] ❌ Nullable type trong Pigeon không được kiểm tra phía Native
    ✅ Native code phải handle null từ Dart → không dùng !! (Kotlin) / force unwrap (Swift)
    Lý do: Pigeon cho phép Dart gửi null → native crash nếu không handle
```
