# Bài 8.3: Shorebird Code Push — Vá lỗi Flutter không qua Store

> **Cấp độ**: Staff Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Đã đọc Bài 8.2 (CI/CD); biết Dart VM cơ bản

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: P0 crash production cần fix trong 10 phút

**E-commerce app — Black Friday incident:**

```
08:45 AM: App release v3.2.0 lên App Store và Play Store (đã review xong)
09:15 AM: Traffic spike (1.2M users online) — Black Friday starts
09:17 AM: Crashlytics alert: 4,200 crashes/phút (normal: 12/phút)
09:18 AM: Root cause found: NullPointerException trong CartBloc khi list empty

Traditional fix path:
  09:18 Fix code (2 min)
  09:20 flutter build (18 min)
  09:38 Submit to App Store Review
  09:38 Android: Upload to Play Store → instant (internal test) → promote (30 min)
  
  Apple App Store Review: 24-48 GIỜ
  → Trong 24 giờ: 340,000+ crashes
  → Estimated revenue loss: $1.2M

Shorebird path:
  09:18 Fix code (2 min)
  09:20 shorebird patch android (3 min)
  09:23 shorebird patch ios (3 min)
  09:26 Patch live — users nhận patch khi mở lại app
  → Total downtime: 8 phút
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. Shorebird — Dart VM Bytecode Patching

```
Traditional Flutter Release:
  Dart code → AOT compiler → libapp.so (native machine code)
  
  Không thể patch libapp.so sau khi app đã install:
  - Native code cần resign → App Store review
  - Thay đổi native code = thay đổi app binary

Shorebird Approach:
  Dart code → Shorebird's modified Dart VM interpreter
           → Dart snapshot (bytecode, không phải native)
           
  Patch flow:
  1. Shorebird server lưu snapshot mới (chỉ Dart code changes)
  2. App launch → check Shorebird server
  3. Download delta patch (chỉ phần thay đổi, nhỏ hơn full snapshot)
  4. Apply patch → chạy updated Dart code
  
  TẠI SAO APP STORE CHO PHÉP?
  → Shorebird chỉ patch Dart code (interpreted bytecode)
  → KHÔNG patch: native code, native plugins, assets, permissions
  → Tương tự React Native CodePush — đã được App Store approve từ 2016
  → Shorebird review từng app để đảm bảo compliance
```

### 2.2. Giới hạn của Shorebird Patch

```
CÓ THỂ patch qua Shorebird:
  ✅ Dart code changes (business logic, UI code)
  ✅ String changes (typo fix, copy change)
  ✅ Bug fixes trong Dart layer
  ✅ Feature flags toggle (Dart side)
  
KHÔNG THỂ patch:
  ❌ Native plugin code (Kotlin/Swift/Java/Obj-C)
  ❌ Flutter engine (C++ layer) — cần store release
  ❌ Assets (images, fonts, JSON config files)
  ❌ AndroidManifest.xml / Info.plist changes
  ❌ New permissions
  ❌ Changes cần new native API
  
QUYẾT ĐỊNH: Patch hay Full Release?
  Crash trong Dart code (CartBloc.dart) → Patch ✅
  Crash trong camera native plugin → Full release
  Add new feature với new permission → Full release
  Fix typo trong string → Patch ✅
  Change API endpoint URL (nếu hardcoded trong Dart) → Patch ✅
```

### 2.3. Patch Download Strategy

```
App Launch:
  1. App start với current Dart snapshot
  2. Background: check Shorebird server cho new patch
  3. If patch available: download (usually <2MB delta)
  4. Apply patch on NEXT app launch (không restart ngay)
  
  Lý do không apply ngay:
  → App đang chạy với current Dart VM state
  → Apply patch giữa chừng = risk crash (partially applied)
  → Safer: download silently, apply on clean launch
  
  Override behavior (cho critical patches):
  → Show "Update available" dialog → "Restart now"
  → User restart → patch applied → clean launch
```

---

## Phần 3 — Production Code Implementation

### 3.1. Shorebird Setup

```bash
# Install Shorebird CLI
curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash

# Login
shorebird login

