# Bài 3.3 — setState — Dùng Đúng Cách

## Phần 1 — Khái Niệm & Mục Tiêu Bài Học

### Tại sao bài này quan trọng?

`setState()` là API đơn giản nhất trong Flutter, nhưng cũng là nguồn gốc của nhiều vấn đề hiệu năng. Một `setState()` không cẩn thận có thể rebuild hàng chục widget không cần thiết.

```dart
// Tình huống: chỉ thay đổi 1 item trong list 100 item
setState(() {
  items[5].isSelected = true;
});
// Kết quả: toàn bộ Widget tree từ chỗ gọi setState() rebuild
// 100 ListTile, Header, Footer... tất cả rebuild dù chỉ 1 item thay đổi!
```

### Bạn sẽ hiểu được sau bài này:
- `setState()` hoạt động như thế nào internally
- Tại sao phải tách widget con để giảm rebuild scope
- Anti-patterns gây rebuild không cần thiết
- Đo rebuild với `debugPrintRebuildDirtyWidgets`

---

## Phần 2 — Cơ Chế Hoạt Động (Under the Hood)

### setState Internal Flow

```dart
// Từ Flutter source code (framework.dart), simplified:
void setState(VoidCallback fn) {
  assert(mounted, 'setState called after dispose!');
  // 1. Thực thi callback để modify state
  fn();
  // 2. Đánh dấu Element này là "dirty"
  _element!.markNeedsBuild();
}
```

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant State
    participant Element as StatefulElement
    participant Scheduler
    participant Build as Build Phase

    Dev->>State: setState(() { _count++; })
    State->>Element: markNeedsBuild()
    Element->>Scheduler: Thêm vào dirty list
    Note over Scheduler: Chờ end of frame

    Scheduler->>Build: buildScope()
    Build->>Element: element.rebuild()
    Element->>State: state.build(context)
    Note over State: Chỉ rebuild từ Element này xuống!
    State-->>Build: Widget subtree mới
```

### Rebuild Scope — Tại sao quan trọng

```
setState() trong _TopLevelState → rebuild TẤT CẢ từ TopLevel xuống:

TopLevelWidget (setState gọi ở đây)
├── Header (rebuild không cần thiết)
├── ProductList (rebuild không cần thiết)
│   ├── ProductItem 1 (rebuild không cần thiết)
│   ├── ProductItem 2 (rebuild không cần thiết)
│   └── ...
└── Counter (thứ thực sự thay đổi)

Giải pháp: Tách Counter thành widget riêng với State riêng
→ setState trong _CounterState chỉ rebuild Counter và con của nó
```

---

## Phần 3 — Code Mẫu Chuẩn Google

### 3.1 — setState cơ bản đúng cách

```dart
class ProductListScreen extends StatefulWidget {
  const ProductListScreen({super.key});
  @override State<ProductListScreen> createState() => _ProductListState();
}

class _ProductListState extends State<ProductListScreen> {
  List<Product> _products = [];
  bool _isLoading = false;
  String? _error;
  String _searchQuery = '';

  // ✅ setState với multiple state updates trong một lần
  // → Chỉ 1 rebuild thay vì 3 rebuild riêng lẻ
  Future<void> _loadProducts() async {
    setState(() {
      _isLoading = true;
      _error = null;
      // Không clear _products ngay — giữ cũ trong lúc load mới
    });

    try {
      final products = await ProductRepository().fetchAll(query: _searchQuery);
      if (!mounted) return;
      setState(() {
        _products = products;
        _isLoading = false;
      });
    } catch (e) {
      if (!mounted) return;
      setState(() {
        _error = e.toString();
        _isLoading = false;
      });
    }
  }

  // ✅ Filter không cần setState nếu dùng getter
  // (computed from state, không phải state itself)
  List<Product> get _filteredProducts => _products
      .where((p) => p.name.toLowerCase().contains(_searchQuery.toLowerCase()))
      .toList();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          // Header — không thay đổi theo _products
          const _PageHeader(), // const → không rebuild
          // Search field
          SearchBar(
            onChanged: (query) => setState(() => _searchQuery = query),
          ),
          // Content
          Expanded(
            child: _isLoading
                ? const CircularProgressIndicator()
                : _error != null
                    ? Text(_error!)
                    : _ProductGrid(products: _filteredProducts),
          ),
        ],
      ),
    );
  }
}

