# Bài 6.1 — Navigator 1.0 Toàn Diện: Cơ Chế Ngăn Xếp, Truyền Nhận Dữ Liệu & Named Routes

## Phần 1 — Khái Niệm & Ràng Buộc Kiến Trúc (Architecture & Design Philosophy)

### 1.1 — Mô hình Ngăn xếp Mệnh lệnh (Imperative Navigation Stack)

Trong kiến trúc giao diện của Flutter, việc quản lý luồng hiển thị giữa các màn hình ban đầu được hiện thực hóa thông qua **Navigator 1.0**. Đây là một mô hình điều hướng **Mệnh lệnh (Imperative Navigation)** dựa trên cấu trúc dữ liệu kinh điển: **Ngăn xếp LIFO (Last-In, First-Out)**.

```
┌────────────────────────────────────────────────────────────────────────┐
│ NAVIGATOR 1.0 STACK LIFECYCLE (LIFO)                                   │
│                                                                        │
│   Push Screen C          Pop Screen C            PushReplacement D     │
│   ┌──────────────┐       ┌──────────────┐        ┌──────────────┐      │
│   │ [Screen C]   │ ──►   │              │        │ [Screen D]   │      │
│   ├──────────────┤       ├──────────────┤        ├──────────────┤      │
│   │ [Screen B]   │       │ [Screen B]   │  ──►   │ [Screen B]   │      │
│   ├──────────────┤       ├──────────────┤        ├──────────────┤      │
│   │ [Screen A]   │       │ [Screen A]   │        │ [Screen A]   │      │
│   └──────────────┘       └──────────────┘        └──────────────┘      │
│     (Đỉnh Stack)           (B lộ diện)             (C bị thay thế)     │
└────────────────────────────────────────────────────────────────────────┘
```

#### Bản chất kỹ thuật của mô hình LIFO:
- Màn hình khởi đầu của ứng dụng (`home` trong `MaterialApp`) nằm ở đáy ngăn xếp.
- Khi người dùng tương tác mở một trang mới, Route của trang đó được **đẩy (push)** lên đỉnh ngăn xếp và che khuất màn hình trước đó.
- Khi người dùng hoàn thành tác vụ hoặc nhấn nút quay lại, Route trên cùng được **rút (pop)** khỏi ngăn xếp, giải phóng tài nguyên và đưa màn hình liền kề bên dưới trở lại trạng thái hoạt động.

#### Khi nào Navigator 1.0 vẫn là sự lựa chọn tối ưu?
Dù Flutter đã giới thiệu mô hình Declarative (GoRouter / Navigator 2.0) cho các ứng dụng đa nền tảng phức tạp, Navigator 1.0 vẫn đóng vai trò nền tảng không thể thay thế trong các trường hợp:
1. **Các luồng phụ cục bộ (Local Sub-flows):** Hiển thị màn hình chọn ảnh, bộ lọc sản phẩm tạm thời, hoặc màn hình xác nhận mã OTP ngắn hạn.
2. **Hộp thoại và bảng trượt:** Mọi lời gọi `showDialog()`, `showModalBottomSheet()`, `showDatePicker()` trong Flutter về bản chất đều hoạt động dựa trên API `Navigator.of(context).push()`.
3. **Ứng dụng di động thuần túy không cần đồng bộ URL:** Ứng dụng không chạy trên Web và không có các kịch bản Deep Link phức tạp.

---

### 1.2 — Tổng quan các thao tác điều hướng cốt lõi trong Navigator 1.0

Flutter cung cấp bộ API phong phú thông qua lớp `Navigator` tĩnh hoặc extension trên `BuildContext`:

| Phương Thức | Hành Vi Ngăn Xếp (Stack) | Lịch Sử Nút Back | Kịch Bản Điển Hình |
| :--- | :--- | :--- | :--- |
| **`Navigator.push()`** | Thêm Route mới lên đỉnh stack | Back về màn hình vừa gọi push | Mở trang Chi tiết từ Danh sách |
| **`Navigator.pop([result])`** | Gỡ bỏ Route trên đỉnh khỏi stack | Trả về màn hình liền kề bên dưới | Đóng trang, đóng modal, trả kết quả |
| **`Navigator.pushReplacement()`** | Thay thế Route trên đỉnh bằng Route mới | Back về màn hình trước đó của Route cũ | Đăng nhập thành công $\to$ Vào Dashboard |
| **`Navigator.pushAndRemoveUntil()`**| Xóa các Route cũ theo điều kiện rồi push Route mới | Không thể back về các Route đã bị xóa | Đăng xuất $\to$ Xóa sạch stack về Login |
| **`Navigator.popUntil()`** | Pop liên tiếp cho đến khi gặp Route chỉ định | Dừng lại đúng vị trí mong muốn | Hủy wizard nhiều bước $\to$ Về Home |

