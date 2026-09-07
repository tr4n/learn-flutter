# Bài 8.2: CI/CD Pipeline — GitHub Actions & Fastlane

> **Cấp độ**: Senior Engineer / DevOps  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Biết git, cơ bản về iOS signing và Android keystore

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Deploy thủ công mất 4 giờ mỗi release

**Quy trình cũ (manual deployment):**

```
Thứ Sáu — Release Day:
  14:00 Dev: "Code done, sẵn sàng release!"
  
  14:30 Lead: Cập nhật version code manually trong build.gradle và Info.plist
  15:00 Lead: flutter build apk --release (25 phút build)
  15:25 Lead: Sign APK manually với keystore
  15:30 Lead: Upload lên Play Console internal testing
  15:45 Lead: flutter build ipa --release (20 phút)
  16:05 Lead: Open Xcode Archive...
  16:40 Lead: Submit to TestFlight...
  17:10 Lead: Firebase App Distribution upload...
  
  17:30 → "Done! Time to go home"
  
Vấn đề:
- 3.5 giờ manual work
- Error-prone (quên bump version, wrong signing key)
- Bottleneck: chỉ 1 người có keystore + provisioning profiles
- Không thể release nhiều lần/ngày

Mục tiêu CI/CD:
  14:00 Dev: Merge PR vào main
  14:15 CI: Build + Test
  14:30 CI: Deploy to Firebase App Distribution (staging)
  14:45 CI: Deploy to TestFlight + Play Internal Testing (production)
  TOTAL: 45 phút, 0 manual work
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. GitHub Actions — Flutter CI Architecture

```
Event: Push to main branch
         │
         ▼
GitHub Actions Trigger
         │
    ┌────┴────┐
    │         │
    ▼         ▼
Android      iOS
Build        Build
(ubuntu)     (macos)
    │         │
    │ Parallel│
    ▼         ▼
Tests       Tests
    │         │
    ▼         ▼
Sign APK    Archive IPA
    │         │
    ▼         ▼
Firebase    TestFlight
App Dist    + App Store
    │         │
    └────┬────┘
         ▼
    Notify Slack
```

### 2.2. Fastlane — Automation Layer

```
Fastlane lanes:
  lane :test → chạy unit + widget test
  lane :build_android → tạo APK/AAB signed
  lane :build_ios → tạo IPA signed
  lane :deploy_staging → Firebase App Distribution
  lane :deploy_production → Play Store + App Store

Fastlane Match (iOS code signing):
  Lưu tất cả certificates + provisioning profiles trong Git repo riêng
  Encrypt bằng passphrase
  CI fetch và install tự động
  → Không cần Xcode "Automatically manage signing" (không reliable trong CI)
```

---

## Phần 3 — Production Code Implementation

### 3.1. Fastfile — Lane definitions

```ruby
# fastlane/Fastfile
default_platform(:ios)

# Shared helper
def bump_build_number
  # Dùng CI build number để đảm bảo unique + tăng dần
  build_number = ENV['GITHUB_RUN_NUMBER'] || Time.now.to_i.to_s
  
  # Android
  android_set_version_code(
    version_code: build_number.to_i,
    gradle_file: 'android/app/build.gradle',
  )
  
  # iOS
  increment_build_number(
    build_number: build_number,
    xcodeproj: 'ios/Runner.xcodeproj',
  )
end

