# Bài 5.3 — InheritedNotifier & ChangeNotifier Architecture

## Phần 1 — Khái Niệm & Ràng Buộc Kiến Trúc (Architecture & Design Philosophy)

### 1.1 — Mô hình Observer Pattern trong Flutter Framework

Trong khi `InheritedWidget` giải quyết bài toán phổ biến dữ liệu theo chiều dọc trong cây giao diện, việc cập nhật dữ liệu của nó mặc định vẫn phải dựa vào vòng lặp Rebuild của một `StatefulWidget` cha bao bọc ở ngoài. Điều này tạo ra sự cồng kềnh khi lập trình viên phải kết hợp đồng thời 3 lớp: `StatefulWidget` + `State` + `InheritedWidget`.

Để tinh gọn kiến trúc và hỗ trợ mô hình hướng sự kiện (Event-driven Reactive State), Flutter cung cấp cơ chế kết hợp giữa **Observer Pattern** và **Ambient Property Pattern** thông qua:
1. **`ChangeNotifier`:** Một lớp triển khai giao diện `Listenable`, cho phép các đối tượng nghiệp vụ (Business Models) chủ động phát thông báo (`notifyListeners()`) khi trạng thái nội bộ thay đổi.
2. **`InheritedNotifier<T extends Listenable>`:** Một lớp con đặc thù của `InheritedWidget`. Nó tự động đăng ký lắng nghe đối tượng `Listenable`, và mỗi khi đối tượng này phát thông báo, `InheritedNotifier` sẽ tự động kích hoạt chu trình đánh dấu bẩn và thông báo cho toàn bộ các subscriber con trên Element Tree.

```
┌────────────────────────────────────────────────────────┐
│ MODEL NGHIỆP VỤ (ChangeNotifier / ValueNotifier)      │
│   • Quản lý logic và dữ liệu biến đổi                  │
│   • Kích hoạt: notifyListeners() khi có mutation       │
└───────────────────────────┬────────────────────────────┘
                            │ Lắng nghe tín hiệu
                            ▼
┌────────────────────────────────────────────────────────┐
│ INHERITED NOTIFIER (Element Bridge)                   │
│   • Tự động đăng ký làm listener của Model             │
│   • Kích hoạt: notifyClients() xuống Element Tree       │
└───────────────────────────┬────────────────────────────┘
                            │ Phổ biến dữ liệu O(1)
                            ▼
┌────────────────────────────────────────────────────────┐
│ CONSUMER WIDGETS (Subscribed via context)              │
│   • Chỉ các widget thực sự đọc dữ liệu mới bị Rebuild  │
└────────────────────────────────────────────────────────┘
```

---

### 1.2 — Phân cấp họ Listenable trong Flutter SDK

```
                    ┌───────────────────────────┐
                    │    Listenable (Abstract)   │
                    │  addListener / removeListener│
                    └─────────────┬─────────────┘
                                  │
                   ┌──────────────┴──────────────┐
                   ▼                             ▼
       ┌──────────────────────┐      ┌──────────────────────┐
       │    ChangeNotifier    │      │    ValueListenable<T>│
       │  (Thủ công phát tín  │      │  (Interface mang giá │
       │   hiệu notify)       │      │   trị kiểu T)        │
       └──────────┬───────────┘      └──────────┬───────────┘
                  │                             │
                  └──────────────┬──────────────┘
                                 ▼
                     ┌──────────────────────┐
                     │   ValueNotifier<T>   │
                     │ (Tự động phát tín hiệu│
                     │  khi value thay đổi) │
                     └──────────────────────┘
```

- **`ChangeNotifier`:** Thích hợp cho các mô hình trạng thái phức tạp (nhiều trường dữ liệu, nhiều phương thức nghiệp vụ). Đòi hỏi lập trình viên phải gọi hàm `notifyListeners()` thủ công sau mỗi biến đổi dữ liệu.
- **`ValueNotifier<T>`:** Là lớp con của `ChangeNotifier` hiện thực hóa `ValueListenable<T>`. Nó đóng gói một biến đơn lẻ kiểu `T`. Mỗi khi setter `value = newValue` được gọi, nó tự động so sánh giá trị cũ và mới bằng toán tử `operator==`, nếu có sự khác biệt sẽ tự động kích hoạt `notifyListeners()`.

