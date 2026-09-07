# Bài 6.3: Auth Guards & Deep Linking — Android App Links / iOS Universal Links

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Đã đọc Bài 6.1 và 6.2

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Deep link từ notification xử lý sai

**E-commerce app:**

```
Scenario 1: User chưa login nhận push notification "Đơn hàng đã giao"
  Tap notification → deep link: myapp://orders/ORD-456
  
  ❌ BUG: App mở → màn hình trắng trong 2 giây → về Home
  
  Nên xảy ra:
  → App mở → Auth check → Chưa login → Login screen
  → Login thành công → Auto navigate đến /orders/ORD-456

Scenario 2: App đang ở background (không bị kill)
  Nhận deep link myapp://products/ABC-789
  
  ❌ BUG: Navigate đến ProductDetail nhưng không cập nhật URL
  
  Nên xảy ra: 
  → App foregrounded → URL updated → ProductDetail displayed

Scenario 3: App bị kill (terminated)
  Nhận deep link khi user tap notification
  
  ❌ BUG: App launch → HomeScreen (deep link bị bỏ qua)
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. Auth Guard Pattern với GoRouter

```
GoRouter redirect callback chạy khi:
1. App launch (initial route)
2. User navigate tới route mới (context.go/push)
3. refreshListenable.notifyListeners() được gọi

Auth Guard flow:
  User navigate → redirect() gọi
                       │
                  Check auth state (synchronous!)
                  /              \
          Authenticated          Not authenticated
              │                      │
         return null             Lưu intended URL
         (cho phép)              return '/auth/login?redirect=...'
```

### 2.2. Deep Link States — 3 trạng thái app

```
TERMINATED (app bị kill):
  Platform → app launch với link URI
  Flutter engine khởi động
  GoRouter.initialLocation = deep link URI
  (UniLink/firebase_dynamic_links provide URI before app starts)

BACKGROUND (app sống nhưng không hiển thị):
  Platform → flutter engine nhận URI
  GoRouter.router.go(uri) được gọi từ link handler
  Redirect check → navigate đúng screen
  App foreground

FOREGROUND (app đang hiển thị):
  User tap in-app link
  context.go(uri) hoặc router.go(uri)
  Immediate navigation
```

### 2.3. Android App Links vs iOS Universal Links

```
App Links (Android) vs Custom Scheme (Android):
  myapp://orders/123       ← Custom scheme: app-specific, không verify
  https://shop.myapp.com/orders/123 ← App Links: domain-verified, trusted
  
  App Links yêu cầu:
  1. HTTPS domain
  2. /.well-known/assetlinks.json trên server
  3. android:autoVerify="true" trong AndroidManifest
  4. Fingerprint SHA-256 của keystore

Universal Links (iOS):
  https://shop.myapp.com/orders/123
  
  Yêu cầu:
  1. HTTPS domain
  2. /.well-known/apple-app-site-association (AASA) trên server
  3. Associated Domains capability trong Xcode
  4. App ID trong AASA file
```

---

## Phần 3 — Production Code Implementation

### 3.1. Auth Guard với refreshListenable

```dart
// lib/core/auth/auth_state_listenable.dart
// ChangeNotifier bridge giữa Stream và GoRouter

final class AuthStateListenable extends ChangeNotifier {
  AuthStateListenable({required AuthRepository authRepository}) {
    // Subscribe auth stream → notify GoRouter khi auth thay đổi
    _subscription = authRepository.watchAuthState().listen(
      (authState) {
        _isAuthenticated = authState is Authenticated;
        _currentUser = authState is Authenticated ? authState.user : null;
        notifyListeners(); // → GoRouter re-evaluate redirect()
      },
    );
  }

  StreamSubscription<AuthState>? _subscription;
  bool _isAuthenticated = false;
  User? _currentUser;

  bool get isAuthenticated => _isAuthenticated;
  User? get currentUser => _currentUser;

  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }
}

// Trong GoRouter:
GoRouter(
  refreshListenable: authStateListenable,
  redirect: (context, state) {
    final isAuth = authStateListenable.isAuthenticated;
    final location = state.uri.toString();
    final isAuthRoute = location.startsWith('/auth/');
    
    if (!isAuth && !isAuthRoute) {
      // Encode intended destination để redirect sau login
      final encodedRedirect = Uri.encodeComponent(location);
      return '/auth/login?redirect=$encodedRedirect';
    }
    
    if (isAuth && isAuthRoute) {
      // Đã login mà vào auth route → về home
      return '/home';
    }
    
    return null;
  },
)
```

### 3.2. Login Screen — Redirect sau khi login thành công

```dart
// features/auth/presentation/screens/login_screen.dart
class LoginScreen extends ConsumerWidget {
  const LoginScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Đọc redirect parameter từ URL
    final state = GoRouterState.of(context);
    final redirectTo = state.uri.queryParameters['redirect'];

    return BlocListener<AuthBloc, AuthState>(
      listener: (context, authState) {
        if (authState is AuthAuthenticated) {
          // Login thành công → navigate về intended destination
          if (redirectTo != null && redirectTo.isNotEmpty) {
            context.go(Uri.decodeComponent(redirectTo));
          } else {
            context.go('/home');
          }
        }
      },
      child: LoginForm(),
    );
  }
}
```

### 3.3. Android App Links Setup

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity
  android:name=".MainActivity"
  android:launchMode="singleTask"
  android:exported="true">
  
  <!-- App Links intent filter -->
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <!-- Domain được verify qua assetlinks.json -->
    <data
      android:scheme="https"
      android:host="shop.myapp.com"/>
  </intent-filter>
  
  <!-- Fallback: custom scheme cho development -->
  <intent-filter>
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <data android:scheme="myapp"/>
  </intent-filter>
</activity>
```

