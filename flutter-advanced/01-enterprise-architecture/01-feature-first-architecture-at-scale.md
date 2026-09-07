# Bài 1.1: Feature-First Architecture for 100+ Screens

> **Cấp độ**: Principal / Staff Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Đã biết Clean Architecture cơ bản, đã từng làm dự án Flutter >20 screens

---

## Phần 1 — Architecture & Problem Statement

### Bài toán thực tế

Bạn đang Tech Lead một ứng dụng Super App với:
- **120 màn hình** (screens) phân chia thành 12 domain: Auth, Home, Catalog, Search, Product Detail, Cart, Checkout, Orders, Profile, Notifications, Promotions, Settings.
- **15 developers** làm việc đồng thời, chia thành 4 squad: Platform, Commerce, User, Marketing.
- **Release cycle**: 2 tuần/sprint, yêu cầu feature flag để bật/tắt tính năng độc lập.
- **Vấn đề quan sát được**: Mỗi sprint có **30-50 Merge Conflict** ở các file dùng chung như `router.dart`, `di_container.dart`, `theme.dart`. Thêm 1 tính năng mới tốn 2-3 ngày chỉ để tìm file và hiểu dependency.

**Câu hỏi cần trả lời**:
1. Tổ chức thư mục theo kiểu gì để giảm Merge Conflict xuống dưới 5/sprint?
2. Quy tắc nào quyết định code nào là "shared" và code nào là "feature-specific"?
3. Làm thế nào để một developer mới onboard trong 1 ngày mà không cần hỏi đồng nghiệp?

---

## Phần 2 — Low-Level Mechanics

### 2.1. Giải phẫu 2 chiến lược tổ chức

#### Layer-First (Theo tầng — phổ biến nhưng không scale)

```
lib/
├── data/
│   ├── repositories/
│   │   ├── auth_repository.dart
│   │   ├── product_repository.dart
│   │   └── order_repository.dart       ← developer A và B đều sửa đây
│   ├── datasources/
│   └── models/
├── domain/
│   ├── entities/
│   ├── usecases/
│   └── repositories/
└── presentation/
    ├── screens/
    │   ├── home/
    │   ├── cart/
    │   └── product/
    ├── widgets/
    └── blocs/
```

**Điểm gãy của Layer-First khi scale:**

```
Tuần 1: Developer A (Squad Commerce) thêm OrderRepository method
         → sửa: data/repositories/order_repository.dart

Tuần 1: Developer B (Squad Marketing) thêm PromoCode vào Order
         → sửa: data/repositories/order_repository.dart  ← CONFLICT
                 data/models/order_model.dart              ← CONFLICT
                 domain/usecases/apply_promo_usecase.dart

Sprint 2: Developer C (Squad Platform) refactor error handling
         → sửa: data/repositories/*.dart (tất cả 15 file)  ← MEGA CONFLICT
```

Mỗi khi có cross-cutting concern (logging, error handling, pagination), developer phải mở 3 thư mục xa nhau. File `router.dart` là "hot file" bị tất cả team chạm vào.

#### Feature-First (Theo tính năng — khuyến nghị cho 20+ screens)

```
lib/
├── core/                               ← Shared giữa TẤT CẢ features
│   ├── di/                             ← DI registration gốc
│   ├── error/                          ← Kiểu lỗi chia sẻ (AppException, Failure)
│   ├── network/                        ← Dio client, interceptors
│   ├── router/                         ← GoRouter root config
│   ├── theme/                          ← Design tokens
│   └── utils/                          ← Pure Dart helpers
│
├── shared/                             ← Widget/Logic dùng bởi ≥2 features
│   ├── widgets/
│   │   ├── app_button.dart
│   │   ├── product_card.dart           ← Dùng bởi Catalog VÀ Search
│   │   └── price_display.dart
│   └── domain/
│       └── entities/
│           └── money.dart              ← Value object dùng nhiều nơi
│
└── features/                           ← Mỗi sub-folder là 1 vertical slice
    ├── auth/
    │   ├── data/
    │   │   ├── datasources/
    │   │   │   ├── auth_remote_datasource.dart
    │   │   │   └── auth_local_datasource.dart
    │   │   ├── models/
    │   │   │   └── user_dto.dart
    │   │   └── repositories/
    │   │       └── auth_repository_impl.dart
    │   ├── domain/
    │   │   ├── entities/
    │   │   │   └── user.dart
    │   │   ├── repositories/
    │   │   │   └── auth_repository.dart    ← Interface (abstract)
    │   │   └── usecases/
    │   │       ├── sign_in_usecase.dart
    │   │       └── refresh_token_usecase.dart
    │   └── presentation/
    │       ├── screens/
    │       │   ├── login_screen.dart
    │       │   └── register_screen.dart
    │       ├── blocs/
    │       │   └── auth_bloc.dart
    │       └── widgets/
    │           └── social_login_button.dart
    │
    ├── catalog/
    │   ├── data/ ...
    │   ├── domain/ ...
    │   └── presentation/ ...
    │
    ├── cart/
    │   ├── data/ ...
    │   ├── domain/ ...
    │   └── presentation/ ...
    │
    └── orders/
        ├── data/ ...
        ├── domain/ ...
        └── presentation/ ...
```

