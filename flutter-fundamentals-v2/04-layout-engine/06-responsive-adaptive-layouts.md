# Bài 4.6 — Responsive & Adaptive Layouts

## Phần 1 — Khái Niệm & Mục Tiêu Bài Học

### Tại sao bài này quan trọng?

Flutter target multiple platforms (mobile, tablet, web, desktop) — một codebase. Nhưng UI đẹp trên phone không có nghĩa là đẹp trên tablet:

```
Phone (390px wide):
  [List item 1]
  [List item 2]
  [List item 3]

Tablet (900px wide) — nếu không responsive:
  [List item 1                      ]  ← Trông rất lạ, nhiều whitespace
  [List item 2                      ]

Tablet — nếu responsive:
  [List item 1] | [Detail view     ]  ← Master-detail layout!
  [List item 2] |                    
  [List item 3] |                    
```

### Sự khác biệt: Responsive vs Adaptive

- **Responsive**: thay đổi layout theo *kích thước* màn hình (column count, font size)
- **Adaptive**: thay đổi *hành vi* theo *platform* (Material trên Android, Cupertino trên iOS)

### Bạn sẽ hiểu được sau bài này:
- `MediaQuery` — screen size, orientation, text scale
- `LayoutBuilder` — available space từ parent
- Responsive breakpoints: mobile / tablet / desktop
- `OrientationBuilder` — portrait vs landscape
- Flexible spacing với `Flexible` và `FractionallySizedBox`

---

## Phần 2 — Cơ Chế Hoạt Động (Under the Hood)

### MediaQuery vs LayoutBuilder

```mermaid
graph LR
    subgraph mq ["MediaQuery"]
        MQ["MediaQuery.of(context)"]
        MQ --> SW["screenWidth (full screen)"]
        MQ --> SH["screenHeight (full screen)"]
        MQ --> TS["textScaleFactor"]
        MQ --> ORI["orientation"]
        MQ --> PAD["padding (safe area)"]
        MQ --> VI["viewInsets (keyboard)"]
    end

    subgraph lb ["LayoutBuilder"]
        LB["LayoutBuilder"]
        LB --> AW["availableWidth (parent constraint)"]
        LB --> AH["availableHeight (parent constraint)"]
        Note["Có thể khác screenWidth\nnếu widget trong Column/Padding"]
    end

    Prefer["Prefer LayoutBuilder\ncho widget-level responsive\nMediaQuery cho screen-level"]
```

### Breakpoints Strategy

```
Mobile:  0     - 599px
Tablet:  600px - 1023px
Desktop: 1024px+

Nhưng Flutter app trên tablet thường chạy cả hai:
- Phone app trên tablet → MediaQuery.size.width = tablet width → cần responsive
- Native tablet app → phải design cho tablet
```

---

## Phần 3 — Code Mẫu Chuẩn Google

### 3.1 — Breakpoint system đơn giản

```dart
// Định nghĩa breakpoints một lần, dùng khắp app
enum Breakpoint { mobile, tablet, desktop }

extension BreakpointExt on double {
  Breakpoint get breakpoint {
    if (this >= 1024) return Breakpoint.desktop;
    if (this >= 600) return Breakpoint.tablet;
    return Breakpoint.mobile;
  }
}

// Helper extension trên BuildContext
extension ResponsiveContext on BuildContext {
  double get screenWidth => MediaQuery.of(this).size.width;
  Breakpoint get breakpoint => screenWidth.breakpoint;
  bool get isMobile => breakpoint == Breakpoint.mobile;
  bool get isTablet => breakpoint == Breakpoint.tablet;
  bool get isDesktop => breakpoint == Breakpoint.desktop;
}

// Dùng:
Widget build(BuildContext context) {
  return switch (context.breakpoint) {
    Breakpoint.mobile => MobileLayout(),
    Breakpoint.tablet => TabletLayout(),
    Breakpoint.desktop => DesktopLayout(),
  };
}
```

### 3.2 — Responsive Column Count

