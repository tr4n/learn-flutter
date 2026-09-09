# Bài 8.1 — Implicit Animations & Cơ Chế ImplicitlyAnimatedWidget

## Tài Liệu Tham Khảo Chính Thức
- [Flutter Documentation: Implicit animations](https://docs.flutter.dev/ui/animations/implicit-animations)
- [Flutter API: ImplicitlyAnimatedWidget](https://api.flutter.dev/flutter/widgets/ImplicitlyAnimatedWidget-class.html)
- [Flutter API: AnimatedContainer](https://api.flutter.dev/flutter/widgets/AnimatedContainer-class.html)
- [Flutter API: TweenAnimationBuilder](https://api.flutter.dev/flutter/widgets/TweenAnimationBuilder-class.html)
- [Flutter API: AnimatedSwitcher](https://api.flutter.dev/flutter/widgets/AnimatedSwitcher-class.html)
- [Material Design 3: Motion system](https://m3.material.io/styles/motion/overview)

---

## Phần 1 — Khái Niệm & Phân Loại Animation Trong Flutter

### 1.1 — Triết Lý Thiết Kế: Implicit vs Explicit Animations

Trong Flutter framework, hệ thống hoạt họa (Animation System) được chia thành hai nhánh chính dựa trên mức độ kiểm soát:

1. **Implicit Animations (Hoạt họa ngầm định)**:
   - **Triết lý**: Lập trình viên chỉ cần chỉ định giá trị đích (Target Value), thời lượng (`duration`) và đường cong tốc độ (`curve`). Framework sẽ tự động quản lý vòng đời bộ điều khiển, tính toán giá trị nội suy giữa giá trị cũ và mới.
   - **Đặc điểm**: Đóng gói hoàn chỉnh, không cần quản lý `AnimationController`, không cần thêm mixin `TickerProvider`, và tự động giải phóng tài nguyên khi widget bị hủy.
   - **Ví dụ**: `AnimatedContainer`, `AnimatedOpacity`, `AnimatedPadding`, `AnimatedAlign`, `AnimatedPositioned`, `AnimatedSwitcher`.

2. **Explicit Animations (Hoạt họa tường minh)**:
   - **Triết lý**: Lập trình viên trực tiếp khởi tạo và điều khiển một `AnimationController`, tự quản lý các lệnh `forward()`, `reverse()`, `repeat()`, `stop()`.
   - **Đặc điểm**: Cung cấp khả năng kiểm soát tuyệt đối trên từng khung hình, hỗ trợ điều phối nhiều hoạt họa song song hoặc nối tiếp, nhưng đòi hỏi phải quản lý giải phóng tài nguyên (`dispose()`) thủ công.
   - **Ví dụ**: `RotationTransition`, `ScaleTransition`, `SlideTransition`, `AnimatedBuilder`.

#### Bảng so sánh đặc tính kỹ thuật:

| Tiêu Chí Kỹ Thuật | Implicit Animations | Explicit Animations |
| :--- | :--- | :--- |
| **Quản lý AnimationController** | Framework tự quản lý nội bộ | Lập trình viên khởi tạo và quản lý |
| **Yêu cầu `dispose()`** | Không (tự động dọn dẹp) | Bắt buộc gọi `controller.dispose()` |
| **Độ phức tạp mã nguồn** | Thấp (chỉ cần đổi thuộc tính và gọi `setState`) | Trung bình đến cao |
| **Khả năng lặp vô hạn / chạy ngược** | Hạn chế | Hỗ trợ qua `repeat()`, `reverse()` |
| **Điều phối nhiều animation** | Khó đồng bộ chính xác | Hỗ trợ qua `Interval` hoặc `TweenSequence` |
| **Phù hợp với** | Chuyển đổi trạng thái giao diện UI đơn giản | Hiệu ứng loading, game UI, thao tác cử chỉ kéo thả |

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Cơ Chế Hoạt Động Của `ImplicitlyAnimatedWidget`

Tất cả các widget implicit animation trong Flutter đều kế thừa từ lớp trừu tượng `ImplicitlyAnimatedWidget` và được quản lý bởi `ImplicitlyAnimatedWidgetState<T>`:

```
┌────────────────────────────────────────────────────────────────────────┐
│ VÒNG ĐỜI NỘI BỘ CỦA IMPLICITLYANIMATEDWIDGET                           │
│                                                                        │
│ 1. setState() làm thay đổi giá trị thuộc tính (ví dụ: width từ 100 -> 200)│
│      │                                                                 │
│      ▼                                                                 │
│ 2. didUpdateWidget(oldWidget) được framework gọi                       │
│    • So sánh giá trị thuộc tính giữa oldWidget và newWidget            │
│    • Gọi phương thức forEachTween()                                    │
│      │                                                                 │
│      ▼                                                                 │
│ 3. Cập nhật Tween:                                                     │
│    • Tween.begin = Giá trị hiện tại tại thời điểm thay đổi             │
│    • Tween.end   = Giá trị mới của newWidget                           │
│      │                                                                 │
│      ▼                                                                 │
│ 4. Khởi động AnimationController nội bộ:                               │
│    • Controller được reset và chạy từ 0.0 -> 1.0                       │
│    • Mỗi nhịp VSync, Ticker kích hoạt hàm lerp()                       │
│    • Widget tự vẽ lại với giá trị nội suy cho đến khi kết thúc         │
└────────────────────────────────────────────────────────────────────────┘
```

Phương thức quan trọng nhất trong `ImplicitlyAnimatedWidgetState` là `forEachTween`:
```dart
@override
void forEachTween(TweenVisitor<dynamic> visitor) {
  _widthTween = visitor(
    _widthTween,
    widget.width,
    (dynamic value) => Tween<double>(begin: value as double),
  ) as Tween<double>?;
}
```
Khi widget rebuild, `visitor` kiểm tra xem giá trị đích có thay đổi so với giá trị hiện tại hay không. Nếu có, nó sẽ thiết lập lại điểm bắt đầu (`begin`) là giá trị đang hiển thị và điểm kết thúc (`end`) là giá trị mới, đảm bảo hiệu ứng chuyển động diễn ra liền mạch ngay cả khi người dùng thay đổi trạng thái liên tục trước khi animation trước đó kịp kết thúc.

---

### 2.2 — Cơ Chế Nhận Diện Widget Của `AnimatedSwitcher`

`AnimatedSwitcher` thực hiện hiệu ứng chuyển cảnh (mặc định là Cross-fade) giữa hai widget con khác nhau. Cơ chế nhận diện widget mới dựa trên phương thức tĩnh `Widget.canUpdate`:

```dart
static bool canUpdate(Widget oldWidget, Widget newWidget) {
  return oldWidget.runtimeType == newWidget.runtimeType
      && oldWidget.key == newWidget.key;
}
```

- Nếu `canUpdate` trả về `true`: Framework coi đó là **cùng một widget** vừa được cập nhật thuộc tính $\to$ Không kích hoạt chuyển cảnh.
- Nếu `canUpdate` trả về `false`: Framework coi đó là **hai widget riêng biệt** $\to$ Đưa widget cũ vào hiệu ứng thoát dần (Exit Transition) và đưa widget mới vào hiệu ứng xuất hiện (Entry Transition).

> **Hệ quả**: Khi chuyển đổi giữa hai widget có cùng kiểu dữ liệu (ví dụ từ `Text('A')` sang `Text('B')`), lập trình viên bắt buộc phải gán `Key` (như `ValueKey('A')` và `ValueKey('B')`) để `canUpdate` trả về `false`, từ đó kích hoạt chuyển cảnh.

---

### 2.3 — Tiêu Chuẩn Material 3 Motion System

Theo hướng dẫn chuyển động của Material Design 3, các hiệu ứng thị giác cần tuân thủ các quy tắc thời lượng và đường cong tốc độ để đảm bảo tính tự nhiên:

1. **Thời lượng (Durations)**:
   - **Ngắn (Short: 50ms – 200ms)**: Áp dụng cho các thành phần nhỏ như checkbox, icon chuyển đổi, tooltip.
   - **Trung bình (Medium: 250ms – 400ms)**: Áp dụng cho mở rộng thẻ (card expansion), hộp thoại (dialogs), bottom sheets.
   - **Dài (Long: 450ms – 700ms)**: Áp dụng cho chuyển trang toàn màn hình.

2. **Đường cong tốc độ (Curves)**:
   - **`Curves.easeInOutCubic` hoặc `Curves.fastOutSlowIn`**: Sử dụng cho các đối tượng di chuyển hoàn toàn bên trong khung nhìn.
   - **`Curves.easeOutCubic` (Decelerate)**: Sử dụng cho các đối tượng từ bên ngoài tiến vào khung nhìn.
   - **`Curves.easeInCubic` (Accelerate)**: Sử dụng cho các đối tượng rời khỏi khung nhìn.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — `AnimatedContainer`: Hoạt Họa Đa Thuộc Tính

`AnimatedContainer` cho phép thay đổi kích thước, màu sắc, khoảng đệm (padding) và góc bo viền đồng thời chỉ với một lệnh `setState()`:

```dart
import 'package:flutter/material.dart';

class ExpandableCard extends StatefulWidget {
  const ExpandableCard({super.key});

  @override
  State<ExpandableCard> createState() => _ExpandableCardState();
}

class _ExpandableCardState extends State<ExpandableCard> {
  bool _isExpanded = false;

  void _toggleExpand() {
    setState(() {
      _isExpanded = !_isExpanded;
    });
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Center(
      child: GestureDetector(
        onTap: _toggleExpand,
        child: AnimatedContainer(
          duration: const Duration(milliseconds: 300),
          curve: Curves.fastOutSlowIn,
          width: _isExpanded ? 320.0 : 160.0,
          height: _isExpanded ? 200.0 : 90.0,
          padding: EdgeInsets.all(_isExpanded ? 20.0 : 12.0),
          decoration: BoxDecoration(
            color: _isExpanded
                ? theme.colorScheme.primaryContainer
                : theme.colorScheme.surfaceVariant,
            borderRadius: BorderRadius.circular(_isExpanded ? 24.0 : 12.0),
            boxShadow: [
              BoxShadow(
                color: Colors.black.withOpacity(_isExpanded ? 0.15 : 0.05),
                blurRadius: _isExpanded ? 16.0 : 6.0,
                offset: Offset(0, _isExpanded ? 8.0 : 2.0),
              ),
            ],
          ),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Text(
                    _isExpanded ? 'Chi tiết thông tin' : 'Thu gọn',
                    style: theme.textTheme.titleMedium,
                  ),
                  AnimatedRotation(
                    turns: _isExpanded ? 0.5 : 0.0, // Xoay 180 độ
                    duration: const Duration(milliseconds: 300),
                    child: const Icon(Icons.keyboard_arrow_down),
                  ),
                ],
              ),
              if (_isExpanded) ...[
                const SizedBox(height: 12),
                const Text(
                  'Nội dung bổ sung được hiển thị mượt mà khi thẻ mở rộng.',
                  style: TextStyle(fontSize: 13),
                ),
              ],
            ],
          ),
        ),
      ),
    );
  }
}
```

---

### 3.2 — `AnimatedSwitcher`: Chuyển Cảnh Mượt Mà Giữa Các Widget

Sử dụng `AnimatedSwitcher` kết hợp với `ValueKey` để hoán đổi widget kèm hiệu ứng chuyển đổi tùy biến:

```dart
import 'package:flutter/material.dart';

class StatusSwitcher extends StatelessWidget {
  final bool isLoading;
  final VoidCallback onRetry;

  const StatusSwitcher({
    super.key,
    required this.isLoading,
    required this.onRetry,
  });

  @override
  Widget build(BuildContext context) {
    return AnimatedSwitcher(
      duration: const Duration(milliseconds: 250),
      // Tùy biến hiệu ứng: Kết hợp FadeTransition và ScaleTransition
      transitionBuilder: (Widget child, Animation<double> animation) {
        return FadeTransition(
          opacity: animation,
          child: ScaleTransition(
            scale: Tween<double>(begin: 0.85, end: 1.0).animate(animation),
            child: child,
          ),
        );
      },
      child: isLoading
          ? const SizedBox(
              key: ValueKey('loading_indicator'),
              width: 24,
              height: 24,
              child: CircularProgressIndicator(strokeWidth: 2.5),
            )
          : ElevatedButton.icon(
              key: const ValueKey('submit_button'),
              onPressed: onRetry,
              icon: const Icon(Icons.refresh),
              label: const Text('Tải lại dữ liệu'),
            ),
    );
  }
}
```

---

### 3.3 — `TweenAnimationBuilder`: Tạo Hoạt Họa Giá Trị Tùy Biến

Khi không có sẵn widget `Animated...` cho kiểu dữ liệu mong muốn (ví dụ: đếm số nguyên tăng dần từ 0 đến N), `TweenAnimationBuilder` cung cấp giải pháp chuyển động không cần khởi tạo `AnimationController`:

```dart
import 'package:flutter/material.dart';

class CounterTextAnimation extends StatelessWidget {
  final int targetValue;

  const CounterTextAnimation({super.key, required this.targetValue});

  @override
  Widget build(BuildContext context) {
    return TweenAnimationBuilder<double>(
      tween: Tween<double>(begin: 0, end: targetValue.toDouble()),
      duration: const Duration(milliseconds: 800),
      curve: Curves.easeOutExpo,
      builder: (BuildContext context, double value, Widget? child) {
        return Text(
          '${value.toInt()} VNĐ',
          style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                fontWeight: FontWeight.bold,
                color: Theme.of(context).colorScheme.primary,
              ),
        );
      },
    );
  }
}
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Quên khai báo `Key` trong `AnimatedSwitcher`

#### Mô tả vấn đề:
Thay đổi nội dung hiển thị trong `AnimatedSwitcher` nhưng không thấy hiệu ứng chuyển cảnh hoạt động:
```dart
// Lỗi: Không có key, AnimatedSwitcher coi là cùng một widget
AnimatedSwitcher(
  duration: const Duration(milliseconds: 300),
  child: Text(_isLoggedIn ? 'Xin chào' : 'Đăng nhập'),
)
```

#### Nguyên nhân kỹ thuật:
Cả hai trường hợp đều trả về widget có cùng `runtimeType` là `Text`. Hàm `Widget.canUpdate` trả về `true`, framework chỉ cập nhật chuỗi văn bản mới vào `Element` hiện tại mà không kích hoạt chu kỳ chuyển cảnh.

#### Biện pháp khắc phục:
Gán `ValueKey` phân biệt cho từng nhánh giao diện:
```dart
AnimatedSwitcher(
  duration: const Duration(milliseconds: 300),
  child: Text(
    _isLoggedIn ? 'Xin chào' : 'Đăng nhập',
    key: ValueKey<bool>(_isLoggedIn),
  ),
)
```

---

### 4.2 — Thiết lập thời lượng hoạt họa không phù hợp

#### Mô tả vấn đề:
Cài đặt `duration` quá dài (ví dụ $1500\text{ms}$) cho các tương tác thường xuyên, hoặc quá ngắn ($30\text{ms}$) cho các thành phần mở rộng diện tích lớn.

#### Nguyên nhân kỹ thuật:
Thời lượng quá dài làm chậm nhịp độ sử dụng ứng dụng, gây cảm giác lag hoặc phản hồi chậm chạp cho người dùng. Thời lượng quá ngắn không đủ số khung hình (ở màn hình 60Hz, 30ms chỉ hiển thị được chưa đầy 2 frame) khiến mắt người nhận thức chuyển động như một cú giật đột ngột.

#### Biện pháp khắc phục:
Tuân thủ bảng hướng dẫn thời lượng của Material Design 3 ($200\text{ms} - 400\text{ms}$ cho các tương tác chuyển đổi thông thường).

---

### 4.3 — Khởi tạo `Tween` mới liên tục trong `build()` với `TweenAnimationBuilder`

#### Mô tả vấn đề:
Tạo giá trị `begin` thay đổi liên tục ở mỗi lần hàm `build()` chạy:
```dart
// Lỗi: begin luôn được gán lại giá trị cố định ở mỗi lần cha rebuild
TweenAnimationBuilder<double>(
  tween: Tween<double>(begin: 0, end: _currentValue),
  duration: const Duration(milliseconds: 500),
  builder: (context, value, child) => ...,
)
```

#### Nguyên nhân kỹ thuật:
Nếu widget cha kích hoạt rebuild khi animation đang chạy dở ở giá trị `50`, việc truyền một `Tween(begin: 0, end: _currentValue)` mới sẽ ép bộ nội suy nhảy ngược về `0` rồi chạy tiếp, làm gián đoạn chuyển động mượt mà.

#### Biện pháp khắc phục:
Khi giá trị `end` thay đổi, `TweenAnimationBuilder` sẽ tự động lấy giá trị hiện tại làm mốc `begin` mới. Không cần can thiệp lại thuộc tính `begin` sau lần khởi tạo đầu tiên.

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Lớp `ImplicitlyAnimatedWidgetState` quản lý `AnimationController` như thế nào mà lập trình viên không cần gọi `dispose()`?
*Phân tích:*
`ImplicitlyAnimatedWidgetState` là một lớp `State` kế thừa từ `SingleTickerProviderStateMixin`. Trong phương thức `initState()`, nó tự khởi tạo một đối tượng `AnimationController` nội bộ. Trong phương thức `dispose()` của chính lớp `State` đó, framework đã cài đặt sẵn lời gọi `_controller.dispose()`. Do đó, vòng đời của controller được gắn chặt và giải phóng tự động theo vòng đời của Widget trên cây Element.

---

#### Câu hỏi 2: Tại sao `AnimatedOpacity` có thuộc tính `alwaysIncludeSemantics`?
*Phân tích:*
Khi `opacity` đạt giá trị `0.0`, theo mặc định widget con sẽ bị ẩn hoàn toàn khỏi màn hình và bị loại khỏi cây trợ năng (Accessibility/Semantics Tree). Tuy nhiên, nếu widget con chứa các nút bấm điều khiển quan trọng mà người dùng khiếm thị cần nhận biết qua trình đọc màn hình (Screen Reader như TalkBack hoặc VoiceOver), việc bật `alwaysIncludeSemantics: true` sẽ giữ lại thông tin ngữ nghĩa trong cây hỗ trợ tiếp cận ngay cả khi phần tử đang vô hình về mặt thị giác.

---

#### Câu hỏi 3: Thuật toán nội suy `lerp` trong Flutter hoạt động như thế nào?
*Phân tích:*
`lerp` là viết tắt của *Linear Interpolation* (Nội suy tuyến tính). Công thức toán học cơ bản:
$$\text{result} = a + (b - a) \times t$$
Trong đó:
- $a$ là giá trị bắt đầu (`begin`).
- $b$ là giá trị đích (`end`).
- $t$ là giá trị tiến độ thời gian trong khoảng $[0.0, 1.0]$ do `Curve` điều phối.
Flutter triển khai hàm `lerp` tĩnh trên hầu hết các lớp kiểu dữ liệu hình ảnh: `Color.lerp(a, b, t)`, `Rect.lerp(a, b, t)`, `Decoration.lerp(a, b, t)`.

---

#### Câu hỏi 4: Sự khác biệt bản chất giữa `AnimatedWidget` và `ImplicitlyAnimatedWidget`?
*Phân tích:*
- **`AnimatedWidget`**: Là lớp cơ sở cho các widget explicit animation (như `SlideTransition`, `FadeTransition`). Lớp này nhận một đối tượng `Listenable` (thường là `Animation<T>`) từ bên ngoài truyền vào và tự động gọi `setState()` mỗi khi `Listenable` phát tín hiệu thay đổi.
- **`ImplicitlyAnimatedWidget`**: Tự quản lý `AnimationController` bên trong. Lập trình viên không truyền `Animation` mà chỉ truyền giá trị thuần túy (như `double`, `Color`), widget sẽ tự tạo controller và tính toán chuyển động nội bộ.

---

#### Câu hỏi 5: Hạn chế về mặt hiệu năng của `AnimatedContainer` so với các Transition Widget chuyên biệt là gì?
*Phân tích:*
Mỗi khi `AnimatedContainer` thay đổi thuộc tính kích thước hoặc viền, nó buộc Render Tree phải thực hiện lại toàn bộ chu trình **Layout** và **Paint** cho chính nó và các widget con bên trong ở mỗi khung hình ($60\text{fps} - 120\text{fps}$). Ngược lại, các Transition Widget như `Transform.translate` hoặc `FadeTransition` chỉ can thiệp vào giai đoạn **Paint** hoặc cập nhật trực tiếp trên **Compositing Layer**, không kích hoạt lại giai đoạn tính toán kích thước (Layout), do đó tiêu thụ ít tài nguyên CPU/GPU hơn.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho đoạn mã sử dụng `AnimatedOpacity`:

```dart
class DemoOpacity extends StatefulWidget {
  const DemoOpacity({super.key});
  @override State<DemoOpacity> createState() => _DemoOpacityState();
}

class _DemoOpacityState extends State<DemoOpacity> {
  double _opacity = 1.0;

  void _trigger() {
    setState(() => _opacity = 0.0);
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedOpacity(
      opacity: _opacity,
      duration: const Duration(milliseconds: 300),
      curve: Curves.linear,
      child: const ContainerBox(),
    );
  }
}
```

Giả sử màn hình hoạt động ở tần số quét $60\text{Hz}$ (khoảng $16.67\text{ms}$ mỗi frame). Khi phương thức `_trigger()` được gọi tại thời điểm $t = 0\text{ms}$:
1. Tại thời điểm $t = 0\text{ms}$ (ngay sau khi `setState` hoàn tất), giá trị opacity hiển thị trên màn hình là bao nhiêu?
2. Sau bao nhiêu khung hình thì animation hoàn tất?
3. Tại thời điểm $t = 150\text{ms}$, giá trị opacity được gửi xuống tầng RenderObject là bao nhiêu?

#### Kết quả phân tích kỹ thuật:

1. **Tại thời điểm $t = 0\text{ms}$:**
   - Khi `setState()` chạy, `didUpdateWidget` được gọi, `Tween<double>` được cập nhật với `begin = 1.0`, `end = 0.0`.
   - `AnimationController` nội bộ được khởi động lại tại frame kế tiếp. Tại thời điểm hàm `build()` đầu tiên chạy xong ở frame hiện tại, giá trị opacity hiển thị vẫn là **`1.0`**. Chuyển động bắt đầu thay đổi từ frame tiếp theo khi `Ticker` phát tín hiệu nhịp đầu tiên.

2. **Số khung hình để hoàn tất:**
   - Thời lượng animation: $300\text{ms}$.
   - Tần số quét $60\text{Hz}$ tương ứng với chu kỳ: $1000\text{ms} / 60 \approx 16.67\text{ms}$ mỗi frame.
   - Tổng số frame được render trong suốt quá trình:
     $$\text{Số frame} = \frac{300\text{ms}}{16.67\text{ms}} \approx 18 \text{ frames}$$

3. **Tại thời điểm $t = 150\text{ms}$:**
   - Tiến độ thời gian: $t_{\text{progress}} = \frac{150\text{ms}}{300\text{ms}} = 0.5$.
   - Do sử dụng `Curves.linear`, giá trị tiến độ chuyển động bằng chính tiến độ thời gian: $t_{\text{curve}} = 0.5$.
   - Giá trị nội suy:
     $$\text{opacity} = \text{begin} + (\text{end} - \text{begin}) \times t = 1.0 + (0.0 - 1.0) \times 0.5 = \mathbf{0.5}$$
   - Giá trị `0.5` được truyền trực tiếp vào đối tượng `RenderAnimatedOpacity` để cập nhật độ trong suốt của Layer.
