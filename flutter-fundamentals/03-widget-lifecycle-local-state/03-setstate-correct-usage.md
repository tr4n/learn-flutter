# Bài 3.3 — Cơ Chế setState & Quá Trình Rebuild (setState & Rebuild Pipeline)

## Phần 1 — Khái Niệm & Cơ Chế Đánh Dấu Không Hợp Lệ (Concepts & Invalidation Model)

### 1.1 — Mô hình Đánh dấu không hợp lệ (Reactive Invalidation / Push-to-Pull)

Trong Flutter Framework, `setState()` không phải là một lệnh vẽ trực tiếp (Eager Immediate Rendering). Khi `setState()` được gọi, framework không tính toán lại layout hay vẽ lại pixel lên màn hình ngay tại thời điểm đó. Thay vào đó, nó hoạt động theo **Mô hình Đánh dấu không hợp lệ (Reactive Invalidation)** chia thành hai giai đoạn độc lập:

1. **Pha thông báo không hợp lệ (Push / Invalidation Phase):** 
   - Lập trình viên cập nhật các biến trạng thái nội bộ bên trong callback `fn()`.
   - Đối tượng `StatefulElement` tương ứng được đánh dấu là bẩn (`_dirty = true`) và được đăng ký vào danh sách chờ xử lý của `BuildOwner`.
   - `SchedulerBinding` gửi yêu cầu lên hệ điều hành để lên lịch nhận một tín hiệu xung nhịp quét màn hình mới (VSync).
2. **Pha tái dựng khung hình (Pull / Frame Reconstruction Phase):**
   - Khi tín hiệu VSync từ phần cứng xuất hiện, engine kích hoạt chu kỳ render mới.
   - `BuildOwner.buildScope()` duyệt qua danh sách các element bị đánh dấu bẩn, thực thi lại phương thức `build()` trên các đối tượng `State` liên quan để tạo ra cây Widget con mới.
   - Cây Widget mới được chuyển tiếp sang các giai đoạn Layout và Paint trong rendering pipeline.

```
Lập trình viên gọi setState()
       │
       ▼ [Thực thi đồng bộ]
Cập nhật biến trạng thái ──► Element.markNeedsBuild() ──► Đưa vào danh sách _dirtyElements
                                                                    │
                                                      Chờ tín hiệu VSync phần cứng
                                                                    │
                                                                    ▼
PipelineOwner (Layout & Paint) ◄── BuildOwner.buildScope() ◄── Tín hiệu VSync từ Engine
```

---

### 1.2 — Trách nhiệm kỹ thuật và phạm vi ảnh hưởng của setState

- **Phạm vi cục bộ:** `setState()` chỉ đánh dấu dirty chính đối tượng `StatefulElement` đang chứa nó. Quá trình rebuild sẽ lan truyền từ node đó đi xuống tất cả các widget con bên dưới (subtree).
- **Chi phí lan truyền:** Nếu `setState()` được đặt ở một node quá cao trong cây phân cấp (ví dụ: ở cấp màn hình chính `Scaffold`), toàn bộ các widget con bên dưới dù không thay đổi dữ liệu cũng sẽ bị duyệt lại qua hàm `build()`.
- **Ranh giới cô lập:** Việc phân tách các widget có trạng thái biến đổi thành các lớp `StatefulWidget` nhỏ độc lập ("Push State Down") là giải pháp kỹ thuật bắt buộc để thu hẹp phạm vi rebuild.

---

## Phần 2 — Cơ Chế Hoạt Động & Mã Nguồn Đối Chiếu (Framework Internals)

### 2.1 — Giải phẫu mã nguồn `State.setState()` trong `framework.dart`

