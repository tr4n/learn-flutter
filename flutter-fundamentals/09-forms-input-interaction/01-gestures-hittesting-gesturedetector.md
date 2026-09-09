# Bài 9.1 — Hệ Thống Cử Chỉ, Hit Testing & Đấu Trường Gesture Arena

## Tài Liệu Tham Khảo Chính Thức
- [Flutter Documentation: Gestures in Flutter](https://docs.flutter.dev/ui/interactivity/gestures)
- [Flutter API: GestureDetector class](https://api.flutter.dev/flutter/widgets/GestureDetector-class.html)
- [Flutter API: GestureArenaManager class](https://api.flutter.dev/flutter/gestures/GestureArenaManager-class.html)
- [Flutter API: HitTestBehavior enum](https://api.flutter.dev/flutter/rendering/HitTestBehavior.html)
- [Flutter API: Listener class](https://api.flutter.dev/flutter/widgets/Listener-class.html)
- [Flutter API: InkWell class](https://api.flutter.dev/flutter/material/InkWell-class.html)

---

## Phần 1 — Khái Niệm & Phân Cấp Hệ Thống Tương Tác Cảm Ứng

### 1.1 — Mô Hình Phân Tầng Tương Tác Trong Flutter

Hệ thống cử chỉ của Flutter được chia thành 3 tầng kiến trúc độc lập, chuyển hóa tín hiệu từ phần cứng cấp thấp thành các hành vi ngữ nghĩa cấp cao:

```
┌────────────────────────────────────────────────────────────────────────┐
│ PHÂN TẦNG KIẾN TRÚC HỆ THỐNG CỬ CHỈ TRONG FLUTTER                      │
│                                                                        │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ TẦNG 3: WIDGET GIAO TIẾP (High-level Components)                   │ │
│ │  • GestureDetector, InkWell, Dismissible, Draggable                 │ │
│ └──────────────────────────────────┬─────────────────────────────────┘ │
│                                    │ Sử dụng và đăng ký callbacks      │
│                                    ▼                                   │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ TẦNG 2: BỘ NHẬN DIỆN CỬ CHỈ (Gesture Recognizers & Gesture Arena)   │ │
│ │  • TapGestureRecognizer, PanGestureRecognizer, ScaleGestureRecog... │ │
│ │  • GestureArenaManager phân xử xung đột giữa các cử chỉ             │ │
│ └──────────────────────────────────┬─────────────────────────────────┘ │
│                                    │ Tiếp nhận sự kiện con trỏ         │
│                                    ▼                                   │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ TẦNG 1: SỰ KIỆN CON TRỎ CẤP THẤP (Raw Pointer Events)              │ │
│ │  • PointerDownEvent, PointerMoveEvent, PointerUpEvent, PointerCancel│ │
│ │  • Được thu thập qua quá trình Hit Testing                          │ │
│ └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

1. **Tầng 1 — Raw Pointer Events (`Listener`)**:
   - Nhận tín hiệu trực tiếp từ hệ điều hành.
   - Chỉ chứa thông tin vật lý thô: Tọa độ $(x, y)$, lực nhấn (`pressure`), loại con trỏ (`touch`, `mouse`, `stylus`).
   - Không có khái niệm ngữ nghĩa như "Click", "Vuốt", hay "Phóng to".
2. **Tầng 2 — Gesture Recognizers & Gesture Arena**:
   - Gom nhóm chuỗi các `PointerEvent` liên tiếp thành các cử chỉ có ý nghĩa.
   - Khi có nhiều cử chỉ cùng quan tâm đến một chuỗi sự kiện, **Gesture Arena** đóng vai trò là đấu trường phân xử xem cử chỉ nào chiến thắng.
3. **Tầng 3 — Gesture Widgets (`GestureDetector`, `InkWell`)**:
   - Lớp giao diện bao bọc (Wrapper), tiếp nhận các callback (`onTap`, `onDoubleTap`, `onLongPress`, `onPanUpdate`) và cập nhật UI.

#### Bảng so sánh 3 widget tương tác cơ bản:

| Tiêu Chí | `Listener` | `GestureDetector` | `InkWell` |
| :--- | :--- | :--- | :--- |
| **Tầng kiến trúc** | Tầng 1 (Pointer Events) | Tầng 2 & 3 (Gesture Arena) | Tầng 3 (Material Component) |
| **Tham gia Gesture Arena** | Không (nhận sự kiện trực tiếp) | Có (cạnh tranh trong Arena) | Có (dựa trên TapGestureRecognizer) |
| **Phản hồi thị giác** | Không | Không (chỉ có logic) | Có (Hiệu ứng gợn sóng Material InkRipple) |
| **Ngữ cảnh sử dụng** | Bắt tọa độ thô, tracking chuột, joystick | Xử lý cử chỉ nghiệp vụ phức tạp | Các thành phần bấm chuẩn Material Design |

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Giai Đoạn 1: Hit Testing Pipeline

Khi ngón tay chạm vào màn hình (`PointerDownEvent`), Flutter Engine thực hiện giai đoạn **Hit Testing** để xác định danh sách các widget nằm bên dưới điểm chạm:

```
┌────────────────────────────────────────────────────────────────────────┐
│ LUỒNG HIT TESTING TRONG RENDER TREE                                    │
│                                                                        │
│ 1. RendererBinding.hitTest(result, position) kích hoạt từ RenderView   │
│      │                                                                 │
│      ▼                                                                 │
│ 2. Duyệt cây theo chiều sâu (Depth-First Traversal):                   │
│    • RenderBox cha gọi hitTestChildren() trên các con từ trên xuống    │
│    • Nếu con trả về true -> Thêm con vào HitTestResult                 │
│    • Gọi hitTestSelf() của chính cha -> Thêm cha vào HitTestResult     │
│      │                                                                 │
│      ▼                                                                 │
│ 3. Kết quả: Một danh sách HitTestResult chứa các RenderBox trúng đích  │
│    (Sắp xếp theo thứ tự từ lá sâu nhất lên đến gốc RenderView)         │
│      │                                                                 │
│      ▼                                                                 │
│ 4. Phân phối sự kiện (Event Dispatch):                                 │
│    • Gửi PointerDownEvent tuần tự qua tất cả các phần tử trong danh sách│
└────────────────────────────────────────────────────────────────────────┘
```

#### Vai trò của `HitTestBehavior`:
Widget `GestureDetector` (thông qua `RenderPointerListener`) cung cấp thuộc tính `behavior` với 3 giá trị:

1. **`HitTestBehavior.deferToChild` (Mặc định)**:
   - Chỉ nhận diện điểm chạm nếu một trong các widget con của nó trả về `true` trong quá trình hit test.
   - Nếu chạm vào vùng trống (ví dụ khoảng cách giữa hai biểu tượng hoặc `Container` không có màu nền), sự kiện chạm sẽ đi xuyên qua và widget không phản hồi.
2. **`HitTestBehavior.opaque`**:
   - Tự động coi toàn bộ diện tích hình chữ nhật của widget là một vùng cản đặc, ngay cả khi vùng đó hoàn toàn trong suốt.
   - Ngăn chặn các widget nằm phía sau (trên trục Z) nhận được sự kiện chạm.
3. **`HitTestBehavior.translucent`**:
   - Cho phép bản thân widget nhận diện được điểm chạm trên toàn bộ diện tích của nó, **đồng thời vẫn cho phép** các widget nằm bên dưới nó trong cấu trúc `Stack` tiếp tục nhận diện điểm chạm.

---

### 2.2 — Giai Đoạn 2: Đấu Trường Cử Chỉ (Gesture Arena)

Khi nhiều `GestureRecognizer` cùng nhận được chuỗi `PointerEvent` từ một điểm chạm (ví dụ: một `VerticalDragGestureRecognizer` của `ListView` và một `TapGestureRecognizer` của nút bấm), chúng cùng tham gia vào một **Gesture Arena**:

```
┌────────────────────────────────────────────────────────────────────────┐
│ VÒNG ĐỜI CẠNH TRANH TRONG GESTURE ARENA                                │
│                                                                        │
│ 1. PointerDownEvent xuất hiện:                                         │
│    • GestureArenaManager mở một Đấu trường (Arena) mới                 │
│    • Các Recognizer đăng ký tham gia (add member)                      │
│      │                                                                 │
│      ▼                                                                 │
│ 2. PointerMoveEvent xuất hiện:                                         │
│    • Nếu khoảng cách di chuyển vượt ngưỡng kTouchSlop:                 │
│      -> VerticalDragRecognizer tự nhận là THẮNG                        │
│      -> TapRecognizer tự nhận thấy không hợp lệ và rút lui (REJECT)    │
│      │                                                                 │
│      ▼                                                                 │
│ 3. PointerUpEvent xuất hiện (Người dùng nhấc tay mà chưa di chuyển):   │
│    • VerticalDragRecognizer không đủ điều kiện -> Rút lui (REJECT)     │
│    • Chỉ còn duy nhất TapRecognizer trong Arena                        │
│    • Arena kích hoạt phương thức sweep() -> TapRecognizer THẮNG        │
│      │                                                                 │
│      ▼                                                                 │
│ 4. Recognizer chiến thắng gọi callback tương ứng (onTap)               │
│    Arena đóng lại và giải phóng tài nguyên                             │
└────────────────────────────────────────────────────────────────────────┘
```

- **Thắng sớm (Eager Win)**: Một recognizer có thể tự tuyên bố chiến thắng sớm nếu phát hiện hành vi chắc chắn (ví dụ `PanGestureRecognizer` khi ngón tay đã trượt một khoảng cách lớn), các đối thủ còn lại lập tức bị loại (`rejectGesture`).
- **Thắng mặc định qua `sweep()`**: Khi sự kiện `PointerUp` diễn ra mà chưa có ai tự nhận thắng, `GestureArenaManager` sẽ quét qua các thành viên còn lại và trao quyền chiến thắng cho thành viên đầu tiên còn trụ lại trong đấu trường.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — `GestureDetector`: Xử Lý Đa Cử Chỉ

Sử dụng `GestureDetector` kết hợp cấu hình `HitTestBehavior.opaque` để đảm bảo toàn bộ diện tích thẻ đều nhận tương tác:

```dart
import 'package:flutter/material.dart';

class InteractiveCard extends StatefulWidget {
  const InteractiveCard({super.key});

  @override
  State<InteractiveCard> createState() => _InteractiveCardState();
}

class _InteractiveCardState extends State<InteractiveCard> {
  String _lastAction = 'Chưa có tương tác';
  double _scale = 1.0;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: GestureDetector(
        // Đảm bảo toàn bộ khung chữ nhật nhận sự kiện chạm dù nền trong suốt
        behavior: HitTestBehavior.opaque,
        onTap: () {
          setState(() => _lastAction = 'Chạm một lần (Tap)');
        },
        onDoubleTap: () {
          setState(() {
            _lastAction = 'Chạm đúp (Double Tap)';
            _scale = _scale == 1.0 ? 1.2 : 1.0;
          });
        },
        onLongPress: () {
          setState(() => _lastAction = 'Nhấn giữ lâu (Long Press)');
        },
        child: AnimatedScale(
          scale: _scale,
          duration: const Duration(milliseconds: 200),
          child: Container(
            width: 240,
            height: 140,
            padding: const EdgeInsets.all(16),
            decoration: BoxDecoration(
              color: Theme.of(context).colorScheme.primaryContainer,
              borderRadius: BorderRadius.circular(16),
              border: Border.all(color: Theme.of(context).colorScheme.primary),
            ),
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                const Icon(Icons.touch_app, size: 36),
                const SizedBox(height: 8),
                Text(
                  _lastAction,
                  textAlign: TextAlign.center,
                  style: const TextStyle(fontWeight: FontWeight.w600),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

---

### 3.2 — `InkWell`: Hiệu Ứng Gợn Sóng Material Chuẩn Xác

Khi sử dụng `InkWell`, widget con trực tiếp phải nằm trên một đối tượng `Material` để mực (Ink) có thể vẽ lên bề mặt:

```dart
import 'package:flutter/material.dart';

class MaterialRippleButton extends StatelessWidget {
  final VoidCallback onPressed;
  final String label;

  const MaterialRippleButton({
    super.key,
    required this.onPressed,
    required this.label,
  });

  @override
  Widget build(BuildContext context) {
    return Material(
      color: Colors.transparent, // Giữ nền trong suốt cho Material
      child: Ink(
        decoration: BoxDecoration(
          color: Theme.of(context).colorScheme.secondaryContainer,
          borderRadius: BorderRadius.circular(12),
        ),
        child: InkWell(
          borderRadius: BorderRadius.circular(12),
          splashColor: Theme.of(context).colorScheme.primary.withOpacity(0.15),
          highlightColor: Theme.of(context).colorScheme.primary.withOpacity(0.08),
          onTap: onPressed,
          child: Padding(
            padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
            child: Text(
              label,
              style: TextStyle(
                color: Theme.of(context).colorScheme.onSecondaryContainer,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

---

### 3.3 — `Listener`: Bắt Tọa Độ Con Trỏ Thô Không Qua Đấu Trường

Khi cần theo dõi tọa độ liên tục (ví dụ: vẽ Canvas hoặc làm thanh joystick điều khiển):

```dart
import 'package:flutter/material.dart';

class RawPointerTracker extends StatefulWidget {
  const RawPointerTracker({super.key});

  @override
  State<RawPointerTracker> createState() => _RawPointerTrackerState();
}

class _RawPointerTrackerState extends State<RawPointerTracker> {
  Offset _pointerLocation = Offset.zero;
  bool _isDown = false;

  @override
  Widget build(BuildContext context) {
    return Listener(
      // Listener bỏ qua Gesture Arena, bắt mọi chuyển động vật lý của con trỏ
      onPointerDown: (event) {
        setState(() {
          _isDown = true;
          _pointerLocation = event.localPosition;
        });
      },
      onPointerMove: (event) {
        setState(() {
          _pointerLocation = event.localPosition;
        });
      },
      onPointerUp: (event) {
        setState(() => _isDown = false);
      },
      child: Container(
        width: double.infinity,
        height: 200,
        color: _isDown ? Colors.blue.shade50 : Colors.grey.shade100,
        child: Center(
          child: Text(
            'Tọa độ: (${_pointerLocation.dx.toStringAsFixed(1)}, ${_pointerLocation.dy.toStringAsFixed(1)})',
            style: const TextStyle(fontSize: 16, fontFamily: 'monospace'),
          ),
        ),
      ),
    );
  }
}
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Vùng trống trong suốt không nhận diện được cử chỉ

#### Mô tả vấn đề:
Bọc một `Row` hoặc `Container` có các khoảng trống bằng `GestureDetector`. Người dùng nhấn vào chữ thì nhận, nhưng nhấn vào khoảng trống giữa các chữ thì không có phản hồi:
```dart
// Lỗi: behavior mặc định là deferToChild
GestureDetector(
  onTap: () => print('Tapped'),
  child: Container(
    width: 200,
    height: 50,
    child: Text('Bấm vào đây'), // Chỉ có phần chữ nhận hit
  ),
)
```

#### Nguyên nhân kỹ thuật:
`HitTestBehavior.deferToChild` chỉ trả về `true` khi điểm chạm trúng một widget con có thực hiện vẽ (`RenderParagraph`). Khoảng trống trong suốt không vẽ gì nên bị bỏ qua trong Hit Testing.

#### Biện pháp khắc phục:
Thêm `behavior: HitTestBehavior.opaque` hoặc gán màu nền `color: Colors.transparent` cho Container.

---

### 4.2 — Xung đột cử chỉ cuộn khi lồng cử chỉ kéo (Pan/Drag) vào `ListView`

#### Mô tả vấn đề:
Đặt một widget có `onVerticalDragUpdate` bên trong một `ListView`. Người dùng không thể cuộn trang danh sách khi chạm vào widget đó.

#### Nguyên nhân kỹ thuật:
Cả `ListView` (thông qua `Scrollable`) và `GestureDetector` đều đăng ký `VerticalDragGestureRecognizer` vào Gesture Arena. Do `GestureDetector` bắt đầu nhận diện chuyển động kéo trước, nó chiếm quyền chiến thắng và triệt tiêu khả năng cuộn của `ListView`.

#### Biện pháp khắc phục:
Sử dụng `RawGestureDetector` với một custom `GestureRecognizer` tự định nghĩa chính sách phân xử, hoặc chuyển sang cử chỉ kéo ngang (`onHorizontalDragUpdate`) để hai cử chỉ cạnh tranh trên hai trục khác nhau.

---

### 4.3 — Hiệu ứng `InkWell` bị che khuất bởi màu nền của `Container`

#### Mô tả vấn đề:
Sử dụng `InkWell` nhưng khi bấm vào hoàn toàn không thấy hiệu ứng gợn sóng (Ripple Effect).

#### Nguyên nhân kỹ thuật:
Lập trình viên đặt `Container(color: Colors.blue)` làm widget con bên trong `InkWell`. Trong cây RenderObject, tầng vẽ của `Container` nằm đè lên trên tầng vẽ của `Material`, che khuất toàn bộ mực gợn sóng được vẽ ra.

#### Biện pháp khắc phục:
1. Chuyển thuộc tính màu vào widget `Ink` nằm bên ngoài `InkWell`.
2. Hoặc sử dụng thuộc tính `color` của widget `Material` bọc ngoài cùng.

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Lớp `GestureArenaManager` phân định cử chỉ chiến thắng như thế nào khi người dùng chạm và nhấc tay ngay lập tức mà không di chuyển?
*Phân tích:*
1. Khi `PointerDownEvent` xảy ra, cả `TapGestureRecognizer` và `LongPressGestureRecognizer` cùng đăng ký vào Arena.
2. `LongPressGestureRecognizer` đặt một timer nội bộ (thường là $500\text{ms}$).
3. Khi người dùng nhấc tay tại $t = 100\text{ms}$, `PointerUpEvent` kích hoạt.
4. `LongPressGestureRecognizer` nhận thấy ngón tay đã nhấc lên trước khi timer $500\text{ms}$ kích hoạt $\to$ Tự gọi `rejectGesture()` để rút lui khỏi Arena.
5. Lúc này, `TapGestureRecognizer` là thành viên duy nhất còn lại trong Arena $\to$ Được tuyên bố chiến thắng (`acceptGesture()`) và gọi hàm `onTap`.

---

#### Câu hỏi 2: Sự khác biệt bản chất giữa `HitTestBehavior.opaque` và `HitTestBehavior.translucent` khi đặt trong `Stack`?
*Phân tích:*
Giả sử có hai lớp widget A và B nằm trong `Stack`, trong đó A nằm dưới và B nằm đè lên trên A:
- Nếu B có `behavior: HitTestBehavior.opaque`: B nhận diện điểm chạm và chặn đứng quá trình hit test. Widget A nằm phía dưới hoàn toàn không được đưa vào `HitTestResult`.
- Nếu B có `behavior: HitTestBehavior.translucent`: B vẫn nhận diện điểm chạm cho chính mình, **nhưng không chặn luồng duyệt**. Quá trình hit test tiếp tục đi xuống widget A phía dưới. Kết quả là cả A và B cùng có mặt trong `HitTestResult` và cùng nhận được `PointerDownEvent`.

---

#### Câu hỏi 3: Tại sao `Listener` không thể giải quyết được bài toán phân xử cử chỉ (Gesture Disambiguation)?
*Phân tích:*
`Listener` hoạt động độc lập hoàn toàn với `GestureArenaManager`. Nó là một lớp bao bọc mỏng trực tiếp lắng nghe các sự kiện con trỏ (`PointerEvent`) được chuyển giao từ `HitTestResult`. Bất kỳ widget nào có mặt trong `HitTestResult` đều sẽ nhận được toàn bộ các sự kiện `Down`, `Move`, `Up` đồng thời. `Listener` không có cơ chế thương lượng, nhường quyền hay hủy bỏ sự kiện giữa các widget cha-con.

---

#### Câu hỏi 4: Ngưỡng `kTouchSlop` trong Flutter Engine là gì và vai trò của nó trong Gesture Arena?
*Phân tích:*
`kTouchSlop` (mặc định khoảng $18.0\text{ logical pixels}$ trên màn hình cảm ứng) là bán kính dung sai chuyển động cho phép. Khi ngón tay người dùng chạm vào màn hình, mắt người không thể giữ yên tuyệt đối mà luôn có những rung động vi mô. `TapGestureRecognizer` cho phép ngón tay di chuyển trong phạm vi `kTouchSlop` mà vẫn tính là một cú chạm hợp lệ. Khi khoảng cách vượt quá `kTouchSlop`, các cử chỉ dạng Drag/Pan mới kích hoạt và loại bỏ Tap khỏi Arena.

---

#### Câu hỏi 5: Làm thế nào để hai widget cha và con cùng nhận sự kiện `onTap` đồng thời?
*Phân tích:*
Mặc định trong Gesture Arena, chỉ có duy nhất **một** recognizer chiến thắng. Nếu cả hai đều dùng `GestureDetector`, widget con sẽ thắng và widget cha sẽ bị reject. Để cả hai cùng nhận:
1. Thay vì dùng `GestureDetector` ở cả hai, sử dụng `Listener(onPointerUp: ...)` ở widget cha (vì `Listener` không tham gia Arena nên không bị loại).
2. Hoặc sử dụng `RawGestureDetector` với một custom `GestureRecognizer` ghi đè phương thức `rejectGesture()` để tự động chấp nhận cử chỉ bất kể phán quyết của Arena.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho cấu trúc widget lồng nhau như sau:

```dart
GestureDetector(
  onTap: () => print('Cha: onTap'),
  child: Container(
    padding: const EdgeInsets.all(32),
    color: Colors.blue,
    child: GestureDetector(
      onTap: () => print('Con: onTap'),
      child: Container(
        padding: const EdgeInsets.all(16),
        color: Colors.yellow,
        child: const Text('Bấm vào tôi'),
      ),
    ),
  ),
)
```

Khi người dùng thực hiện một cú chạm nhanh (Tap) vào chính giữa dòng chữ `'Bấm vào tôi'`:
1. Quá trình Hit Testing diễn ra như thế nào? Danh sách `HitTestResult` chứa những phần tử nào theo thứ tự?
2. Có bao nhiêu `TapGestureRecognizer` tham gia vào Gesture Arena?
3. Dòng log nào sẽ được in ra console? Giải thích cơ chế quyết định của Arena.

---

#### Kết quả phân tích kỹ thuật:

1. **Quá trình Hit Testing:**
   - Điểm chạm nằm trong diện tích của `Text`, `Container` (Con), `GestureDetector` (Con), `Container` (Cha), `GestureDetector` (Cha).
   - Thuật toán duyệt sâu Depth-first traversal thu thập các node từ lá lên gốc.
   - Thứ tự trong `HitTestResult`:
     1. `RenderParagraph` (chữ 'Bấm vào tôi')
     2. `RenderDecoratedBox` (Container con màu vàng)
     3. `RenderPointerListener` (GestureDetector con)
     4. `RenderDecoratedBox` (Container cha màu xanh)
     5. `RenderPointerListener` (GestureDetector cha)

2. **Các thành viên tham gia Arena:**
   - Cả hai `RenderPointerListener` đều chuyển `PointerDownEvent` cho `TapGestureRecognizer` tương ứng của mình.
   - Có **hai** `TapGestureRecognizer` (một của Con, một của Cha) cùng đăng ký tham gia vào Gesture Arena của điểm chạm này.

3. **Kết quả in ra console:**
   - Console chỉ in ra duy nhất: **`Con: onTap`**.
   - **Giải thích cơ chế**:
     - Do không có chuyển động kéo (Drag) vượt ngưỡng `kTouchSlop`, cả hai Recognizer đều giữ nguyên trạng thái chờ đến khi `PointerUpEvent` xảy ra.
     - Khi `PointerUpEvent` diễn ra, phương thức `sweep()` của `GestureArenaManager` được kích hoạt.
     - `GestureArenaManager` duyệt danh sách các thành viên đăng ký theo thứ tự xuất hiện trong `HitTestResult`. Do widget Con nằm sâu hơn trên cây Widget nên được thêm vào danh sách trước.
     - Thành viên đầu tiên (Recognizer của Con) được tuyên bố chiến thắng (`acceptGesture()`). Ngay lập tức, đấu trường gửi tín hiệu `rejectGesture()` tới tất cả các thành viên còn lại, loại bỏ Recognizer của Cha. Do đó callback của Cha không bao giờ được gọi.
