# Bài 2.2: Modern Riverpod — AsyncNotifier & Code Generation

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Đã biết Riverpod cơ bản (Provider, StateNotifier)

---

## Phần 1 — Architecture & Problem Statement

### Tại sao cần AsyncNotifier? — Câu chuyện StateNotifier bị deprecated

**Riverpod 2.x thay đổi core API:**

```
Riverpod 1.x (cũ — vẫn hoạt động nhưng deprecated):
StateNotifierProvider<ProductNotifier, ProductState>

Vấn đề với StateNotifier:
1. Không có built-in async lifecycle (initState async phải workaround)
2. Phải tự quản lý loading/error/data state thủ công
3. ref.watch() không compile-safe — typo lúc runtime mới biết
4. Boilerplate cao: 50+ dòng để setup 1 notifier

Riverpod 2.x + Code Generation (khuyến nghị):
@riverpod
class ProductList extends _$ProductList {
  @override
  Future<List<Product>> build() => _fetchProducts();
}
// → tự động có loading/error/data, compile-safe, <10 dòng
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. AsyncNotifier Lifecycle

```
AsyncNotifier.build() được gọi khi:
1. Provider được watch lần đầu tiên
2. Dependency (ref.watch) thay đổi
3. ref.invalidate() được gọi thủ công
4. Provider bị AutoDispose và được watch lại

State machine của AsyncValue:
                  ┌─────────────────────────────────┐
                  │                                 │
        build()   ▼                                 │ dependency changed
   ──► loading ──────────────────────► data(value)  │ hoặc invalidate()
                  │                      │          │
                  │ error                │          │
                  ▼                      └──────────┘
               error(err, stackTrace)
                  │
                  │ refresh / retry
                  └──────────────────► loading (previousData được giữ)
```

### 2.2. AutoDispose — Khi nào provider bị hủy?

```
Không có AutoDispose:
Provider tồn tại SUỐT app — dù không có widget nào watch

Có AutoDispose:
Provider bị hủy khi:
  - Không còn widget nào watch nó
  - Sau 1 khoảng delay (configurable)
  
Widget A watch provider P ──► P alive
Widget A dispose           ──► P alive (delay...)
[No more listeners]        ──► P disposed, data cleared
Widget B watch provider P  ──► P created again, build() gọi lại
```

### 2.3. ref.watch vs ref.read vs ref.listen

```dart
// ref.watch(provider) — rebuild khi state thay đổi
// Dùng trong: build(), AsyncNotifier.build(), đọc dependency
final products = ref.watch(productListProvider); // → AsyncValue<List<Product>>

// ref.read(provider) — đọc 1 lần, không rebuild
// Dùng trong: event handlers, callbacks (KHÔNG dùng trong build())
final notifier = ref.read(productListProvider.notifier);

// ref.listen(provider, callback) — side effect khi state thay đổi
// Dùng trong: navigation, snackbar, analytics
ref.listen(productListProvider, (previous, next) {
  if (next is AsyncError) showErrorSnackbar(next.error.toString());
});
```

### 2.4. Family Provider — Parameterized Provider

```
Không dùng Family:
Cần 3 provider riêng cho 3 category:
  electronicProductsProvider
  clothingProductsProvider
  foodProductsProvider
  → Code duplicate, khó maintain

Dùng Family:
  productsByCategoryProvider(CategoryId.electronics)
  productsByCategoryProvider(CategoryId.clothing)
  productsByCategoryProvider(CategoryId.food)
  → 1 provider definition, param khác nhau → instance khác nhau
  → Mỗi instance có lifecycle độc lập
```

---

## Phần 3 — Production Code Implementation

### 3.1. Setup riverpod_generator

```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^2.6.1
  riverpod_annotation: ^2.6.1

dev_dependencies:
  riverpod_generator: ^2.6.2
  build_runner: ^2.4.13
  custom_lint: ^0.7.5       # Riverpod lint rules
  riverpod_lint: ^2.6.2