```dart
// Source code: packages/flutter/lib/src/widgets/framework.dart
@protected
void setState(VoidCallback fn) {
  // 1. Kiểm tra trạng thái tồn tại của State
  assert(mounted, 'setState() called after dispose(): $this');
  assert(_debugLifecycleState != _StateLifecycle.defunct,
      'setState() called after dispose(): $this');

  // 2. Chặn việc gọi setState trong khi pipeline đang thực thi build
  assert(!_debugBuilding,
      'setState() or markNeedsBuild() called during build.');

  // 3. Thực thi callback cập nhật trạng thái đồng bộ
  final Object? result = fn() as dynamic;

  // 4. Ràng buộc callback không được là hàm bất đồng bộ (async)
  assert(() {
    if (result is Future) {
      throw FlutterError(
        'Callbacks provided to setState() generally shouldn\'t return a Future.\n'
        'Maybe an async function was used where a synchronous callback was expected?'
      );
    }
    return true;
  }());

  // 5. Chuyển giao việc đánh dấu dirty cho Element
  _element!.markNeedsBuild();
}
```

#### Phân tích các ràng buộc assertion:

1. **`assert(!_debugBuilding)`**: Cờ `_debugBuilding` mang giá trị `true` trong suốt thời gian `BuildOwner.buildScope()` đang duyệt qua cây. Nếu một widget gọi `setState()` trực tiếp bên trong phương thức `build()`, assertion này sẽ kích hoạt ngay lập tức để ngăn chặn vòng lặp vô hạn (Infinite Build Loop: Build -> SetState -> Build...).
2. **`assert(result is! Future)`**: Tại sao callback của `setState()` không được phép là `async`?
   - `setState()` yêu cầu quá trình đột biến dữ liệu phải hoàn thành **đồng bộ** trước khi `_element!.markNeedsBuild()` được gọi.
   - Nếu truyền vào một `async` function, thân hàm sẽ trả về một `Future` và việc gán dữ liệu thực sự sẽ bị hoãn lại vào Microtask Queue hoặc Event Queue trong tương lai. Khung hình VSync kế tiếp sẽ render với dữ liệu cũ chưa kịp cập nhật, và khi dữ liệu mới được gán sau đó, framework lại không nhận được thông báo để vẽ lại.

---

### 2.2 — Cơ chế hoạt động của `Element.markNeedsBuild()` và `BuildOwner`

Khi `_element!.markNeedsBuild()` được kích hoạt:

```dart
void markNeedsBuild() {
  assert(_lifecycleState != _ElementLifecycle.defunct);
  if (!_active) return;
  
  // Kiểm tra cờ dirty: Nếu đã bẩn, bỏ qua không làm gì thêm
  if (dirty) return;
  
  _dirty = true;
  owner!.scheduleBuildFor(this);
}
```

#### Thuật toán sắp xếp theo độ sâu cây (`depth` order) trong `BuildOwner`:
Bên trong `BuildOwner`, các element bẩn được tập hợp trong danh sách `_dirtyElements`. Trước khi bắt đầu duyệt để gọi `build()`, framework thực hiện sắp xếp danh sách này:

```dart
_dirtyElements.sort((Element a, Element b) => a.depth - b.depth);
```

**Nguyên nhân kỹ thuật của giải thuật sắp xếp:**
Trong cây phân cấp Element Tree, thuộc tính `depth` của node cha luôn nhỏ hơn thuộc tính `depth` của node con.
- Nếu cả node cha và node con đều bị đánh dấu dirty trong cùng một khung hình: Node cha (`depth` nhỏ hơn) sẽ luôn được rebuild trước.
- Khi node cha rebuild, nó gọi `updateChild()` xuống node con, truyền vào cấu hình widget mới. Node con được cập nhật luôn trong lượt này và cờ `_dirty` của nó được xóa bỏ.
- Nếu không sắp xếp theo `depth` mà xử lý node con trước: Node con sẽ build lần 1; ngay sau đó node cha build lại ép node con phải build lần 2. Việc sắp xếp theo độ sâu triệt tiêu hoàn toàn hiện tượng duplicate rebuilds.

---

### 2.3 — Cơ chế Gom cụm khung hình (Frame Coalescing / Batching)

Xem xét chuỗi thao tác đồng bộ:

```dart
void updateMetrics() {
  setState(() => _width = 100);
  setState(() => _height = 200);
  setState(() => _color = Colors.blue);
}
```

