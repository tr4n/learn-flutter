# Bài 4.6 — Responsive & Adaptive Layout Architecture

## Phần 1 — Khái Niệm & Ràng Buộc Kiến Trúc (Architecture & Design Philosophy)

### 1.1 — Phân định ranh giới kiến trúc: Responsive vs Adaptive

Flutter là một bộ công cụ đa nền tảng (Multi-platform Toolkit) cho phép biên dịch cùng một codebase lên điện thoại, máy tính bảng, màn hình gập, trình duyệt web và máy tính để bàn. Để đạt được trải nghiệm người dùng tự nhiên trên mọi thiết bị, hệ thống kiến trúc phân biệt rõ ràng hai khái niệm thường bị nhầm lẫn:

```
┌────────────────────────────────────────────────────────────────────────┐
│ RESPONSIVE LAYOUT (Thích ứng hình học & Không gian hiển thị)          │
│   • Phản ứng với:  Chiều rộng (Width), Chiều cao (Height), Tỷ lệ khung │
│                    hình (Aspect Ratio), Hướng xoay (Orientation).       │
│   • Mục tiêu:      Sắp xếp lại cấu trúc giao diện để tận dụng tối đa   │
│                    diện tích khả dụng (ví dụ: chuyển từ 1 cột sang    │
│                    Master-Detail 2 cột khi màn hình mở rộng).          │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│ ADAPTIVE LAYOUT (Thích ứng bản chất nền tảng & Phương thức tương tác) │
│   • Phản ứng với:  Hệ điều hành (Android, iOS, macOS, Windows, Linux), │
│                    Thiết bị ngoại vi (Touchscreen, Chuột, Bàn phím).   │
│   • Mục tiêu:      Cung cấp đúng hành vi bản địa (Native Idioms) của   │
│                    từng OS (ví dụ: Cuộn overscroll kiểu iOS vs Android, │
│                    Context Menu chuột phải trên Desktop, Back gesture). │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 1.2 — Nguyên lý Window Size Classes và Breakpoint Invariants

Theo tiêu chuẩn thiết kế hiện đại (Material 3 Adaptive Design), kích thước không gian hiển thị được phân chia thành ba lớp chuẩn mực (**Window Size Classes**) dựa trên chiều rộng khả dụng:

$$\text{Window Width} \begin{cases} < 600\text{ dp} & \longrightarrow \mathbf{Compact} \text{ (Hầu hết Smartphone dạng dọc)} \\ 600\text{ dp} \le \text{Width} < 840\text{ dp} & \longrightarrow \mathbf{Medium} \text{ (Smartphone xoay ngang, Tablet nhỏ, Màn hình gập)} \\ \ge 840\text{ dp} & \longrightarrow \mathbf{Expanded} \text{ (Tablet lớn, Màn hình Desktop, Web)} \end{cases}$$

Bất biến kiến trúc: **Một thiết kế responsive chất lượng cao không được phụ thuộc vào tên thiết bị (như "iPhone 15" hay "iPad Pro"), mà chỉ được phép phụ thuộc thuần túy vào miền giá trị hình học của Window Size Class hiện tại.**

---

## Phần 2 — Cơ Chế Hoạt Động & Mã Nguồn Đối Chiếu (Under the Hood / Deep-Dive)

### 2.1 — Cơ chế phụ thuộc dữ liệu và Chi phí Rebuild của `MediaQuery`

`MediaQuery` là một `InheritedWidget` tầng gốc lưu trữ cấu trúc `MediaQueryData`. Trước phiên bản Flutter 3.10, lời gọi truyền thống:

```dart
final Size size = MediaQuery.of(context).size;
```

Tạo ra một **đăng ký phụ thuộc toàn phần (Full Dependency Registration)**. `MediaQueryData` là một đối tượng dữ liệu phức tạp chứa hàng chục thuộc tính:
- `size`: Kích thước cửa sổ.
- `orientation`: Hướng xoay màn hình.
- `padding`: Vùng đệm an toàn (tai thỏ, camera nốt ruồi).
- `viewInsets`: Khoảng không gian bị che phủ bởi bàn phím ảo hệ thống.
- `textScaler`: Tỷ lệ phóng to phông chữ của người dùng.
- `platformBrightness`: Chế độ giao diện sáng/tối.

```
[Bàn phím ảo xuất hiện] ──► viewInsets.bottom thay đổi
                                   │
                                   ▼
          Toàn bộ các Widget gọi MediaQuery.of(context) bị REBUILD!
          (Kể cả widget chỉ cần đọc size.width để chia cột!)
