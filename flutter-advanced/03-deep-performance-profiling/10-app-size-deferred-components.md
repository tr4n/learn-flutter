# Bài 3.4: App Size Optimization & Deferred Components

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Biết cơ bản về Android/iOS build process

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: APK 62MB — 40% user bỏ qua vì file quá lớn

**Nghiên cứu của Google Play (2021)**: Mỗi 6MB tăng thêm kích thước app → giảm 1% conversion rate khi install.

**E-commerce app trước optimize:**

```
flutter build apk --analyze-size
Output: app-release.apk = 62.3MB

Breakdown (app-size.json):
├── Dart code (AOT): 18.2MB  (29%)
├── Native libs (flutter engine): 8.1MB (13%)  
├── Assets (images, fonts): 26.4MB (42%) ← CULPRIT
│   ├── product_images/ (WebP): 18.2MB
│   ├── fonts (không dùng hết): 4.1MB
│   └── animations (Lottie): 4.1MB
└── Res (Android resources): 9.6MB (16%)

Target: <25MB APK để optimize install conversion
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. Flutter App Size Architecture

```
App bundle (AAB) gồm:
┌─────────────────────────────────────────────────────┐
│                  App Bundle                          │
├─────────────────────────────────────────────────────┤
│ Base Module (luôn download khi install):             │
│  ├── Flutter engine (libflutter.so): ~4MB/ABI        │
│  ├── libapp.so (Dart AOT snapshot): biến thiên        │
│  ├── Assets: fonts, icons cần ngay                   │
│  └── Android resources                              │
├─────────────────────────────────────────────────────┤
│ Dynamic Feature Module (download khi cần):           │
│  ├── feature_ar_camera/ (AR features)               │
│  ├── feature_video_player/ (HD video)               │
│  └── feature_offline_maps/ (offline maps)           │
└─────────────────────────────────────────────────────┘

Google Play phân phối APK đã split:
  - Chỉ ABI phù hợp (arm64-v8a cho 64-bit device)
  - Chỉ density phù hợp (xxhdpi cho Full HD screen)
  → User thực tế nhận <25MB thay vì 62MB
```

### 2.2. R8/ProGuard và Tree Shaking

```
Dart Tree Shaking (build time):
  Dart AOT compiler phân tích tất cả code paths
  → Loại bỏ function/class không bao giờ được gọi
  → Giảm libapp.so 15-40%
  
  VÍ DỤ:
  import 'package:intl/intl.dart';
  // Chỉ dùng DateFormat — nhưng intl có 500KB+ code
  // Dart tree shaker chỉ include code path của DateFormat
  // → Thực tế chỉ ~80KB được include

R8 (Android Java/Kotlin code shrinking):
  - Code shrinking: loại bỏ unused Java/Kotlin code
  - Obfuscation: rename classes/methods → ngắn hơn
  - Optimization: inline small methods
  
  Áp dụng cho: Flutter plugin native code (Kotlin/Java)
  Ví dụ: firebase_messaging plugin có 800KB Kotlin code
         Sau R8 shrinking: ~320KB
```

### 2.3. Deferred Components — Dynamic Feature Delivery

```
Deferred Loading trong Dart:
  Tương tự JavaScript dynamic import()
  
  KHÔNG dùng Deferred:
  import 'package:ar_camera/ar_camera.dart';
  // ar_camera code được include trong base APK
  // Dù 95% user không dùng AR → tất cả phải download

  DÙNG Deferred:
  import 'package:ar_camera/ar_camera.dart' deferred as arCamera;
  
  // Chỉ load khi user thực sự muốn dùng AR:
  Future<void> openAR() async {
    await arCamera.loadLibrary(); // Download AR module (~8MB) on-demand
    arCamera.ARCameraScreen.show(context);
  }
  
  → Base APK giảm 8MB
  → 5% user dùng AR mới download 8MB đó
```

---

## Phần 3 — Production Code Implementation

### 3.1. Phân tích App Size chi tiết

```bash
# Build với size analysis
flutter build apk --analyze-size --target-platform android-arm64

# Hoặc App Bundle (khuyến nghị cho production)
flutter build appbundle --analyze-size

# Output: ${project}/.dart_tool/flutter_build/*/app.android-arm64.json
# Mở trong DevTools: Dart DevTools → App Size tab
```

```bash
# So sánh 2 build để thấy delta
flutter build apk --analyze-size --output /tmp/before-optimize.apk
# ... thực hiện optimization ...
flutter build apk --analyze-size --output /tmp/after-optimize.apk