```

```yaml
# analysis_options.yaml — bật riverpod lint
analyzer:
  plugins:
    - custom_lint

custom_lint:
  rules:
    - avoid_manual_providers_as_generated_provider_dependency
    - provider_dependencies
    - riverpod_avoid_empty_functional_providers
```

### 3.2. AsyncNotifier cho Infinite Scroll

```dart
// features/catalog/presentation/providers/product_list_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'product_list_provider.g.dart';

@immutable
final class ProductListState {
  const ProductListState({
    required this.products,
    required this.hasNextPage,
    required this.currentPage,
    this.isLoadingMore = false,
  });

  final List<Product> products;
  final bool hasNextPage;
  final int currentPage;
  final bool isLoadingMore;

  ProductListState copyWith({
    List<Product>? products,
    bool? hasNextPage,
    int? currentPage,
    bool? isLoadingMore,
  }) {
    return ProductListState(
      products: products ?? this.products,
      hasNextPage: hasNextPage ?? this.hasNextPage,
      currentPage: currentPage ?? this.currentPage,
      isLoadingMore: isLoadingMore ?? this.isLoadingMore,
    );
  }
}

// @riverpod annotation → code gen tạo productListProvider tự động
@riverpod
class ProductList extends _$ProductList {
  // build() = "constructor" của provider
  // Được gọi khi provider được watch lần đầu hoặc dependency thay đổi
  @override
  Future<ProductListState> build() async {
    // ref.onDispose: cleanup khi provider bị hủy
    ref.onDispose(() => debugPrint('[ProductList] Provider disposed'));

    // ref.watch: khi filterProvider thay đổi → build() được gọi lại tự động
    final filter = ref.watch(productFilterProvider);
    
    return _fetchPage(page: 1, filter: filter);
  }

  Future<ProductListState> _fetchPage({
    required int page,
    required ProductFilter filter,
  }) async {
    final repository = ref.read(productRepositoryProvider);
    final result = await repository.getProducts(
      page: page,
      filter: filter,
      pageSize: 20,
    );

    return switch (result) {
      Success(:final value) => ProductListState(
          products: value.items,
          hasNextPage: value.totalPages > page,
          currentPage: page,
        ),
      Failure_(:final failure) => throw failure, // AsyncNotifier convert thành AsyncError
    };
  }

  /// Load trang tiếp theo — không thay thế, APPEND vào list
  Future<void> loadNextPage() async {
    final currentState = state.valueOrNull;
    
    // Guard: không load nếu đang loading hoặc không còn trang
    if (currentState == null || !currentState.hasNextPage) return;
    if (currentState.isLoadingMore) return;

    // Update state để hiển thị loading indicator ở cuối list
    // Giữ nguyên data cũ — không set state = AsyncLoading()
    state = AsyncData(currentState.copyWith(isLoadingMore: true));

    final filter = ref.read(productFilterProvider);
    final nextPage = currentState.currentPage + 1;

    final result = await AsyncValue.guard(
      () => _fetchPage(page: nextPage, filter: filter),
    );

    state = switch (result) {
      AsyncData(:final value) => AsyncData(
          ProductListState(
            // APPEND sản phẩm mới vào cuối list hiện tại
            products: [...currentState.products, ...value.products],
            hasNextPage: value.hasNextPage,
            currentPage: nextPage,
            isLoadingMore: false,
          ),
        ),
      AsyncError(:final error, :final stackTrace) => AsyncError(error, stackTrace),
      _ => state, // AsyncLoading — không thay đổi
    };
  }

  /// Refresh toàn bộ list về trang đầu
  Future<void> refresh() async {
    // ref.invalidateSelf() → trigger build() gọi lại → reset state
    ref.invalidateSelf();
    // Đợi build() hoàn thành để caller có thể await
    await future;
  }
}
```

### 3.3. Family Provider — Product Detail theo ID

```dart
// features/catalog/presentation/providers/product_detail_provider.dart

