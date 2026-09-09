# Bài 8.3 — Hero Animations & Chuyển Trang Tùy Biến (Page Transitions)

## Tài Liệu Tham Khảo Chính Thức
- [Flutter Documentation: Hero animations](https://docs.flutter.dev/ui/animations/hero-animations)
- [Flutter API: Hero class](https://api.flutter.dev/flutter/widgets/Hero-class.html)
- [Flutter API: PageRouteBuilder class](https://api.flutter.dev/flutter/widgets/PageRouteBuilder-class.html)
- [Flutter Package: animations (Material Motion System)](https://pub.dev/packages/animations)

---

## Phần 1 — Khái Niệm & Vai Trò Của Hero Animation

### 1.1 — Khái Niệm Hero Animation

Trong thiết kế giao diện di động, khi người dùng chuyển hướng từ danh sách sang màn hình chi tiết, các thao tác chuyển trang thông thường (push route) sẽ thay thế toàn bộ màn hình một cách đột ngột. 

**Hero Animation** giải quyết vấn đề này bằng cách tạo ra tính **liên tục về thị giác (Visual Continuity)**:
- Một phần tử giao diện (ví dụ ảnh đại diện sản phẩm, avatar người dùng) xuất hiện ở cả hai màn hình.
- Khi điều hướng, phần tử này dường như "bay" (fly) từ vị trí và kích thước ban đầu ở màn hình nguồn đến vị trí và kích thước mới ở màn hình đích.

```
┌────────────────────────────────────────────────────────────────────────┐
│ NGUYÊN LÝ HOẠT ĐỘNG CỦA HERO ANIMATION                                │
│                                                                        │
│ Màn hình A (Source Route)                   Màn hình B (Destination)   │
│ ┌──────────────────────┐                    ┌──────────────────────┐   │
│ │ [Thumbnail (80x80)]  │ ──► [CHUYẾN BAY] ──► │                      │   │
│ │ Hero(tag: 'item_01') │     trên Overlay   │ [Large Image(300x300)]│  │
│ └──────────────────────┘                    │ Hero(tag: 'item_01') │   │
│                                             └──────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

- **Điều kiện tiên quyết**: Cả hai widget ở hai màn hình khác nhau phải được bọc trong widget `Hero` và sở hữu cùng một thuộc tính `tag` đồng nhất.

---

### 1.2 — Chuyển Trang Tùy Biến Với `PageRouteBuilder`

Mặc định, Flutter sử dụng hiệu ứng chuyển trang theo chuẩn của từng nền tảng:
- **Android (`MaterialPageRoute`)**: Slide từ dưới lên trên hoặc Fade Through theo Material Design.
- **iOS (`CupertinoPageRoute`)**: Slide ngang từ phải sang trái kết hợp hiệu ứng vuốt quay lại (Back Swipe Gesture).

Khi ứng dụng yêu cầu hiệu ứng chuyển trang riêng biệt (như phóng to từ tâm, mờ dần toàn diện hoặc trượt theo trục tùy biến), `PageRouteBuilder` cung cấp một route linh hoạt cho phép lập trình viên định nghĩa các `TransitionBuilder` tùy biến.

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Cơ Chế Chuyến Bay Hero (Hero Flight Mechanism)

Khi thao tác chuyển route diễn ra (`Navigator.push` hoặc `Navigator.pop`), luồng xử lý bên dưới của Flutter Engine được vận hành như sau:

```
┌────────────────────────────────────────────────────────────────────────┐
│ VÒNG ĐỜI NỘI BỘ CỦA MỘT CHUYẾN BAY HERO                                │
│                                                                        │
│ 1. Navigator phát tín hiệu chuyển route qua HeroController             │
│      │                                                                 │
│      ▼                                                                 │
│ 2. HeroController duyệt RenderTree tìm cặp Hero có cùng tag            │
│    • Xác định Rect nguồn (vị trí (x, y) và kích thước (w, h) ở Màn A)  │
│    • Xác định Rect đích (vị trí (x, y) và kích thước (w, h) ở Màn B)   │
│      │                                                                 │
│      ▼                                                                 │
│ 3. Tạo chuyến bay (_HeroFlight):                                       │
│    • Tạm thời ẩn widget gốc ở cả hai màn hình                          │
│    • Đưa một bản sao widget vào OverlayEntry của Navigator             │
│      (Overlay là tầng hiển thị trên cùng, độc lập với các route)       │
│      │                                                                 │
│      ▼                                                                 │
│ 4. Thực thi RectTween nội suy từ Rect nguồn -> Rect đích              │
│    • Sử dụng đường cong tốc độ theo thời lượng chuyển route            │
│      │                                                                 │
│      ▼                                                                 │
│ 5. Kết thúc chuyến bay:                                                │
│    • Gỡ bỏ widget khỏi OverlayEntry                                    │
│    • Hiển thị widget đích trên Màn B tại vị trí cố định                │
└────────────────────────────────────────────────────────────────────────┘
```

- **Lớp `Overlay`**: Đây là mấu chốt kỹ thuật giúp Hero có thể bay xuyên qua ranh giới giữa hai màn hình mà không bị giới hạn bởi phạm vi cắt (clip) của từng Route riêng lẻ.
- **`RectTween`**: Lớp toán học chịu trách nhiệm nội suy tọa độ 4 chiều $(x, y, \text{width}, \text{height})$ giữa hai khung chữ nhật trong suốt quá trình bay.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Cấu Hình Hero Animation Cơ Bản

Đảm bảo thuộc tính `tag` là duy nhất trên mỗi đối tượng dữ liệu:

```dart
import 'package:flutter/material.dart';

class ProductItem {
  final String id;
  final String title;
  final String imageUrl;

  const ProductItem({
    required this.id,
    required this.title,
    required this.imageUrl,
  });
}

// 1. Màn hình danh sách (Source)
class ProductListScreen extends StatelessWidget {
  final List<ProductItem> products;

  const ProductListScreen({super.key, required this.products});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Danh Sách Sản Phẩm')),
      body: ListView.builder(
        itemCount: products.length,
        itemBuilder: (context, index) {
          final product = products[index];
          return ListTile(
            leading: Hero(
              // Tag duy nhất theo ID của sản phẩm
              tag: 'product-image-${product.id}',
              child: ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: Image.network(
                  product.imageUrl,
                  width: 56,
                  height: 56,
                  fit: BoxFit.cover,
                ),
              ),
            ),
            title: Text(product.title),
            onTap: () {
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (_) => ProductDetailScreen(product: product),
                ),
              );
            },
          );
        },
      ),
    );
  }
}