platform :android do
  desc "Build và sign APK cho staging"
  lane :build_staging do
    bump_build_number
    
    gradle(
      task: 'bundle',
      build_type: 'Release',
      flavor: 'staging',
      project_dir: 'android/',
      properties: {
        'android.injected.signing.store.file' => ENV['KEYSTORE_PATH'],
        'android.injected.signing.store.password' => ENV['KEYSTORE_PASSWORD'],
        'android.injected.signing.key.alias' => ENV['KEY_ALIAS'],
        'android.injected.signing.key.password' => ENV['KEY_PASSWORD'],
      },
    )
  end
  
  desc "Deploy lên Firebase App Distribution"
  lane :deploy_staging do
    build_staging
    
    firebase_app_distribution(
      app: ENV['FIREBASE_APP_ID_ANDROID'],
      apk_path: 'android/app/build/outputs/apk/staging/release/app-staging-release.apk',
      groups: 'internal-testers, qa-team',
      release_notes: "Build #{ENV['GITHUB_RUN_NUMBER']}: #{ENV['COMMIT_MESSAGE']}",
      firebase_cli_token: ENV['FIREBASE_TOKEN'],
    )
  end
  
  desc "Deploy lên Google Play Internal Testing"
  lane :deploy_production do
    # Build AAB (không APK) cho Play Store
    gradle(
      task: 'bundle',
      build_type: 'Release',
      flavor: 'production',
      project_dir: 'android/',
    )
    
    upload_to_play_store(
      track: 'internal', # internal → alpha → beta → production
      aab: 'android/app/build/outputs/bundle/productionRelease/app-production-release.aab',
      json_key: ENV['PLAY_STORE_JSON_KEY'], # Service account JSON
      skip_upload_screenshots: true,
      skip_upload_metadata: true,
    )
  end
end

platform :ios do
  before_all do
    # Setup signing certificates từ Match (encrypted git repo)
    setup_ci # Configure keychain cho CI environment
  end
  
  desc "Fetch certificates và provisioning profiles"
  lane :match_certificates do |options|
    match(
      type: options[:type] || 'appstore', # 'development', 'adhoc', 'appstore'
      app_identifier: 'com.mycompany.myapp',
      git_url: ENV['MATCH_GIT_URL'],          # Private git repo
      git_basic_authorization: ENV['MATCH_GIT_TOKEN'],
      password: ENV['MATCH_PASSWORD'],         # Encryption passphrase
      readonly: true,                          # CI chỉ đọc, không tạo mới
      keychain_name: 'CI_Keychain',
      keychain_password: ENV['KEYCHAIN_PASSWORD'],
    )
  end
  
  desc "Build IPA"
  lane :build_ios do |options|
    bump_build_number
    match_certificates(type: options[:signing_type] || 'appstore')
    
    gym(
      scheme: options[:scheme] || 'Runner-Production',
      workspace: 'ios/Runner.xcworkspace',
      configuration: 'Release',
      export_method: options[:export_method] || 'app-store',
      output_directory: 'build/ios/',
      output_name: 'Runner.ipa',
      clean: true,
    )
  end
  
  desc "Deploy lên TestFlight"
  lane :deploy_testflight do
    build_ios(signing_type: 'appstore', export_method: 'app-store')
    
    upload_to_testflight(
      ipa: 'build/ios/Runner.ipa',
      api_key_path: ENV['APP_STORE_CONNECT_API_KEY'], # .p8 key path
      skip_waiting_for_build_processing: true, # Không block CI
      changelog: "Build #{ENV['GITHUB_RUN_NUMBER']}",
      distribute_external: false,
      notify_external_testers: false,
    )
  end
end
```

### 3.2. GitHub Actions Workflow — Hoàn chỉnh

```yaml
# .github/workflows/deploy.yml
name: Flutter CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  FLUTTER_VERSION: '3.27.0'