```

#### Kiến trúc trích xuất vi mô (Fine-Grained Aspects) trong Flutter hiện đại:
Bắt đầu từ Flutter 3.10, `MediaQuery` được tái cấu trúc thành một `InheritedModel`. Lớp này cho phép các widget chỉ đăng ký lắng nghe chính xác lát cắt dữ liệu (Aspect) mà chúng quan tâm thông qua các static method chuyên biệt:

```dart
// CHUẨN TỐI ƯU HIỆU NĂNG
final Size size = MediaQuery.sizeOf(context);            // Chỉ rebuild khi SIZE đổi
final EdgeInsets insets = MediaQuery.viewInsetsOf(context); // Chỉ rebuild khi BÀN PHÍM đổi
final EdgeInsets padding = MediaQuery.paddingOf(context);   // Chỉ rebuild khi SAFE AREA đổi
final Orientation orientation = MediaQuery.orientationOf(context);
final TextScaler textScaler = MediaQuery.textScalerOf(context);
```

Khi bàn phím ảo bật lên hoặc ẩn đi, chỉ các widget gọi `MediaQuery.viewInsetsOf(context)` mới bị đưa vào danh sách bẩn của `BuildOwner`. Toàn bộ các widget bố cục khác gọi `MediaQuery.sizeOf(context)` được bảo toàn 100%, loại bỏ hoàn toàn các chu kỳ Rebuild thừa.

---

### 2.2 — `LayoutBuilder` vs `MediaQuery`: Phạm vi không gian

| Tiêu chí | `MediaQuery.sizeOf(context)` | `LayoutBuilder` |
| :--- | :--- | :--- |
| **Phạm vi không gian** | Toàn cầu (Global Window / Screen Geometry). | Cục bộ (Local Parent Constraints). |
| **Pha thực thi** | Pha Build (Build Phase). | Pha Bố cục (Layout Phase via `performLayout`). |
| **Khả năng tái sử dụng (Reusability)** | Thấp: Phụ thuộc vào kích thước của cả màn hình thiết bị. | Rất cao: Tự thích ứng với mọi không gian chứa nó. |
| **Ngữ cảnh sử dụng chuẩn** | Điều hướng cấp màn hình (Top-level Page Routing, Scaffold Adaptive Layout). | Thành phần giao diện cấp linh kiện (Component-level adaptation, Widget cards). |

```
┌─────────────────────────────────────────────────────────────┐
│ MÀN HÌNH MÁY TÍNH ĐỂ BÀN (MediaQuery.sizeOf = 1440x900)     │
│ ┌──────────────────────┐ ┌────────────────────────────────┐ │
│ │ SIDEBAR (Width: 300) │ │ MAIN CONTENT AREA (Width: 1140)│ │
│ │                      │ │                                │ │
│ │ [Widget Card]        │ │ [Widget Card]                  │ │
│ │ LayoutBuilder nhận   │ │ LayoutBuilder nhận             │ │
│ │ maxWidth = 300px     │ │ maxWidth = 1140px              │ │
│ └──────────────────────┘ └────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```
Nếu `Widget Card` sử dụng `MediaQuery.sizeOf(context).width`, nó sẽ lầm tưởng rằng nó đang có $1440px$ không gian và hiển thị bố cục dạng ngang nhiều cột ngay cả khi đang bị co hẹp trong Sidebar $300px$, gây ra lỗi vỡ khung hình nghiêm trọng.

---

### 2.3 — Màn hình gập (Foldables) và API DisplayFeatures

Với các thiết bị màn hình gập (như Samsung Galaxy Z Fold hoặc Microsoft Surface Duo), giao diện có thể bị cắt đôi bởi một nếp gập (Fold) hoặc một bản lề cơ học (Hinge).

Framework cung cấp thông tin này thông qua thuộc tính `displayFeatures` của `MediaQueryData`:
```dart
final List<DisplayFeature> features = MediaQuery.displayFeaturesOf(context);
for (final DisplayFeature feature in features) {
  if (feature.type == DisplayFeatureType.hinge) {
    // Bản lề cơ học vật lý: Tuyệt đối không vẽ nội dung quan trọng đè lên vùng bounds này
    final Rect hingeBounds = feature.bounds;
  }
}
```

Widget `DisplayFeatureSubScreen` tự động chia nhỏ vùng hiển thị thành các không gian độc lập, ngăn chặn hiện tượng chữ hoặc nút bấm bị gãy đôi ở chính giữa bản lề thiết bị.

---

## Phần 3 — Hướng Dẫn Thực Hành Chuẩn (Production-Ready Implementations)

### 3.1 — Kiến trúc Canonical Master-Detail Layout

Mô hình chuẩn mực triển khai một màn hình hiển thị danh sách - chi tiết tự động chuyển đổi giữa Mobile (Navigation Stack push màn hình mới) và Tablet/Desktop (Hiển thị song song hai cột):

```dart
import 'package:flutter/material.dart';