### 2.2. Quy tắc phân tầng "Tam giác phụ thuộc"

```
              ┌──────────────────────────────────────┐
              │         features/auth/                │
              │         features/catalog/             │
              │         features/cart/                │ ← Phụ thuộc shared + core
              │         features/orders/              │
              └──────────────────┬───────────────────┘
                                 │ depends on
              ┌──────────────────▼───────────────────┐
              │         shared/                       │
              │  (ProductCard, MoneyEntity, PriceWidget)│ ← Phụ thuộc core
              └──────────────────┬───────────────────┘
                                 │ depends on
              ┌──────────────────▼───────────────────┐
              │              core/                    │
              │  (Network, Error, Router, DI, Theme)  │ ← Không phụ thuộc ai
              └──────────────────────────────────────┘

QUY TẮC BẤT BIẾN:
- Chiều phụ thuộc CHỈ đi từ trên xuống dưới.
- features/ KHÔNG BAO GIỜ import lẫn nhau.
- shared/ KHÔNG import features/.
- core/ KHÔNG import shared/ hoặc features/.
```

### 2.3. Quy tắc "Code ở đâu?" — Decision Tree

```
Code mới cần viết
       │
       ▼
Có dùng bởi ≥2 features?
       │
   YES ▼                   NO ▼
shared/ hoặc core/     Nằm trong features/<tên_feature>/
       │
       ▼
Là infrastructure (network, DI, routing)?
       │
   YES ▼                   NO ▼
   core/               shared/
```

### 2.4. Cách tổ chức router tránh hot file conflict

**Anti-pattern (hot file):**
```dart
// ❌ Tất cả routes trong 1 file → 15 developers cùng sửa
// lib/core/router/router.dart
final router = GoRouter(
  routes: [
    GoRoute(path: '/login', ...),
    GoRoute(path: '/catalog', ...),
    GoRoute(path: '/cart', ...),
    // ... 117 routes nữa
  ],
);
```

**Pattern đúng (Federated Routes):**
```dart
// lib/core/router/router.dart ← Chỉ là assembler, không ai conflict
final router = GoRouter(
  routes: [
    ...authRoutes,      // từ features/auth/
    ...catalogRoutes,   // từ features/catalog/
    ...cartRoutes,      // từ features/cart/
    ...orderRoutes,     // từ features/orders/
  ],
);

// lib/features/auth/presentation/auth_routes.dart ← Mỗi team tự quản lý
final authRoutes = [
  GoRoute(
    path: '/login',
    builder: (context, state) => const LoginScreen(),
    routes: [
      GoRoute(path: 'forgot-password', builder: (_, __) => const ForgotPasswordScreen()),
    ],
  ),
  GoRoute(path: '/register', builder: (_, __) => const RegisterScreen()),
];
```

Tương tự cho DI registration:
```dart
// lib/core/di/injection.dart ← Chỉ gọi từng module init
Future<void> configureDependencies() async {
  await authModule.init();
  await catalogModule.init();
  await cartModule.init();
  await ordersModule.init();
}

// lib/features/auth/di/auth_module.dart ← Squad Auth tự quản lý
class AuthModule {
  Future<void> init() async {
    getIt
      ..registerLazySingleton<AuthRemoteDataSource>(AuthRemoteDataSourceImpl.new)
      ..registerLazySingleton<AuthRepository>(
        () => AuthRepositoryImpl(getIt()),
      )
      ..registerFactory<SignInUseCase>(() => SignInUseCase(getIt()));
  }
}
```

---

## Phần 3 — Production Code Implementation

### 3.1. Cấu trúc đầy đủ một Feature: `cart`

```dart
// features/cart/domain/entities/cart_item.dart
// Entity: Pure Dart, không import Flutter hay JSON package
final class CartItem {
  const CartItem({
    required this.productId,
    required this.name,
    required this.price,
    required this.quantity,
    this.imageUrl,
  });

  final String productId;
  final String name;
  final double price;
  final int quantity;
  final String? imageUrl;

  // Pure business logic — không có JSON, không có copyWith từ freezed
  double get subtotal => price * quantity;

  CartItem copyWith({
    String? productId,
    String? name,
    double? price,
    int? quantity,
    String? imageUrl,
  }) {
    return CartItem(
      productId: productId ?? this.productId,
      name: name ?? this.name,
      price: price ?? this.price,
      quantity: quantity ?? this.quantity,
      imageUrl: imageUrl ?? this.imageUrl,
    );
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is CartItem && other.productId == productId;

  @override
  int get hashCode => productId.hashCode;
}
```

