# Bài 6.2 — GoRouter & Declarative Routing: Hướng Dẫn Tính Năng & Ứng Dụng Thực Chiến

## Phần 1 — Khái Niệm & Ràng Buộc Kiến Trúc (Architecture & Design Philosophy)

### 1.1 — Sự chuyển dịch từ Imperative sang Declarative Navigation

Trong các bài học trước, chúng ta đã làm quen với **Navigator 1.0**. Đây là mô hình điều hướng **Mệnh lệnh (Imperative Navigation)**: lập trình viên trực tiếp can thiệp và ra lệnh cho framework thay đổi ngăn xếp màn hình:

```dart
// Mệnh lệnh (Imperative - Navigator 1.0): Ra lệnh "Đẩy màn hình này lên đỉnh stack"
Navigator.push(context, MaterialPageRoute(builder: (_) => const ProductDetailScreen()));
Navigator.pop(context);
```

Mặc dù đơn giản và trực quan đối với các ứng dụng di động cơ bản, mô hình Navigator 1.0 bộc lộ những hạn chế kiến trúc nghiêm trọng khi ứng dụng mở rộng quy mô, đặc biệt là khi hỗ trợ đa nền tảng (Web, Desktop, Mobile Deep Linking):
1. **Mất đồng bộ thanh địa chỉ URL (Web Browser):** Trên nền tảng Web, khi người dùng thực hiện `Navigator.push()`, thanh URL của trình duyệt không được cập nhật tương ứng. Người dùng không thể copy đường link để chia sẻ (shareable URL) hoặc bookmark trang.
2. **Nút Back của trình duyệt hoạt động sai lệch:** Trình duyệt dựa vào Browser History, trong khi Flutter quản lý một Stack riêng biệt trong bộ nhớ RAM. Khi người dùng nhấn nút Back trên trình duyệt, hành vi thường bị gián đoạn hoặc thoát ứng dụng bất thường.
3. **Phức tạp hóa Deep Linking:** Khi ứng dụng nhận một liên kết từ bên ngoài (ví dụ: `https://myapp.com/products/42`), hệ thống phải viết mã phân tích cú pháp (parsing) thủ công rất phức tạp để tuần tự push các màn hình nền trước khi hiển thị màn hình đích.
4. **Khó kiểm thử và bảo vệ luồng (Auth Guard):** Việc kiểm tra quyền truy cập (ví dụ: người dùng chưa đăng nhập thì không được vào `/checkout`) buộc phải phân tán rải rác bên trong logic `build()` hoặc sự kiện `onTap` của từng nút bấm.

Để giải quyết triệt để các vấn đề trên, Flutter giới thiệu mô hình **Điều hướng Khai báo (Declarative Navigation - Navigator 2.0)**:
- Giao diện và các màn hình hiển thị là một hàm số của trạng thái URL hiện tại:
  $$\text{Navigation Stack} = f(\text{App State, Current URL})$$
- Thay vì ra lệnh push/pop từng màn hình, lập trình viên chỉ cần **thay đổi trạng thái URL** (`/products/42`). Router Engine sẽ tự động tính toán và cấu trúc lại toàn bộ ngăn xếp màn hình cho phù hợp với trạng thái đó.

---

### 1.2 — GoRouter: Công cụ điều hướng khai báo tiêu chuẩn

Mặc dù Flutter cung cấp sẵn API Navigator 2.0 thuần (`RouterDelegate`, `RouteInformationParser`), việc tự viết và duy trì các lớp này đòi hỏi hàng trăm dòng mã phức tạp (boilerplate).

**`GoRouter`** là package định tuyến chính thức do Google Flutter Team phát triển và duy trì, đóng vai trò là một lớp bao bọc (wrapper) hoàn hảo trên nền Navigator 2.0:
- **Cung cấp API khai báo URL-first trực quan, gọn gàng.**
- **Tự động đồng bộ hai chiều giữa thanh URL trình duyệt và ứng dụng.**
- **Hỗ trợ Deep Linking out-of-the-box cho cả Android, iOS và Web.**
- **Tích hợp sẵn Nested Navigation, ShellRoute, StatefulShellRoute cho Bottom Navigation Bar.**
- **Hỗ trợ cơ chế bảo vệ tuyến đường (Redirects / Auth Guards) tự động phản ứng theo State.**