---

## Phần 2 — Cơ Chế Hoạt Động Tầng Sâu (Under the Hood Deep-Dive)

### 2.1 — Bản chất của Navigator trên Render Tree: `Overlay` & `OverlayEntry`

Nhiều lập trình viên lầm tưởng `Navigator` trực tiếp lưu trữ một danh sách các `Widget`. Trong thực tế, kiến trúc tầng lõi của Flutter phân tách thành các thành phần sau:

```
┌────────────────────────────────────────────────────────────────────────┐
│ KIẾN TRÚC TẦNG LÕI CỦA NAVIGATOR WIDGET                                │
│                                                                        │
│  NavigatorState                                                        │
│   ├── List<_RouteEntry> _history   (Danh sách logic các Route)         │
│   └── Overlay                      (Widget hiển thị trực quan)         │
│        ├── OverlayEntry (Route 0: HomePage - Hiển thị ở đáy)           │
│        ├── OverlayEntry (Route 1: DetailPage - Hiển thị ở giữa)        │
│        └── OverlayEntry (ModalBarrier / Dialog - Hiển thị trên cùng)   │
└────────────────────────────────────────────────────────────────────────┘
```

1. **`Overlay` Widget:** `NavigatorState` sở hữu một `Overlay`. `Overlay` là một widget đặc biệt tương tự như `Stack`, có khả năng quản lý danh sách các `OverlayEntry` độc lập với cây widget thông thường.
2. **`Route` và `PageRoute`:** Khi gọi `Navigator.push(context, route)`, `Route` sẽ:
   - Khởi tạo `AnimationController` điều khiển hiệu ứng chuyển cảnh.
   - Tạo ra một hoặc nhiều `OverlayEntry` đại diện cho giao diện trang và lớp phủ mờ (`ModalBarrier`).
   - Yêu cầu `Overlay` chèn các entry này lên trên cùng.
3. **Giải phóng bộ nhớ khi Pop:** Khi gọi `Navigator.pop()`, Route bắt đầu chạy animation đảo ngược (reverse animation). Chỉ khi animation kết thúc hoàn toàn, `OverlayEntry` mới bị tháo dỡ (`remove()`) khỏi Render Tree và `State` của trang mới chính thức được gọi hàm `dispose()`.

---

### 2.2 — Kiểm soát nút Back phần cứng & Cử chỉ vuốt với `PopScope`

Trước phiên bản Flutter 3.12, việc chặn nút Back được thực hiện qua `WillPopScope`. Tuy nhiên, `WillPopScope` không hỗ trợ tính năng **Predictive Back Gesture** (xem trước màn hình khi vuốt cạnh trên Android 14+).

Từ Flutter 3.12+, `PopScope` là giải pháp chuẩn hóa thay thế hoàn toàn:

```dart
PopScope(
  // 1. canPop = false: Chặn đứng hành vi pop mặc định của hệ thống
  canPop: !hasUnsavedChanges,
  
  // 2. Callback được kích hoạt khi có cử chỉ hoặc sự kiện pop
  onPopInvokedWithResult: (bool didPop, dynamic result) async {
    // Nếu didPop = true, nghĩa là trang đã được pop thành công, không làm gì thêm
    if (didPop) return;

    // Nếu didPop = false, hiển thị hộp thoại xác nhận người dùng có muốn thoát không
    final bool shouldLeave = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Rời khỏi trang?'),
        content: const Text('Các thay đổi chưa lưu sẽ bị mất.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Ở lại'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('Rời đi'),
          ),
        ],
      ),
    ) ?? false;

    // Nếu người dùng đồng ý rời đi, chủ động gọi Navigator.pop()
    if (shouldLeave && context.mounted) {
      Navigator.of(context).pop(result);
    }
  },
  child: const FormContentWidget(),
)
```

---

### 2.3 — Phân biệt Root Navigator vs Local Navigator (`rootNavigator: true`)

