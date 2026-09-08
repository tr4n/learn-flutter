# Chuyên Đề 01 - Bài 04: Collections & Các Toán Tử UI Trong Dart

> **Trọng tâm**: Collection if, Collection for, Spread operator (`...`), Null-aware spread (`...?`), Các phương thức hàm học (`map`, `where`, `fold`, `expand`), Lazy Evaluation của Iterable, Danh sách bất biến (`List.unmodifiable`), và bộ câu hỏi thẩm định năng lực kỹ thuật chuyên sâu.

---

## 1. Bộ 3 Toán Tử UI Quyền Lực: `if`, `for`, `...`

Trước Dart 2.3, để thêm một Widget con có điều kiện vào `Column`, lập trình viên phải tạo một mảng tạm thời bên ngoài rồi dùng lệnh `.add()` cổ điển mang tính mệnh lệnh (imperative).

Ngày nay, Dart hỗ trợ **Collection operators** trực tiếp ngay bên trong danh sách con của Widget:

### 1.1. Collection `if` (Hiển thị widget có điều kiện)
```dart
Column(
  children: [
    const HeaderWidget(),
    // Chỉ render banner nếu người dùng chưa xác thực email
    if (!user.isEmailVerified)
      const WarningBanner(text: 'Vui lòng xác thực email!'),
    
    // if-else ngay trong mảng
    if (user.isVip)
      const VipBadge()
    else
      const StandardBadge(),
  ],
)
```

### 1.2. Collection `for` (Lặp qua danh sách để sinh widget)
```dart
Row(
  children: [
    // Lặp trực tiếp không cần map() hay toList()
    for (final tag in post.tags)
      Chip(label: Text('#$tag')),
  ],
)
```

### 1.3. Spread Operator (`...`) & Null-aware Spread (`...?`)
Dùng để "trải" các phần tử của một danh sách khác vào danh sách hiện tại:

```dart
List<Widget> buildActionButtons() {
  return [
    ElevatedButton(onPressed: () {}, child: const Text('Lưu')),
    OutlinedButton(onPressed: () {}, child: const Text('Hủy')),
  ];
}

List<Widget>? optionalExtraActions;

// Trải vào danh sách con của AppBar:
AppBar(
  actions: [
    // 1. Trải danh sách cố định
    ...buildActionButtons(),
    
    // 2. Null-aware spread: Nếu danh sách là null, tự động bỏ qua không gây crash!
    ...?optionalExtraActions,
    
    const IconButton(icon: Icon(Icons.more_vert), onPressed: null),
  ],
)
```

---

## 2. Các Phương Thức Hàm Học (Functional Methods) Thiết Yếu

Dart `Iterable` cung cấp các phương thức duyệt mảng dạng **Lazy Evaluation** (lười biếng - chỉ tính toán từng phần tử khi thực sự có người lặp qua nó).

### 2.1. `map()` & `where()`: Biến Đổi & Lọc Dữ Liệu
```dart
final List<Product> products = fetchProducts();

// Lọc sản phẩm còn hàng và biến đổi thành Widget Card:
final inStockCards = products
    .where((p) => p.quantity > 0)
    .map((p) => ProductCard(key: ValueKey(p.id), product: p))
    .toList(); // Ép thực thi lazy stream thành List cụ thể
```

### 2.2. `fold()` vs `reduce()`: Tính Toán Giá Trị Tổng Hợp
```dart
final List<CartItem> cart = [
  CartItem(name: 'Áo thun', price: 150000, quantity: 2),
  CartItem(name: 'Quần jean', price: 350000, quantity: 1),
];

// fold nhận giá trị khởi tạo (initialValue = 0.0)
final double totalPrice = cart.fold(0.0, (sum, item) => sum + (item.price * item.quantity));
```

### 2.3. `expand()`: Trải Phẳng Danh Sách Lồng Nhau (Flattening)
Khi bạn có một danh mục chứa danh sách các thẻ bài viết, và muốn rút trích ra tất cả các thẻ tag duy nhất thành một danh sách đơn:

```dart
final List<Category> categories = fetchCategories();

// Mỗi category có List<String> tags. expand() sẽ gộp tất cả thành 1 Iterable<String> phẳng:
final List<String> allTags = categories
    .expand((category) => category.tags)
    .toSet() // Loại bỏ trùng lặp
    .toList();
```

---

## 3. Bảo Vệ Dữ Liệu State Với `List.unmodifiable()`

Khi lưu trữ một danh sách trong BLoC State hoặc ChangeNotifier, nếu trả về trực tiếp `List<Item>`, code bên ngoài có thể gọi `state.items.clear()` làm biến đổi ngầm dữ liệu mà không thông qua cơ chế phát sự kiện!