**Kết quả thực thi:** Hàm `build()` chỉ được triệu gọi **đúng 1 lần duy nhất**.

**Cơ chế kỹ thuật:**
1. Ở lệnh `setState()` đầu tiên: Cờ `_dirty` chuyển từ `false` sang `true`. Element được thêm vào `_dirtyElements`, và `SchedulerBinding` gửi một yêu cầu VSync.
2. Ở các lệnh `setState()` thứ hai và thứ ba: Do cờ `_dirty` đã là `true`, điều kiện `if (dirty) return;` trong `markNeedsBuild()` trả về ngay lập tức.
3. Khi luồng JavaScript/Dart đồng bộ hoàn tất, khung hình VSync kế tiếp xuất hiện, `BuildOwner.buildScope()` mới chạy và thực thi hàm `build()` với toàn bộ các giá trị cuối cùng (`_width = 100`, `_height = 200`, `_color = Colors.blue`).

---

### 2.4 — So sánh kỹ thuật: `setState(() { a = 1; })` vs `a = 1; setState(() {})`

Về mặt kỹ thuật thuần túy trên máy ảo Dart VM, cả hai cách viết đều cập nhật biến và đánh dấu `_dirty = true`. Tuy nhiên, cú pháp chuẩn của Flutter yêu cầu bao bọc mutation trong callback:

```dart
// Mẫu chuẩn khuyến nghị
setState(() {
  _value = 10;
});

// Mẫu không khuyến nghị
_value = 10;
setState(() {});
```

#### Lý do kỹ thuật:
1. **Tính gắn kết về thời gian (Temporal Cohesion):** Khối callback xác lập ranh giới giao dịch rõ ràng: dữ liệu bên trong block này thay đổi chính là nguyên nhân làm giao diện cần cập nhật.
2. **Quản lý biệt lệ (Exception Boundary):** Nếu logic tính toán dữ liệu ném ra Exception, framework có thể khoanh vùng lỗi trong phạm vi callback thay vì để trạng thái hệ thống bị biến dạng cục bộ.
3. **Phân tích hiệu năng qua DevTools:** Công cụ Timeline Profiler của Flutter đo đạc thời gian thực thi của chính callback `fn()`, giúp xác định các thao tác tính toán nặng bị đặt nhầm trong luồng UI.

---

### 2.5 — Bảng so sánh các giải pháp quản lý trạng thái cục bộ

| Tiêu chí | `setState()` | `ValueNotifier` + `ValueListenableBuilder` | `ListenableBuilder` (Flutter 3.10+) | `StatefulBuilder` |
| :--- | :--- | :--- | :--- | :--- |
| **Phạm vi tái dựng (Rebuild Scope)** | Toàn bộ subtree của `State` | Chỉ phạm vi closure của `builder` | Chỉ phạm vi closure của `builder` | Chỉ phạm vi closure của `builder` |
| **Yêu cầu `StatefulWidget`?** | Bắt buộc | Không bắt buộc (sử dụng được trong `StatelessWidget`) | Không bắt buộc | Không bắt buộc |
| **Cơ chế tái sử dụng cây tĩnh (`child`)** | Thủ công qua biến thành viên | Hỗ trợ trực tiếp qua tham số `child` | Hỗ trợ trực tiếp qua tham số `child` | Không hỗ trợ |
| **Cơ chế giải phóng tài nguyên** | Framework tự động theo vòng đời State | Lập trình viên bắt buộc gọi `notifier.dispose()` | Lập trình viên bắt buộc gọi `listenable.dispose()` | Không yêu cầu giải phóng riêng |

---

## Phần 3 — Mẫu Triển Khai Chuẩn (Standard Implementation Patterns)

### 3.1 — Kỹ thuật cô lập phạm vi Rebuild (Pushing State Down)