Trong các ứng dụng có cấu trúc lồng nhau (ví dụ một màn hình có `BottomNavigationBar` và mỗi Tab lại có một `Navigator` con riêng):

```dart
// Mặc định: Tìm Navigator gần nhất trong cây phân cấp ngược lên trên
Navigator.of(context).push(...);

// rootNavigator = true: Bỏ qua mọi Navigator con, tìm lên Navigator cao nhất của MaterialApp
Navigator.of(context, rootNavigator: true).push(...);
```

- **Khi nào bắt buộc dùng `rootNavigator: true`?**
  Khi bạn muốn hiển thị một màn hình hoặc `Dialog` che phủ hoàn toàn màn hình, bao gồm cả thanh `BottomNavigationBar`. Nếu không bật cờ này, Dialog sẽ bị giới hạn không gian chỉ trong phần thân của tab hiện tại!

---

## Phần 3 — Hướng Dẫn Thực Hành & Toàn Bộ Tính Năng Cốt Lõi (Production-Ready Implementations)

### 3.1 — Kỹ thuật Truyền & Nhận Dữ Liệu Hai Chiều

#### 1. Truyền dữ liệu đi (Forward Passing):
Flutter hỗ trợ 2 phương pháp truyền dữ liệu chính:

```dart
// CÁCH 1: Constructor Injection (Khuyên Dùng - Type-safe 100%)
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (context) => ProductDetailScreen(
      productId: 'P-100',
      productName: 'Bàn phím cơ',
    ),
  ),
);

// CÁCH 2: RouteSettings.arguments (Thường dùng cho Named Routes)
Navigator.pushNamed(
  context,
  '/product-detail',
  arguments: ProductArgs(id: 'P-100', name: 'Bàn phím cơ'),
);
```

#### 2. Nhận kết quả phản hồi bất đồng bộ (Pop with Result):
Khi màn hình B muốn gửi dữ liệu ngược lại cho màn hình A (ví dụ: chọn màu sắc, xác nhận xóa):

```dart
// Màn hình A: Gọi push và await kết quả kiểu Future<T?>
final String? selectedColor = await Navigator.push<String>(
  context,
  MaterialPageRoute(builder: (context) => const ColorPickerScreen()),
);

if (selectedColor != null && context.mounted) {
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(content: Text('Đã chọn màu: $selectedColor')),
  );
}

// Màn hình B (ColorPickerScreen): Trả kết quả về khi pop
ElevatedButton(
  onPressed: () {
    // Đóng màn hình và gửi giá trị về cho A
    Navigator.pop(context, 'Xanh Dương');
  },
  child: const Text('Xác nhận Xanh Dương'),
);
```

---

### 3.2 — Định Tuyến Theo Tên (Named Routes) & `onGenerateRoute`

#### 1. Hạn chế của bảng ánh xạ tĩnh `routes`:
Khai báo mảng `routes: <String, WidgetBuilder>{}` trong `MaterialApp` rất phổ biến ở mức cơ bản nhưng bộc lộ 2 nhược điểm chí mạng:
- Không thể truyền tham số qua constructor một cách trực tiếp.
- Không thể trích xuất tham số động (Dynamic parameters) hoặc tùy biến Animation.

#### 2. Giải pháp chuẩn mực: Sử dụng `onGenerateRoute`
`onGenerateRoute` cho phép quản lý tập trung toàn bộ logic khởi tạo Route, trích xuất `RouteSettings.arguments`, kiểm tra bảo vệ và gán fallback trang 404:

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Navigator 1.0 Enterprise',
      initialRoute: AppRoutes.home,
      
      // 1. Quản lý route tập trung và linh hoạt
      onGenerateRoute: (RouteSettings settings) {
        switch (settings.name) {
          case AppRoutes.home:
            return MaterialPageRoute(
              settings: settings,
              builder: (_) => const HomeScreen(),
            );

          case AppRoutes.productDetail:
            // Trích xuất an toàn arguments
            final args = settings.arguments as ProductDetailArgs?;
            if (args == null) {
              return _errorRoute('Thiếu thông số sản phẩm');
            }
            return MaterialPageRoute(
              settings: settings,
              builder: (_) => ProductDetailScreen(args: args),
            );

          case AppRoutes.profile:
            // Kỹ thuật Route Guard (Kiểm tra đăng nhập)
            final bool isAuthenticated = AuthService.isLoggedIn;
            if (!isAuthenticated) {
              return MaterialPageRoute(
                settings: settings,
                builder: (_) => const LoginScreen(redirectTo: AppRoutes.profile),
              );
            }
            return MaterialPageRoute(
              settings: settings,
              builder: (_) => const ProfileScreen(),
            );

          default:
            return null; // Chuyển giao tiếp cho onUnknownRoute
        }
      },

      // 2. Fallback khi không tìm thấy bất kỳ route nào
      onUnknownRoute: (RouteSettings settings) {
        return MaterialPageRoute(
          settings: settings,
          builder: (_) => Scaffold(
            appBar: AppBar(title: const Text('Lỗi 404')),
            body: Center(
              child: Text('Không tìm thấy tuyến đường: ${settings.name}'),
            ),
          ),
        );
      },
    );
  }

  static Route<dynamic> _errorRoute(String message) {
    return MaterialPageRoute(
      builder: (_) => Scaffold(
        appBar: AppBar(title: const Text('Lỗi Dữ Liệu')),
        body: Center(child: Text(message)),
      ),
    );
  }
}

