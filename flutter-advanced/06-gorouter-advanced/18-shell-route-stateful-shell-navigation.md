# Bài 6.2: StatefulShellRoute — Nested Navigation & State Persistence

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Đã đọc Bài 6.1

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Bottom Nav bị reset state khi switch tab

```
User scenario (Super App - 5 tabs):
1. User ở Tab "Tìm kiếm" → search "iPhone" → scroll xuống 50 items
2. User tap Tab "Giỏ hàng"
3. User tap Tab "Tìm kiếm" lại
4. ❌ BUG: Search bị reset về trống, phải search lại

Root cause: Dùng ShellRoute thay vì StatefulShellRoute
ShellRoute rebuild toàn bộ tab khi switch
→ BLoC bị dispose → state mất → UI reset
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. ShellRoute vs StatefulShellRoute

```
ShellRoute:
Widget Tree khi ở Tab "Tìm kiếm":
  ScaffoldWithBottomNav
  └─ SearchScreen (Active)
  
Khi switch sang Tab "Giỏ hàng":
  ScaffoldWithBottomNav
  └─ CartScreen (Active)
  [SearchScreen bị UNMOUNT → BLoC dispose]
  
Khi switch lại Tab "Tìm kiếm":
  ScaffoldWithBottomNav
  └─ SearchScreen (Rebuilt từ đầu → BLoC created mới)
  [SearchBLoC.search("iPhone") = empty state]

──────────────────────────────────────────────

StatefulShellRoute:
Widget Tree khi ở Tab "Tìm kiếm":
  ScaffoldWithBottomNav
  └─ IndexedStack (index: 1 = Search tab)
      ├─ [0] HomeScreen (ALIVE nhưng offstage)
      ├─ [1] SearchScreen (ACTIVE, hiển thị)
      ├─ [2] CartScreen (ALIVE nhưng offstage)
      ├─ [3] OrdersScreen (ALIVE nhưng offstage)
      └─ [4] ProfileScreen (ALIVE nhưng offstage)

Khi switch sang Tab "Giỏ hàng" (index: 2):
  IndexedStack (index: 2)
  ├─ [0] HomeScreen (ALIVE, offstage)
  ├─ [1] SearchScreen (ALIVE, offstage) ← KHÔNG dispose
  ├─ [2] CartScreen (ACTIVE)
  ...

Khi switch lại Tab "Tìm kiếm":
  IndexedStack (index: 1)
  ├─ [1] SearchScreen (ALIVE → tiếp tục state cũ)
  → BLoC vẫn có "iPhone" search results
  → ScrollPosition được khôi phục
```

### 2.2. Memory Trade-off của IndexedStack

```
IndexedStack giữ tất cả tab widget trees trong memory:
  5 tabs × avg 5MB per tab = 25MB tổng
  
Nếu tất cả tabs đều nặng (infinite list):
  5 tabs × 30MB = 150MB → OOM trên thiết bị cũ

Giải pháp: Lazy initialization
  - Chỉ tạo tab widget khi user visit lần đầu
  - Dùng ValueKey để control creation
  - PageStorageKey để restore scroll position
```

---

## Phần 3 — Production Code Implementation

### 3.1. StatefulShellRoute hoàn chỉnh

```dart
// lib/features/shell/presentation/shell_routes.dart