@riverpod
class ProductDetail extends _$ProductDetail {
  // build nhận param qua arg — type-safe, không dùng Map<String, dynamic>
  @override
  Future<Product> build(String productId) async {
    // Tự động cancel HTTP request khi provider dispose
    final cancelToken = CancelToken();
    ref.onDispose(cancelToken.cancel);

    final repository = ref.read(productRepositoryProvider);
    final result = await repository.getProductById(
      id: productId,
      cancelToken: cancelToken,
    );

    return switch (result) {
      Success(:final value) => value,
      Failure_(:final failure) => throw failure,
    };
  }

  /// Optimistic update — update UI ngay, rollback nếu server fail
  Future<void> addToWishlist() async {
    final previousState = state;
    
    // 1. Update local state ngay lập tức (optimistic)
    state = state.whenData(
      (product) => product.copyWith(isWishlisted: true),
    );

    // 2. Gửi lên server
    final result = await ref
        .read(wishlistRepositoryProvider)
        .addToWishlist(productId: ref.read(productDetailProvider(arg).notifier).arg);

    // 3. Rollback nếu server fail
    if (result case Failure_()) {
      state = previousState;
    }
  }
}

// Sử dụng Family provider trong Widget
class ProductDetailScreen extends ConsumerWidget {
  const ProductDetailScreen({super.key, required this.productId});
  final String productId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Mỗi productId có provider instance riêng biệt
    final productAsync = ref.watch(productDetailProvider(productId));

    return productAsync.when(
      loading: () => const ProductDetailSkeleton(),
      error: (error, _) => ErrorView(
        message: error is Failure ? error.message : 'Lỗi không xác định',
        onRetry: () => ref.invalidate(productDetailProvider(productId)),
      ),
      data: (product) => ProductDetailView(product: product),
    );
  }
}
```

### 3.4. ref.invalidate vs ref.refresh — Phân biệt

```dart
// ref.invalidate(provider): 
//   - Đánh dấu provider là "stale" (cũ)
//   - Provider chỉ rebuild khi có widget watch nó
//   - Nếu không có widget nào watch: dispose luôn (nếu AutoDispose)
//   - KHÔNG chờ rebuild xong

// ref.refresh(provider):
//   - Tương đương invalidate + đồng bộ trigger rebuild ngay
//   - Trả về Future<T> — có thể await
//   - Dùng khi cần đảm bảo data mới trước khi tiếp tục

// ref.read(provider.notifier).refresh():
//   - Gọi method refresh() trong notifier — có thể thêm custom logic

// Ví dụ thực tế:
// Pull-to-refresh: cần đợi xong → dùng ref.refresh()
Future<void> onRefresh() async {
  await ref.refresh(productListProvider.future);
}

// Sau khi edit profile: không cần đợi → dùng ref.invalidate()
void onProfileUpdated() {
  ref.invalidate(userProfileProvider);
  // Không await — navigate ngay, data sẽ reload khi cần
}
```

### 3.5. keepAlive — Tránh rebuild tốn kém

```dart
// Provider bình thường (AutoDispose mặc định với @riverpod):
// → Dispose khi không có listener → rebuild từ đầu khi vào lại màn hình

// Giữ alive cho data không thay đổi thường xuyên:
@riverpod
class AppConfig extends _$AppConfig {
  @override
  Future<AppConfig> build() async {
    // keepAlive: không dispose dù không có widget nào watch
    // → Data được cache cho toàn bộ app session
    ref.keepAlive();
    
    return ref.read(configRepositoryProvider).getConfig();
  }
}