```
┌────────────────────────────────────────────────────────────────────────┐
│ GOROUTER ARCHITECTURE OVERVIEW                                         │
│                                                                        │
│  Browser URL / Deep Link         User Action (context.go('/cart'))     │
│             │                                    │                     │
│             ▼                                    ▼                     │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ GOROUTER ENGINE (Declarative URL-Driven Matching)                  │ │
│ │  • So khớp Route pattern (/products/:id, /cart, /login)            │ │
│ │  • Trích xuất pathParameters & queryParameters                     │ │
│ │  • Chạy chuỗi kiểm tra chuyển hướng (Global & Route Redirects)     │ │
│ └──────────────────────────────────┬─────────────────────────────────┘ │
│                                    │                                   │
│                                    ▼                                   │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ NAVIGATOR STACK (Được cấu hình tự động)                            │ │
│ │  • Tự động build Page List tương ứng với URL                       │ │
│ │  • Tự động đồng bộ nút Back phần cứng & Browser History            │ │
│ └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Phần 2 — Cơ Chế Hoạt Động & Cấu Trúc Lõi (Under the Hood / Deep-Dive)

### 2.1 — Cấu trúc định tuyến phân cấp (Routing Hierarchy)

Trong GoRouter, hệ thống màn hình được định nghĩa tập trung dưới dạng một cây phân cấp các đối tượng `RouteBase` (chủ yếu là `GoRoute` và `ShellRoute`):

```dart
final GoRouter appRouter = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path: '/',
      name: 'home',
      builder: (context, state) => const HomeScreen(),
      routes: [
        // Sub-route: Đường dẫn con kế thừa từ cha
        GoRoute(
          path: 'products', // Kết quả URL: /products
          name: 'products',
          builder: (context, state) => const ProductsScreen(),
          routes: [
            // Nested param: /products/:id
            GoRoute(
              path: ':id', // Đường dẫn tương đối: /products/:id
              name: 'product_detail',
              builder: (context, state) {
                final id = state.pathParameters['id']!;
                return ProductDetailScreen(id: id);
              },
            ),
          ],
        ),
      ],
    ),
    GoRoute(
      path: '/login',
      name: 'login',
      builder: (context, state) => const LoginScreen(),
    ),
  ],
);
```

#### Quy tắc đường dẫn Tuyệt đối vs Tương đối:
- **Đường dẫn bắt đầu bằng dấu gạch chéo `/` (Absolute Path):** Là đường dẫn tuyệt đối tính từ gốc (Root). Chỉ được khai báo ở tầng ngoài cùng của mảng `routes`.
- **Đường dẫn không có dấu `/` ở đầu (Relative Path):** Được định nghĩa bên trong mảng `routes` của một route cha. Framework sẽ tự động ghép nối đường dẫn: `products` nằm trong `/` $\to$ `/products`; `:id` nằm trong `products` $\to$ `/products/:id`.

---

### 2.2 — Bộ tứ điều hướng cốt lõi: `go()` vs `push()` vs `replace()` vs `pop()`

Hiểu rõ sự khác biệt giữa các phương thức điều hướng là yếu tố quan trọng nhất để tránh các lỗi logic và xung đột ngăn xếp navigation:

```
context.go('/product/123')             context.push('/product/123')
┌─────────────────────────┐            ┌─────────────────────────┐
│ [ProductDetail(123)]    │            │ [ProductDetail(123)]    │
│ (Cấu trúc lại toàn bộ   │            │ [HomeScreen]            │
│  stack theo URL mới)    │            │ (Đẩy đè lên stack cũ,   │
│                         │            │  bảo toàn lịch sử)      │
└─────────────────────────┘            └─────────────────────────┘
```

#### Bảng so sánh toàn diện các phương thức điều hướng:

| Phương Thức | Cơ Chế Ngăn Xếp (Stack) | Đồng Bộ URL | Trình Duyệt Web | Ngữ Cảnh Sử Dụng Phù Hợp |
| :--- | :--- | :--- | :--- | :--- |
| **`context.go(path)`** | **Tái cấu trúc Stack:** Xóa stack cũ, dựng stack mới chuẩn xác theo cấu trúc URL | Cập nhật URL hiện tại | Thay thế hoặc cấu trúc lại lịch sử | Chuyển Tab chính, đăng nhập thành công về Home, đổi phân hệ |
| **`context.push(path)`** | **Đẩy thêm trang:** Thêm một Page mới lên đỉnh stack hiện hành bất kể vị trí URL | Cập nhật URL hiện tại | Thêm một mục mới vào Browser History | Mở trang chi tiết, form nhập liệu tạm thời muốn Back lại |
| **`context.replace(path)`**| **Thay thế đỉnh stack:** Đổi trang trên đỉnh hiện tại bằng trang mới | Cập nhật URL hiện tại | Thay thế entry trên cùng của Browser History | Bước tiếp theo trong Wizard/Onboarding (không cho back lại bước trước) |
| **`context.pop([result])`**| **Rút đỉnh stack:** Hủy bỏ trang trên đỉnh, quay về màn hình bên dưới | Cập nhật lùi về URL trước| Quay lại trang trước (như nút Back) | Đóng modal, đóng trang chi tiết, trả kết quả về caller |

---

### 2.3 — Cơ chế phân giải dữ liệu (Parameters Resolution)

GoRouter hỗ trợ 4 phương thức truyền nhận dữ liệu giữa các màn hình:

```
┌────────────────────────────────────────────────────────────────────────┐
│ 4 CƠ CHẾ TRUYỀN NHẬN DỮ LIỆU TRONG GOROUTER                            │
├───────────────────┬───────────────────────────────┬────────────────────┤
│ 1. Path Param     │ /products/:id                 │ Xác định danh tính │
├───────────────────┼───────────────────────────────┼────────────────────┤
│ 2. Query Param    │ /search?category=shoes&page=2 │ Lọc, tìm kiếm, tab │
├───────────────────┼───────────────────────────────┼────────────────────┤
│ 3. Extra Object   │ extra: ProductModel(...)      │ Đối tượng phức tạp │
├───────────────────┼───────────────────────────────┼────────────────────┤
│ 4. Pop with Result│ context.pop(selectedItem)     │ Trả kết quả ngược  │
└───────────────────┴───────────────────────────────┴────────────────────┘
```

1. **Path Parameters (`state.pathParameters`):**
   - Được định nghĩa bằng dấu hai chấm `:name` trong path.
   - Thường dùng để chỉ định **định danh (ID / Slug)** của tài nguyên.
   - Luôn tồn tại trên URL, thân thiện với SEO và Deep Link.
2. **Query Parameters (`state.uri.queryParameters`):**
   - Nằm sau dấu `?` trên URL (ví dụ: `?filter=active&sort=asc`).
   - Thường dùng cho các trạng thái không bắt buộc như bộ lọc, từ khóa tìm kiếm, phân trang.
3. **Extra Parameter (`state.extra`):**
   - Cho phép truyền trực tiếp một object Dart phức tạp (ví dụ cả một entity `User` hoặc `ProductModel`) mà không cần chuyển thành String.
   - *Lưu ý quan trọng:* Đối tượng `extra` chỉ tồn tại trong bộ nhớ RAM của phiên chạy hiện tại. Trên nền tảng Web, nếu người dùng nhấn **F5 (Reload trang)** hoặc mở link từ tab mới, `state.extra` sẽ có giá trị `null`!
4. **Pop with Result (`context.pop<T>(result)`):**
   - Tương đương `Navigator.pop(context, result)`. Màn hình trước đón nhận kết quả qua `await context.push<T>(...)`.

---

## Phần 3 — Hướng Dẫn Thực Hành & Code Mẫu Trọng Tâm (Production-Ready Implementations)

### 3.1 — Cài đặt và Khởi tạo với `MaterialApp.router`

Khai báo phụ thuộc trong `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  go_router: ^14.2.0 # Sử dụng phiên bản ổn định mới nhất
```

Thiết lập router tập trung và gắn vào ứng dụng:

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'GoRouter Production Demo',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
      ),
      // Gắn cấu hình GoRouter vào MaterialApp
      routerConfig: appRouter,
    );
  }
}
```