class CanonicalMasterDetailScreen extends StatefulWidget {
  const CanonicalMasterDetailScreen({super.key});

  @override
  State<CanonicalMasterDetailScreen> createState() => _CanonicalMasterDetailScreenState();
}

class _CanonicalMasterDetailScreenState extends State<CanonicalMasterDetailScreen> {
  int? _selectedItemId;

  @override
  Widget build(BuildContext context) {
    // Đọc kích thước màn hình thông qua API vi mô
    final double screenWidth = MediaQuery.sizeOf(context).width;
    final bool isExpandedLayout = screenWidth >= 840.0;

    return Scaffold(
      appBar: AppBar(title: const Text('Quản Lý Đơn Hàng')),
      body: isExpandedLayout
          ? Row(
              children: [
                // Cột Master (Danh sách) cố định 360px
                SizedBox(
                  width: 360.0,
                  child: OrderListView(
                    selectedId: _selectedItemId,
                    onSelect: (id) => setState(() => _selectedItemId = id),
                  ),
                ),
                const VerticalDivider(width: 1.0),
                // Cột Detail (Chi tiết) mở rộng chiếm toàn bộ không gian còn lại
                Expanded(
                  child: _selectedItemId != null
                      ? OrderDetailView(orderId: _selectedItemId!)
                      : const Center(child: Text('Vui lòng chọn một đơn hàng để xem chi tiết')),
                ),
              ],
            )
          : OrderListView(
              selectedId: _selectedItemId,
              onSelect: (id) {
                // Trên màn hình Compact: Push route mới sang trang chi tiết
                Navigator.of(context).push(
                  MaterialPageRoute(
                    builder: (context) => Scaffold(
                      appBar: AppBar(title: Text('Đơn hàng #$id')),
                      body: OrderDetailView(orderId: id),
                    ),
                  ),
                );
              },
            ),
    );
  }
}

class OrderListView extends StatelessWidget {
  final int? selectedId;
  final ValueChanged<int> onSelect;