```dart
import 'package:flutter/material.dart';

/// Màn hình chính là StatelessWidget tĩnh.
/// State thay đổi thường xuyên (bộ đếm) được đẩy xuống widget con CounterBadge.
class CatalogOverviewScreen extends StatelessWidget {
  const CatalogOverviewScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Danh Mục Sản Phẩm'),
        actions: const [
          // Widget con tự quản lý State nội bộ, cô lập phạm vi rebuild
          CartCounterBadge(),
        ],
      ),
      body: ListView.builder(
        itemCount: 500,
        itemBuilder: (context, index) => ListTile(title: Text('Sản phẩm #$index')),
      ),
    );
  }
}

class CartCounterBadge extends StatefulWidget {
  const CartCounterBadge({super.key});

  @override
  State<CartCounterBadge> createState() => _CartCounterBadgeState();
}

class _CartCounterBadgeState extends State<CartCounterBadge> {
  int _count = 0;

  void increment() {
    setState(() {
      _count++;
    });
  }

  @override
  Widget build(BuildContext context) {
    // Chỉ có widget CartCounterBadge thực thi lại build() khi _count thay đổi.
    // Màn hình chính CatalogOverviewScreen và ListView 500 item hoàn toàn không bị ảnh hưởng.
    return IconButton(
      icon: Badge(
        label: Text('$_count'),
        child: const Icon(Icons.shopping_cart),
      ),
      onPressed: increment,
    );
  }
}
```

---

### 3.2 — Tối ưu hóa cập nhật cục bộ với ValueNotifier và ListenableBuilder

```dart
import 'package:flutter/material.dart';

/// Sử dụng ListenableBuilder kết hợp tham số child để tối ưu hóa hiệu năng render.
class OptimizedSearchHeader extends StatefulWidget {
  const OptimizedSearchHeader({super.key});

  @override
  State<OptimizedSearchHeader> createState() => _OptimizedSearchHeaderState();
}

class _OptimizedSearchHeaderState extends State<OptimizedSearchHeader> {
  final ValueNotifier<int> _resultCountNotifier = ValueNotifier<int>(0);

  @override
  void dispose() {
    _resultCountNotifier.dispose(); // Bắt buộc giải phóng Notifier
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        TextField(
          onChanged: (text) {
            // Cập nhật giá trị Notifier mà không cần gọi setState trên widget cha
            _resultCountNotifier.value = text.length * 2;
          },
        ),
        // ListenableBuilder chỉ thực thi lại builder khi Notifier phát tín hiệu
        ListenableBuilder(
          listenable: _resultCountNotifier,
          builder: (context, child) {
            return Row(
              children: [
                Text('Kết quả: ${_resultCountNotifier.value}'),
                const SizedBox(width: 8),
                // Tái sử dụng đối tượng child tĩnh, không tạo lại qua mỗi lần cập nhật
                child!,
              ],
            );
          },
          child: const Icon(Icons.check_circle, color: Colors.green),
        ),
      ],
    );
  }
}
```

---

### 3.3 — Giá trị dẫn xuất (Derived State) qua Getter

```dart
class FilterModelExample extends StatefulWidget {
  const FilterModelExample({super.key});

  @override
  State<FilterModelExample> createState() => _FilterModelExampleState();
}

class _FilterModelExampleState extends State<FilterModelExample> {
  String _query = '';
  bool _inStockOnly = false;

  // Thuộc tính tính toán (Computed property) — Tránh lưu trữ trạng thái dư thừa
  bool get hasActiveFilter => _query.isNotEmpty || _inStockOnly;

  void _resetFilters() {
    // Gom cụm nhiều thay đổi trạng thái vào duy nhất một lần gọi setState
    setState(() {
      _query = '';
      _inStockOnly = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        if (hasActiveFilter)
          TextButton(onPressed: _resetFilters, child: const Text('Đặt lại')),
      ],
    );
  }
}
```

---

## Phần 4 — Các Bẫy Kỹ Thuật & Giải Pháp (Common Pitfalls & Mitigations)

### 4.1 — Triệu gọi setState trong quá trình thực thi hàm build()