// ✅ Header tách riêng, không có internal state → const
class _PageHeader extends StatelessWidget {
  const _PageHeader();

  @override
  Widget build(BuildContext context) {
    return const Padding(
      padding: EdgeInsets.all(16),
      child: Text('Danh Sách Sản Phẩm',
          style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold)),
    );
  }
}

// ✅ ProductGrid tách riêng
class _ProductGrid extends StatelessWidget {
  final List<Product> products;
  const _ProductGrid({required this.products});

  @override
  Widget build(BuildContext context) {
    return GridView.builder(
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 2),
      itemCount: products.length,
      itemBuilder: (_, i) => ProductCard(product: products[i]),
    );
  }
}
```

### 3.2 — Tách widget để giảm rebuild scope

```dart
// ❌ TRƯỚC: Counter State ở Screen level → mọi child rebuild
class ShopScreenBefore extends StatefulWidget { ... }
class _ShopScreenStateBefore extends State<ShopScreenBefore> {
  int _cartCount = 0; // State ở đây

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Shop'),
        actions: [
          // Counter badge ở đây
          Stack(children: [
            const Icon(Icons.shopping_cart),
            if (_cartCount > 0) Positioned(
              right: 0, top: 0,
              child: Container(
                padding: const EdgeInsets.all(2),
                decoration: const BoxDecoration(
                  color: Colors.red, shape: BoxShape.circle,
                ),
                child: Text('$_cartCount'),
              ),
            ),
          ]),
        ],
      ),
      // Body: cực nhiều widget không liên quan đến _cartCount
      body: const ProductCatalog(), // Rebuild vô ích mỗi khi _cartCount thay đổi!
    );
  }
}

// ✅ SAU: CartBadge là StatefulWidget riêng
class ShopScreen extends StatelessWidget {
  const ShopScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Shop'),
        actions: const [
          CartBadge(), // Có State riêng → setState chỉ rebuild CartBadge
        ],
      ),
      body: const ProductCatalog(), // KHÔNG rebuild khi cart thay đổi
    );
  }
}

class CartBadge extends StatefulWidget {
  const CartBadge({super.key});
  @override State<CartBadge> createState() => _CartBadgeState();
}

class _CartBadgeState extends State<CartBadge> {
  int _count = 0;

  void _addToCart() => setState(() => _count++);

  @override
  Widget build(BuildContext context) {
    return Stack(children: [
      IconButton(icon: const Icon(Icons.shopping_cart), onPressed: () {}),
      if (_count > 0) Positioned(
        right: 4, top: 4,
        child: CircleAvatar(
          radius: 8,
          backgroundColor: Colors.red,
          child: Text('$_count', style: const TextStyle(fontSize: 10, color: Colors.white)),
        ),
      ),
    ]);
  }
}
```

### 3.3 — Callback pattern để "lift state up"

```dart
// Khi State cần share giữa siblings → lift up to common ancestor

class CartScreen extends StatefulWidget {
  const CartScreen({super.key});
  @override State<CartScreen> createState() => _CartScreenState();
}

class _CartScreenState extends State<CartScreen> {
  final List<CartItem> _cartItems = [];

  void _addItem(Product product) {
    setState(() {
      final existingIndex = _cartItems.indexWhere((i) => i.productId == product.id);
      if (existingIndex >= 0) {
        // Tạo object mới để đảm bảo immutability
        final existing = _cartItems[existingIndex];
        _cartItems[existingIndex] = existing.copyWith(quantity: existing.quantity + 1);
      } else {
        _cartItems.add(CartItem(productId: product.id, quantity: 1));
      }
    });
  }

  void _removeItem(String productId) {
    setState(() => _cartItems.removeWhere((i) => i.productId == productId));
  }