# Dùng devtools để diff:
flutter pub global run devtools --appSizeBase=/tmp/before-optimize.apk \
                                 --appSizeTest=/tmp/after-optimize.apk
```

### 3.2. Asset Optimization

```yaml
# pubspec.yaml — chỉ include fonts cần thiết
flutter:
  fonts:
    - family: Roboto
      fonts:
        # ❌ Include toàn bộ weight (thêm ~1MB không cần)
        # - asset: fonts/Roboto-Thin.ttf       (weight: 100)
        # - asset: fonts/Roboto-Light.ttf      (weight: 300)
        - asset: fonts/Roboto-Regular.ttf      # weight: 400 — BẮT BUỘC
        - asset: fonts/Roboto-Medium.ttf       # weight: 500 — BẮT BUỘC
        - asset: fonts/Roboto-Bold.ttf         # weight: 700 — BẮT BUỘC
        # - asset: fonts/Roboto-Black.ttf      (weight: 900) — không dùng
```

```bash
# Convert PNG → WebP để giảm ~30-40% kích thước images
# Dùng cwebp tool (Google WebP encoder)
find assets/images -name "*.png" -exec \
  cwebp -q 85 {} -o {}.webp \;

# Convert PNG → AVIF (mới hơn, nhỏ hơn 50% so với WebP)
# Nhưng AVIF chỉ support Flutter 3.22+
```

```dart
// Không bundle large assets vào app — load từ CDN
// features/catalog/data/datasources/image_datasource.dart
class ImageDataSource {
  // ❌ Không đặt product images trong assets/
  // ✅ Load từ CDN với caching
  static Widget productImage(String productId, {required double width}) {
    return CachedNetworkImage(
      imageUrl: '${Config.cdnBaseUrl}/products/$productId/thumb-${width.toInt()}.webp',
      width: width,
      memCacheWidth: width.toInt(), // Resize trong memory → giảm RAM
      placeholder: (_, __) => const ProductImageSkeleton(),
      errorWidget: (_, __, ___) => const ProductImageError(),
    );
  }
}
```

### 3.3. Deferred Components Setup (Android)

```yaml
# pubspec.yaml
flutter:
  deferred-components:
    - name: ar-camera
      libraries:
        - package:ar_camera/ar_camera.dart
      assets:
        - assets/ar/
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<!-- Thêm Dynamic Feature manifest -->
<manifest>
  <dist:module
    dist:instant="false"
    dist:title="@string/title_ar_camera">
    <dist:delivery>
      <dist:on-demand/>
    </dist:delivery>
    <dist:fusing dist:include="true"/>
  </dist:module>
</manifest>
```

```dart
// features/ar/presentation/ar_launcher.dart
import 'package:ar_camera/ar_camera.dart' deferred as arCamera;

class ARLauncher {
  static bool _isLoaded = false;

  /// Gọi trước khi show AR — download module nếu chưa có
  static Future<void> preloadIfNeeded() async {
    if (_isLoaded) return;
    
    // Kiểm tra: user có muốn AR không? (optional pre-download)
    // Chỉ pre-download nếu user trên WiFi
    final connectivity = await Connectivity().checkConnectivity();
    if (connectivity != ConnectivityResult.wifi) return;
    
    await arCamera.loadLibrary();
    _isLoaded = true;
  }

  /// Mở AR với loading indicator
  static Future<void> launch(BuildContext context) async {
    // Show loading nếu chưa load
    if (!_isLoaded) {
      showDialog(
        context: context,
        barrierDismissible: false,
        builder: (_) => const _ARLoadingDialog(),
      );
      
      try {
        await arCamera.loadLibrary(); // Download ~8MB on-demand
        _isLoaded = true;
        if (context.mounted) Navigator.of(context).pop(); // Close loading dialog
      } on Exception catch (e) {
        if (context.mounted) {
          Navigator.of(context).pop();
          ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(content: Text('Không thể tải tính năng AR')),
          );
        }
        return;
      }
    }
    
    if (context.mounted) {
      await arCamera.ARCameraScreen.show(context);
    }
  }
}

class _ARLoadingDialog extends StatelessWidget {
  const _ARLoadingDialog();
  
  @override
  Widget build(BuildContext context) {
    return const AlertDialog(
      content: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          CircularProgressIndicator(),
          SizedBox(height: 16),
          Text('Đang tải tính năng AR...'),
        ],
      ),
    );
  }
}
```

### 3.4. Obfuscation + Split Debug Info

```bash
# Build với obfuscation — rename classes/functions thành a, b, c...
flutter build apk --obfuscate --split-debug-info=./debug-symbols/

