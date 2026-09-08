# Bài 4.2 — BoxConstraints: Tight, Loose, Bounded & Unbounded

## Phần 1 — Khái Niệm & Ràng Buộc Kiến Trúc (Architecture & Design Philosophy)

### 1.1 — Mô hình toán học của BoxConstraints

Trong hệ thống kết xuất hai chiều của Flutter, lớp `BoxConstraints` định nghĩa một không gian trạng thái hình học đóng vai trò là ranh giới kích thước mà một `RenderBox` bắt buộc phải tuân thủ. Về mặt giải tích, `BoxConstraints` là tích Descartes của hai đoạn số thực trên trục hoành ($W$) và trục tung ($H$):

$$\mathbb{C} = [minWidth, maxWidth] \times [minHeight, maxHeight]$$

Trong đó, một `RenderBox` có quyền tự do lựa chọn một điểm kích thước $(w, h) \in \mathbb{R}^2$ khi và chỉ khi điểm đó thỏa mãn đồng thời hai hệ bất đẳng thức:

$$0.0 \le minWidth \le w \le maxWidth \le \infty$$

$$0.0 \le minHeight \le h \le maxHeight \le \infty$$

Dựa trên cấu trúc biên của tập hợp $\mathbb{C}$, Flutter Framework phân loại `BoxConstraints` thành 4 trạng thái hình học cơ bản:

```
┌────────────────────────────────────────────────────────────────────────┐
│ PHÂN LOẠI HÌNH HỌC CỦA BOXCONSTRAINTS                                  │
├────────────────────────────────────────────────────────────────────────┤
│ 1. TIGHT CONSTRAINTS                                                   │
│    minWidth == maxWidth  VÀ  minHeight == maxHeight                    │
│    └─► Không gian suy biến thành một điểm duy nhất. Bậc tự do = 0.     │
│        RenderBox con bị ép buộc lấy chính xác kích thước này.          │
├────────────────────────────────────────────────────────────────────────┤
│ 2. LOOSE CONSTRAINTS                                                   │
│    minWidth == 0.0  VÀ  minHeight == 0.0                               │
│    └─► RenderBox con có toàn quyền co giãn từ kích thước cực tiểu      │
│        (0x0) đến giới hạn trần khả dụng của cha (maxWidth x maxHeight).│
├────────────────────────────────────────────────────────────────────────┤
│ 3. BOUNDED CONSTRAINTS                                                 │
│    maxWidth < infinity  VÀ  maxHeight < infinity                       │
│    └─► Cả hai chiều đều có cận trên hữu hạn. Cho phép tính toán tọa độ  │
│        cố định và căn chỉnh lề an toàn.                                │
├────────────────────────────────────────────────────────────────────────┤
│ 4. UNBOUNDED CONSTRAINTS                                               │
│    maxWidth == infinity  HOẶC  maxHeight == infinity                   │
│    └─► Không có giới hạn trần trên ít nhất một trục. RenderBox con     │
│        tự do mở rộng vô hạn theo trục đó (thường gặp trong Scroll).    │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 1.2 — Bất biến chuẩn hóa (Normalization Invariants)

Trong mã nguồn Flutter Engine và Framework, một `BoxConstraints` hợp lệ bắt buộc phải thỏa mãn tính chất **chuẩn hóa (normalized)**. Thuộc tính `isNormalized` trong `packages/flutter/lib/src/rendering/box.dart` kiểm tra bất biến này thông qua các assertion nghiêm ngặt:

```dart
bool get isNormalized {
  return minWidth >= 0.0 &&
         minWidth <= maxWidth &&
         minHeight >= 0.0 &&
         minHeight <= maxHeight;
}
```

Nếu một RenderObject tính toán và truyền xuống một đối tượng `BoxConstraints` có giá trị âm hoặc có cận dưới lớn hơn cận trên ($min > max$), framework sẽ lập tức ngắt pipeline và ném ra ngoại lệ `FlutterError`.

---

## Phần 2 — Cơ Chế Hoạt Động & Mã Nguồn Đối Chiếu (Under the Hood / Deep-Dive)

### 2.1 — Giải phẫu các phương thức biến đổi trong `box.dart`

Toàn bộ các phép biến đổi không gian ràng buộc đều được tối ưu hóa thành các hàm toán học nguyên tử bên trong `packages/flutter/lib/src/rendering/box.dart`:

#### 1. Hàm tạo Tight và Loose:
```dart
// Khóa cứng không gian thành một điểm kích thước cố định
BoxConstraints.tight(Size size)
  : minWidth = size.width,
    maxWidth = size.width,
    minHeight = size.height,
    maxHeight = size.height;