---

### 3.2 — Kỹ thuật Điều hướng & Truyền nhận dữ liệu thực tế

Dưới đây là kịch bản hoàn chỉnh: Danh sách sản phẩm $\to$ Chi tiết sản phẩm (với Path Param & Extra) $\to$ Tìm kiếm (với Query Param) $\to$ Chọn bộ lọc (Trả kết quả về):

```dart
// Định nghĩa router với các kiểu tham số
final GoRouter appRouter = GoRouter(
  initialLocation: '/products',
  debugLogDiagnostics: true, // In log điều hướng chi tiết trong console debug
  routes: [
    GoRoute(
      path: '/products',
      name: 'products',
      builder: (context, state) {
        // Đọc Query Parameters: /products?category=electronics
        final category = state.uri.queryParameters['category'] ?? 'all';
        return ProductListScreen(category: category);
      },
      routes: [
        GoRoute(
          path: ':id', // Đường dẫn tương đối: /products/:id
          name: 'product_detail',
          builder: (context, state) {
            // 1. Đọc Path Parameter
            final productId = state.pathParameters['id']!;
            
            // 2. Đọc Extra Object (nếu có)
            final productExtra = state.extra as ProductItem?;

            return ProductDetailScreen(
              productId: productId,
              fallbackProduct: productExtra,
            );
          },
        ),
      ],
    ),
    GoRoute(
      path: '/filter-selector',
      name: 'filter_selector',
      builder: (context, state) => const FilterSelectorScreen(),
    ),
  ],
);

// Model dữ liệu mẫu
class ProductItem {
  final String id;
  final String name;
  final double price;

  const ProductItem({required this.id, required this.name, required this.price});
}

// 1. Màn hình danh sách sản phẩm
class ProductListScreen extends StatelessWidget {
  final String category;

  const ProductListScreen({super.key, required this.category});

  @override
  Widget build(BuildContext context) {
    final sampleProduct = const ProductItem(id: 'macbook-m3', name: 'MacBook Pro M3', price: 1999.0);

    return Scaffold(
      appBar: AppBar(
        title: Text('Sản Phẩm ($category)'),
        actions: [
          IconButton(
            icon: const Icon(Icons.filter_list),
            onPressed: () async {
              // Mở màn hình chọn bộ lọc và chờ kết quả trả về
              final selectedCategory = await context.push<String>('/filter-selector');
              if (selectedCategory != null && context.mounted) {
                // Cập nhật Query Param: /products?category=...
                context.go('/products?category=$selectedCategory');
              }
            },
          ),
        ],
      ),
      body: Center(
        child: Card(
          margin: const EdgeInsets.all(16),
          child: ListTile(
            title: Text(sampleProduct.name),
            subtitle: Text('\$${sampleProduct.price}'),
            trailing: const Icon(Icons.arrow_forward_ios),
            onTap: () {
              // Cách 1: Điều hướng bằng path URL kèm extra
              context.go(
                '/products/${sampleProduct.id}',
                extra: sampleProduct,
              );

              // Hoặc Cách 2: Điều hướng bằng Route Name (Type-safe)
              // context.goNamed(
              //   'product_detail',
              //   pathParameters: {'id': sampleProduct.id},
              //   extra: sampleProduct,
              // );
            },
          ),
        ),
      ),
    );
  }
}

// 2. Màn hình chi tiết sản phẩm
class ProductDetailScreen extends StatelessWidget {
  final String productId;
  final ProductItem? fallbackProduct;

  const ProductDetailScreen({super.key, required this.productId, this.fallbackProduct});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(fallbackProduct?.name ?? 'Sản Phẩm $productId')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('ID Sản Phẩm: $productId', style: Theme.of(context).textTheme.titleMedium),
            if (fallbackProduct != null) ...[
              const SizedBox(height: 8),
              Text('Giá bán: \$${fallbackProduct!.price}'),
            ],
            const Spacer(),
            SizedBox(
              width: double.infinity,
              child: FilledButton.icon(
                icon: const Icon(Icons.arrow_back),
                label: const Text('Quay Lại Danh Sách'),
                onPressed: () => context.pop(),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// 3. Màn hình chọn bộ lọc (Trả kết quả với pop)
class FilterSelectorScreen extends StatelessWidget {
  const FilterSelectorScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final categories = ['electronics', 'clothing', 'books', 'home'];

    return Scaffold(
      appBar: AppBar(title: const Text('Chọn Danh Mục')),
      body: ListView.builder(
        itemCount: categories.length,
        itemBuilder: (context, index) {
          final cat = categories[index];
          return ListTile(
            title: Text(cat.toUpperCase()),
            onTap: () {
              // Trả kết quả về cho trang trước
              context.pop(cat);
            },
          );
        },
      ),
    );
  }
}
```