```dart
// Lỗi: Gọi setState trực tiếp trong build()
@override
Widget build(BuildContext context) {
  if (_items.isEmpty) {
    // Kích hoạt assertion failure: setState() or markNeedsBuild() called during build
    setState(() {
      _isLoading = true;
    });
  }
  return const SizedBox.shrink();
}

// Giải pháp: Chuyển logic kích hoạt bất đồng bộ vào initState hoặc dùng addPostFrameCallback
@override
void initState() {
  super.initState();
  _loadInitialData();
}

// Hoặc lên lịch thực thi sau khi frame hiện tại kết thúc:
void _scheduleUpdate() {
  WidgetsBinding.instance.addPostFrameCallback((_) {
    if (mounted) {
      setState(() => _isLoading = true);
    }
  });
}
```

---

### 4.2 — Khai báo callback của setState ở dạng bất đồng bộ (async)

```dart
// Lỗi: Callback của setState trả về một Future
void _handleLikeAction() {
  setState(() async { // Warning: Callbacks provided to setState() shouldn't return a Future
    await apiService.submitLike();
    _likeCount++;
  });
}

// Giải pháp: Thực thi thao tác bất đồng bộ bên ngoài, chỉ cập nhật trạng thái đồng bộ khi hoàn tất
Future<void> _handleLikeAction() async {
  try {
    final newCount = await apiService.submitLike();
    if (!mounted) return;
    setState(() {
      _likeCount = newCount;
    });
  } catch (e) {
    // Xử lý lỗi
  }
}
```

---

### 4.3 — Đột biến tập hợp tại chỗ (In-place Collection Mutation)

```dart
// Bẫy kỹ thuật: Đột biến phần tử trong danh sách nhưng giữ nguyên tham chiếu đối tượng
setState(() {
  _userList.add(newUser); // Tham chiếu _userList không đổi
});

// Giải pháp chuẩn: Luôn tạo một collection mới (Immutable update pattern)
setState(() {
  _userList = [..._userList, newUser]; // Tạo instance danh sách mới trên Heap
});
```

---

## Phần 5 — Câu Hỏi Kỹ Thuật & Phân Tích Thực Thi (Technical Analysis & Code Tracing)

---

#### Q1 — "Phương thức `setState()` có thực thi lại phương thức `build()` ngay tại thời điểm gọi hay không?"

**Phân tích kỹ thuật:**
Không. `setState()` là một phương thức đồng bộ nhưng nhiệm vụ của nó chỉ giới hạn ở việc:
1. Thực thi callback `fn()`.
2. Gọi `_element!.markNeedsBuild()`, đặt cờ `_dirty = true` trên Element và đưa Element vào danh sách `_dirtyElements` của `BuildOwner`.

Phương thức `build()` thực sự sẽ chỉ được triệu gọi trong pha render tiếp theo khi engine nhận được tín hiệu VSync từ phần cứng thông qua phương thức `BuildOwner.buildScope()`.

---

#### Q2 — "Tại sao `BuildOwner` bắt buộc phải sắp xếp danh sách `_dirtyElements` theo chiều sâu cây (`depth`) trước khi rebuild?"

**Phân tích kỹ thuật:**
Trong Element Tree, node cha luôn có giá trị `depth` nhỏ hơn node con của nó.
- Nếu cả cha và con đều bị đánh dấu dirty trong cùng một frame: Nếu duyệt không theo thứ tự và node con build trước, node con sẽ tốn tài nguyên chạy hàm `build()`. Ngay sau đó, node cha chạy hàm `build()`, dẫn đến việc cha khởi tạo widget mới cho con và gọi `updateChild()`, buộc node con phải chạy lại hàm `build()` lần thứ hai.
- Bằng cách sắp xếp `_dirtyElements.sort((a, b) => a.depth - b.depth)`, framework đảm bảo node cha luôn build trước. Khi cha build xong, cấu hình của con được cập nhật và cờ dirty của con được xóa bỏ, loại bỏ hoàn toàn hiện tượng duplicate rebuilds.

---

#### Q3 — "Nếu triệu gọi `setState()` 5 lần liên tiếp trong một hàm đồng bộ, framework sẽ yêu cầu bao nhiêu khung hình VSync?"