// Giải phóng cận dưới về 0, giữ nguyên cận trên
BoxConstraints loosen() {
  assert(isNormalized);
  return BoxConstraints(
    minWidth: 0.0,
    maxWidth: maxWidth,
    minHeight: 0.0,
    maxHeight: maxHeight,
  );
}
```

#### 2. Phép chiếu không gian `constrain()`:
Khi một RenderBox tự tính toán kích thước mong muốn (`Size size`), nó bắt buộc phải đưa kích thước đó qua hàm `constrain()` để ép kích thước rơi vào miền cho phép:

```dart
Size constrain(Size size) {
  Size result = Size(constrainWidth(size.width), constrainHeight(size.height));
  assert(isSatisfiedBy(result));
  return result;
}

double constrainWidth([ double width = double.infinity ]) {
  assert(isNormalized);
  return clampDouble(width, minWidth, maxWidth);
}
```

Về mặt toán học, `constrain()` là phép chiếu trực giao (orthogonal projection) của một điểm $(w, h)$ bất kỳ lên tập lồi $\mathbb{C}$.

#### 3. Phép giao ràng buộc `enforce()`:
Khi hai bộ ràng buộc từ hai tầng widget lồng nhau tương tác, `enforce()` thực hiện phép giao hình học giữa hai miền:

```dart
BoxConstraints enforce(BoxConstraints ancestor) {
  return BoxConstraints(
    minWidth: clampDouble(minWidth, ancestor.minWidth, ancestor.maxWidth),
    maxWidth: clampDouble(maxWidth, ancestor.minWidth, ancestor.maxWidth),
    minHeight: clampDouble(minHeight, ancestor.minHeight, ancestor.maxHeight),
    maxHeight: clampDouble(maxHeight, ancestor.minHeight, ancestor.maxHeight),
  );
}
```

---

### 2.2 — Căn nguyên kiến trúc của các ngoại lệ Layout kinh điển

#### 1. Ngoại lệ: `A RenderFlex overflowed by X pixels`
```
════╡ EXCEPTION CAUGHT BY RENDERING LIBRARY ╞═════════════════════════════════
The following assertion was thrown during performLayout():
A RenderFlex overflowed by 42 pixels on the right.
```
* **Căn nguyên kiến trúc:** 
  - `RenderFlex` (nền tảng của `Row` và `Column`) nhận một bounded constraint từ cha (ví dụ $maxWidth = 400$).
  - Khi duyệt qua các con không có `Flexible` hoặc `Expanded`, `RenderFlex` truyền cho chúng một **unbounded constraint** trên trục chính (`maxWidth = double.infinity`).
  - Mỗi phần tử con tự do báo cáo kích thước tự nhiên của nó ($size_i$).
  - Sau khi cộng dồn: $\sum size_i > \text{maxAvailableSpace}$.
  - Do `RenderFlex` không có quyền tự ý cắt gọt (clip) kích thước con trừ khi được chỉ định rõ ràng, phần sai lệch vượt ngưỡng $X = \sum size_i - \text{maxAvailableSpace}$ được báo cáo lên và vẽ dải sọc vàng đen cảnh báo.

#### 2. Ngoại lệ: `Vertical viewport was given unbounded height`
```
════╡ EXCEPTION CAUGHT BY RENDERING LIBRARY ╞═════════════════════════════════
Vertical viewport was given unbounded height.
Viewports expand in the scrolling direction to fill their container.
```
* **Căn nguyên kiến trúc:**
  - `RenderViewport` (nền tảng của `ListView`, `GridView`) được thiết kế để mở rộng kích thước vô hạn theo trục cuộn nhằm hiển thị một cửa sổ trượt trên một danh sách dài.
  - Tuy nhiên, để xác định được khung nhìn hiển thị (Viewport Box), bản thân `RenderViewport` **bắt buộc phải nhận được một bounded constraint từ cha trên trục cuộn**:
    ```dart
    // Mã nguồn kiểm tra assertion trong RenderViewport.performLayout():
    assert(constraints.hasBoundedHeight, 'Vertical viewport was given unbounded height.');
    ```
  - Khi đặt `ListView` trực tiếp bên trong `Column`, `Column` truyền cho các con của nó một `maxHeight = double.infinity`. Khi `RenderViewport` nhận được ràng buộc vô hạn này, assertion kích hoạt và chương trình bị crash ngay lập tức.

---

### 2.3 — Ma trận chuyển đổi ràng buộc của các Widget nền tảng

| Widget | Ràng buộc nhận từ cha ($\mathbb{C}_{in}$) | Ràng buộc truyền cho con ($\mathbb{C}_{out}$) | Kích thước tự thân báo cáo lên cha ($Size$) |
| :--- | :--- | :--- | :--- |
| **`Scaffold.body`** | Tight / Bounded (Màn hình) | Tight (Đã trừ AppBar, BottomBar) | Khớp chính xác với kích thước màn hình khả dụng. |
| **`Center` / `Align`** | Bất kỳ | `constraints.loosen()` ($\min W = 0, \min H = 0$) | Nhận kích thước lớn nhất có thể của cha (`constraints.biggest`). |
| **`SizedBox(w, h)`** | Bất kỳ | `constraints.enforce(BoxConstraints.tightFor(w, h))` | Khớp với $w, h$ sau khi đã chiếu qua $\mathbb{C}_{in}$. |
| **`UnconstrainedBox`** | Bất kỳ | `BoxConstraints()` (Hoàn toàn Unbounded $0..\infty$) | Thu nhỏ theo kích thước con, cho phép con vẽ tràn. |
| **`FractionallySizedBox`**| Bounded | Tight constraint theo tỷ lệ phần trăm của cha | Khớp chính xác với kích thước đã nhân tỷ lệ. |
| **`Container()` (Rỗng)** | Bất kỳ | Không có con | Nếu Tight $\to$ Lấy max; Nếu Loose $\to$ Lấy min ($0 \times 0$). |

---

## Phần 3 — Hướng Dẫn Thực Hành Chuẩn (Production-Ready Implementations)

### 3.1 — Thuần hóa Unbounded Constraints trong Flex Layout

Khi xây dựng giao diện với các cấu trúc lồng nhau (Row/Column chứa Text hoặc các widget co giãn), kỹ thuật tiêu chuẩn là biến đổi Unbounded Constraint thành Bounded / Tight Constraint bằng `Expanded` hoặc `Flexible`:

```dart
import 'package:flutter/material.dart';