---

### 3.3 — Nested Navigation & Bottom Bar: `StatefulShellRoute.indexedStack`

Một trong những tính năng mạnh mẽ nhất của GoRouter là **`StatefulShellRoute.indexedStack`**. 

Khác với `ShellRoute` thông thường (vốn hủy và nạp lại toàn bộ widget mỗi khi chuyển tab, làm mất vị trí cuộn và dữ liệu đang nhập), `StatefulShellRoute.indexedStack` duy trì một **Navigator Stack độc lập cho từng nhánh (Branch)** và ẩn/hiện chúng qua `IndexedStack`:

```
┌────────────────────────────────────────────────────────────────────────┐
│ STATEFUL SHELL ROUTE (Branch Isolation)                                │
│                                                                        │
│ ┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────┐ │
│ │ Branch 0: Trang Chủ  │ │ Branch 1: Khám Phá   │ │ Branch 2: Hồ Sơ  │ │
│ │  • Navigator Stack 0 │ │  • Navigator Stack 1 │ │ • Nav Stack 2   │ │
│ │  • Giữ vị trí cuộn   │ │  • Giữ bộ lọc search │ │ • Giữ form input│ │
│ └──────────────────────┘ └──────────────────────┘ └──────────────────┘ │
│                            ▲ Active Branch                             │
│ ┌──────────────────────────┴─────────────────────────────────────────┐ │
│ │ MATERIAL 3 NAVIGATION BAR (Thanh điều hướng chung cố định bên dưới)│ │
│ └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

#### Mã nguồn hoàn chỉnh triển khai `StatefulShellRoute.indexedStack`:

```dart
final GoRouter navigationBarRouter = GoRouter(
  initialLocation: '/home',
  routes: [
    // Khởi tạo StatefulShellRoute với IndexedStack
    StatefulShellRoute.indexedStack(
      builder: (context, state, navigationShell) {
        // navigationShell chứa index của tab đang hoạt động và phương thức chuyển tab
        return ScaffoldWithNavBar(navigationShell: navigationShell);
      },
      branches: [
        // NHÁNH 1: Trang Chủ (/home)
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/home',
              builder: (context, state) => const HomeTabScreen(),
              routes: [
                GoRoute(
                  path: 'details',
                  builder: (context, state) => const HomeDetailScreen(),
                ),
              ],
            ),
          ],
        ),

        // NHÁNH 2: Khám Phá (/explore)
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/explore',
              builder: (context, state) => const ExploreTabScreen(),
            ),
          ],
        ),

        // NHÁNH 3: Hồ Sơ Cá Nhân (/profile)
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/profile',
              builder: (context, state) => const ProfileTabScreen(),
            ),
          ],
        ),
      ],
    ),
  ],
);

// Scaffold bao bọc chứa Bottom Navigation Bar
class ScaffoldWithNavBar extends StatelessWidget {
  final StatefulNavigationShell navigationShell;