  double get _total => _cartItems.fold(0, (sum, item) => sum + item.subtotal);

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // CartSummary nhận _total — rebuild khi _total thay đổi
        CartSummary(total: _total),
        Expanded(
          child: CartItemList(
            items: _cartItems,
            onRemove: _removeItem, // Callback lift state up
          ),
        ),
        CheckoutButton(
          isEnabled: _cartItems.isNotEmpty,
          onPressed: () { /* checkout */ },
        ),
      ],
    );
  }
}
```

---

## Phần 4 — Lỗi Sai Phổ Biến & Best Practices

### ❌ Anti-pattern 1: setState trong build()

```dart
// ❌ Nguy hiểm: Vòng lặp vô hạn build → setState → build → ...
@override
Widget build(BuildContext context) {
  if (_data == null) {
    setState(() => _loadData()); // 💥 Sẽ gây vòng lặp vô hạn
  }
  return Text(_data ?? 'Loading');
}

// ✅ Đúng: Load data trong initState hoặc didChangeDependencies
@override
void initState() {
  super.initState();
  _loadData(); // Gọi future, setState bên trong _loadData
}
```

### ❌ Anti-pattern 2: setState sau dispose

```dart
// ❌ Crash: setState sau khi widget bị dispose
Future<void> _loadData() async {
  final data = await fetchData(); // Có thể mất nhiều thời gian
  setState(() => _data = data); // 💥 Widget có thể đã dispose!
}

// ✅ Guard với mounted
Future<void> _loadData() async {
  final data = await fetchData();
  if (!mounted) return;
  setState(() => _data = data);
}
```

### ❌ Anti-pattern 3: setState với empty callback

```dart
// ❌ Sai: setState với empty callback vẫn trigger rebuild!
setState(() {}); // Trigger rebuild không cần thiết

// ✅ Đúng: Nếu muốn force rebuild (rất hiếm khi cần):
// Cân nhắc xem có cần không, thường là dấu hiệu design sai
```

### ❌ Anti-pattern 4: Modify list/map trực tiếp trong setState

```dart
// ❌ Sai: Modify object trong list mà không tạo mới
setState(() {
  _products[0].isSelected = true; // Modify object cũ
  // Flutter không detect thay đổi này nếu list reference không đổi
});

// ✅ Đúng: Tạo list/object mới (immutable update pattern)
setState(() {
  _products = [
    for (final p in _products)
      if (p.id == targetId) p.copyWith(isSelected: true)
      else p,
  ];
});
```

---

## Phần 5 — Bài Tập Củng Cố Tư Duy

### Challenge: Refactor Counter App — Giảm 50 Rebuild → <5 Rebuild

**Code hiện tại:**
```dart
class HomeScreen extends StatefulWidget { ... }
class _HomeScreenState extends State<HomeScreen> {
  int _count = 0;
  String _username = 'User';
  List<String> _history = [];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Hello $_username')),
      body: Column(children: [
        const AppLogo(), // StatelessWidget, không thay đổi
        WelcomeText(name: _username), // Thay đổi khi _username thay đổi
        const FeatureList(), // StatelessWidget, không thay đổi
        CounterDisplay(count: _count), // Thay đổi khi _count thay đổi
        HistoryList(history: _history), // Thay đổi khi _history thay đổi
        Row(children: [
          ElevatedButton(
            onPressed: () => setState(() {
              _count++;
              _history.add('Added at ${DateTime.now()}');
            }),
            child: const Text('+'),
          ),
        ]),
      ]),
    );
  }
}
```

**Nhiệm vụ:**
1. Xác định widget nào rebuild không cần thiết khi nhấn button
2. Tách State thành nhiều widget nhỏ hơn để giảm rebuild scope
3. Bật `debugPrintRebuildDirtyWidgets` và đếm số rebuild trước/sau

### Câu Hỏi Phỏng Vấn

> **[Junior]** — nắm khái niệm | **[Middle]** — hiểu cơ chế | **[Senior]** — hiểu Flutter internals | **[Trace Code]** — đọc code và dự đoán output

---

#### Q1 [Junior] — "`setState()` gây ra gì trong Flutter?"

**Trả lời chuẩn:**

`setState()` không immediately rebuild widget — nó **đánh dấu Element là dirty** và yêu cầu Flutter schedule một rebuild vào cuối frame hiện tại (hoặc frame tiếp theo):

```
setState(() { _count++; })
  1. Thực thi callback (mutation xảy ra ở đây)
  2. element.markNeedsBuild()    → Element được mark dirty
  3. BuildOwner.scheduleBuildFor(element)
  4. SchedulerBinding.scheduleFrame()   → yêu cầu vsync nếu chưa có
  --- (frame boundary) ---
  5. BuildOwner.buildScope()    → rebuild mọi dirty element
  6. State.build(context) được gọi → widget tree mới