```dart
class CartState {
  final List<CartItem> _items;

  CartState(List<CartItem> items) 
      : _items = List.unmodifiable(items); // ✅ Khóa danh sách thành bất biến!

  // Bên ngoài chỉ được đọc, nếu gọi .add() hoặc .remove() sẽ ném UnsupportedError!
  List<CartItem> get items => _items;
}
```

---

## 🎯 4. Thẩm Định Năng Lực & Phân Tích Chuyên Sâu (Technical Competency & Deep Dive)

### Câu hỏi 1: Sự khác biệt bản chất về cơ chế thực thi và hiệu năng giữa `[for (var x in list) Widget(x)]` và `list.map((x) => Widget(x)).toList()`?
**Trả lời chuẩn 10/10**:
- **Cơ chế thực thi**:
  - `[for (var x in list) Widget(x)]` (Collection `for`) là một cú pháp nội tại của trình biên dịch Dart (Compiler syntax sugar). Nó tạo trực tiếp mảng đích và đẩy từng phần tử vào thông qua một vòng lặp đơn, **không sinh thêm bất kỳ đối tượng trung gian nào**.
  - `list.map(...).toList()`: Đầu tiên phương thức `.map()` tạo ra một đối tượng trung gian `MappedIterable` (Lazy generator). Sau đó, lệnh `.toList()` duyệt qua đối tượng này để phân bổ một mảng mới.
- **Về mặt hiệu năng**:
  - Đối với các mảng nhỏ trong Widget tree (dưới 20 phần tử), Collection `for` nhanh hơn một chút và tốn ít bộ nhớ RAM hơn do loại bỏ đối tượng `Iterable` trung gian.
  - Về tính dễ đọc (Readability), Google khuyên dùng Collection `for` khi lồng ghép trực tiếp bên trong danh sách con của Widget tree, và chỉ dùng `.map()` khi cần xâu chuỗi (chaining) nhiều thao tác như `.where().map().take()`.

---

### Câu hỏi 2: Tại sao `Iterable` trong Dart lại hoạt động theo cơ chế Lazy Evaluation? Cạm bẫy thực tế của nó là gì?
**Trả lời chuẩn 10/10**:
- **Bản chất**: `Iterable` không lưu trữ các phần tử trong bộ nhớ RAM như `List`. Khi bạn gọi `var mapped = items.map((x) => heavyComputation(x));`, hàm `heavyComputation` **hoàn toàn chưa chạy một lần nào**! Phép tính chỉ thực sự được kích hoạt khi có ai đó lặp qua phần tử đầu tiên bằng `for-in` hoặc gọi `.toList()`.
- **Lợi ích**: Tối ưu hóa hiệu năng vượt trội. Nếu bạn viết `items.map(...).first`, hàm tính toán chỉ chạy duy nhất 1 lần cho phần tử đầu tiên rồi dừng lại, không tính toán lãng phí 999 phần tử còn lại.
- **Cạm bẫy nguy hiểm**: Nếu `heavyComputation` có tác dụng phụ (side-effects) hoặc bạn lặp qua `mapped` 3 lần ở 3 nơi khác nhau, **hàm tính toán sẽ bị chạy lại đúng 3 lần từ đầu**! Để tránh điều này, hãy gọi `.toList()` một lần duy nhất để lưu kết quả vào bộ nhớ (Eager caching).

---

### Câu hỏi 3: Làm thế nào để so sánh 2 List có cùng giá trị trong Dart? Tại sao `[1, 2] == [1, 2]` lại trả về `false`?
**Trả lời chuẩn 10/10**:
- **Lý do trả về `false`**: Trong Dart, toán tử `==` mặc định của `List` kiểm tra **sự đồng nhất về ô nhớ tham chiếu (Identity Equality)** thông qua hàm `identical(listA, listB)`. Vì hai mảng được cấp phát tại hai địa chỉ ô nhớ khác nhau trên Heap, phép so sánh luôn là `false`.
- **Cách so sánh theo giá trị (Structural / Value Equality)**:
  1. Sử dụng hàm chuẩn của thư viện nền tảng `package:collection`:
     ```dart
     import 'package:collection/collection.dart';
     final areEqual = const ListEquality().equals(listA, listB);
     ```
  2. Hoặc dùng `const`: Nếu cả hai danh sách đều là hằng số biên dịch `const [1, 2] == const [1, 2]`, Dart áp dụng Canonicalization và trả về `true` vì cùng trỏ vào một ô nhớ duy nhất!