// 2. Màn hình chi tiết (Destination)
class ProductDetailScreen extends StatelessWidget {
  final ProductItem product;

  const ProductDetailScreen({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(product.title)),
      body: Column(
        children: [
          Hero(
            // Tag phải khớp chính xác với tag ở màn hình danh sách
            tag: 'product-image-${product.id}',
            child: Image.network(
              product.imageUrl,
              width: double.infinity,
              height: 300,
              fit: BoxFit.cover,
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(16.0),
            child: Text(
              product.title,
              style: Theme.of(context).textTheme.headlineSmall,
            ),
          ),
        ],
      ),
    );
  }
}
```

---

### 3.2 — Tùy Biến Chuyến Bay Với `flightShuttleBuilder`

Khi một widget chứa văn bản (`Text`) tham gia vào Hero flight, trong quá trình bay trên `Overlay`, nó tạm thời mất liên kết với `ThemeData` và `Material` của Route. Thuộc tính `flightShuttleBuilder` cho phép lập trình viên định nghĩa cấu trúc widget hiển thị riêng biệt trong lúc đang bay:

```dart
Hero(
  tag: 'card-title-${product.id}',
  flightShuttleBuilder: (
    BuildContext flightContext,
    Animation<double> animation,
    HeroFlightDirection flightDirection,
    BuildContext fromHeroContext,
    BuildContext toHeroContext,
  ) {
    // Đảm bảo kiểu chữ giữ nguyên nền Material và màu chữ trong lúc bay
    return Material(
      color: Colors.transparent,
      child: DefaultTextStyle(
        style: TextStyle(
          fontSize: flightDirection == HeroFlightDirection.push ? 20.0 : 16.0,
          fontWeight: FontWeight.bold,
          color: Colors.black,
        ),
        child: Text(product.title),
      ),
    );
  },
  child: Text(product.title),
)
```

---

### 3.3 — Tạo Hiệu Ứng Chuyển Trang Tùy Biến Với `PageRouteBuilder`

Thay thế hiệu ứng chuyển trang mặc định bằng hiệu ứng trượt kết hợp mờ dần:

```dart
import 'package:flutter/material.dart';

Route createCustomPageRoute(Widget page) {
  return PageRouteBuilder(
    pageBuilder: (context, animation, secondaryAnimation) => page,
    transitionDuration: const Duration(milliseconds: 400),
    reverseTransitionDuration: const Duration(milliseconds: 350),
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      // 1. Đường cong gia tốc chuyển động
      final curvedAnimation = CurvedAnimation(
        parent: animation,
        curve: Curves.fastOutSlowIn,
        reverseCurve: Curves.easeInOut,
      );

      // 2. Hiệu ứng trượt từ dưới lên
      final slideTransition = Tween<Offset>(
        begin: const Offset(0.0, 0.1),
        end: Offset.zero,
      ).animate(curvedAnimation);

      // 3. Hiệu ứng mờ dần
      final fadeTransition = Tween<double>(
        begin: 0.0,
        end: 1.0,
      ).animate(curvedAnimation);

      return SlideTransition(
        position: slideTransition,
        child: FadeTransition(
          opacity: fadeTransition,
          child: child,
        ),
      );
    },
  );
}
```

---

### 3.4 — Tích Hợp Material Motion System (`package:animations`)

Gói thư viện chính thức `animations` cung cấp các chuyển động theo chuẩn Material 3:

```dart
import 'package:animations/animations.dart';
import 'package:flutter/material.dart';