```dart
class ResponsiveProductGrid extends StatelessWidget {
  final List<Product> products;
  const ResponsiveProductGrid({super.key, required this.products});

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        // Tính column count dựa trên available width
        // (không phải screen width!)
        final columnCount = switch (constraints.maxWidth) {
          >= 1024 => 4,   // Desktop: 4 columns
          >= 600  => 3,   // Tablet: 3 columns
          >= 400  => 2,   // Large phone: 2 columns
          _       => 1,   // Small phone: 1 column
        };

        return GridView.builder(
          gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: columnCount,
            mainAxisSpacing: 8,
            crossAxisSpacing: 8,
            childAspectRatio: 0.75,
          ),
          itemCount: products.length,
          itemBuilder: (_, i) => ProductCard(product: products[i]),
        );
      },
    );
  }
}
```

### 3.3 — Master-Detail Layout cho tablet

```dart
class ProductMasterDetail extends StatefulWidget {
  final List<Product> products;
  const ProductMasterDetail({super.key, required this.products});
  @override State<ProductMasterDetail> createState() => _ProductMasterDetailState();
}

class _ProductMasterDetailState extends State<ProductMasterDetail> {
  Product? _selectedProduct;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final isWide = constraints.maxWidth >= 600;

        if (isWide) {
          // Tablet: Master-Detail side by side
          return Row(
            children: [
              // Master: 40% width
              SizedBox(
                width: constraints.maxWidth * 0.4,
                child: _MasterList(
                  products: widget.products,
                  selectedId: _selectedProduct?.id,
                  onSelect: (p) => setState(() => _selectedProduct = p),
                ),
              ),
              // Divider
              const VerticalDivider(width: 1),
              // Detail: remaining 60%
              Expanded(
                child: _selectedProduct != null
                    ? ProductDetailView(product: _selectedProduct!)
                    : const Center(child: Text('Chọn sản phẩm để xem chi tiết')),
              ),
            ],
          );
        } else {
          // Mobile: List only, detail on push
          return _MasterList(
            products: widget.products,
            selectedId: null,
            onSelect: (p) {
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (_) => Scaffold(
                    appBar: AppBar(title: Text(p.name)),
                    body: ProductDetailView(product: p),
                  ),
                ),
              );
            },
          );
        }
      },
    );
  }
}

class _MasterList extends StatelessWidget {
  final List<Product> products;
  final String? selectedId;
  final ValueChanged<Product> onSelect;

  const _MasterList({
    required this.products,
    required this.selectedId,
    required this.onSelect,
  });

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: products.length,
      itemBuilder: (_, i) {
        final product = products[i];
        final isSelected = product.id == selectedId;
        return ListTile(
          selected: isSelected,
          selectedTileColor: Theme.of(context).colorScheme.primaryContainer,
          title: Text(product.name),
          subtitle: Text('${product.price}đ'),
          onTap: () => onSelect(product),
        );
      },
    );
  }
}
```

### 3.4 — OrientationBuilder và Text Scale

```dart
class AdaptiveScreen extends StatelessWidget {
  const AdaptiveScreen({super.key});

  @override
  Widget build(BuildContext context) {
    // MediaQuery cung cấp nhiều thông tin hữu ích
    final mediaQuery = MediaQuery.of(context);
    final textScale = mediaQuery.textScaler;
    final padding = mediaQuery.padding; // Safe area (notch, home indicator)

    return OrientationBuilder(
      builder: (context, orientation) {
        // Thay đổi layout theo portrait/landscape
        final isPortrait = orientation == Orientation.portrait;

        return Padding(
          // Tôn trọng safe area
          padding: EdgeInsets.only(
            top: padding.top,
            bottom: padding.bottom,
          ),
          child: isPortrait ? _PortraitLayout() : _LandscapeLayout(),
        );
      },
    );
  }
}

class _PortraitLayout extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Flexible width cho portrait
        const Expanded(child: ProductList()),
        const _BottomBar(),
      ],
    );
  }
}

class _LandscapeLayout extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        // Sidebar cho landscape
        const SizedBox(width: 200, child: ProductList()),
        const VerticalDivider(width: 1),
        const Expanded(child: MainContent()),
      ],
    );
  }
}
```