```json
// Đặt tại: https://shop.myapp.com/.well-known/assetlinks.json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.myapp.shop",
    "sha256_cert_fingerprints": [
      "AB:CD:EF:...:12:34"  // SHA-256 của release keystore
    ]
  }
}]
```

### 3.4. iOS Universal Links Setup

```json
// Đặt tại: https://shop.myapp.com/.well-known/apple-app-site-association
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.myapp.shop",
        "paths": [
          "/orders/*",
          "/products/*",
          "/profile",
          "/cart"
        ]
      }
    ]
  }
}
```

```xml
<!-- ios/Runner/Runner.entitlements -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "...">
<plist version="1.0">
<dict>
  <key>com.apple.developer.associated-domains</key>
  <array>
    <string>applinks:shop.myapp.com</string>
  </array>
</dict>
</plist>
```

### 3.5. Deep Link Handler — 3 trạng thái app

```dart
// lib/core/deep_link/deep_link_handler.dart
import 'package:app_links/app_links.dart';

@singleton
final class DeepLinkHandler {
  DeepLinkHandler({required AppRouter appRouter})
      : _appRouter = appRouter;
  
  final AppRouter _appRouter;
  final _appLinks = AppLinks();
  StreamSubscription<Uri>? _subscription;
  
  /// Gọi trong main() sau configureDependencies()
  Future<void> initialize() async {
    // Case 1: TERMINATED → app launch với link
    final initialLink = await _appLinks.getInitialLink();
    if (initialLink != null) {
      // Delay nhỏ để GoRouter init xong trước khi navigate
      await Future.delayed(const Duration(milliseconds: 100));
      _handleLink(initialLink);
    }
    
    // Case 2 & 3: BACKGROUND / FOREGROUND → nhận link khi đang chạy
    _subscription = _appLinks.uriLinkStream.listen(
      _handleLink,
      onError: (Object error) {
        debugPrint('[DeepLink] Error: $error');
      },
    );
  }
  
  void _handleLink(Uri uri) {
    // Normalize: https://shop.myapp.com/orders/123 → /orders/123
    final path = _normalizePath(uri);
    
    if (path != null) {
      // GoRouter xử lý route matching và auth redirect tự động
      _appRouter.router.go(path);
    }
  }
  
  String? _normalizePath(Uri uri) {
    // Custom scheme: myapp://orders/123 → /orders/123
    if (uri.scheme == 'myapp') {
      return '/${uri.host}${uri.path}';
    }
    
    // Universal link: https://shop.myapp.com/orders/123 → /orders/123
    if (uri.host == 'shop.myapp.com') {
      return uri.path + (uri.query.isNotEmpty ? '?${uri.query}' : '');
    }
    
    return null; // Unknown scheme → ignore
  }
  
  void dispose() {
    _subscription?.cancel();
  }
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Deep Link handling latency

```
TERMINATED state (worst case):
  App boot time: 1,200ms (cold start)
  DeepLinkHandler.initialize(): 50ms
  GoRouter redirect: 5ms
  Navigate to screen: 80ms
  Total: ~1,335ms

Optimization: Splash screen delay đến khi deep link resolved
  → User thấy splash → sau đó jump thẳng vào đúng màn hình
  → Không bao giờ thấy Home rồi jump → UX tốt hơn

BACKGROUND state (best case):
  App resume: 50ms
  Link handling: 5ms
  Navigate: 80ms
  Total: ~135ms
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ redirect() function gọi await (async)
    ✅ redirect() phải synchronous — đọc cached auth state
    Lý do: GoRouter không hỗ trợ async redirect → exception hoặc loop

[ ] ❌ Không có refreshListenable cho Auth guard
    ✅ authStateListenable notifyListeners() khi auth thay đổi
    Lý do: Không có refreshListenable → login/logout không trigger redirect

[ ] ❌ Android autoVerify="false" hoặc thiếu
    ✅ android:autoVerify="true" + assetlinks.json đúng fingerprint
    Lý do: App Links không verified → Android hiển thị chooser dialog (bad UX)

[ ] ❌ AASA file không được serve với Content-Type: application/json
    ✅ Server phải serve với Header: Content-Type: application/json
    Lý do: iOS reject AASA nếu Content-Type sai → Universal Links không work

[ ] ❌ Deep link handler không có delay trước khi navigate (terminated state)
    ✅ Future.delayed(100ms) để GoRouter init hoàn tất trước navigate
    Lý do: Quá sớm → router chưa ready → navigate bị ignore hoặc crash

[ ] ❌ Không handle deep link khi app ở foreground
    ✅ Subscribe cả uriLinkStream (foreground) VÀ getInitialLink() (terminated)
    Lý do: getInitialLink() chỉ return cho terminated state

[ ] ❌ Không test deep link với tất cả 3 app states
    ✅ Test matrix: terminated/background/foreground × logged-in/logged-out
    Lý do: Bug thường xuất hiện ở "terminated + not logged in" — trường hợp ít test nhất
```