---

## Phần 2 — Cơ Chế Hoạt Động & Mã Nguồn Đối Chiếu (Under the Hood / Deep-Dive)

### 2.1 — Mổ xẻ mã nguồn `ChangeNotifier` và Giải thuật Concurrent Modification

Một trong những thách thức lớn nhất của mẫu thiết kế Observer là hiện tượng **Sửa đổi đồng thời (Concurrent Modification)**: Trong lúc danh sách listeners đang được duyệt để phát thông báo, một listener nào đó lại thực hiện gọi `removeListener()` hoặc `addListener()`.

Bên trong `packages/flutter/lib/src/foundation/change_notifier.dart`, Flutter giải quyết bài toán này mà không cần deep-clone danh sách qua mỗi lần thông báo:

```dart
class ChangeNotifier implements Listenable {
  int _count = 0;
  // Sử dụng mảng động có kích thước co giãn theo lũy thừa của 2
  List<VoidCallback?>? _listeners = _emptyListeners;
  int _notificationCallStackDepth = 0;

  @override
  void addListener(VoidCallback listener) {
    // ... Cấp phát mảng hoặc mở rộng dung lượng
    _listeners![_count++] = listener;
  }

  @override
  void removeListener(VoidCallback listener) {
    for (int i = 0; i < _count; i++) {
      final VoidCallback? listenerAtIndex = _listeners![i];
      if (listenerAtIndex == listener) {
        if (_notificationCallStackDepth > 0) {
          // BẢO VỆ CONCURRENT: Không xóa ngay làm dịch chuyển chỉ mục mảng!
          // Thay vào đó, đánh dấu bằng con trỏ null (Tombstone)
          _listeners![i] = null;
        } else {
          // Nếu không nằm trong vòng lặp notify, dồn mảng ngay lập tức
          _count--;
          if (i < _count) {
            _listeners![i] = _listeners![_count];
          }
          _listeners![_count] = null;
        }
        break;
      }
    }
  }

  @protected
  @visibleForTesting
  void notifyListeners() {
    assert(_debugAssertNotDisposed());
    if (_count == 0) {
      return;
    }

    _notificationCallStackDepth++;
    final int end = _count;
    for (int i = 0; i < end; i++) {
      try {
        // Bỏ qua các listener đã bị đánh dấu null trong lúc duyệt
        _listeners![i]?.call();
      } catch (exception, stack) {
        FlutterError.reportError(...);
      }
    }
    _notificationCallStackDepth--;

    // Khi thoát khỏi toàn bộ các tầng đệ quy notify, tiến hành nén dọn nulls
    if (_notificationCallStackDepth == 0 && _reifiedRemovals > 0) {
      _compact();
    }
  }
}
```

#### Ý nghĩa kiến trúc:
Kỹ thuật **Tombstone (`null` placeholder)** giúp việc triệu hồi `notifyListeners()` đạt hiệu năng cực cao: Duyệt mảng tuần tự liên tục trên bộ nhớ đệm CPU (Cache Locality), hoàn toàn không cần cấp phát thêm bộ nhớ tạm trong mỗi khung hình.

---

### 2.2 — Mổ xẻ mã nguồn `InheritedNotifier` và `InheritedNotifierElement`

Phương thức liên kết giữa thế giới Listenable và Element Tree được hiện thực hóa trong `packages/flutter/lib/src/widgets/inherited_notifier.dart`:

```dart
class InheritedNotifier<T extends Listenable> extends InheritedWidget {
  const InheritedNotifier({
    super.key,
    this.notifier,
    required super.child,
  });

  final T? notifier;

  @override
  bool updateShouldNotify(InheritedNotifier<T> oldWidget) {
    // Luôn trả về true nếu có notifier để kích hoạt việc kiểm tra các con
    return notifier != null;
  }

  @override
  InheritedElement createElement() => InheritedNotifierElement<T>(this);
}

class InheritedNotifierElement<T extends Listenable> extends InheritedElement {
  InheritedNotifierElement(InheritedNotifier<T> super.widget);

  @override
  void mount(Element? parent, Object? newSlot) {
    super.mount(parent, newSlot);
    // Tự động đăng ký lắng nghe ngay khi mount vào cây
    (widget as InheritedNotifier<T>).notifier?.addListener(_handleUpdate);
  }

  @override
  void update(InheritedNotifier<T> newWidget) {
    final T? oldNotifier = (widget as InheritedNotifier<T>).notifier;
    final T? newNotifier = newWidget.notifier;
    if (oldNotifier != newNotifier) {
      oldNotifier?.removeListener(_handleUpdate);
      newNotifier?.addListener(_handleUpdate);
    }
    super.update(newWidget);
  }

  void _handleUpdate() {
    // Khi notifier phát tín hiệu, Element tự đánh dấu bẩn và thông báo xuống con
    markNeedsBuild();
  }

  @override
  void unmount() {
    (widget as InheritedNotifier<T>).notifier?.removeListener(_handleUpdate);
    super.unmount();
  }
}
```

---

### 2.3 — So sánh 3 giải pháp tiêu thụ Listenable ở tầng cục bộ

| Tiêu chí | `ListenableBuilder` (Flutter 3.7+) | `AnimatedBuilder` | `ValueListenableBuilder<T>` |
| :--- | :--- | :--- | :--- |
| **Kiểu dữ liệu hỗ trợ** | Bất kỳ đối tượng nào hiện thực `Listenable`. | Bất kỳ đối tượng nào hiện thực `Listenable`. | Chỉ chấp nhận `ValueListenable<T>`. |
| **Chữ ký hàm Builder** | `(BuildContext, Widget? child)` | `(BuildContext, Widget? child)` | `(BuildContext, T value, Widget? child)` |
| **Bản chất mã nguồn** | Là một `StatefulWidget` tự động quản lý `addListener` và `removeListener`. | Thực chất `AnimatedBuilder` là một subclass kế thừa hoặc cấu hình tương đương `ListenableBuilder`. | Tự động giải nén giá trị `.value` và truyền trực tiếp vào tham số thứ hai của builder. |
| **Tối ưu hóa Subtree** | Hỗ trợ tham số `child` tĩnh để chống rebuild thừa. | Hỗ trợ tham số `child` tĩnh để chống rebuild thừa. | Hỗ trợ tham số `child` tĩnh để chống rebuild thừa. |

---

## Phần 3 — Hướng Dẫn Thực Hành Chuẩn (Production-Ready Implementations)

### 3.1 — Kiến trúc Quản lý Giỏ hàng với `ChangeNotifier` và `InheritedNotifier`

Dưới đây là một hệ thống quản lý giỏ hàng thương mại điện tử hoàn chỉnh, thể hiện sự kết hợp chuẩn mực giữa Business Model và Element Tree:

```dart
import 'package:flutter/material.dart';

/// 1. Business Logic Model độc lập với UI
class CartItem {
  final String id;
  final String title;
  final double price;

  const CartItem({required this.id, required this.title, required this.price});
}

class CartModel extends ChangeNotifier {
  final List<CartItem> _items = [];

  List<CartItem> get items => List.unmodifiable(_items);

  double get totalPrice => _items.fold(0.0, (sum, item) => sum + item.price);

  int get totalCount => _items.length;

  void addItem(CartItem item) {
    _items.add(item);
    notifyListeners(); // Thông báo cho UI cập nhật
  }

  void removeItem(String id) {
    _items.removeWhere((item) => item.id == id);
    notifyListeners();
  }

  void clear() {
    _items.clear();
    notifyListeners();
  }
}

/// 2. InheritedNotifier đóng vai trò cầu nối Element Tree
class CartScope extends InheritedNotifier<CartModel> {
  const CartScope({
    super.key,
    required CartModel cartModel,
    required super.child,
  }) : super(notifier: cartModel);

  /// Đọc dữ liệu và ĐĂNG KÝ phụ thuộc (Rebuild khi giỏ hàng thay đổi)
  static CartModel of(BuildContext context) {
    final CartScope? scope =
        context.dependOnInheritedWidgetOfExactType<CartScope>();
    assert(scope != null, 'Không tìm thấy CartScope trong cây tổ tiên');
    return scope!.notifier!;
  }

  /// Đọc dữ liệu MỘT LẦN (Không đăng ký phụ thuộc - Dùng cho nút bấm)
  static CartModel read(BuildContext context) {
    final CartScope? scope =
        context.getInheritedWidgetOfExactType<CartScope>();
    assert(scope != null, 'Không tìm thấy CartScope trong cây tổ tiên');
    return scope!.notifier!;
  }
}
```