```

**Kết quả:** UI được update tại frame tiếp theo, không phải ngay lập tức. Nhiều `setState()` trong cùng frame → chỉ một lần rebuild.

---

#### Q2 [Junior] — "Làm thế nào để giảm rebuild scope khi dùng `setState()`?"

**Trả lời chuẩn:**

**1. Tách StatefulWidget nhỏ hơn:**
```dart
// ❌ Bad: Toàn bộ screen rebuild khi counter thay đổi
class _HomeState extends State<Home> {
  int _counter = 0;
  @override Widget build(context) => Column(children: [
    HeavyList(),          // rebuild không cần thiết!
    Text('$_counter'),
    ElevatedButton(onPressed: () => setState(() => _counter++), ...)
  ]);
}

// ✅ Good: Tách counter ra component riêng
class CounterWidget extends StatefulWidget { ... }
class _CounterState extends State<CounterWidget> {
  int _counter = 0;
  @override Widget build(context) => Column(children: [
    Text('$_counter'),
    ElevatedButton(onPressed: () => setState(() => _counter++), ...)
  ]);
}
// HeavyList() trong parent không rebuild khi counter thay đổi
```

**2. Dùng `const` cho widget static:** `const HeavyList()` → Flutter skip rebuild dù parent rebuild.

**3. Dùng state management:** `ValueNotifier + ValueListenableBuilder` chỉ rebuild đúng phần cần thiết.

---

#### Q3 [Middle] — "Tại sao gọi `setState()` nhiều lần trong cùng frame không gây nhiều rebuild?"

**Trả lời chuẩn:**

`setState()` chỉ **mark** element là dirty và call `scheduleFrame()`. Nếu element đã dirty, gọi lần hai chỉ là no-op cho `scheduleBuildFor()`. `scheduleFrame()` cũng idempotent — nếu đã có frame scheduled, không schedule thêm.

```dart
void onButtonTap() {
  setState(() => _a = 1); // mark dirty, schedule frame
  setState(() => _b = 2); // element đã dirty → scheduleBuildFor no-op
  setState(() => _c = 3); // tương tự
  // Kết quả: chỉ 1 rebuild với _a=1, _b=2, _c=3
}
```

**Điều này có nghĩa là** gọi `setState()` nhiều lần trong một event handler là OK và thường là pattern tốt để tách biệt mutations:
```dart
setState(() => _isLoading = true);
final data = await fetch(); // await không ở đây — setState sync
setState(() { _data = data; _isLoading = false; }); // merge vào 1 setState sau await
```

---

#### Q4 [Senior] — "Bên trong `setState()`, Flutter thực sự làm gì? `BuildOwner` hoạt động ra sao?"

**Trả lời chuẩn:**

```dart
// Flutter source (simplified) — State.setState()
void setState(VoidCallback fn) {
  assert(mounted, 'setState called after dispose!');
  assert(_debugLifecycleState == _StateLifecycle.ready);
  
  final dynamic result = fn(); // thực thi mutation callback
  assert(() {
    if (result is Future) throw 'setState callback must not be async';
    return true;
  }());
  
  _element!.markNeedsBuild(); // delegate lên Element
}

// Element.markNeedsBuild()
void markNeedsBuild() {
  if (_dirty) return; // đã dirty rồi → bỏ qua
  _dirty = true;
  owner!.scheduleBuildFor(this);
}