// Quản lý hằng số Route tập trung tránh typo
abstract final class AppRoutes {
  static const home = '/';
  static const productDetail = '/product-detail';
  static const profile = '/profile';
  static const login = '/login';
}

class ProductDetailArgs {
  final String productId;
  final String title;

  const ProductDetailArgs({required this.productId, required this.title});
}
```

---

### 3.3 — Tùy biến Hoạt Ảnh Chuyển Trang với `PageRouteBuilder`

Mặc định, `MaterialPageRoute` sử dụng animation trượt từ phải sang trái hoặc từ dưới lên (tùy nền tảng). Sử dụng `PageRouteBuilder` giúp bạn tạo ra mọi hiệu ứng chuyển động mong muốn:

```dart
// 1. Hiệu ứng Mờ dần (Fade Transition)
Route createFadeRoute(Widget page) {
  return PageRouteBuilder(
    pageBuilder: (context, animation, secondaryAnimation) => page,
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      return FadeTransition(
        opacity: animation,
        child: child,
      );
    },
    transitionDuration: const Duration(milliseconds: 300),
  );
}

// 2. Hiệu ứng Trượt từ dưới lên (Slide Up Transition)
Route createSlideUpRoute(Widget page) {
  return PageRouteBuilder(
    pageBuilder: (context, animation, secondaryAnimation) => page,
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      const begin = Offset(0.0, 1.0);
      const end = Offset.zero;
      const curve = Curves.easeOutCubic;

      final tween = Tween(begin: begin, end: end).chain(CurveTween(curve: curve));

      return SlideTransition(
        position: animation.drive(tween),
        child: child,
      );
    },
  );
}

