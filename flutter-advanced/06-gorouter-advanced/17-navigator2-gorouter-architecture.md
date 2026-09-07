# Bài 6.1: Navigator 2.0 & GoRouter Architecture Deep Dive

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Biết GoRouter cơ bản; đọc qua Navigator 1.0

---

## Phần 1 — Architecture & Problem Statement

### Tại sao cần Navigator 2.0?

**Navigator 1.0 — Imperative (push/pop) giới hạn:**

```
Vấn đề thực tế:
1. Web support: URL bar không sync với app state
   User copy URL /products/123 → back button về home thay vì product page
   
2. Deep linking: App nhận universal link → không điều hướng đúng
   Notification: "Đơn hàng #456 đã giao" → tap → về Home, không vào Order #456
   
3. Back button: Android physical back → pop stack, không restore state
   User ở /cart/checkout → back → về /home thay vì /cart

4. State restoration: App killed → reopen → về home, mất navigation state
```

Navigator 2.0 giải quyết tất cả bằng cách làm cho navigation **declarative và URL-driven**.

---

## Phần 2 — Low-Level Mechanics

### 2.1. Navigator 2.0 — 3 thành phần cốt lõi

```dart
// RouterDelegate: Quyết định Widget nào được render dựa trên Router State
// → Giống như "build() cho Navigator"
abstract class RouterDelegate<T> {
  Widget build(BuildContext context); // Build Navigator widget
  Future<void> setNewRoutePath(T configuration); // URL thay đổi
  T? get currentConfiguration; // State hiện tại (để sync URL)
}

// RouteInformationParser: Parse URL string → typed Route config
abstract class RouteInformationParser<T> {
  Future<T> parseRouteInformation(RouteInformation routeInformation);
  RouteInformation? restoreRouteInformation(T configuration);
}

// RouteInformationProvider: Nguồn URL (OS URL scheme, in-memory)
// Flutter tự cung cấp: PlatformRouteInformationProvider
```

**Vấn đề của Navigator 2.0 thuần:** Cực kỳ verbose — 200+ dòng cho app đơn giản. GoRouter wrap phức tạp này thành declarative config 30 dòng.

### 2.2. GoRouter — Cách GoRouterDelegate hoạt động

```
GoRouter.go('/products/123')
    │
    ▼
GoRouterDelegate.go()
    │ update internal _currentConfiguration
    ▼
GoRouterDelegate.notifyListeners()
    │
    ▼
Router widget rebuild
    │
    ▼
GoRouterDelegate.build(context)
    │ parse current location → tìm matching GoRoute
    ▼
Navigator(
  pages: [
    MaterialPage(child: HomeScreen()),      // /
    MaterialPage(child: ProductsScreen()),  // /products
    MaterialPage(child: ProductDetailScreen(id: '123')), // /products/123
  ],
)
    │
    ▼
URL bar update (Web/deep link) via RouteInformationProvider
```

### 2.3. GoRoute vs ShellRoute vs StatefulShellRoute

```
GoRoute (single screen):
  /home  → HomeScreen()
  /home/detail → DetailScreen()
  
  Navigator stack: [HomeScreen, DetailScreen]
  Layout:         Mỗi screen độc lập, không shared UI

ShellRoute (shared UI wrapper):
  ShellRoute(
    builder: (ctx, state, child) => ScaffoldWithBottomNav(body: child),
    routes: [
      GoRoute(path: '/home', builder: ...),
      GoRoute(path: '/search', builder: ...),
    ]
  )
  
  Mọi route con được wrap trong ScaffoldWithBottomNav
  NHƯNG: tab state bị RESET khi switch tab (không keep-alive)

StatefulShellRoute (keep state between tabs):
  → Dùng IndexedStack để giữ widget tree của mỗi tab
  → Tab A scroll xuống → switch Tab B → switch lại Tab A → vẫn ở vị trí cũ
  → BLoC/Provider trong Tab A không bị dispose khi switch tab
```

---

## Phần 3 — Production Code Implementation

### 3.1. GoRouter Setup Production-grade

```dart
// lib/core/router/app_router.dart
import 'package:go_router/go_router.dart';
import 'package:injectable/injectable.dart';

@singleton
final class AppRouter {
  AppRouter({
    required AuthStateRepository authRepository,
    required AppLoggerService logger,
  })  : _authRepo = authRepository,
        _logger = logger;

  final AuthStateRepository _authRepo;
  final AppLoggerService _logger;
  
  late final GoRouter router = GoRouter(
    initialLocation: '/home',
    debugLogDiagnostics: kDebugMode, // Log route changes trong debug
    
    // Error handling — route không tìm thấy
    errorBuilder: (context, state) => ErrorScreen(
      error: state.error,
      onGoHome: () => context.go('/home'),
    ),
    
    // Redirect: Auth guard + route guards
    redirect: _handleRedirect,
    
    // Refresh: Khi auth state thay đổi → re-evaluate redirect
    refreshListenable: _authRepo.authStateListenable,
    
    routes: [
      // Assemble routes từ từng feature module
      ...authRoutes,
      ...shellRoutes,    // Bottom nav với StatefulShellRoute
      ...onboardingRoutes,
    ],
  );

  String? _handleRedirect(BuildContext context, GoRouterState state) {
    final isLoggedIn = _authRepo.isAuthenticated;
    final isOnAuthRoute = state.matchedLocation.startsWith('/auth');
    final isOnOnboarding = state.matchedLocation.startsWith('/onboarding');
    
    _logger.logNavigation(state.matchedLocation);
    
    // Chưa login → redirect về login (trừ khi đang ở auth route)
    if (!isLoggedIn && !isOnAuthRoute && !isOnOnboarding) {
      // Lưu intended destination để redirect sau khi login
      return '/auth/login?redirect=${Uri.encodeComponent(state.uri.toString())}';
    }
    
    // Đã login → không cho vào auth route
    if (isLoggedIn && isOnAuthRoute) {
      return '/home';
    }
    
    return null; // Không redirect
  }
}

// Khai báo route names là const để tránh typo
abstract final class AppRouteNames {
  static const home = 'home';
  static const search = 'search';
  static const cart = 'cart';
  static const orders = 'orders';
  static const profile = 'profile';
  static const productDetail = 'product-detail';
  static const checkout = 'checkout';
  static const login = 'login';
  static const register = 'register';
}
```