// BuildOwner.scheduleBuildFor()
void scheduleBuildFor(Element element) {
  _dirtyElements.add(element);
  if (!_scheduledFlushDirtyElements) {
    _scheduledFlushDirtyElements = true;
    SchedulerBinding.instance.scheduleFrame();
  }
}
```

**Sau vsync:** `BuildOwner.buildScope()` sort `_dirtyElements` theo depth (đảm bảo parent build trước child), sau đó loop và gọi `element.rebuild()` trên mỗi element.

---

#### Q5 [Middle] — "`setState(() {})` rỗng có gây rebuild không? Khi nào hợp lý?"

**Trả lời chuẩn:**

**Có** — `setState(() {})` vẫn mark element dirty và trigger rebuild, dù callback không làm gì.

**Khi nào hợp lý:**

```dart
// 1. Mutation đã xảy ra bên ngoài setState
_list.add(item); // mutate directly
setState(() {}); // chỉ cần trigger rebuild

// 2. ChangeNotifier-style notification
_model.update();   // model tự cập nhật
setState(() {}); // rebuild để reflect changes

// 3. Force rebuild sau async (pattern cũ, không khuyến khích)
await Future.delayed(duration);
if (mounted) setState(() {}); // force rebuild
```

**Tốt hơn nên làm:** Mutation trong callback để code rõ ràng hơn:
```dart
setState(() { _list.add(item); }); // explicit — ai đọc code biết ngay tại sao rebuild
```

---

#### Q6 [Middle] — "Tại sao không được gọi `setState()` trong `build()` method?"

**Trả lời chuẩn:**

Gọi `setState()` trong `build()` tạo **infinite rebuild loop**:

```
build() được gọi
  → setState() trong build()
    → mark dirty
      → build() được gọi lại
        → setState() lại
          → infinite loop
```

Flutter có assertion trong `BuildOwner.buildScope()` phát hiện điều này trong debug mode và throw `FlutterError: setState() or markNeedsBuild() called during build`.

**Tương tự:** Không gọi `setState()` trong `didUpdateWidget()` hoặc `didChangeDependencies()` mà không có điều kiện guard — vì các lifecycle này được gọi từ bên trong build scope.

```dart
// ❌ Sai — infinite loop
@override Widget build(context) {
  setState(() { _value = 42; }); // CRASH
  return Text('$_value');
}

// ❌ Sai — vẫn có thể loop nếu không có guard
@override void didUpdateWidget(old) {
  super.didUpdateWidget(old);
  setState(() { }); // loop nếu luôn trigger
}

// ✅ Đúng — có điều kiện
@override void didUpdateWidget(old) {
  super.didUpdateWidget(old);
  if (old.value != widget.value) setState(() { _internal = widget.value; });
}
```

---

#### Q7 [Trace Code] — "`setState` trong async callback sau dispose: lỗi gì xảy ra?"

```dart
class TimerWidget extends StatefulWidget {
  const TimerWidget({super.key});
  @override
  State<TimerWidget> createState() => _TimerWidgetState();
}

class _TimerWidgetState extends State<TimerWidget> {
  int _seconds = 0;

  @override
  void initState() {
    super.initState();
    // Bug: Timer không được cancel trong dispose
    Timer.periodic(const Duration(seconds: 1), (timer) {
      setState(() { // Gọi sau widget đã dispose?
        _seconds++;
      });
    });
  }

  @override
  Widget build(BuildContext context) => Text('$_seconds seconds');
  
  // Không có dispose() → Timer chạy mãi
}
```

**Điều gì xảy ra khi navigate away từ widget này?**

1. `_TimerWidgetState.dispose()` được gọi (default `super.dispose()` chạy)
2. `mounted = false`, `_element = null`
3. Timer **vẫn chạy** — nó không biết widget đã bị dispose
4. Sau 1 giây: Timer callback chạy → `setState()` được gọi
5. `setState()` check: `assert(mounted)` → `mounted = false` → **throw assertion error** (debug) hoặc **silent exception** (release)
6. Memory leak: Timer giữ reference đến State → State không được GC

**Fix:**
```dart
Timer? _timer;

@override
void initState() {
  super.initState();
  _timer = Timer.periodic(const Duration(seconds: 1), (timer) {
    if (!mounted) { timer.cancel(); return; }
    setState(() { _seconds++; });
  });
}

@override
void dispose() {
  _timer?.cancel(); // luôn cancel trong dispose
  super.dispose();
}
```