# Init trong project (chỉ 1 lần)
shorebird init
# Tạo: shorebird.yaml với app_id
```

```yaml
# shorebird.yaml
app_id: your-app-id-here  # Từ Shorebird console

# Optional: environment-specific app IDs
flavors:
  staging: staging-app-id
  production: production-app-id
```

### 3.2. Release workflow với Shorebird

```bash
# RELEASE (thay cho flutter build) — lần đầu release 1 version
shorebird release android \
  --flavor production \
  --target lib/main_production.dart
  
shorebird release ios \
  --flavor production \
  --target lib/main_production.dart

# Output: app được build với Shorebird runtime
# Submit lên stores như bình thường (APK/AAB/IPA từ Shorebird release)
```

```bash
# PATCH (sau khi đã có release) — hotfix
# Chỉ fix Dart code, không thêm native deps

# 1. Fix bug trong Dart code
# 2. Create patch
shorebird patch android \
  --flavor production \
  --release-version 3.2.0  # Phải match với release version

shorebird patch ios \
  --flavor production \
  --release-version 3.2.0

# Patch available ngay lập tức cho tất cả users có app version 3.2.0
# Không cần store review!
```

### 3.3. Tích hợp vào CI/CD pipeline

```yaml
# .github/workflows/shorebird_patch.yml
# Trigger thủ công hoặc khi push hotfix branch

name: Shorebird Hotfix Patch

on:
  workflow_dispatch:
    inputs:
      release_version:
        description: 'Release version to patch (e.g., 3.2.0)'
        required: true
      patch_notes:
        description: 'What does this patch fix?'
        required: true

jobs:
  patch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.27.0'
      
      - name: Install Shorebird
        uses: shorebirdtech/setup-shorebird@v1
        with:
          cache: true
      
      - name: Create Android Patch
        run: |
          shorebird patch android \
            --flavor production \
            --release-version ${{ github.event.inputs.release_version }}
        env:
          SHOREBIRD_TOKEN: ${{ secrets.SHOREBIRD_TOKEN }}
      
      - name: Create iOS Patch
        run: |
          shorebird patch ios \
            --flavor production \
            --release-version ${{ github.event.inputs.release_version }}
        env:
          SHOREBIRD_TOKEN: ${{ secrets.SHOREBIRD_TOKEN }}
      
      - name: Notify Team
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: |
            🚨 *Shorebird Hotfix Deployed*
            Version: v${{ github.event.inputs.release_version }}
            Fix: ${{ github.event.inputs.patch_notes }}
            Status: ${{ job.status }}
            Patch will be applied on next app launch for all users.
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 3.4. Dart Code — Detect và force-apply patch

```dart
// lib/core/update/shorebird_update_service.dart
// Sử dụng shorebird_code_push package

import 'package:shorebird_code_push/shorebird_code_push.dart';

@singleton
final class ShorebirdUpdateService {
  ShorebirdUpdateService() : _updater = const ShorebirdUpdater();
  final ShorebirdUpdater _updater;

  /// Kiểm tra và download patch ở background
  Future<void> checkAndDownloadUpdate() async {
    // Chỉ check nếu Shorebird supported (release build)
    if (!await _updater.isNewPatchAvailableForDownload()) return;
    
    // Download patch silently
    try {
      await _updater.downloadUpdateIfAvailable();
      debugPrint('[Shorebird] Patch downloaded, will apply on next launch');
    } catch (e) {
      debugPrint('[Shorebird] Download failed: $e');
    }
  }

  /// Kiểm tra có patch mới đã download chưa (sẵn sàng apply)
  Future<bool> hasReadyPatch() async {
    return _updater.isNewPatchReadyToInstall();
  }

  /// Hiển thị dialog yêu cầu restart cho critical patches
  Future<void> promptRestartForCriticalPatch(BuildContext context) async {
    if (!await hasReadyPatch()) return;
    
    if (!context.mounted) return;
    
    final shouldRestart = await showDialog<bool>(
      context: context,
      barrierDismissible: false, // Critical → không thể dismiss
      builder: (context) => AlertDialog(
        title: const Text('Cập nhật quan trọng'),
        content: const Text(
          'Một bản vá lỗi quan trọng đã sẵn sàng. '
          'Vui lòng khởi động lại ứng dụng để áp dụng.',
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Để sau'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('Khởi động lại'),
          ),
        ],
      ),
    );
    
    if (shouldRestart == true) {
      // Restart app để apply patch
      _updater.restart();
    }
  }
}

// Gọi trong App lifecycle:
class _AppState extends State<App> {
  @override
  void initState() {
    super.initState();
    // Check update sau khi app ready (không block launch)
    WidgetsBinding.instance.addPostFrameCallback((_) {
      _checkForUpdates();
    });
  }
  
  Future<void> _checkForUpdates() async {
    final updateService = getIt<ShorebirdUpdateService>();
    await updateService.checkAndDownloadUpdate();
    
    // Nếu có critical patch (có thể check từ remote config)
    final hasCriticalPatch = await updateService.hasReadyPatch() &&
        await RemoteConfig.instance.getBool('force_patch_update');
    
    if (hasCriticalPatch && mounted) {
      await updateService.promptRestartForCriticalPatch(context);
    }
  }
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Shorebird vs Traditional Release — So sánh

```
Kịch bản: P0 crash fix cần deploy ngay