**Phân tích kỹ thuật:**
Framework chỉ yêu cầu **duy nhất 1 khung hình VSync**.
Bên trong mã nguồn của `Element.markNeedsBuild()`:
```dart
if (dirty) return;
```
Ở lần gọi đầu tiên, cờ `_dirty` chuyển thành `true` và yêu cầu lên lịch frame đã được gửi tới `SchedulerBinding`. Ở 4 lần gọi tiếp theo trong cùng chu kỳ đồng bộ, do cờ `_dirty` đã là `true`, phương thức trả về ngay lập tức. Tất cả các đột biến trạng thái được tích lũy lại và chỉ kết xuất trong một lần build duy nhất khi frame VSync kế tiếp xuất hiện (Frame Coalescing).

---

#### Q4 — "Phân tích nguyên nhân kỹ thuật của lỗi: `setState() or markNeedsBuild() called during build`."

**Phân tích kỹ thuật:**
Trong khi `BuildOwner.buildScope()` đang duyệt qua cây, framework thiết lập cờ kiểm tra nội bộ `_debugBuilding = true`.
- Nếu trong phương thức `build()` của một widget (hoặc trong một callback được kích hoạt đồng bộ từ `build()`), xuất hiện lời gọi `setState()`: Phương thức `State.setState()` sẽ kiểm tra `assert(!_debugBuilding)`. Do cờ này đang bật, assertion exception được ném ra ngay lập tức.
- **Ràng buộc kiến trúc:** Trong Declarative UI, hàm `build()` phải là một hàm thuần túy chuyển đổi từ State sang UI descriptor ($UI = f(State)$). Việc tính toán giao diện không được phép phát sinh side-effect làm thay đổi State, vì hành vi này sẽ phá vỡ tính ổn định của đồ thị cây đang được dựng và có thể gây ra vòng lặp vô hạn.

---

#### Q5 (Trace Code) — "Dự đoán số lần thực thi hàm `build()` khi kết hợp lệnh đồng bộ, microtask và VSync boundary"

Xem xét đoạn mã sau:

```dart
class CoalescingTraceWidget extends StatefulWidget {
  const CoalescingTraceWidget({super.key});
  @override State<CoalescingTraceWidget> createState() => _CoalescingTraceWidgetState();
}

class _CoalescingTraceWidgetState extends State<CoalescingTraceWidget> {
  int _counter = 0;

  void _executeSequence() {
    print('1. Bắt đầu hàm');
    setState(() => _counter++);
    setState(() => _counter++);
    
    Future.microtask(() {
      print('2. Trong microtask');
      setState(() => _counter++);
    });
    
    print('3. Kết thúc hàm');
  }

  @override
  Widget build(BuildContext context) {
    print('===> build() thực thi: counter = $_counter');
    return ElevatedButton(
      onPressed: _executeSequence,
      child: Text('Count: $_counter'),
    );
  }
}
```

**Thứ tự in log khi khởi chạy ban đầu:**
```
===> build() thực thi: counter = 0
```

**Thứ tự in log khi người dùng kích hoạt `_executeSequence`:**
```
1. Bắt đầu hàm
3. Kết thúc hàm
2. Trong microtask
===> build() thực thi: counter = 3
```

**Phân tích kỹ thuật:**
1. Hai lệnh `setState()` đầu tiên chạy đồng bộ, đưa Element vào danh sách dirty (chỉ đánh dấu 1 lần).
2. Lệnh `Future.microtask()` được đưa vào hàng đợi Microtask Queue của Dart Event Loop.
3. Luồng đồng bộ kết thúc in ra log `3. Kết thúc hàm`.
4. Event loop lập tức xử lý Microtask Queue trước khi frame render tiếp theo diễn ra, in ra `2. Trong microtask` và thực thi lệnh `setState()` thứ ba.
5. Cả 3 lần đột biến trạng thái đều hoàn thành trước khi tín hiệu VSync kích hoạt `BuildOwner.buildScope()`. Do đó, phương thức `build()` chỉ thực thi **đúng 1 lần** với giá trị cuối cùng là `_counter = 3`.