#### 3. Tích hợp và Tối ưu hóa Rebuild trên giao diện:
```dart
class CartAppRoot extends StatefulWidget {
  const CartAppRoot({super.key});

  @override
  State<CartAppRoot> createState() => _CartAppRootState();
}

class _CartAppRootState extends State<CartAppRoot> {
  final CartModel _cartModel = CartModel();

  @override
  void dispose() {
    _cartModel.dispose(); // Bắt buộc giải phóng tài nguyên
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return CartScope(
      cartModel: _cartModel,
      child: const MaterialApp(
        home: CatalogScreen(),
      ),
    );
  }
}

// Widget này CHỈ rebuild số lượng badge, không làm rebuild toàn bộ AppBar
class CartBadgeIcon extends StatelessWidget {
  const CartBadgeIcon({super.key});

  @override
  Widget build(BuildContext context) {
    final cart = CartScope.of(context); // Đăng ký lắng nghe
    return Badge(
      label: Text('${cart.totalCount}'),
      child: const Icon(Icons.shopping_cart),
    );
  }
}

// Nút bấm thêm sản phẩm: Dùng CartScope.read() để KHÔNG BỊ REBUILD khi giỏ hàng đổi
class AddToCartButton extends StatelessWidget {
  final CartItem item;

  const AddToCartButton({super.key, required this.item});

  @override
  Widget build(BuildContext context) {
    debugPrint('Build AddToCartButton: ${item.id}');
    return ElevatedButton(
      onPressed: () {
        // Đọc một lần duy nhất trong callback sự kiện
        CartScope.read(context).addItem(item);
      },
      child: const Text('Thêm vào giỏ'),
    );
  }
}
```

---

### 3.2 — Tối ưu hóa với `ValueNotifier` và `ListenableBuilder` cho trạng thái cục bộ

Đối với các biến trạng thái đơn lẻ (như bộ đếm, trạng thái đóng mở menu), sử dụng `ValueNotifier` kết hợp `ListenableBuilder` giúp loại bỏ hoàn toàn việc gọi `setState()` ở widget cha:

```dart
class SearchFilterWidget extends StatefulWidget {
  const SearchFilterWidget({super.key});

  @override
  State<SearchFilterWidget> createState() => _SearchFilterWidgetState();
}

class _SearchFilterWidgetState extends State<SearchFilterWidget> {
  final ValueNotifier<bool> _isFilterExpanded = ValueNotifier<bool>(false);

  @override
  void dispose() {
    _isFilterExpanded.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Nút toggle: Chỉ thay đổi giá trị của notifier
        ElevatedButton(
          onPressed: () => _isFilterExpanded.value = !_isFilterExpanded.value,
          child: const Text('Bộ lọc nâng cao'),
        ),
        
        // Chỉ duy nhất khối này được rebuild khi _isFilterExpanded thay đổi
        ListenableBuilder(
          listenable: _isFilterExpanded,
          builder: (context, child) {
            if (!_isFilterExpanded.value) return const SizedBox.shrink();
            return child!;
          },
          child: const Padding(
            padding: EdgeInsets.all(8.0),
            child: Text('Nội dung các tiêu chí lọc chi tiết...'),
          ),
        ),
      ],
    );
  }
}
```

---

## Phần 4 — Lỗi Thường Gặp & Giải Pháp Khắc Phục (Anti-Patterns & Pitfalls)

### 4.1 — Đột biến đối tượng trong `ValueNotifier` mà không gán lại tham chiếu mới

#### Mô tả lỗi:
Lập trình viên sử dụng `ValueNotifier<List<T>>` hoặc `ValueNotifier<Map<K, V>>`, thực hiện thêm phần tử trực tiếp vào mảng nhưng giao diện không hề cập nhật:

```dart
// SAI LẦM: Không kích hoạt rebuild
final ValueNotifier<List<String>> tagsNotifier = ValueNotifier<List<String>>(['Flutter']);

void addNewTag(String tag) {
  tagsNotifier.value.add(tag); // Đột biến nội dung mảng trực tiếp
  // Setter của ValueNotifier kiểm tra: if (_value == newValue) return;
  // Vì tagsNotifier.value vẫn trỏ vào cùng một đối tượng mảng trong bộ nhớ,
  // phép so sánh _value == newValue trả về true -> notifyListeners() BỊ HỦY BỎ!
}
```

#### Giải pháp:
Luôn gán một tham chiếu danh sách mới (Immutable pattern):
```dart
void addNewTag(String tag) {
  tagsNotifier.value = [...tagsNotifier.value, tag]; // Tạo danh sách mới
}
```

---

### 4.2 — Rò rỉ bộ nhớ do không giải phóng `ChangeNotifier`

#### Mô tả lỗi:
Khởi tạo `ChangeNotifier` hoặc `ValueNotifier` trong `StatefulWidget` nhưng quên triệu hồi `dispose()` trong phương thức `dispose()` của State.

#### Hậu quả kỹ thuật:
Các listener đã đăng ký (hoặc chính instance notifier) sẽ tiếp tục bị giữ lại trên bộ nhớ Heap bởi các tham chiếu tĩnh hoặc stream listener, gây thất thoát RAM nghiêm trọng. Đồng thời, nếu một tác vụ bất đồng bộ sau đó kích hoạt `notifyListeners()`, framework sẽ ném ngoại lệ:
```
A ChangeNotifier was used after being disposed.
```

---

## Phần 5 — Câu Hỏi Kiểm Tra Kiến Thức Chuyên Sâu & Bài Tập Phân Tích Mã Nguồn (Technical Assessment & Code Tracing)

### 5.1 — Câu hỏi khảo sát kiến trúc

#### Câu 1: Cơ chế Tombstone trong `ChangeNotifier`
*Đề bài:* Tại sao Flutter không sử dụng cấu trúc dữ liệu `Set<VoidCallback>` hoặc `LinkedList` để lưu trữ listeners trong `ChangeNotifier` mà lại sử dụng mảng động `List<VoidCallback?>` kết hợp với con trỏ `null` tombstone?

*Phân tích kỹ thuật:*
1. **Hiệu năng lặp (Iteration Performance):** Trong mỗi khung hình kết xuất, thao tác `notifyListeners()` diễn ra liên tục hàng nghìn lần. Duyệt một mảng phẳng (Contiguous Array) tận dụng tối đa kiến trúc CPU Cache Line, nhanh hơn nhiều lần so với việc duyệt con trỏ phân tán của `LinkedList` hoặc băm của `HashSet`.
2. **Bảo toàn tính toàn vẹn chỉ mục khi xóa:** Nếu xóa trực tiếp phần tử khỏi mảng trong lúc vòng lặp `for (int i = 0; i < count; i++)` đang chạy, các phần tử phía sau sẽ bị dịch chuyển chỉ mục sang trái ($i \gets i - 1$), dẫn đến việc một listener có thể bị bỏ qua hoặc duyệt hai lần. Bằng cách gán `_listeners[i] = null`, chỉ mục giữ nguyên tuyệt đối cho đến khi kết thúc toàn bộ chu kỳ duyệt mới dọn dẹp một lần duy nhất.

---

#### Câu 2: Sự khác biệt bản chất về cơ chế kích hoạt Rebuild giữa `InheritedWidget` và `InheritedNotifier`
*Đề bài:* Phân tích sự khác biệt về nguồn gốc phát động chu trình Rebuild giữa `InheritedWidget` truyền thống và `InheritedNotifier`.