// Giữ alive có điều kiện — với timeout
@riverpod
class UserProfile extends _$UserProfile {
  @override
  Future<UserProfile> build() async {
    // Giữ alive 5 phút sau khi không có listener
    final link = ref.keepAlive();
    Timer(const Duration(minutes: 5), link.close);
    
    return ref.read(userRepositoryProvider).getCurrentUser();
  }
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Đo lường: Rebuild count với watch patterns khác nhau

```
Test: ProductListScreen với 20 items
      Trigger: 1 item trong list được update (wishlist toggle)

ref.watch(productListProvider):
  → Rebuild TOÀN BỘ ProductListScreen + tất cả 20 ProductCard
  → Rebuild count: 21 widgets
  → Thời gian build: ~12ms

ref.watch(productListProvider.select((s) => s.products)):
  → Chỉ rebuild khi products list thực sự thay đổi
  → Rebuild count: 21 widgets (vì list được replaced)
  → Thời gian build: ~12ms (không cải thiện vì list thay đổi)

ConsumerWidget per item + productDetailProvider(id):
  → Mỗi item watch provider riêng
  → Chỉ rebuild item có thay đổi: 1 widget
  → Rebuild count: 1 widget
  → Thời gian build: ~0.8ms
  → Cải thiện: 15x ít rebuild hơn

KHUYẾN NGHỊ: Granular providers thay vì 1 provider khổng lồ
```

### AutoDispose vs keepAlive — Memory trade-off

```
Scenario: User navigate Home → Catalog → Product Detail → back → Catalog

AutoDispose (không keepAlive):
  Catalog data bị dispose khi vào Product Detail
  → Khi back về Catalog: fetch lại API (latency 300-500ms)
  → Memory footprint thấp: ~2MB

keepAlive (hoặc cache 5 phút):
  Catalog data giữ trong memory
  → Back về Catalog: render ngay, no loading (0ms)
  → Memory footprint: ~8-15MB tùy data size

NGƯỠNG QUYẾT ĐỊNH:
- Data thay đổi thường xuyên (cart, notifications): AutoDispose
- Data ít thay đổi (catalog, config, user profile): keepAlive + TTL
- Data lớn (image list, full product catalog): AutoDispose với cache layer (Hive/Isar)
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ ref.watch() gọi trong callback/event handler
    ✅ ref.watch() chỉ trong build() hoặc AsyncNotifier.build()
    ✅ ref.read() trong event handlers, button callbacks
    Lý do: ref.watch trong callback gây memory leak

[ ] ❌ StateNotifier với manual AsyncState management
    ✅ AsyncNotifier với AsyncValue.guard() cho async operations
    Lý do: AsyncNotifier có built-in loading/error/data, ít boilerplate

[ ] ❌ Một provider khổng lồ quản lý toàn bộ app state
    ✅ Chia nhỏ: productListProvider, productFilterProvider, productDetailProvider(id)
    Lý do: Granular provider → ít rebuild → hiệu năng tốt hơn

[ ] ❌ Không set keepAlive cho provider cần persist qua navigation
    ✅ Phân tích: data nào cần cache qua navigation → keepAlive với TTL
    Lý do: AutoDispose không có keepAlive → refetch mỗi lần navigate → UX kém

[ ] ❌ Dùng ref.refresh() trong build() method
    ✅ ref.refresh() chỉ trong event handlers, refresh callbacks
    Lý do: Gọi trong build() → infinite rebuild loop

[ ] ❌ throw Exception trong AsyncNotifier.build()
    ✅ Throw exception → Riverpod tự chuyển thành AsyncError → widget .error case xử lý
    ✅ Hoặc return value (AsyncData) và quản lý error state thủ công
    Lý do: Không throw → error bị nuốt, state mãi là AsyncLoading

[ ] ❌ Không có ref.onDispose để cancel HTTP request
    ✅ Luôn tạo CancelToken, cancel trong ref.onDispose
    Lý do: Memory leak khi provider dispose nhưng HTTP call vẫn chạy

[ ] ❌ Quên chạy build_runner sau khi thêm @riverpod annotation
    ✅ Tích hợp watch mode: dart run build_runner watch --delete-conflicting-outputs
    Lý do: Generated file cũ → compile error, runtime crash
```