  const OrderListView({super.key, required this.selectedId, required this.onSelect});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: 20,
      itemBuilder: (context, index) {
        final bool isSelected = selectedId == index;
        return ListTile(
          selected: isSelected,
          title: Text('Đơn hàng #$index'),
          subtitle: const Text('Chờ thanh toán • 500.000 đ'),
          onTap: () => onSelect(index),
        );
      },
    );
  }
}

class OrderDetailView extends StatelessWidget {
  final int orderId;
  const OrderDetailView({super.key, required this.orderId});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Text(
        'Chi tiết đơn hàng #$orderId',
        style: Theme.of(context).textTheme.headlineMedium,
      ),
    );
  }
}
```

---

### 3.2 — Triển khai Adaptive Navigation Bar đa nền tảng

Tự động chuyển đổi cấu trúc điều hướng giữa các kích thước màn hình theo khuyến nghị của Material 3:

```dart
import 'package:flutter/material.dart';

class AdaptiveNavigationScaffold extends StatefulWidget {
  final List<NavigationDestination> destinations;
  final List<Widget> pages;

  const AdaptiveNavigationScaffold({
    super.key,
    required this.destinations,
    required this.pages,
  });

  @override
  State<AdaptiveNavigationScaffold> createState() => _AdaptiveNavigationScaffoldState();
}

class _AdaptiveNavigationScaffoldState extends State<AdaptiveNavigationScaffold> {
  int _currentIndex = 0;