*Phân tích kỹ thuật:*
1. **`InheritedWidget` truyền thống (Top-Down Pull):** Hoàn toàn bị động. Nó chỉ có thể phát tín hiệu rebuild cho các con khi **chính widget cha của nó bị rebuild** (thường do `setState` ở StatefulWidget tổ tiên).
2. **`InheritedNotifier` (Active Push-to-Pull):** Mang tính chủ động độc lập. Khi một sự kiện nghiệp vụ xảy ra ở bất kỳ đâu, chỉ cần gọi `notifier.notifyListeners()`, `InheritedNotifierElement` sẽ lập tức nhận được callback và tự đưa chính nó vào hàng đợi `_dirtyElements`. Quá trình này kích hoạt việc thông báo xuống các con **mà hoàn toàn không cần widget cha của `InheritedNotifier` phải rebuild**.

---

### 5.2 — Bài tập phân tích luồng thực thi (Code Tracing)

#### Đề bài:
Cho cấu trúc chương trình sau:

```dart
final ValueNotifier<int> counterNotifier = ValueNotifier<int>(0);

class TracingRootScreen extends StatelessWidget {
  const TracingRootScreen({super.key});

  @override
  Widget build(BuildContext context) {
    debugPrint('1. Build TracingRootScreen');
    return Scaffold(
      body: Column(
        children: [
          const StaticHeaderWidget(), // (Node 1)
          ValueListenableBuilder<int>( // (Node 2)
            valueListenable: counterNotifier,
            builder: (context, value, child) {
              debugPrint('2. Build ValueListenableBuilder: $value');
              return Row(
                children: [
                  Text('Counter: $value'),
                  child!, // Reusable static child
                ],
              );
            },
            child: const SubStaticIconWidget(), // (Node 3)
          ),
        ],
      ),
    );
  }
}

class StaticHeaderWidget extends StatelessWidget {
  const StaticHeaderWidget({super.key});
  @override
  Widget build(BuildContext context) {
    debugPrint('3. Build StaticHeaderWidget');
    return const Text('Header');
  }
}

class SubStaticIconWidget extends StatelessWidget {
  const SubStaticIconWidget({super.key});
  @override
  Widget build(BuildContext context) {
    debugPrint('4. Build SubStaticIconWidget');
    return const Icon(Icons.star);
  }
}
```

Giả sử ứng dụng đã hoàn tất lượt render đầu tiên. Sau đó, lập trình viên thực hiện liên tiếp 2 thao tác:
- **Thao tác 1:** `counterNotifier.value = 0;` (Gán lại chính xác giá trị cũ).
- **Thao tác 2:** `counterNotifier.value = 5;` (Gán giá trị mới).

Hãy xác định chính xác:
1. Những thông báo log nào xuất hiện sau Thao tác 1?
2. Những thông báo log nào xuất hiện sau Thao tác 2?
3. `SubStaticIconWidget` có bị rebuild lại ở Thao tác 2 không? Giải thích tại sao.

---

#### Đáp án phân tích:

**1. Kết quả sau Thao tác 1 (`counterNotifier.value = 0`):**
- **Không có bất kỳ log nào được in ra**.
- *Giải thích:* Setter của `ValueNotifier` kiểm tra:
  ```dart
  if (_value == newValue) return;
  ```
  Vì `0 == 0` trả về `true`, phương thức ngắt sớm và không gọi `notifyListeners()`.

**2. Kết quả sau Thao tác 2 (`counterNotifier.value = 5`):**
- Console in ra duy nhất một dòng log:
  ```
  2. Build ValueListenableBuilder: 5
  ```

**3. Phân tích trạng thái của `SubStaticIconWidget`:**
- `SubStaticIconWidget` **hoàn toàn KHÔNG bị rebuild** (không in dòng log số 4).
- *Giải thích:* `SubStaticIconWidget` được khởi tạo và truyền vào tham số `child` tĩnh của `ValueListenableBuilder`. 
- Khi `counterNotifier` phát tín hiệu, chỉ có hàm callback `builder(context, value, child)` được thực thi lại. Khối widget `child` được truyền nguyên vẹn vào cây con mới mà không trải qua quá trình khởi tạo hay gọi hàm `build()` mới, giúp tiết kiệm tối đa chi phí kết xuất cho các thành phần tĩnh phức tạp.
- Cả `TracingRootScreen` và `StaticHeaderWidget` cũng không bị ảnh hưởng.