  const ScaffoldWithNavBar({super.key, required this.navigationShell});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: navigationShell, // Render nội dung của branch hiện tại
      bottomNavigationBar: NavigationBar(
        selectedIndex: navigationShell.currentIndex,
        onDestinationSelected: (int index) {
          // Chuyển nhánh mượt mà. Nếu tap lại vào tab đang mở, tự động scroll về đỉnh
          navigationShell.goBranch(
            index,
            initialLocation: index == navigationShell.currentIndex,
          );
        },
        destinations: const [
          NavigationDestination(
            icon: Icon(Icons.home_outlined),
            selectedIcon: Icon(Icons.home),
            label: 'Trang chủ',
          ),
          NavigationDestination(
            icon: Icon(Icons.explore_outlined),
            selectedIcon: Icon(Icons.explore),
            label: 'Khám phá',
          ),
          NavigationDestination(
            icon: Icon(Icons.person_outline),
            selectedIcon: Icon(Icons.person),
            label: 'Hồ sơ',
          ),
        ],
      ),
    );
  }
}

class HomeTabScreen extends StatelessWidget {
  const HomeTabScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Trang Chủ')),
      body: Center(
        child: ElevatedButton(
          onPressed: () => context.go('/home/details'),
          child: const Text('Vào Trang Chi Tiết (Vẫn giữ Bottom Bar)'),
        ),
      ),
    );
  }
}

class HomeDetailScreen extends StatelessWidget {
  const HomeDetailScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Chi Tiết Trang Chủ')),
      body: const Center(child: Text('Nội dung chi tiết sâu bên trong Tab 1')),
    );
  }
}

class ExploreTabScreen extends StatelessWidget {
  const ExploreTabScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Khám Phá')),
      body: ListView.builder(
        itemCount: 100,
        itemBuilder: (context, index) => ListTile(title: Text('Mục khám phá số #$index')),
      ),
    );
  }
}

class ProfileTabScreen extends StatelessWidget {
  const ProfileTabScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hồ Sơ')),
      body: const Center(child: Text('Thông tin cá nhân')),
    );
  }
}
```

---

### 3.4 — Bảo vệ Tuyến Đường (Auth Guards) kết hợp `refreshListenable`

Trong ứng dụng thực tế, bài toán phổ biến nhất là: **Nếu chưa đăng nhập, người dùng truy cập trang nhạy cảm (như `/checkout`, `/settings`) phải bị chuyển hướng về `/login`. Khi đăng nhập thành công, tự động chuyển về trang họ định truy cập.**

GoRouter cung cấp thuộc tính `redirect` kết hợp hoàn hảo với **`refreshListenable`**:

```dart
import 'package:flutter/foundation.dart';

// Dịch vụ xác thực phát tín hiệu thông báo
class AuthService extends ChangeNotifier {
  static final AuthService instance = AuthService._();
  AuthService._();

  bool _isLoggedIn = false;
  bool get isLoggedIn => _isLoggedIn;

  void login() {
    _isLoggedIn = true;
    notifyListeners(); // Kích hoạt GoRouter chạy lại redirect tự động!
  }

  void logout() {
    _isLoggedIn = false;
    notifyListeners(); // Tự động đẩy người dùng về màn hình đăng nhập!
  }
}

// Khởi tạo router với Auth Guard
final GoRouter securedRouter = GoRouter(
  initialLocation: '/dashboard',
  // 1. Đăng ký lắng nghe thay đổi trạng thái từ AuthService
  refreshListenable: AuthService.instance,

  // 2. Logic kiểm tra và chuyển hướng toàn cục
  redirect: (BuildContext context, GoRouterState state) {
    final bool loggedIn = AuthService.instance.isLoggedIn;
    final bool isLoggingIn = state.matchedLocation == '/login';

    // Trường hợp 1: Chưa đăng nhập và đang cố truy cập trang khác ngoài /login
    if (!loggedIn && !isLoggingIn) {
      // Lưu lại URL người dùng muốn tới vào query param để redirect ngược lại sau khi login
      final target = state.matchedLocation;
      return '/login?from=$target';
    }

    // Trường hợp 2: Đã đăng nhập nhưng vẫn đứng ở màn hình /login
    if (loggedIn && isLoggingIn) {
      // Nếu có query param 'from', ưu tiên trở lại trang đó, ngược lại về /dashboard
      final from = state.uri.queryParameters['from'];
      return from ?? '/dashboard';
    }

    // Trường hợp hợp lệ: Không cần chuyển hướng
    return null;
  },

  routes: [
    GoRoute(
      path: '/login',
      builder: (context, state) => const LoginScreen(),
    ),
    GoRoute(
      path: '/dashboard',
      builder: (context, state) => const DashboardScreen(),
    ),
  ],
);

class LoginScreen extends StatelessWidget {
  const LoginScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Đăng Nhập')),
      body: Center(
        child: FilledButton(
          onPressed: () {
            // Chỉ cần gọi login() trong AuthService, GoRouter tự động redirect nhờ refreshListenable!
            AuthService.instance.login();
          },
          child: const Text('Xác Thực Đăng Nhập'),
        ),
      ),
    );
  }
}