class SafeHorizontalCard extends StatelessWidget {
  final String title;
  final String description;
  final VoidCallback onTap;

  const SafeHorizontalCard({
    super.key,
    required this.title,
    required this.description,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Row(
          children: [
            const Icon(Icons.info, size: 40.0),
            const SizedBox(width: 16.0),
            // Expanded chuyển đổi Unbounded width của Row thành Tight width cụ thể
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                mainAxisSize: MainAxisSize.min, // Thu nhỏ chiều cao theo nội dung
                children: [
                  Text(
                    title,
                    style: Theme.of(context).textTheme.titleMedium,
                    maxLines: 1,
                    overflow: TextOverflow.ellipsis,
                  ),
                  const SizedBox(height: 4.0),
                  Text(
                    description,
                    style: Theme.of(context).textTheme.bodySmall,
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                  ),
                ],
              ),
            ),
            const SizedBox(width: 8.0),
            IconButton(
              icon: const Icon(Icons.arrow_forward),
              onPressed: onTap,
            ),
          ],
        ),
      ),
    );
  }
}
```

---

### 3.2 — Kỹ thuật áp đặt ranh giới bảo vệ bằng `LimitedBox` và `ConstrainedBox`

Khi thiết kế các thành phần widget tái sử dụng (Reusable Components) có thể được nhúng vào cả môi trường Bounded (màn hình cố định) lẫn Unbounded (bên trong `ListView`), việc áp dụng `LimitedBox` giúp bảo vệ component không bị lỗi kích thước:

```dart
import 'package:flutter/widgets.dart';

class AdaptiveHeaderBanner extends StatelessWidget {
  final Widget child;

  const AdaptiveHeaderBanner({super.key, required this.child});