### 3.2. Type-safe Navigation với GoRouterState

```dart
// Extra object approach — type-safe, không dùng Map<String, dynamic>
@immutable
class ProductDetailArgs {
  const ProductDetailArgs({
    required this.productId,
    this.referrerScreen,
    this.initialTab = ProductDetailTab.overview,
  });
  
  final String productId;
  final String? referrerScreen;
  final ProductDetailTab initialTab;
}

// Navigate với type-safe args:
context.pushNamed(
  AppRouteNames.productDetail,
  pathParameters: {'productId': product.id},
  extra: ProductDetailArgs(
    productId: product.id,
    referrerScreen: 'catalog',
    initialTab: ProductDetailTab.reviews,
  ),
);

// Nhận trong builder:
GoRoute(
  path: '/products/:productId',
  name: AppRouteNames.productDetail,
  builder: (context, state) {
    final productId = state.pathParameters['productId']!;
    final args = state.extra as ProductDetailArgs?;
    return ProductDetailScreen(
      productId: productId,
      initialTab: args?.initialTab ?? ProductDetailTab.overview,
    );
  },
),
```

### 3.3. Phân tích GoRouter Source — Cách build Navigator pages

```dart
// Simplified version của GoRouterDelegate._buildPages() để hiểu cơ chế

List<Page<dynamic>> _buildPages(String location) {
  final matches = _router.routeConfiguration.findMatch(location);
  
  // Mỗi route match → 1 Page trong Navigator stack
  return matches.map((match) {
    final route = match.route as GoRoute;
    final state = GoRouterState.fromMatch(match);
    
    return route.buildPage(context, state);
    // buildPage trả về MaterialPage, CupertinoPage, hoặc custom Page
  }).toList();
}

// Navigator nhận pages list → tạo Element stack tương ứng
Navigator(
  pages: _buildPages(currentLocation),
  onDidRemovePage: (page) {
    // User pop → GoRouter sync state với URL
    _router.pop();
  },
)
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Navigate overhead

```
context.go('/products/123') [GoRouter]:
  Parse location string:        ~0.1ms
  Match route tree:             ~0.2ms (O(n) routes)
  Build Page list:              ~0.5ms
  Navigator rebuild:            ~2-5ms (widget build)
  Total:                        ~3-6ms

Với 200 routes:
  Route matching: O(n) = ~200 comparisons ≈ 0.5ms
  → Không ảnh hưởng UX ngay cả với 500 routes
  
QUAN TRỌNG: Minimize rebuild khi navigate
  - Dùng StatefulShellRoute để không rebuild parent shell
  - Dùng const constructors cho shell widget
  - Cache screen instance trong IndexedStack (StatefulShellRoute tự làm)
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Dùng context.push() cho tab navigation (Bottom Navigation Bar)
    ✅ context.go() cho tab navigation (replace stack, không push)
    Lý do: push thêm vào stack → back button behavior không đúng trên tab nav

[ ] ❌ Route name là String literal rải rác trong code ('product-detail')
    ✅ const trong AppRouteNames class → refactor safe, no typo
    Lý do: Typo trong route name → runtime crash, không phát hiện compile time

[ ] ❌ extra: Map<String, dynamic> cho route arguments
    ✅ extra: typed class (ProductDetailArgs) — implement Equatable
    Lý do: Map không type-safe; args không survive URL restore (web/deep link)

[ ] ❌ Redirect function gọi async API (await) → exception
    ✅ redirect phải SYNCHRONOUS — đọc cached auth state, không call API
    Lý do: GoRouter redirect không hỗ trợ async → throw exception

[ ] ❌ Không set refreshListenable → redirect không chạy lại khi auth thay đổi
    ✅ refreshListenable: authStateListenable → redirect re-evaluate khi login/logout
    Lý do: Không có refreshListenable → user login nhưng vẫn ở login screen

[ ] ❌ errorBuilder không được implement
    ✅ Luôn có errorBuilder để handle 404/unmatched routes gracefully
    Lý do: Deep link sai format → app crash thay vì hiển thị friendly error
```