### 3.5 — FractionallySizedBox — Percentage-based sizing

```dart
Widget build(BuildContext context) {
  return Column(
    children: [
      // 80% chiều rộng parent
      FractionallySizedBox(
        widthFactor: 0.8, // 80%
        child: TextField(decoration: InputDecoration(labelText: 'Email')),
      ),

      const SizedBox(height: 16),

      // 100% chiều rộng, 50% chiều cao
      FractionallySizedBox(
        widthFactor: 1.0,
        heightFactor: 0.5,
        child: Image.network(bannerUrl, fit: BoxFit.cover),
      ),

      // Flexible spacing: responsive gap
      const Spacer(), // Fill remaining

      // Safe button width
      FractionallySizedBox(
        widthFactor: 0.9,
        child: FilledButton(
          onPressed: () {},
          child: const Text('Tiếp tục'),
        ),
      ),
    ],
  );
}
```

---

## Phần 4 — Lỗi Sai Phổ Biến & Best Practices

### ❌ Anti-pattern 1: Hardcode pixel size

```dart
// ❌ Sai: hardcode pixel → vỡ trên màn hình khác
Container(
  width: 390, // Chỉ đúng trên iPhone 14!
  height: 844,
)

// ✅ Đúng: Responsive
LayoutBuilder(
  builder: (context, constraints) => Container(
    width: constraints.maxWidth, // Fill available
    height: constraints.maxHeight * 0.5, // 50% available
  ),
)
```

### ❌ Anti-pattern 2: `MediaQuery.of(context).size` trong deep widget

```dart
// ❌ Không chính xác: screenWidth ≠ available width nếu trong Drawer/Dialog
Widget build(BuildContext context) {
  final screenWidth = MediaQuery.of(context).size.width; // Screen size!
  return Container(width: screenWidth * 0.5); // Sai khi widget trong Drawer
}

// ✅ Đúng: LayoutBuilder cho accurate available space
LayoutBuilder(
  builder: (context, constraints) => Container(
    width: constraints.maxWidth * 0.5, // Actual available width
  ),
)
```

### ❌ Anti-pattern 3: Không test tablet layout

```dart
// Dùng Flutter DevTools Device Toolbar:
// Chrome DevTools → Device Toolbar → chọn tablet
// Hoặc trong Android Studio: AVD Manager → Pixel Tablet

// Minimum testing:
// - Phone portrait: 390x844
// - Phone landscape: 844x390
// - Tablet portrait: 768x1024
// - Tablet landscape: 1024x768
```

---

## Phần 5 — Bài Tập Củng Cố Tư Duy

### Challenge: Layout Thay Đổi Từ 1 Cột (Mobile) → 2 Cột (Tablet)

**Yêu cầu:**
- Mobile (<600px): danh sách 1 cột dọc
- Tablet (>=600px): grid 2 cột
- Desktop (>=1024px): grid 3 cột
- Khi chọn item trên tablet: hiện detail ở bên phải (master-detail)
- Khi chọn item trên mobile: navigate sang màn hình mới

**Gợi ý:**
- `LayoutBuilder` cho column count
- `Row` + `VerticalDivider` cho master-detail
- `Navigator.push` cho mobile detail

### Câu hỏi phỏng vấn liên quan:

1. **"Responsive vs Adaptive trong Flutter?"**
   - Responsive: thay đổi layout theo screen size
   - Adaptive: thay đổi widget theo platform (Material vs Cupertino)

2. **"Khi nào dùng `MediaQuery` vs `LayoutBuilder`?"**
   - `MediaQuery`: screen-level decisions (safe area, text scale, orientation)
   - `LayoutBuilder`: component-level decisions (column count, show/hide sidebar)

3. **"`FractionallySizedBox` dùng khi nào?"**
   - Khi cần size theo phần trăm của parent
   - Button width = 90% screen width, banner height = 30% screen height