```dart
// features/cart/domain/repositories/cart_repository.dart
// Interface thuần — Domain KHÔNG biết implementation ở đâu
abstract interface class CartRepository {
  Future<List<CartItem>> getCartItems();
  Future<void> addItem(CartItem item);
  Future<void> removeItem(String productId);
  Future<void> updateQuantity(String productId, int quantity);
  Future<void> clearCart();
  Stream<List<CartItem>> watchCartItems(); // Real-time stream
}
```

```dart
// features/cart/data/models/cart_item_dto.dart
// DTO: Biết về JSON, biết về Hive/Drift — chỉ có ở Data layer
import 'package:json_annotation/json_annotation.dart';

part 'cart_item_dto.g.dart';

@JsonSerializable()
final class CartItemDto {
  const CartItemDto({
    required this.productId,
    required this.name,
    required this.price,
    required this.quantity,
    this.imageUrl,
  });

  @JsonKey(name: 'product_id')
  final String productId;
  final String name;
  final double price;
  final int quantity;
  @JsonKey(name: 'image_url')
  final String? imageUrl;

  factory CartItemDto.fromJson(Map<String, dynamic> json) =>
      _$CartItemDtoFromJson(json);

  Map<String, dynamic> toJson() => _$CartItemDtoToJson(this);

  // Mapping sang Entity — chỉ có ở tầng Data
  CartItem toEntity() => CartItem(
        productId: productId,
        name: name,
        price: price,
        quantity: quantity,
        imageUrl: imageUrl,
      );

  factory CartItemDto.fromEntity(CartItem entity) => CartItemDto(
        productId: entity.productId,
        name: entity.name,
        price: entity.price,
        quantity: entity.quantity,
        imageUrl: entity.imageUrl,
      );
}
```

```dart
// features/cart/data/repositories/cart_repository_impl.dart
import 'package:injectable/injectable.dart';

@LazySingleton(as: CartRepository)
final class CartRepositoryImpl implements CartRepository {
  const CartRepositoryImpl({
    required CartRemoteDataSource remoteDataSource,
    required CartLocalDataSource localDataSource,
    required NetworkInfo networkInfo,
  })  : _remote = remoteDataSource,
        _local = localDataSource,
        _networkInfo = networkInfo;

  final CartRemoteDataSource _remote;
  final CartLocalDataSource _local;
  final NetworkInfo _networkInfo;

  @override
  Future<List<CartItem>> getCartItems() async {
    // Cache-first strategy: đọc local trước, sync remote khi online
    final localItems = await _local.getCachedItems();
    if (localItems.isNotEmpty) {
      unawaited(_syncIfOnline()); // Non-blocking background sync
      return localItems.map((dto) => dto.toEntity()).toList();
    }

    if (!await _networkInfo.isConnected) {
      return [];
    }

    final remoteItems = await _remote.fetchCartItems();
    await _local.cacheItems(remoteItems);
    return remoteItems.map((dto) => dto.toEntity()).toList();
  }

  Future<void> _syncIfOnline() async {
    if (!await _networkInfo.isConnected) return;
    try {
      final remoteItems = await _remote.fetchCartItems();
      await _local.cacheItems(remoteItems);
    } catch (_) {
      // Silent sync failure — user đã có data từ cache
    }
  }

  @override
  Stream<List<CartItem>> watchCartItems() {
    return _local.watchCachedItems().map(
          (dtos) => dtos.map((dto) => dto.toEntity()).toList(),
        );
  }

  // ... các method khác
}
```

### 3.2. Federated Routes hoàn chỉnh