class DashboardScreen extends StatelessWidget {
  const DashboardScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Bảng Điều Khiển'),
        actions: [
          IconButton(
            icon: const Icon(Icons.logout),
            onPressed: () => AuthService.instance.logout(),
          ),
        ],
      ),
      body: const Center(child: Text('Nội dung bảo mật dành riêng cho thành viên')),
    );
  }
}
```

---

### 3.5 — Tùy biến Hiệu ứng Chuyển cảnh với `CustomTransitionPage`

Để tạo hiệu ứng mượt mà (Fade, Slide từ phải sang trái hoặc từ dưới lên), thay vì dùng thuộc tính `builder`, chúng ta sử dụng **`pageBuilder`** kết hợp với `CustomTransitionPage`:

```dart
GoRoute(
  path: '/settings',
  pageBuilder: (context, state) {
    return CustomTransitionPage(
      key: state.pageKey, // Bắt buộc để Flutter phân biệt trang trên Tree
      child: const SettingsScreen(),
      transitionsBuilder: (context, animation, secondaryAnimation, child) {
        // Hiệu ứng trượt từ dưới lên (Slide Up Transition)
        const begin = Offset(0.0, 1.0);
        const end = Offset.zero;
        const curve = Curves.easeOutCubic;

        final tween = Tween(begin: begin, end: end).chain(CurveTween(curve: curve));

        return SlideTransition(
          position: animation.drive(tween),
          child: FadeTransition(
            opacity: animation,
            child: child,
          ),
        );
      },
      transitionDuration: const Duration(milliseconds: 350),
    );
  },
)
```

---

### 3.6 — Xử lý Trang lỗi 404 & Cấu hình Web URL Strategy

#### 1. Tùy biến trang báo lỗi (Error Screen):
Khi người dùng nhập sai URL hoặc Deep Link không tồn tại:

```dart
final GoRouter router = GoRouter(
  routes: [...],
  errorBuilder: (context, state) => Scaffold(
    appBar: AppBar(title: const Text('Không Tìm Thấy Trang')),
    body: Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          const Icon(Icons.error_outline, size: 64, color: Colors.redAccent),
          const SizedBox(height: 16),
          Text('Lỗi 404: Tuyến đường không hợp lệ (${state.error?.message})'),
          const SizedBox(height: 16),
          ElevatedButton(
            onPressed: () => context.go('/'),
            child: const Text('Về Trang Chủ'),
          ),
        ],
      ),
    ),
  ),
);
```

#### 2. Cấu hình Web URL Strategy (Bỏ dấu `#` trên thanh trình duyệt):
Mặc định trên Flutter Web, URL sẽ có dạng `http://localhost:8080/#/products`. Để chuyển sang dạng chuẩn `http://localhost:8080/products`:

```dart
import 'package:flutter_web_plugins/url_strategy.dart';

void main() {
  // Loại bỏ ký tự '#' trên URL của Web
  usePathUrlStrategy();
  runApp(const MyApp());
}
```

---

## Phần 4 — Cạm Bẫy Thường Gặp & Giải Pháp Khắc Phục (Anti-Patterns & Pitfalls)

### 4.1 — Trộn lẫn Navigator 1.0 (`Navigator.push`) với `GoRouter`

#### Mô tả lỗi:
Ứng dụng đang dùng GoRouter nhưng ở một màn hình sâu bên trong, lập trình viên vẫn dùng `Navigator.push(context, MaterialPageRoute(...))`.

#### Hậu quả kỹ thuật:
- `Navigator.push()` đẩy một Route lên stack nhưng **hoàn toàn nằm ngoài tầm kiểm soát của GoRouter**.
- Thanh URL trên trình duyệt không được cập nhật.
- Khi người dùng nhấn nút Back vật lý hoặc reload trang, trạng thái navigation bị lệch pha hoàn toàn, gây lỗi màn hình trắng hoặc crash.

#### Giải pháp:
Luôn sử dụng đồng nhất các API mở rộng của GoRouter trên `BuildContext`:
```dart
// ❌ SAI LẦM
Navigator.push(context, MaterialPageRoute(builder: (_) => const DetailScreen()));

// ✅ ĐÚNG
context.push('/detail');
// Hoặc
context.go('/detail');
```

---

### 4.2 — Lạm dụng `extra` trên nền tảng Web gây Crash khi Refresh trang

#### Mô tả lỗi:
Truyền toàn bộ model dữ liệu qua `extra` và ép kiểu non-null `!` trong màn hình con:

```dart
// Màn hình con
final product = state.extra! as ProductItem; // CRASH KHI F5 TRÊN WEB!
```

#### Nguyên nhân kỹ thuật:
Trên Mobile, app duy trì instance trong suốt phiên chạy. Nhưng trên Web, khi người dùng nhấn **F5 (Refresh)**, trình duyệt sẽ gửi lại yêu cầu khởi động ứng dụng từ đầu dựa trên URL. Đối tượng `extra` trong RAM trước đó đã biến mất và trở thành `null`.