// Mở trang sử dụng SharedAxisTransition theo trục Z
void navigateWithSharedAxis(BuildContext context, Widget destinationPage) {
  Navigator.push(
    context,
    PageRouteBuilder(
      transitionDuration: const Duration(milliseconds: 350),
      pageBuilder: (context, animation, secondaryAnimation) => destinationPage,
      transitionsBuilder: (context, animation, secondaryAnimation, child) {
        return SharedAxisTransition(
          animation: animation,
          secondaryAnimation: secondaryAnimation,
          transitionType: SharedAxisTransitionType.scaled, // Trục Z
          child: child,
        );
      },
    ),
  );
}
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Trùng lặp `Hero tag` trong cùng một Route

#### Mô tả vấn đề:
Khởi tạo nhiều widget `Hero` với cùng một chuỗi `tag` tĩnh (ví dụ: `tag: 'product_image'`) trên cùng một màn hình danh sách:

```dart
// Lỗi: Nhiều item dùng chung một tag tĩnh
Hero(
  tag: 'product_avatar',
  child: Image.network(item.url),
)
```

#### Nguyên nhân kỹ thuật:
Khi `HeroController` tìm kiếm widget tương ứng để bắt đầu chuyến bay, nó phát hiện có từ hai widget trở lên cùng chia sẻ một tag. Framework sẽ ném ra ngoại lệ nghiêm trọng:
`There are multiple heroes that share the same tag within a subtree`.

#### Biện pháp khắc phục:
Luôn gắn kèm ID định danh duy nhất vào tag: `tag: 'product_avatar_${item.id}'`.

---

### 4.2 — Chữ bị gạch chân màu vàng kép trong lúc bay

#### Mô tả vấn đề:
Khi bọc một `Text` widget trong `Hero`, trong suốt thời gian bay, dòng chữ xuất hiện hai vạch gạch chân màu vàng và phông chữ bị méo.

#### Nguyên nhân kỹ thuật:
Trong suốt chuyến bay, widget nằm trực tiếp trên tầng `Overlay`. Tầng `Overlay` không tự động cung cấp một đối tượng `Material` cha, dẫn đến việc `Text` không tìm thấy `DefaultTextStyle` từ `ThemeData` của Scaffold và fallback về kiểu văn bản thô của hệ thống.

#### Biện pháp khắc phục:
Bọc `Text` bên trong một widget `Material` với màu nền trong suốt:
```dart
Hero(
  tag: 'title-${item.id}',
  child: Material(
    color: Colors.transparent,
    child: Text(item.title),
  ),
)
```

---

### 4.3 — Vấn đề Hero trong danh sách tái sử dụng phần tử (`ListView`)

#### Mô tả vấn đề:
Khi người dùng cuộn danh sách ở màn hình A, sau đó mở màn hình B, cuộn tiếp một đoạn dài rồi bấm nút Back quay lại: Hiệu ứng Hero bay về bị giật hoặc biến mất giữa chừng.

#### Nguyên nhân kỹ thuật:
Do cơ chế ảo hóa danh sách (Virtualization), khi người dùng cuộn, phần tử gốc ở màn hình A đã bị tháo gỡ khỏi Render Tree để giải phóng bộ nhớ. Khi bay ngược về, `HeroController` không thể xác định được tọa độ `Rect` nguồn trên màn hình A.

#### Biện pháp khắc phục:
Đảm bảo item nguồn vẫn nằm trong tầm hiển thị hoặc sử dụng `keepAlive: true` thông qua `AutomaticKeepAliveClientMixin` cho các item quan trọng.

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Lớp `HeroController` phát hiện và theo dõi các Hero widget trong ứng dụng như thế nào?
*Phân tích:*
`HeroController` là một `NavigatorObserver`. Khi `Navigator.push` hoặc `pop` được gọi, `HeroController` nhận được sự kiện `didPush` / `didPop`. Nó kích hoạt một chu kỳ duyệt qua cây `Element` của cả Route cũ và Route mới thông qua phương thức `findHero()`, thu thập các `RenderBox` tương ứng để đo đạc tọa độ hình học `Rect` toàn cục trước khi khung hình mới kịp hiển thị.

---