jobs:
  # Job 1: Test (nhanh, chạy đầu tiên)
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true  # Cache Flutter SDK
      
      - name: Get dependencies
        run: flutter pub get
      
      - name: Generate code (build_runner)
        run: dart run build_runner build --delete-conflicting-outputs
      
      - name: Analyze
        run: flutter analyze --no-pub
      
      - name: Unit & Widget Tests
        run: flutter test --coverage
      
      - name: Coverage Report
        uses: codecov/codecov-action@v4
        with:
          file: coverage/lcov.info
          fail_ci_if_error: true
          threshold: 80  # Fail nếu coverage < 80%

  # Job 2: Android Build + Deploy
  deploy-android:
    needs: test  # Chỉ chạy sau khi test pass
    if: github.ref == 'refs/heads/main'  # Chỉ deploy từ main
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true
      
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      
      - name: Setup Ruby for Fastlane
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
      
      - name: Decode Keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > android/app/keystore.jks
      
      - name: Deploy to Firebase App Distribution
        env:
          KEYSTORE_PATH: android/app/keystore.jks
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          FIREBASE_APP_ID_ANDROID: ${{ secrets.FIREBASE_APP_ID_ANDROID }}
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
          GITHUB_RUN_NUMBER: ${{ github.run_number }}
          COMMIT_MESSAGE: ${{ github.event.commits[0].message }}
        run: bundle exec fastlane android deploy_staging
      
      - name: Deploy to Play Store Internal Testing
        if: startsWith(github.ref, 'refs/tags/v')  # Chỉ deploy khi có tag vX.Y.Z
        env:
          PLAY_STORE_JSON_KEY: ${{ secrets.PLAY_STORE_JSON_KEY }}
        run: |
          echo "$PLAY_STORE_JSON_KEY" > /tmp/play_store_key.json
          bundle exec fastlane android deploy_production

  # Job 3: iOS Build + Deploy
  deploy-ios:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: macos-latest  # iOS PHẢI dùng macOS runner
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
      
      - name: Install CocoaPods
        run: cd ios && pod install
      
      - name: Deploy to TestFlight
        env:
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
          MATCH_GIT_TOKEN: ${{ secrets.MATCH_GIT_TOKEN }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
          GITHUB_RUN_NUMBER: ${{ github.run_number }}
        run: bundle exec fastlane ios deploy_testflight

  # Job 4: Notify kết quả
  notify:
    needs: [deploy-android, deploy-ios]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Notify Slack
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          channel: '#deployments'
          text: |
            *Flutter Deploy* - Build #${{ github.run_number }}
            Status: ${{ needs.deploy-android.result }} (Android) / ${{ needs.deploy-ios.result }} (iOS)
            Branch: ${{ github.ref_name }}
            Commit: ${{ github.event.commits[0].message }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### CI Build Time Optimization

```
Baseline (không optimize):
  Test: 8 min
  Android build: 18 min
  iOS build: 25 min
  Total sequential: 51 min

Với optimization:
  Cache Flutter SDK:          -3 min  (subosito/flutter-action cache: true)
  Cache Gradle:               -5 min  (actions/cache for .gradle)
  Cache CocoaPods:            -4 min  (actions/cache for Pods)
  Parallel Android + iOS:     Android=13min, iOS=17min (parallel)
  Total: Test(5min) + max(Android,iOS) = 5 + 17 = 22 min

Từ 51 phút → 22 phút (-57%)
Cost: macOS runner ($0.08/min) × 17 min = $1.36/deploy
      Ubuntu runner ($0.008/min) × 5+13 = $0.14/deploy
      Total: ~$1.50/deploy — rất hợp lý
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Keystore/certificates lưu trong git repo
    ✅ Keystore trong GitHub Secrets (base64 encoded)
    ✅ iOS certs trong Fastlane Match (encrypted private git repo)
    Lý do: Secret leak trong git = game over cho security

[ ] ❌ Không pin exact Flutter version trong CI
    ✅ flutter-version: '3.27.0' — không dùng 'latest'
    Lý do: Flutter update có thể break build/golden tests unexpectedly

[ ] ❌ Build version code thủ công (quên bump → app store reject)
    ✅ version_code = GITHUB_RUN_NUMBER (tự tăng, unique)
    Lý do: Tự động = không bao giờ duplicate, không bao giờ quên

[ ] ❌ Deploy thẳng lên production khi push main
    ✅ main → staging (Firebase App Distribution) → tag → production
    Lý do: Cần QA test trên staging trước khi production release

[ ] ❌ Không có test coverage gate trong CI
    ✅ codecov với threshold: 80 — fail nếu coverage giảm
    Lý do: Coverage giảm theo thời gian nếu không enforce

[ ] ❌ iOS build thất bại vì signing certificate hết hạn
    ✅ Fastlane Match + schedule weekly check certificate expiry
    Lý do: Certificate hết hạn → CI dead → không thể release emergency fix

[ ] ❌ Slack notification không phân biệt thành công/thất bại
    ✅ Color-coded: green = success, red = failure + link đến failed job
    Lý do: Team không biết deploy thất bại → issue không được fix sớm
```