#### Giải pháp:
1. Luôn truyền định danh chính (ID) qua **Path Parameter** (`/products/:id`).
2. Sử dụng `extra` như một tầng cache tạm thời để hiển thị tức thì. Nếu `state.extra == null`, hãy dùng `id` từ `state.pathParameters` để gọi API / Database tải lại dữ liệu:

```dart
final id = state.pathParameters['id']!;
final product = state.extra as ProductItem? ?? await fetchProductById(id);
```

---

### 4.3 — Nhầm lẫn giữa Đường dẫn Tuyệt đối và Tương đối trong Sub-routes

#### Mô tả lỗi:
Khai báo đường dẫn con bắt đầu bằng dấu gạch chéo `/`:

```dart
// ❌ SAI: Gây exception 'A sub-route cannot start with a slash'
GoRoute(
  path: '/products',
  routes: [
    GoRoute(
      path: '/details', // LỖI CRASH LÚC KHỞI CHẠY!
      builder: (context, state) => const DetailScreen(),
    ),
  ],
)

// ✅ ĐÚNG: Đường dẫn con phải là tương đối (không có / ở đầu)
GoRoute(
  path: '/products',
  routes: [
    GoRoute(
      path: 'details', // Sẽ được ghép thành: /products/details
      builder: (context, state) => const DetailScreen(),
    ),
  ],
)
```

---

## Phần 5 — Khảo Sát Kỹ Thuật Chuyên Sâu & Bài Tập Phân Tích Mã Nguồn (Technical Assessment & Code Tracing)

### 5.1 — Bộ câu hỏi khảo sát kỹ thuật chuyên sâu (Deep-Dive Q&A)

#### Câu 1: Phân biệt bản chất hành vi giữa `context.go()` và `context.push()` trên cây Navigation Stack và browser history?
*Phân tích bản chất:*
- **`context.go('/target')`**: Là cơ chế điều hướng thuần khai báo (**Declarative Navigation**). GoRouter phân tích chuỗi URL đích, tìm kiếm đường đi từ gốc của cây định tuyến và **tái cấu trúc lại toàn bộ danh sách `pages`** của Navigator. Nếu route đích không phải là sub-route của trang hiện tại, toàn bộ các màn hình trung gian trước đó sẽ bị dọn dẹp khỏi stack. Trên Web, nó cập nhật URL và tái cơ cấu lại history.
- **`context.push('/target')`**: Là cơ chế điều hướng mệnh lệnh (**Imperative Push**). Framework giữ nguyên toàn bộ các trang đang có trên stack hiện hành và đẩy thêm một trang mới đè lên đỉnh. Ngay cả khi trên Web URL thay đổi, khi bấm nút Back, màn hình ngay phía trước chắc chắn sẽ được phục hồi.
- *Quy tắc áp dụng:* Dùng `go()` cho các luồng chuyển đổi trạng thái lớn (Home, Profile tabs, Login redirect). Dùng `push()` cho các màn hình chi tiết, flow popup hoặc wizard mà người dùng cần back lại từng bước.

---

#### Câu 2: Tại sao `StatefulShellRoute.indexedStack` vượt trội hơn `ShellRoute` thông thường trong ứng dụng có Bottom Navigation Bar?
*Phân tích bản chất:*
- **Với `ShellRoute` thông thường:** Khi người dùng chuyển từ Tab A sang Tab B rồi quay lại Tab A, toàn bộ cây con của Tab A bị tháo dỡ (unmounted) và khởi tạo lại từ đầu. Kết quả là: vị trí cuộn danh sách (scroll offset) bị nhảy về 0, nội dung người dùng đang nhập dở trên Form bị xóa sạch.
- **Với `StatefulShellRoute.indexedStack`:** Mỗi nhánh (branch) được gắn với một `GlobalKey<NavigatorState>` riêng biệt và được quản lý bên dưới một `IndexedStack`. Khi đổi tab, cây widget của tab cũ chỉ bị ẩn đi (tắt paint/hit-test) chứ không bị unmount. Nhờ đó, toàn bộ **Element Tree, State và RenderObject của từng tab được bảo tồn nguyên vẹn** trong bộ nhớ, mang lại trải nghiệm mượt mà tức thì khi chuyển tab.

---

#### Câu 3: Cạm bẫy của việc dùng `extra` để truyền object trên nền tảng Web là gì, và giải pháp kiến trúc thay thế chuẩn mực?
*Phân tích bản chất:*
- `state.extra` truyền tham chiếu đối tượng trực tiếp trong bộ nhớ Dart Heap. Trên ứng dụng Web, URL là nguồn chân lý duy nhất (Single Source of Truth). Khi người dùng chia sẻ link cho bạn bè, hoặc chính người dùng nhấn nút F5 Reload trang, trình duyệt khởi động lại ứng dụng chỉ với chuỗi URL trên thanh địa chỉ. Mọi biến trong Heap trước đó bị reset hoàn toàn, khiến `state.extra` trả về `null`. Nếu mã nguồn ép kiểu ép buộc (`state.extra as MyData`), ứng dụng sẽ crash màn hình đỏ ngay lập tức.
- *Giải pháp kiến trúc:* Thiết kế các route quan trọng luôn định danh qua **Path Parameters** (ví dụ `/user/:userId`). Sử dụng `extra` như một tầng tối ưu hóa hiệu năng (nếu có sẵn thì render ngay không cần loading). Nếu `extra == null`, kích hoạt fallback gọi Repository/API bằng `userId` để nạp dữ liệu.