  @override
  Widget build(BuildContext context) {
    final double width = MediaQuery.sizeOf(context).width;

    // Compact: NavigationBar ở đáy màn hình
    if (width < 600.0) {
      return Scaffold(
        body: widget.pages[_currentIndex],
        bottomNavigationBar: NavigationBar(
          selectedIndex: _currentIndex,
          onDestinationSelected: (idx) => setState(() => _currentIndex = idx),
          destinations: widget.destinations,
        ),
      );
    }

    // Medium: NavigationRail dạng cột hẹp bên trái
    if (width < 840.0) {
      return Scaffold(
        body: Row(
          children: [
            NavigationRail(
              selectedIndex: _currentIndex,
              onDestinationSelected: (idx) => setState(() => _currentIndex = idx),
              labelType: NavigationRailLabelType.selected,
              destinations: widget.destinations
                  .map((d) => NavigationRailDestination(
                        icon: d.icon,
                        selectedIcon: d.selectedIcon,
                        label: Text(d.label),
                      ))
                  .toList(),
            ),
            const VerticalDivider(width: 1.0),
            Expanded(child: widget.pages[_currentIndex]),
          ],
        ),
      );
    }

    // Expanded: NavigationDrawer mở rộng có đầy đủ nhãn văn bản
    return Scaffold(
      body: Row(
        children: [
          NavigationDrawer(
            selectedIndex: _currentIndex,
            onDestinationSelected: (idx) => setState(() => _currentIndex = idx),
            children: [
              const Padding(
                padding: EdgeInsets.all(16.0),
                child: Text('Hệ Thống Quản Trị', style: TextStyle(fontWeight: FontWeight.bold)),
              ),
              ...widget.destinations.map(
                (d) => NavigationDrawerDestination(
                  icon: d.icon,
                  selectedIcon: d.selectedIcon,
                  label: Text(d.label),
                ),
              ),
            ],
          ),
          const VerticalDivider(width: 1.0),
          Expanded(child: widget.pages[_currentIndex]),
        ],
      ),
    );
  }
}
```

---

## Phần 4 — Lỗi Thường Gặp & Giải Pháp Khắc Phục (Anti-Patterns & Pitfalls)

### 4.1 — Sử dụng `MediaQuery.of(context)` tại các node lá của Widget Tree

#### Mô tả lỗi:
Chèn lời gọi `MediaQuery.of(context).size` vào các widget hiển thị sâu bên trong cây (như Button, ListTile, Card). Khi người dùng chạm vào một trường nhập liệu `TextField`, bàn phím ảo trượt lên khiến toàn bộ cây giao diện bị lag và giật khung hình.

#### Giải pháp:
Thay thế ngay lập tức bằng `MediaQuery.sizeOf(context)` hoặc chuyển logic tính toán kích thước ra node cha ở tầng Route và truyền dữ liệu thuần túy xuống thông qua tham số constructor.

---

### 4.2 — Hardcode kích thước văn bản và không xử lý `TextScaler`

#### Mô tả lỗi:
Gán chiều cao cố định cho một container chứa văn bản (`SizedBox(height: 40, child: Text(...))`). Khi người dùng là người khiếm thị kích hoạt tính năng Accessibility của hệ điều hành, phóng to font chữ lên $1.5\times$ hoặc $2.0\times$, văn bản lập tức bị tràn và xuất hiện sọc vàng đen `A RenderFlex overflowed...`.

#### Giải pháp:
1. Tránh hardcode `height` trên các container chứa văn bản tự do.
2. Kiểm tra giới hạn co giãn với `textScaler`:
   ```dart
   Text(
     'Văn bản an toàn',
     textScaler: MediaQuery.textScalerOf(context).clamp(minScaleFactor: 1.0, maxScaleFactor: 1.3),
   )
   ```

---

## Phần 5 — Câu Hỏi Kiểm Tra Kiến Thức Chuyên Sâu & Bài Tập Phân Tích Mã Nguồn (Technical Assessment & Code Tracing)

### 5.1 — Câu hỏi khảo sát kiến trúc

#### Câu 1: Cơ chế hoạt động của `InheritedModel` trong `MediaQuery`
*Đề bài:* Phân tích cơ chế nội bộ của `InheritedModel.inheritFrom(context, aspect: ...)` cho phép framework lọc bớt các thông báo thay đổi (Notification Dispatching). Tại sao việc chia nhỏ thành `MediaQuery.sizeOf` lại tối ưu hơn về mặt cấu trúc dữ liệu so với một `InheritedWidget` truyền thống?

*Phân tích kỹ thuật:*
1. Trong một `InheritedWidget` thông thường, khi đối tượng dữ liệu thay đổi, phương thức `updateShouldNotify()` trả về `true` sẽ đưa **toàn bộ mọi Element** đã từng gọi `dependOnInheritedWidgetOfExactType()` vào danh sách `_dirtyElements` của `BuildOwner`.
2. `InheritedModel` mở rộng cơ chế này bằng cách gán cho mỗi Element phụ thuộc một tập hợp các `aspect` (nhãn khía cạnh quan tâm).
3. Trong phương thức `updateShouldNotifyDependent(InheritedModel oldWidget, Set<Object> dependencies)`:
   - Framework so sánh từng trường dữ liệu cụ thể.
   - Nếu chỉ có trường `viewInsets` thay đổi (do bàn phím), framework chỉ đánh dấu bẩn các Element có `dependencies.contains(_MediaQueryAspect.viewInsets)`.
   - Các Element chỉ đăng ký `_MediaQueryAspect.size` hoàn toàn bị bỏ qua, không bị thêm vào hàng đợi build.

---

#### Câu 2: Tác động của `LayoutBuilder` lên Pipeline Bố Cục
*Đề bài:* Tại sao việc lạm dụng `LayoutBuilder` ở quá nhiều cấp con lồng nhau sâu có thể gây suy giảm hiệu năng kết xuất so với việc truyền tham số tĩnh?

*Phân tích kỹ thuật:*
1. `LayoutBuilder` ngắt nhịp đường ống xử lý thông thường: Nó trì hoãn pha Build của một cây con cho đến tận khi Render Tree thực thi phương thức `performLayout()` của `RenderConstrainedLayoutBuilder`.
2. Việc triệu hồi `builder(context, constraints)` ngay bên trong pha Layout đòi hỏi Engine phải tạm dừng chu trình xử lý hình học để quay lại thực thi Dart code của hàm build.
3. Khi lồng ghép nhiều tầng `LayoutBuilder`, framework phải liên tục chuyển đổi ngữ cảnh giữa Layout Pass và Build Pass, làm mất đi khả năng tối ưu hóa song song và làm tăng thời gian xử lý khung hình của CPU.

---

### 5.2 — Bài tập phân tích luồng thực thi (Code Tracing)

#### Đề bài:
Cho cấu trúc Widget Tree sau:

```dart
class RootScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          HeaderTitleWidget(),     // (Node A)
          ContentGridWidget(),      // (Node B)
          FooterInputWidget(),      // (Node C)
        ],
      ),
    );
  }
}

class HeaderTitleWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Lời gọi 1: Đọc size
    final Size size = MediaQuery.sizeOf(context);
    return Container(width: size.width, height: 60.0, child: Text('Header'));
  }
}

class ContentGridWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Lời gọi 2: Đọc orientation
    final Orientation orientation = MediaQuery.orientationOf(context);
    return Expanded(
      child: Center(child: Text('Chế độ: $orientation')),
    );
  }
}

class FooterInputWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Lời gọi 3: Đọc viewInsets bàn phím
    final EdgeInsets insets = MediaQuery.viewInsetsOf(context);
    return Container(
      padding: EdgeInsets.only(bottom: insets.bottom),
      child: const TextField(),
    );
  }
}
```

Giả sử người dùng đang mở ứng dụng ở chế độ màn hình dọc (Portrait), sau đó chạm tay vào `TextField` tại `FooterInputWidget` làm bàn phím ảo xuất hiện trên màn hình (thay đổi `viewInsets.bottom` từ $0.0 \to 300.0$, chiều rộng và chiều cao cửa sổ không đổi).

Hãy xác định chính xác:
1. Node nào trong số các Node A, Node B, Node C sẽ bị kích hoạt lại phương thức `build()`?
2. Node nào được framework bỏ qua hoàn toàn?
3. Giải thích cơ chế nội bộ của `InheritedModel` đã quyết định hành vi trên như thế nào.

---

#### Đáp án phân tích:

**1. Các Node bị kích hoạt lại phương thức `build()`:**
- **Duy nhất Node C (`FooterInputWidget`)** bị kích hoạt lại hàm `build()`.
- *Nguyên nhân:* Node C sử dụng `MediaQuery.viewInsetsOf(context)`. Khi bàn phím xuất hiện, `viewInsets.bottom` thay đổi từ $0.0$ lên $300.0$. `InheritedModel` phát hiện khía cạnh phụ thuộc `_MediaQueryAspect.viewInsets` của Node C bị kích hoạt, do đó chỉ đưa Node C vào danh sách cần rebuild.

**2. Các Node được bỏ qua hoàn toàn:**
- **Node A (`HeaderTitleWidget`)** và **Node B (`ContentGridWidget`)** hoàn toàn **KHÔNG bị rebuild**.
- Cả `RootScreen` cũng **KHÔNG bị rebuild**.

**3. Cơ chế giải thích nội bộ:**
- Node A đăng ký aspect: `_MediaQueryAspect.size`. Vì kích thước tổng thể của cửa sổ ứng dụng không đổi, `size` không đổi $\longrightarrow$ Bỏ qua Node A.
- Node B đăng ký aspect: `_MediaQueryAspect.orientation`. Thiết bị vẫn duy trì hướng dọc (Portrait) $\longrightarrow$ Bỏ qua Node B.
- Đây chính là minh chứng rõ ràng nhất cho sức mạnh của mô hình Fine-Grained Aspects trong Flutter hiện đại: Việc bàn phím ảo xuất hiện chỉ làm tiêu tốn CPU để render lại đúng khu vực nhập liệu, bảo vệ toàn bộ phần còn lại của giao diện không bị giật lag.