#### Câu hỏi 2: Tại sao widget Hero gốc ở cả hai màn hình đều bị ẩn trong suốt thời gian diễn ra chuyến bay?
*Phân tích:*
Nếu widget gốc ở màn hình A và B vẫn hiển thị bình thường, người dùng sẽ nhìn thấy 3 đối tượng cùng một lúc: một đối tượng đứng yên ở Màn A, một đối tượng đứng yên ở Màn B, và một đối tượng đang bay ở giữa. Do đó, `_HeroFlight` thiết lập thuộc tính `_placeholder` hoặc ẩn tạm thời RenderObject của cả hai đầu để chỉ có duy nhất thực thể bay trên `Overlay` hiển thị trước mắt người dùng.

---

#### Câu hỏi 3: Thuộc tính `placeholderBuilder` của Hero được sử dụng cho mục đích gì?
*Phân tích:*
Trong khi chuyến bay đang diễn ra, vị trí nguồn ở màn hình ban đầu sẽ để lại một khoảng trống. Mặc định, Flutter đặt một `SizedBox` có kích thước bằng đúng widget gốc để giữ nguyên cấu trúc layout không bị sụp đổ (layout shift). Lập trình viên có thể dùng `placeholderBuilder` để tùy biến phần giữ chỗ này (ví dụ hiển thị một khung xương xám mờ - skeleton loading) thay vì một khoảng trống vô hình.

---

#### Câu hỏi 4: Sự khác biệt về mặt kiến trúc giữa `Hero` và `AnimatedContainer`?
*Phân tích:*
- `AnimatedContainer`: Hoạt động cục bộ bên trong một Route duy nhất, phụ thuộc vào việc thay đổi trạng thái của chính màn hình đó.
- `Hero`: Hoạt động ở tầng liên Route (Cross-route). Nó không tự animate các thuộc tính bên trong của widget mà đưa toàn bộ widget lên tầng `Overlay` và sử dụng một ma trận biến đổi tọa độ toàn cục (`Matrix4` / `RectTween`) để di chuyển cả khối giao diện giữa hai màn hình độc lập.

---

#### Câu hỏi 5: Tác động của `secondaryAnimation` trong `PageRouteBuilder` là gì?
*Phân tích:*
- `animation`: Là tiến độ chuyển động khi **chính route này** đang được push lên hoặc pop về ($0.0 \to 1.0$).
- `secondaryAnimation`: Là tiến độ chuyển động khi **một route mới khác** được push đè lên trên route hiện tại. Tham số này cho phép route hiện tại tự tạo hiệu ứng rút lui (như mờ dần nhẹ hoặc co nhỏ lại về phía sau) khi có màn hình khác xuất hiện phía trước.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Một lập trình viên cấu hình màn hình `HomeScreen` có hai nút bấm hiển thị ảnh đại diện của cùng một người dùng:

```dart
class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Hero(
          tag: 'user_avatar',
          child: Image.asset('avatar.png', width: 50, height: 50),
        ),
        const Spacer(),
        Hero(
          tag: 'user_avatar',
          child: Image.asset('avatar.png', width: 100, height: 100),
        ),
      ],
    );
  }
}
```

Khi người dùng nhấn một nút để chuyển sang `DetailScreen` (nơi cũng có một `Hero(tag: 'user_avatar', ...)`):
1. Điều gì sẽ xảy ra tại thời điểm `Navigator.push` được kích hoạt?
2. Hãy giải thích nguyên nhân dựa trên cơ chế của `HeroController`.
3. Giải pháp kỹ thuật chuẩn xác để giải quyết trường hợp này là gì?

#### Kết quả phân tích kỹ thuật:

1. **Hiện tượng xảy ra:**
   - Framework sẽ ném ra ngoại lệ nghiêm trọng và dừng chuyển cảnh:
     `FlutterError: There are multiple heroes that share the same tag within a subtree.`

2. **Nguyên nhân kỹ thuật:**
   - Trong quá trình chuẩn bị chuyến bay tại `didPush`, `HeroController` gọi hàm `_discoverHeroes()` để quét cây widget của `HomeScreen` và tạo một bảng ánh xạ `Map<Object, _HeroState>`.
   - Khi phát hiện tag `'user_avatar'` đã tồn tại trong Map và gặp lại lần thứ hai, `HeroController` không thể xác định được đối tượng nào (nút trên 50x50 hay nút dưới 100x100) là mốc tọa độ bắt đầu của chuyến bay. Do vi phạm tính toàn vẹn của bảng ánh xạ 1-1, framework ném ngoại lệ để ngăn chặn hành vi không xác định.

3. **Giải pháp kỹ thuật:**
   - Mỗi Hero trong cùng một Route bắt buộc phải có một tag duy nhất. Nếu có hai ảnh đại diện ở hai vị trí khác nhau, cần phân biệt rõ ngữ cảnh: `tag: 'user_avatar_top'` và `tag: 'user_avatar_bottom'`.