---

#### Câu 4: Cơ chế `refreshListenable` trong `GoRouter` hoạt động như thế nào để xử lý Auth State Changes tự động?
*Phân tích bản chất:*
- `refreshListenable` nhận một đối tượng kế thừa từ `Listenable` (như `ChangeNotifier` hoặc `ValueNotifier`).
- GoRouter đăng ký một listener nội bộ vào `Listenable` này. Mỗi khi phương thức `notifyListeners()` được gọi (ví dụ: khi người dùng vừa đăng nhập hoặc token hết hạn), GoRouter lập tức kích hoạt lại toàn bộ chuỗi phương thức **`redirect(context, state)`**.
- Tại đây, hệ thống so khớp lại trạng thái xác thực mới với URL hiện hành. Nếu phát hiện vi phạm quyền truy cập, Router tự động điều hướng sang trang tương ứng mà lập trình viên không cần phải viết các hàm `context.go('/login')` thủ công rải rác ở khắp các tầng bloc hay controller.

---

#### Câu 5: Làm thế nào để cấu hình Custom Transition Page khác nhau cho từng Route cụ thể trong GoRouter?
*Phân tích bản chất:*
- Mặc định, `builder` của `GoRoute` sẽ sử dụng chuyển cảnh mặc định của hệ điều hành (`MaterialPageRoute` trên Android, `CupertinoPageRoute` trên iOS).
- Để tùy biến riêng cho từng màn hình, sử dụng thuộc tính **`pageBuilder`** thay cho `builder`, và trả về một instance của **`CustomTransitionPage`**.
- Trong `CustomTransitionPage`, cung cấp `state.pageKey` cho thuộc tính `key` và định nghĩa hàm `transitionsBuilder(context, animation, secondaryAnimation, child)` với các hiệu ứng như `FadeTransition`, `SlideTransition`, `ScaleTransition` hoặc kết hợp nhiều Tween lại với nhau.

---

### 5.2 — Bài tập phân tích luồng thực thi (Code Tracing)

#### Đề bài:
Cho cấu hình GoRouter sau:

```dart
final router = GoRouter(
  initialLocation: '/home',
  routes: [
    GoRoute(
      path: '/home',
      builder: (_, __) => const HomeScreen(),
      routes: [
        GoRoute(
          path: 'feed',
          builder: (_, __) => const FeedScreen(),
        ),
      ],
    ),
    GoRoute(
      path: '/profile',
      builder: (_, __) => const ProfileScreen(),
      routes: [
        GoRoute(
          path: 'settings',
          builder: (_, __) => const SettingsScreen(),
        ),
      ],
    ),
  ],
);
```

Giả sử người dùng thực hiện chuỗi thao tác sau:
1. Ứng dụng khởi động tại `/home`.
2. Người dùng nhấn nút kích hoạt lệnh: `context.go('/home/feed')`.
3. Người dùng tiếp tục nhấn nút kích hoạt lệnh: `context.push('/profile/settings')`.
4. Người dùng bấm nút Back vật lý trên điện thoại (hoặc gọi `context.pop()`).

**Câu hỏi:**
1. Sau bước 3, ngăn xếp màn hình (Navigator Stack) bao gồm những màn hình nào? URL hiện tại là gì?
2. Sau bước 4, màn hình nào sẽ hiển thị? URL hiện tại sẽ trở thành gì? Có thể bấm Back tiếp được không và sẽ về đâu?

---

#### Đáp án phân tích:

**1. Sau bước 3:**
- Bước 1: Ứng dụng ở `/home`. Stack: `[HomeScreen]`.
- Bước 2: `context.go('/home/feed')` cấu trúc lại stack theo cây phân cấp route cha-con. Do `feed` là con của `/home`, Stack lúc này là: `[HomeScreen, FeedScreen]`. URL là `/home/feed`.
- Bước 3: `context.push('/profile/settings')` thực hiện lệnh **Push đè** lên stack hiện tại mà không làm mất lịch sử cũ.
  - Stack hiện tại: `[HomeScreen, FeedScreen, SettingsScreen]`.
  - URL hiện tại được cập nhật thành: **`/profile/settings`**.

**2. Sau bước 4 (Bấm Back):**
- Lệnh `pop()` rút trang trên cùng (`SettingsScreen`) ra khỏi stack.
- Màn hình hiển thị tiếp theo là: **`FeedScreen`**.
- URL tự động đồng bộ lùi lại thành: **`/home/feed`**.
- Người dùng **hoàn toàn có thể bấm Back tiếp một lần nữa**. Khi đó, `FeedScreen` bị pop và ứng dụng sẽ quay trở về **`HomeScreen`** với URL là `/home`.