# Kết quả:
# - app-release.apk: smaller + harder to reverse-engineer
# - debug-symbols/: chứa mapping file để deobfuscate stack trace

# Upload debug symbols lên Firebase Crashlytics:
firebase crashlytics:symbols:upload --app=<APP_ID> ./debug-symbols/

# Hoặc tích hợp vào CI:
# .github/workflows/deploy.yml
- name: Upload debug symbols
  run: |
    flutter build apk --obfuscate --split-debug-info=./debug-symbols/
    firebase crashlytics:symbols:upload \
      --app=${{ secrets.FIREBASE_APP_ID }} \
      ./debug-symbols/
```

### 3.5. R8 Configuration cho Flutter plugin

```
# android/app/proguard-rules.pro

# Flutter engine — không obfuscate
-keep class io.flutter.** { *; }
-keep class io.flutter.plugins.** { *; }

# Keep plugin entrypoints
-keep class com.google.firebase.** { *; }
-keepattributes *Annotation*

# Giữ class serialize/deserialize (JSON)
-keepclassmembers class * {
    @com.google.gson.annotations.SerializedName <fields>;
}

# Xóa logging trong release
-assumenosideeffects class android.util.Log {
    public static *** d(...);
    public static *** v(...);
    public static *** i(...);
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Kết quả optimization E-commerce app

```
TRƯỚC → SAU optimization:

                        TRƯỚC       SAU         GIẢM
────────────────────────────────────────────────────────
APK size (full)         62.3MB      28.1MB      -55%
Download size*          38.5MB      16.2MB      -58%
Base module             62.3MB      18.1MB      -71%
AR feature (deferred)   -           8.4MB       (on-demand)
Install time (3G)       ~45s        ~18s        -60%
Install conversion      -           +12%        (Google data)

* Download size = Google Play split APK (đã bỏ unused ABI/density)

Breakdown giảm:
  Images → WebP:           26.4MB → 11.2MB  (-58%)
  Font subset:              4.1MB →  1.8MB  (-56%)
  Lottie → Rive:            4.1MB →  1.9MB  (-54%)
  R8 shrinking (plugins):   9.6MB →  5.2MB  (-46%)
  AR → Deferred:            8.1MB →  0MB    (removed from base)
```

### Cold Start Time sau App Size reduce

```
Mối liên hệ: App size ↓ → DEX load time ↓ → Cold start ↓

TRƯỚC (62MB APK):
  Cold start (Pixel 6): 2,800ms TTID

SAU (28MB APK):
  Cold start (Pixel 6): 1,940ms TTID (-31%)
  
Lý do: Ít code hơn → DEX optimization ít hơn → ART start nhanh hơn
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Build APK (monolithic) để release lên Google Play
    ✅ Build AAB (App Bundle): flutter build appbundle
    Lý do: AAB cho phép Google Play split APK theo ABI/density → user download ít hơn

[ ] ❌ Include tất cả font weight (Thin, Light, Regular, Medium, Bold, Black)
    ✅ Chỉ include weight thực sự dùng trong design
    Lý do: Mỗi font weight ~200-400KB — unused weight = waste

[ ] ❌ Bundle product images / user avatars vào assets/
    ✅ Load từ CDN, chỉ bundle static assets (icons, background)
    Lý do: Product catalog thay đổi thường xuyên — bundle = stale và bloat

[ ] ❌ Dùng PNG cho assets không cần transparency
    ✅ Convert sang WebP (80-85% quality) bằng cwebp
    Lý do: WebP nhỏ hơn PNG ~30-40% với quality tương đương

[ ] ❌ Build release mà không có --obfuscate
    ✅ Luôn: flutter build apk --obfuscate --split-debug-info=./debug-symbols/
    Lý do: Không obfuscate → dễ reverse engineer business logic

[ ] ❌ Không upload debug symbols lên Crashlytics sau khi obfuscate
    ✅ Upload symbols vào CI/CD pipeline ngay sau build
    Lý do: Obfuscated stack trace không đọc được trong Crashlytics

[ ] ❌ Không test Deferred Component loading trên thiết bị thực
    ✅ Test với Internal Testing track trên Google Play Console
    Lý do: Deferred loading cần Play Store infrastructure — không test được trên local

[ ] ❌ Không so sánh app-size.json giữa các build
    ✅ Tích hợp app size comparison vào CI — fail nếu tăng > threshold
    Lý do: Regression không phát hiện → app phình to dần qua từng release
```