  @override
  Widget build(BuildContext context) {
    // LimitedBox CHỈ áp đặt maxHeight khi và chỉ khi nhận maxHeight vô hạn từ cha
    // Nếu cha đã truyền maxHeight hữu hạn, LimitedBox hoàn toàn vô hiệu hóa
    return LimitedBox(
      maxHeight: 250.0,
      child: ConstrainedBox(
        constraints: const BoxConstraints(
          minHeight: 100.0,
        ),
        child: child,
      ),
    );
  }
}
```

---

## Phần 4 — Lỗi Thường Gặp & Giải Pháp Khắc Phục (Anti-Patterns & Pitfalls)

### 4.1 — Đặt ListView bên trong Column mà không xác lập ranh giới Bounded

#### Mô tả lỗi:
Chương trình sụp đổ ngay khi khởi tạo với lỗi `Vertical viewport was given unbounded height`.

```dart
// SAI LẦM PHỔ BIẾN
Widget build(BuildContext context) {
  return Scaffold(
    body: Column(
      children: [
        const HeaderWidget(),
        ListView.builder( // LỖI CRASH: Column truyền maxHeight: infinity
          itemCount: 50,
          itemBuilder: (context, index) => ListTile(title: Text('Item $index')),
        ),
      ],
    ),
  );
}
```

#### Nguyên nhân kỹ thuật:
`Column` mặc định tính toán kích thước bằng cách cho phép các con mở rộng tự do trên trục tung (`maxHeight = infinity`). `ListView` là một Scrollable Viewport yêu cầu `maxHeight` phải có giới hạn xác định để thiết lập không gian cuộn.

#### Giải pháp khắc phục:

* **Giải pháp 1: Sử dụng `Expanded` (Nếu danh sách cuộn độc lập với Header):**
  `Expanded` can thiệp vào pha tính toán của `RenderFlex`, chiếm toàn bộ không gian còn lại của `Column` và áp đặt một Tight Constraint xác định cho `ListView`.

```dart
Expanded(
  child: ListView.builder(
    itemCount: 50,
    itemBuilder: (context, index) => ListTile(title: Text('Item $index')),
  ),
)
```

* **Giải pháp 2: Vô hiệu hóa tính chất Scroll của ListView với `shrinkWrap` (Nếu danh sách ngắn):**
  Chỉ định `shrinkWrap: true` và `physics: const NeverScrollableScrollPhysics()`. Khi đó, `RenderViewport` chuyển đổi hành vi đo đạc để co cụm vừa khít tổng chiều cao của các con.
  *(Lưu ý: Giải pháp này làm mất tính năng Virtualization, không sử dụng cho danh sách dài).*

* **Giải pháp 3: Tái cấu trúc thành `CustomScrollView` với `SliverToBoxAdapter`:**
  Đưa cả Header và List về cùng một hệ điều phối cuộn thống nhất trên Render Tree.

---

### 4.2 — Sử dụng `SizedBox.expand()` hoặc `Spacer()` trong ngữ cảnh Unbounded

#### Mô tả lỗi:
Ném ra ngoại lệ `BoxConstraints forces an infinite width / height` trong console.

```dart
// SAI LẦM
Row(
  children: [
    const Text('Start'),
    Spacer(), // LỖI: Nếu Row nằm trong một horizontal scroll view (Unbounded width)
    const Text('End'),
  ],
)
```

#### Nguyên nhân kỹ thuật:
`Spacer` thực chất là một `Expanded(child: SizedBox())`. Khi `Row` nằm trong một vùng chứa có `maxWidth = infinity` (ví dụ `SingleChildScrollView(scrollDirection: Axis.horizontal)`), `freeSpace` là vô hạn. Phép nhân tỷ lệ flex trên một giá trị vô hạn dẫn đến việc gán constraint vô hạn cho `SizedBox`, vi phạm bất biến của framework.

---

## Phần 5 — Câu Hỏi Kiểm Tra Kiến Thức Chuyên Sâu & Bài Tập Phân Tích Mã Nguồn (Technical Assessment & Code Tracing)

### 5.1 — Câu hỏi khảo sát kiến trúc

#### Câu 1: Phép toán hình học trong `BoxConstraints.enforce()`
*Đề bài:* Phân tích tình huống khi hai bộ ràng buộc không có giao điểm hình học: Giả sử widget con có mong muốn `BoxConstraints(minW: 200, maxW: 300)` nhưng widget cha áp đặt `BoxConstraints(minW: 50, maxW: 100)`. Phương thức `enforce()` xử lý xung đột này như thế nào và kết quả cuối cùng là gì?

*Phân tích kỹ thuật:*
1. Dựa vào mã nguồn của `enforce()`:
   $$\text{minWidth} = \text{clampDouble}(200.0, 50.0, 100.0) = 100.0$$
   $$\text{maxWidth} = \text{clampDouble}(300.0, 50.0, 100.0) = 100.0$$
2. Kết quả thu được là một Tight Constraint: `BoxConstraints(minW: 100.0, maxW: 100.0)`.
3. Bất biến kiến trúc: **Quyền lực của tổ tiên là tuyệt đối.** Nếu mong muốn của con vượt quá trần của cha, con bị ép buộc thu nhỏ về cận trên của cha. Ràng buộc sau phép `enforce()` luôn được đảm bảo chuẩn hóa và nằm trọn vẹn bên trong không gian của cha.

---

#### Câu 2: Hành vi của `Container` rỗng trong các môi trường khác nhau
*Đề bài:* Tại sao cùng một khai báo `Container(color: Colors.red)` lại có kích thước chiếm toàn bộ màn hình khi là con trực tiếp của `Scaffold.body`, nhưng lại biến mất hoàn toàn ($0 \times 0$) khi đặt bên trong `Center`?

*Phân tích kỹ thuật:*
1. Mã nguồn của `Container` ủy quyền logic định cỡ cho `DecoratedBox` và `ConstrainedBox`.
2. Khi không có tham số `width`, `height` và không có `child`:
   - `Container` cố gắng mở rộng lớn nhất có thể nếu ràng buộc từ cha là Bounded/Tight: `size = constraints.biggest`.
   - `Container` co về nhỏ nhất nếu ràng buộc là Loose: `size = constraints.smallest`.
3. Trong `Scaffold.body`: Cha truyền Tight Constraint kích thước màn hình ($W_{screen} \times H_{screen}$) $\to$ `constraints.biggest` chính là kích thước toàn màn hình.
4. Trong `Center`: `Center` thực hiện phương thức `constraints.loosen()`, biến cận dưới về $0.0$. Do không có child nào bên trong để đẩy kích thước lên, `Container` rơi về trạng thái `constraints.smallest` là $(0.0, 0.0)$.

---

### 5.2 — Bài tập phân tích luồng thực thi (Code Tracing)

#### Đề bài:
Cho cây widget lồng nhau dưới đây trên màn hình thiết bị có kích thước vật lý $400 \times 800$:

```dart
Scaffold(
  body: Center(                                            // (Node 1)
    child: UnconstrainedBox(                               // (Node 2)
      child: SizedBox(                                     // (Node 3)
        width: 500,
        height: 100,
        child: Container(color: Colors.amber),
      ),
    ),
  ),
)
```

Hãy xác định:
1. `BoxConstraints` mà từng Node (từ 1 đến 3) truyền cho con trực tiếp của nó.
2. `Size` mà Node 3 báo cáo lên Node 2, và `Size` mà Node 2 báo cáo lên Node 1.
3. Hiện tượng gì sẽ xảy ra trên giao diện khi render? Ứng dụng có bị crash bởi assertion không?

---

#### Đáp án phân tích:

**1. Chuỗi truyền BoxConstraints:**
- **Node 1 (`Center`):** 
  - Nhận từ Scaffold: Tight constraint `BoxConstraints(w: 400..400, h: 800..800)`.
  - Truyền cho Node 2 (`UnconstrainedBox`): Loose constraint `BoxConstraints(w: 0..400, h: 0..800)` sau khi gọi `loosen()`.
- **Node 2 (`UnconstrainedBox`):**
  - Bỏ qua toàn bộ ràng buộc của cha, truyền cho Node 3 (`SizedBox`): Unbounded constraint hoàn toàn `BoxConstraints(w: 0..∞, h: 0..∞)`.
- **Node 3 (`SizedBox`):**
  - Nhận $0..\infty$, thực hiện ép kiểu tight cho con: `BoxConstraints(w: 500..500, h: 100..100)`.

**2. Kích thước (Sizes Go Up):**
- Node 3 (`SizedBox`) chọn kích thước chính xác: `Size(500.0, 100.0)` và trả về cho Node 2.
- Node 2 (`UnconstrainedBox`) lấy kích thước bằng kích thước của con nhưng ép vào giới hạn của cha: Nó báo cáo kích thước `Size(400.0, 100.0)` lên Node 1 (hoặc `Size(500.0, 100.0)` kèm cờ overflow tùy phiên bản render).

**3. Hiện tượng hiển thị trên màn hình:**
- Chiều rộng của con ($500px$) lớn hơn chiều rộng tối đa khả dụng của màn hình ($400px$).
- **Ứng dụng không bị crash**, vì `UnconstrainedBox` được thiết kế đặc thù để cho phép con layout với kích thước tự nhiên vượt ranh giới cha.
- Tuy nhiên, trong chế độ Debug, `UnconstrainedBox` phát hiện con tràn ra ngoài vùng hiển thị của nó và vẽ **dải sọc vàng đen (Overflow stripes)** cảnh báo tràn 100 pixel bên phải màn hình.