final shellRoutes = StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) {
    // navigationShell là shell controller để switch tab
    return AppShell(navigationShell: navigationShell);
  },
  branches: [
    // Branch 0: Home
    StatefulShellBranch(
      // navigatorKey quan trọng: mỗi branch có navigator riêng
      navigatorKey: _homeNavigatorKey,
      routes: [
        GoRoute(
          path: '/home',
          name: AppRouteNames.home,
          pageBuilder: (context, state) => const NoTransitionPage(
            child: HomeScreen(),
          ),
          routes: [
            // Sub-routes của Home — nằm trên HOME navigator stack
            GoRoute(
              path: 'featured/:id',
              name: AppRouteNames.featuredDetail,
              builder: (context, state) => FeaturedDetailScreen(
                id: state.pathParameters['id']!,
              ),
            ),
          ],
        ),
      ],
    ),
    
    // Branch 1: Search
    StatefulShellBranch(
      navigatorKey: _searchNavigatorKey,
      routes: [
        GoRoute(
          path: '/search',
          name: AppRouteNames.search,
          pageBuilder: (context, state) => const NoTransitionPage(
            child: SearchScreen(),
          ),
        ),
      ],
    ),
    
    // Branch 2: Cart
    StatefulShellBranch(
      navigatorKey: _cartNavigatorKey,
      routes: [
        GoRoute(
          path: '/cart',
          name: AppRouteNames.cart,
          pageBuilder: (context, state) => const NoTransitionPage(
            child: CartScreen(),
          ),
          routes: [
            GoRoute(
              path: 'checkout',
              name: AppRouteNames.checkout,
              builder: (context, state) => const CheckoutScreen(),
            ),
          ],
        ),
      ],
    ),
    
    // Branch 3: Orders
    StatefulShellBranch(
      navigatorKey: _ordersNavigatorKey,
      routes: [
        GoRoute(
          path: '/orders',
          name: AppRouteNames.orders,
          pageBuilder: (context, state) => const NoTransitionPage(
            child: OrdersScreen(),
          ),
          routes: [
            GoRoute(
              path: ':orderId',
              name: AppRouteNames.orderDetail,
              builder: (context, state) => OrderDetailScreen(
                orderId: state.pathParameters['orderId']!,
              ),
            ),
          ],
        ),
      ],
    ),
    
    // Branch 4: Profile
    StatefulShellBranch(
      navigatorKey: _profileNavigatorKey,
      routes: [
        GoRoute(
          path: '/profile',
          name: AppRouteNames.profile,
          pageBuilder: (context, state) => const NoTransitionPage(
            child: ProfileScreen(),
          ),
        ),
      ],
    ),
  ],
);

// NavigatorKeys — mỗi branch cần key riêng
final _homeNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'home');
final _searchNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'search');
final _cartNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'cart');
final _ordersNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'orders');
final _profileNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'profile');
```

### 3.2. AppShell Widget — Bottom Navigation

```dart
// lib/features/shell/presentation/widgets/app_shell.dart
class AppShell extends StatelessWidget {
  const AppShell({super.key, required this.navigationShell});
  final StatefulNavigationShell navigationShell;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: navigationShell, // IndexedStack của các branches
      bottomNavigationBar: _AppBottomNavBar(
        currentIndex: navigationShell.currentIndex,
        onTap: _onTabTap,
      ),
    );
  }

  void _onTabTap(int index) {
    // goBranch: switch sang branch khác mà giữ state
    navigationShell.goBranch(
      index,
      // initialLocation: true = khi double-tap tab → về tab root
      // (giống iOS tab behavior)
      initialLocation: index == navigationShell.currentIndex,
    );
  }
}

class _AppBottomNavBar extends StatelessWidget {
  const _AppBottomNavBar({
    required this.currentIndex,
    required this.onTap,
  });

  final int currentIndex;
  final ValueChanged<int> onTap;

  static const _tabs = [
    (icon: Icons.home_outlined, activeIcon: Icons.home, label: 'Trang chủ'),
    (icon: Icons.search_outlined, activeIcon: Icons.search, label: 'Tìm kiếm'),
    (icon: Icons.shopping_cart_outlined, activeIcon: Icons.shopping_cart, label: 'Giỏ hàng'),
    (icon: Icons.receipt_long_outlined, activeIcon: Icons.receipt_long, label: 'Đơn hàng'),
    (icon: Icons.person_outline, activeIcon: Icons.person, label: 'Tài khoản'),
  ];

  @override
  Widget build(BuildContext context) {
    return NavigationBar(
      selectedIndex: currentIndex,
      onDestinationSelected: onTap,
      destinations: [
        for (final tab in _tabs)
          NavigationDestination(
            icon: Icon(tab.icon),
            selectedIcon: Icon(tab.activeIcon),
            label: tab.label,
          ),
      ],
    );
  }
}
```

### 3.3. Lazy Tab Initialization — Tối ưu memory

```dart
// Chỉ tạo tab widget khi user visit lần đầu
class LazyTabWidget extends StatefulWidget {
  const LazyTabWidget({
    super.key,
    required this.isActive,
    required this.child,
  });
  final bool isActive;
  final Widget child;

