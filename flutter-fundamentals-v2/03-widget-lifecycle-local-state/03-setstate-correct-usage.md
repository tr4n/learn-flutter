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

### Câu hỏi phỏng vấn liên quan:

1. **"setState gây ra gì trong Flutter?"**
   - Mark Element hiện tại dirty → scheduler thêm vào dirty list
   - Cuối frame: buildScope() rebuild element dirty và subtree của nó

2. **"Làm thế nào để giảm rebuild scope khi setState?"**
   - Tách StatefulWidget con cho phần thay đổi
   - Đưa State lên/xuống cây phù hợp với scope cần thiết
   - Dùng `const` cho widget không thay đổi

3. **"Có thể gọi setState bên ngoài callback không?"**
   - Được, nhưng callback rỗng `setState(() {})` vẫn trigger rebuild
   - Thường sẽ cần là `setState(() { _field = value; })` để có mutation