// Cách gọi sử dụng:
Navigator.of(context).push(createSlideUpRoute(const FilterScreen()));
```

---

## Phần 4 — Cạm Bẫy Thường Gặp & Anti-Patterns (Pitfalls)

### 4.1 — Dùng `context` trước khi Navigator được Mount trên Widget Tree

#### Mô tả lỗi:
Gọi `Navigator.of(context)` bên trong hàm `main()` hoặc trực tiếp tại constructor của `MyApp`:

```dart
// ❌ SAI LẦM: Crash với exception 'Navigator operation requested with a context that does not include a Navigator'
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Context ở đây là cha của MaterialApp, chưa hề có Navigator nào tồn tại bên trên nó!
    Navigator.of(context).pushNamed('/home'); 
    return MaterialApp(...);
  }
}
```

#### Giải pháp:
Chỉ sử dụng `BuildContext` của các widget con nằm **bên dưới** `MaterialApp` (ví dụ bên trong `Builder` widget hoặc method `build()` của một màn hình con).

---

### 4.2 — Lạm dụng Biến Toàn Cục (Global State) để truyền tham số tạm

#### Mô tả lỗi:
Tạo các biến `static var selectedProductId = ''` ở một file toàn cục để màn hình sau đọc dữ liệu:
- **Hậu quả:** Phá vỡ tính bao đóng, dễ sinh ra Race Condition khi màn hình bị rebuild, rò rỉ bộ nhớ khi không giải phóng biến tĩnh.
- **Giải pháp:** Luôn sử dụng Constructor Injection hoặc `RouteSettings.arguments` để đảm bảo vòng đời dữ liệu gắn liền với Route.

---

### 4.3 — Ép kiểu không an toàn (`as Type`) với `RouteSettings.arguments`

#### Mô tả lỗi:
```dart
// ❌ NGUY HIỂM
final id = ModalRoute.of(context)!.settings.arguments as String;
```
Nếu vô tình gọi `Navigator.pushNamed('/target')` mà quên truyền tham số, ứng dụng sẽ văng lỗi `TypeError: null is not a subtype of type 'String'`.
- **Giải pháp:** Luôn kiểm tra `args is ProductArgs` hoặc sử dụng kiểu nullable `as ProductArgs?` kết hợp xử lý fallback hiển thị thông báo lỗi thân thiện.

---

## Phần 5 — Khảo Sát Kỹ Thuật Chuyên Sâu & Bài Tập Phân Tích Mã Nguồn (Technical Assessment & Code Tracing)

### 5.1 — Bộ câu hỏi khảo sát kỹ thuật chuyên sâu (Deep-Dive Q&A)

#### Câu 1: Sự khác biệt bản chất giữa `push`, `pushReplacement` và `pushAndRemoveUntil` trên Navigation Stack?
*Phân tích bản chất:*
- **`push(route)`**: Tăng kích thước ngăn xếp lên $+1$. Đưa route mới lên đỉnh, giữ nguyên toàn bộ lịch sử các route trước đó. Khi pop, quyền điều khiển được trả lại cho route ngay phía dưới.
- **`pushReplacement(route)`**: Kích thước ngăn xếp không đổi. Nó kích hoạt quá trình dỡ bỏ (dispose) route trên đỉnh hiện tại và thay thế tức thì bằng route mới. Thường dùng trong luồng xác thực (Login $\to$ Home).
- **`pushAndRemoveUntil(route, predicate)`**: Duyệt ngược từ đỉnh ngăn xếp xuống đáy, liên tục pop và giải phóng tất cả các route cho đến khi hàm `predicate(Route<dynamic> r)` trả về `true`, sau đó push route mới vào. Nếu `predicate = (route) => false`, toàn bộ lịch sử sẽ bị xóa sạch, biến route mới thành gốc duy nhất của ứng dụng.

---

#### Câu 2: Tại sao `PopScope` thay thế hoàn toàn `WillPopScope` từ Flutter 3.12?
*Phân tích bản chất:*
- `WillPopScope` hoạt động theo cơ chế bất đồng bộ chặn trước (`Future<bool> onWillPop()`). Khi người dùng bắt đầu vuốt từ cạnh màn hình, framework buộc phải đợi hàm Future này resolve mới biết có được phép pop hay không. Điều này làm tê liệt hoàn toàn tính năng **Predictive Back Gesture** trên Android 14+ (vốn yêu cầu hệ điều hành phải vẽ animation xem trước màn hình cha ngay thời điểm ngón tay vừa chạm vào cạnh).
- `PopScope` chuyển đổi sang mô hình đồng bộ hướng sự kiện: cờ `canPop` cho phép hệ điều hành biết ngay tức thì liệu thao tác vuốt có hợp lệ hay không. Nếu `canPop = false`, sự kiện pop bị triệt tiêu và callback `onPopInvokedWithResult(false, result)` được gọi để ứng dụng tự hiển thị dialog xác nhận.

---

#### Câu 3: Navigator Stack quản lý việc hiển thị bằng Widget nào dưới lõi framework?
*Phân tích bản chất:*
- `Navigator` hiện thực hóa việc vẽ các route thông qua widget **`Overlay`**. Mỗi `Route` nắm giữ một hoặc nhiều **`OverlayEntry`**.
- Khi push, các `OverlayEntry` được nạp vào `Overlay` tương tự các con của một `Stack`. Điểm đặc biệt là `Overlay` cho phép các entry tồn tại độc lập với cây widget chính, cho phép hiển thị các lớp phủ (Floating Tooltip, SnackBar, Dialog) đè lên toàn bộ ứng dụng mà không cần thay đổi cấu trúc widget tree hiện hành.

---

#### Câu 4: Khi nào bắt buộc phải dùng `Navigator.of(context, rootNavigator: true)`?
*Phân tích bản chất:*
- Trong các ứng dụng phức tạp có nhiều tầng Navigator (ví dụ: mỗi Tab trong `BottomNavigationBar` có một `Navigator` riêng để chuyển trang nội bộ bên trong Tab).
- Mặc định, `Navigator.of(context)` chỉ tìm kiếm instance `NavigatorState` gần nhất theo chiều ngược lên cây tổ tiên (Local Navigator). Nếu bạn hiển thị Dialog hoặc push một màn hình tràn viền mà không dùng `rootNavigator: true`, trang mới sẽ bị "giam lỏng" bên trong kích thước của Tab đó và thanh BottomNavigationBar vẫn sẽ hiển thị đè lên trên.
- Sử dụng `rootNavigator: true` buộc framework bỏ qua mọi Local Navigator để truy xuất thẳng lên Root Navigator cao nhất của `MaterialApp`, đảm bảo màn hình mới phủ kín 100% diện tích thiết bị.

---

### 5.2 — Bài tập phân tích luồng thực thi (Code Tracing)

#### Đề bài:
Một ứng dụng khởi chạy với Stack ban đầu:
$$\text{Stack} = [\text{SplashRoute}, \text{LoginRoute}]$$

Người dùng thực hiện tuần tự các thao tác sau:
1. Đăng nhập thành công, hệ thống thực thi:
   ```dart
   Navigator.pushReplacementNamed(context, '/home');
   ```
2. Từ trang Home, người dùng mở trang danh mục sản phẩm:
   ```dart
   Navigator.pushNamed(context, '/products');
   ```
3. Từ trang Products, người dùng mở chi tiết sản phẩm:
   ```dart
   Navigator.pushNamed(context, '/product-detail');
   ```
4. Người dùng bấm Đăng Xuất trên trang chi tiết sản phẩm, hệ thống kích hoạt:
   ```dart
   Navigator.pushAndRemoveUntil(
     context,
     MaterialPageRoute(builder: (_) => const LoginScreen()),
     ModalRoute.withName('/home'),
   );
   ```
5. Người dùng nhấn nút Back vật lý trên điện thoại.

**Câu hỏi:**
1. Sau bước 4, danh sách các Route còn lại trên ngăn xếp là gì?
2. Sau bước 5, màn hình nào sẽ hiển thị? Trạng thái của ứng dụng là gì?

---

#### Đáp án phân tích:

**1. Sau bước 4:**
- Trạng thái ban đầu: `[SplashRoute, LoginRoute]`
- Sau bước 1 (`pushReplacementNamed('/home')`): `LoginRoute` bị xóa và thay bằng `HomeRoute`. Stack lúc này:
  $$\text{Stack} = [\text{SplashRoute}, \text{HomeRoute}]$$
- Sau bước 2 (`pushNamed('/products')`): Thêm `ProductsRoute`. Stack:
  $$\text{Stack} = [\text{SplashRoute}, \text{HomeRoute}, \text{ProductsRoute}]$$
- Sau bước 3 (`pushNamed('/product-detail')`): Thêm `DetailRoute`. Stack:
  $$\text{Stack} = [\text{SplashRoute}, \text{HomeRoute}, \text{ProductsRoute}, \text{DetailRoute}]$$
- Sau bước 4 (`pushAndRemoveUntil` với predicate `ModalRoute.withName('/home')`):
  - Framework duyệt từ đỉnh: Xóa `DetailRoute`, xóa `ProductsRoute`.
  - Khi gặp `HomeRoute`, điều kiện `ModalRoute.withName('/home')` trả về `true` $\to$ Dừng xóa và giữ lại `HomeRoute` (cùng các route bên dưới nó).
  - Push màn hình mới `LoginScreen` lên trên `HomeRoute`.
  - **Kết quả Stack sau bước 4:**
    $$\mathbf{\text{Stack} = [\text{SplashRoute}, \text{HomeRoute}, \text{LoginRoute}]}$$

**2. Sau bước 5 (Nhấn Back):**
- Thao tác nhấn nút Back sẽ thực hiện pop `LoginRoute` ra khỏi đỉnh ngăn xếp.
- Màn hình hiển thị tiếp theo sẽ là **`HomeRoute` (HomeScreen)**.
- *Lưu ý kiến trúc:* Nếu mục tiêu khi đăng xuất là ngăn hoàn toàn người dùng quay lại các trang cũ, lập trình viên phải truyền điều kiện `(route) => false` để xóa sạch toàn bộ stack thay vì giữ lại đến `/home`.