  @override
  State<LazyTabWidget> createState() => _LazyTabWidgetState();
}

class _LazyTabWidgetState extends State<LazyTabWidget> {
  bool _hasBeenActive = false;

  @override
  void didUpdateWidget(LazyTabWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.isActive && !_hasBeenActive) {
      setState(() => _hasBeenActive = true);
    }
  }

  @override
  Widget build(BuildContext context) {
    if (!_hasBeenActive) {
      // Tab chưa được visit → không render → tiết kiệm memory
      return const SizedBox.shrink();
    }
    
    return Offstage(
      offstage: !widget.isActive,
      child: TickerMode(
        // Dừng tất cả animation ở tab không active → tiết kiệm CPU
        enabled: widget.isActive,
        child: widget.child,
      ),
    );
  }
}
```

### 3.4. Deep Link vào nested tab route

```dart
// User nhận notification deep link: myapp://orders/ORD-456
// → Cần vào Tab "Đơn hàng" → OrderDetailScreen

// GoRouter tự động handle:
GoRouter(
  initialLocation: '/home',
  routes: [
    ...shellRoutes, // Bao gồm /orders/:orderId
  ],
)

// Khi nhận deep link myapp://orders/ORD-456:
// 1. GoRouter parse → location: /orders/ORD-456
// 2. Match: StatefulShellRoute(branch: orders) / GoRoute(path: ':orderId')
// 3. GoRouter switch sang orders branch (index 3)
// 4. Navigate đến OrderDetailScreen(orderId: 'ORD-456') trong orders navigator
// 5. Back button → pop OrderDetail → về OrdersScreen (trong cùng tab)
// → Đúng behavior, không về Home
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Rebuild count: ShellRoute vs StatefulShellRoute

```
Test: Switch tab 10 lần (Home → Search → Cart → Orders → Profile → lặp lại 2x)
Widget inspector: đếm rebuild events

ShellRoute:
  Mỗi tab switch: toàn bộ tab widget tree rebuild
  10 tab switches × 50 widgets/tab = 500 widget rebuilds
  BLoC dispose + recreate: 10 × 5 BLoC = 50 BLoC lifecycle events
  Data refetch: 10 lần gọi API

StatefulShellRoute:
  Tab switch đầu tiên đến mỗi tab: 1 lần build (lazy)
  Sau đó: 0 rebuild (widget tree persisted in IndexedStack)
  BLoC: 5 BLoC created once, never disposed on tab switch
  Data refetch: 0 (data cached trong BLoC state)

Memory:
  ShellRoute: ~15MB (1 tab active)
  StatefulShellRoute: ~60MB (5 tabs in memory)
  Trade-off: +45MB để đổi lấy zero rebuild + instant tab switch
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Dùng ShellRoute cho Bottom Navigation Bar
    ✅ StatefulShellRoute.indexedStack cho persistent tab state
    Lý do: ShellRoute dispose tab khi switch → state mất → bad UX

[ ] ❌ Thiếu navigatorKey cho từng StatefulShellBranch
    ✅ Mỗi branch cần GlobalKey<NavigatorState> riêng biệt với debugLabel
    Lý do: Shared navigator key → pop() ảnh hưởng sai navigator

[ ] ❌ context.go() với path route trong nested navigator
    ✅ context.goNamed() với route name để tránh path conflict
    Lý do: /home và /orders có thể có sub-route trùng tên

[ ] ❌ Không có TickerMode(enabled: false) cho inactive tabs
    ✅ Wrap tab content với TickerMode khi offstage
    Lý do: Animation trong inactive tab vẫn consume CPU → battery drain

[ ] ❌ Tất cả 5 tabs load đầy đủ khi app launch
    ✅ Implement lazy initialization — chỉ tạo tab khi user visit
    Lý do: 5 tabs × full content = heavy app launch time và memory

[ ] ❌ Double-tap tab không navigate về tab root
    ✅ navigationShell.goBranch(index, initialLocation: sameIndex)
    Lý do: iOS/Android standard: double-tap tab → scroll to top + pop to root
```