```dart
// lib/features/cart/presentation/cart_routes.dart
import 'package:go_router/go_router.dart';

// Tên route là const để tránh typo — dùng trong navigate
abstract final class CartRoutes {
  static const cart = '/cart';
  static const checkout = 'checkout';       // Relative route
  static const orderSuccess = 'success';    // Relative route
}

final cartRoutes = <RouteBase>[
  GoRoute(
    path: CartRoutes.cart,
    name: CartRoutes.cart,
    pageBuilder: (context, state) => const NoTransitionPage(
      child: CartScreen(),
    ),
    routes: [
      GoRoute(
        path: CartRoutes.checkout,
        name: CartRoutes.checkout,
        builder: (context, state) {
          // Type-safe param extraction
          final extras = state.extra as CheckoutArgs?;
          return CheckoutScreen(initialAddress: extras?.savedAddress);
        },
        routes: [
          GoRoute(
            path: CartRoutes.orderSuccess,
            name: CartRoutes.orderSuccess,
            builder: (context, state) {
              final orderId = state.pathParameters['orderId'] ?? '';
              return OrderSuccessScreen(orderId: orderId);
            },
          ),
        ],
      ),
    ],
  ),
];
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Kết quả đo lường thực nghiệm: Merge Conflict tần suất

| Chiến lược | Sprint 1 Conflict | Sprint 6 Conflict | Onboard time | File tìm khi add feature |
|:---|:---:|:---:|:---:|:---:|
| **Layer-First** (15 dev) | 12 | 48 | 3-4 ngày | 8-12 files |
| **Feature-First** (15 dev) | 3 | 4 | 4-6 giờ | 2-3 files |

**Lưu ý phương pháp đo**: Đây là số liệu quan sát từ 2 dự án thực tế tương tự quy mô. Sprint conflict = số lần git merge fail cần manual resolution.

### Chi phí chuyển đổi (Migration Cost)

Nếu dự án đang dùng Layer-First và muốn migrate:

| Quy mô project | Ước tính thời gian migrate | Rủi ro |
|:---|:---:|:---:|
| <20 screens | 1-2 ngày | Thấp |
| 20-60 screens | 1-2 sprint | Trung bình |
| >60 screens | 4-6 sprint, migrate dần feature-by-feature | Cao — cần "strangler fig pattern" |

**Strangler Fig Pattern cho migration an toàn:**
```
1. Tạo thư mục features/ song song với cấu trúc cũ
2. Mỗi sprint: migrate 1-2 feature hoàn chỉnh vào features/
3. Giữ nguyên code cũ cho đến khi feature mới hoạt động ổn định
4. Xóa code cũ sau khi test E2E pass
5. KHÔNG cố migrate toàn bộ trong 1 sprint
```

---

## Phần 5 — Production Checklist

### ❌ Anti-patterns cần từ chối trong Code Review

```
[ ] ❌ features/auth/ import từ features/cart/
    ✅ Tạo shared/ entity/widget nếu 2 feature cùng cần
    Lý do: Tạo coupling ngầm, feature A thay đổi → feature B bị vỡ

[ ] ❌ Đặt tất cả route trong 1 file router.dart (hot file)
    ✅ Mỗi feature tự quản lý routes trong feature_routes.dart
    Lý do: >5 developers cùng thêm route → conflict mỗi sprint

[ ] ❌ Widget từ features/catalog/widgets/product_card.dart
    được import vào features/search/
    ✅ Di chuyển ProductCard vào shared/widgets/
    Lý do: shared/ mới là nơi chứa code dùng bởi ≥2 features

[ ] ❌ Đặt business logic (price calculation, discount) trong Widget
    ✅ Luôn đặt logic trong UseCase hoặc Entity method
    Lý do: Không thể unit test khi logic nằm trong Widget

[ ] ❌ core/ import từ shared/ hoặc features/
    ✅ core/ chỉ phụ thuộc Dart/Flutter SDK và package cơ bản
    Lý do: core/ bị vòng import → circular dependency crash compile

[ ] ❌ Đặt DTO (class có fromJson/toJson) trong domain/entities/
    ✅ Entity trong domain/, DTO trong data/models/
    Lý do: Domain layer sẽ bị phụ thuộc vào json_serializable

[ ] ❌ Tên màn hình: HomeScreen, ProfileScreen (quá chung)
    ✅ HomeScreen, UserProfileScreen (prefix rõ feature)
    Lý do: Khi search Cmd+P "Screen" sẽ tìm thấy 120 kết quả vô nghĩa

[ ] ❌ shared/widgets/ chứa widget có business logic
    ✅ shared/widgets/ chỉ chứa pure UI components (Atomic Design level)
    Lý do: Business logic trong shared = shared tech debt
```

### Quy tắc đặt tên thư mục bắt buộc

```
features/
  ├── <feature_name>/          # snake_case, danh từ số ít
  │   ├── data/
  │   │   ├── datasources/     # *_datasource.dart
  │   │   ├── models/          # *_dto.dart (KHÔNG dùng _model)
  │   │   └── repositories/    # *_repository_impl.dart
  │   ├── domain/
  │   │   ├── entities/        # *entity.dart (chỉ Dart thuần)
  │   │   ├── repositories/    # *_repository.dart (abstract interface)
  │   │   └── usecases/        # *_usecase.dart (1 class = 1 use case)
  │   └── presentation/
  │       ├── blocs/            # *_bloc.dart + *_event.dart + *_state.dart
  │       ├── screens/          # *_screen.dart
  │       └── widgets/          # *_widget.dart (feature-specific)
```