Traditional Release:
  Code fix:     2 min
  Build:        18 min
  Play Store:   30 min (internal → production track)
  App Store:    24-48 GIỜ (review)
  Total iOS:    24-48 giờ
  Total Android: 50 phút

Shorebird Patch:
  Code fix:     2 min
  Build patch:  3 min (android) + 3 min (ios) = parallel
  Deploy:       1 min (API call)
  Live for users: <5 min sau khi open app
  Total:        8-10 phút (cả iOS lẫn Android!)

Shorebird pricing (2024):
  Free tier: 5,000 monthly active users (MAU)
  Pro: $20/month → 50,000 MAU
  Team: $200/month → 200,000 MAU
  Scale: $1,000/month → unlimited

ROI: $20/month vs $1.2M revenue lost → 60,000x ROI trong 1 incident
```

### Performance impact của Shorebird runtime

```
Shorebird dùng Dart interpreter thay vì AOT compiled code:
  Startup time increase: +50-150ms cold start
  Steady state performance: -5-15% (interpreter vs JIT warmup)
  
Sau khi patch applied (Dart snapshot):
  Interpreter mode: ~same as non-patched Shorebird
  
So với pure native AOT (non-Shorebird):
  Startup: +100ms (acceptable)
  CPU-intensive tasks: -10% (significant nếu app heavy compute)
  
Recommendation:
  Dùng Shorebird cho: Business apps, CRUD apps, E-commerce
  Cẩn thận với: Game apps, heavy compute apps, real-time apps
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Patch thêm new native plugin dependency
    ✅ Chỉ patch thuần Dart code changes
    Lý do: Native plugin không thể patch → app crash khi call non-existent native code

[ ] ❌ Shorebird patch version không match với release version
    ✅ --release-version phải match chính xác với version đã submit store
    Lý do: Mismatch → patch không được apply cho users đang dùng app

[ ] ❌ Không test patch trên staging trước khi apply production
    ✅ shorebird patch staging → test trên internal testers → patch production
    Lý do: Patch buggy còn tệ hơn không patch

[ ] ❌ Không có monitoring sau khi apply patch
    ✅ Watch Crashlytics: crash rate giảm sau 30 phút = patch effective
    Lý do: Cần confirm patch hoạt động; có thể rollback nếu không hiệu quả

[ ] ❌ Rollback patch khi cần
    ✅ shorebird patch revert --release-version X.Y.Z
    Lý do: Patch có thể gây ra vấn đề mới → cần revert về version cũ

[ ] ❌ Cho toàn bộ team patch production mà không có approval
    ✅ Chỉ Tech Lead/SRE có SHOREBIRD_TOKEN trong production secrets
    Lý do: Patch sai → ảnh hưởng toàn bộ users → phải có approval gate

[ ] ❌ Không giải thích cho App Store reviewer về Shorebird
    ✅ Shorebird tự handle App Store compliance — không cần thêm gì
    Lý do: Shorebird đã được Apple review và approve — không phải JavaScript injection
```
