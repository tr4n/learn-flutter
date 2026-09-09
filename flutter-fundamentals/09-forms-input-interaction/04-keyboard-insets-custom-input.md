# Bài 9.4 — Xử Lý Bàn Phím Ảo (Keyboard Insets) & Xây Dựng Custom FormField<T>

## Tài Liệu Tham Khảo Chính Thức
- [Flutter Documentation: Creating responsive and adaptive apps](https://docs.flutter.dev/ui/adaptive-responsive)
- [Flutter API: MediaQueryData class (viewInsets, viewPadding, padding)](https://api.flutter.dev/flutter/widgets/MediaQueryData-class.html)
- [Flutter API: Scaffold.resizeToAvoidBottomInset property](https://api.flutter.dev/flutter/material/Scaffold/resizeToAvoidBottomInset.html)
- [Flutter API: FormField class](https://api.flutter.dev/flutter/widgets/FormField-class.html)
- [Flutter API: FormFieldState class](https://api.flutter.dev/flutter/widgets/FormFieldState-class.html)

---

## Phần 1 — Khái Niệm & Đo Lường Không Gian Màn Hình

### 1.1 — Phân Biệt Bộ Ba: `viewInsets`, `viewPadding`, Và `padding`

Khi bàn phím ảo xuất hiện, nó chiếm dụng một phần diện tích hiển thị của ứng dụng. Để xử lý bố cục chính xác, lập trình viên cần phân biệt rõ 3 thuộc tính đo lường trong đối tượng `MediaQueryData`:

```
┌────────────────────────────────────────────────────────────────────────┐
│ PHÂN BỐ CÁC VÙNG KHÔNG GIAN HỆ THỐNG TRÊN MÀN HÌNH                     │
│                                                                        │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ Status Bar (Tai thỏ / Camera nốt ruồi)                             │ │
│ │  -> Đo bằng: MediaQuery.padding.top / viewPadding.top              │ │
│ ├────────────────────────────────────────────────────────────────────┤ │
│ │                                                                    │ │
│ │                                                                    │ │
│ │ VÙNG HIỂN THỊ CHÍNH CỦA ỨNG DỤNG                                   │ │
│ │                                                                    │ │
│ │                                                                    │ │
│ ├────────────────────────────────────────────────────────────────────┤ │
│ │ BÀN PHÍM ẢO HỆ THỐNG (Khi bật lên)                                 │ │
│ │  -> Đo bằng: MediaQuery.viewInsets.bottom                          │ │
│ ├────────────────────────────────────────────────────────────────────┤ │
│ │ Thanh điều hướng hệ thống (Home Indicator / Navigation Bar)        │ │
│ │  -> Đo bằng: MediaQuery.padding.bottom / viewPadding.bottom        │ │
│ └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

1. **`MediaQuery.of(context).viewInsets`**:
   - Phần diện tích bị che khuất hoàn toàn bởi các thành phần giao diện hệ điều hành xuất hiện đè lên trên (chủ yếu là **bàn phím ảo**).
   - Khi bàn phím ẩn: `viewInsets.bottom = 0.0`.
   - Khi bàn phím hiện: `viewInsets.bottom` tương ứng với chiều cao chính xác của bàn phím (thường từ $250\text{px} - 350\text{px}$).
2. **`MediaQuery.of(context).padding`**:
   - Vùng an toàn (SafeArea) để tránh các phần cứng che khuất (thanh trạng thái phía trên và thanh điều hướng phía dưới).
   - Khi bàn phím mở lên, `padding.bottom` tự động giảm về `0.0` vì bàn phím đã che đè lên thanh điều hướng.
3. **`MediaQuery.of(context).viewPadding`**:
   - Tương tự như `padding`, nhưng **giữ nguyên giá trị không đổi** kể cả khi bàn phím ảo xuất hiện.

---

### 1.2 — Vai Trò Của `Custom FormField<T>`

Hầu hết các biểu mẫu không chỉ có văn bản (`TextFormField`), mà còn bao gồm nhiều kiểu dữ liệu phức tạp khác:
- Đánh giá chất lượng dịch vụ (Star Rating: `int`).
- Chọn ngày tháng (Date Range Picker: `DateTimeRange`).
- Danh sách nhãn tag (Multi-select Chips: `List<String>`).
- Tải ảnh đính kèm (File / Image Picker: `List<File>`).

Lớp trừu tượng `FormField<T>` cho phép đóng gói bất kỳ widget giao diện nào thành một phần tử của hệ thống Form, hỗ trợ đầy đủ các tính năng: gán giá trị khởi tạo (`initialValue`), kiểm tra tính hợp lệ (`validator`), lưu dữ liệu (`onSaved`), và khôi phục trạng thái (`reset()`).

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Cơ Chế Co Giãn Của `Scaffold.resizeToAvoidBottomInset`

Thuộc tính `resizeToAvoidBottomInset` trên widget `Scaffold` điều khiển cách khung nhìn phản ứng với sự thay đổi của `viewInsets.bottom`:

```
┌────────────────────────────────────────────────────────────────────────┐
│ CƠ CHẾ CO GIÃN CỦA SCAFFOLD KHI BÀN PHÍM XUẤT HIỆN                     │
│                                                                        │
│ 1. Hệ điều hành đẩy bàn phím ảo lên (ví dụ cao 300px)                  │
│      │                                                                 │
│      ▼                                                                 │
│ 2. Flutter Engine cập nhật viewInsets.bottom = 300                     │
│      │                                                                 │
│      ▼                                                                 │
│ 3. Scaffold đọc viewInsets.bottom:                                     │
│    • Nếu resizeToAvoidBottomInset == true (Mặc định):                  │
│      -> Chiều cao vùng body = Chiều cao màn hình - 300px               │
│      -> Nếu nội dung là Column cố định -> Ném lỗi Bottom Overflow!     │
│      -> Nếu nội dung bọc trong ScrollView -> Tự động co giãn vùng cuộn │
│                                                                        │
│    • Nếu resizeToAvoidBottomInset == false:                            │
│      -> Chiều cao vùng body giữ nguyên không đổi                       │
│      -> Bàn phím trượt đè lên trên nội dung phía dưới                  │
└────────────────────────────────────────────────────────────────────────┘
```

> **Nguyên nhân lỗi Sọc Vàng Đen (RenderFlex Overflow)**: Mặc định `Scaffold` thu nhỏ chiều cao khả dụng. Nếu trang sử dụng `Column` có tổng chiều cao các widget con lớn hơn không gian còn lại sau khi trừ đi chiều cao bàn phím, Flutter Engine sẽ báo lỗi tràn khung (Overflow) ngay lập tức.

---

### 2.2 — Cấu Trúc Hoạt Động Của `FormFieldState<T>`

Lớp `FormField<T>` quản lý trạng thái thông qua `FormFieldState<T>`:
- Phương thức `didChange(T? newValue)`: Cập nhật giá trị nội bộ `_value`, kích hoạt vẽ lại giao diện và tự động thực thi lại hàm `validator` nếu `autovalidateMode` được kích hoạt.
- Phương thức `save()`: Kích hoạt callback `widget.onSaved?.call(_value)`.
- Phương thức `reset()`: Đưa giá trị về lại `widget.initialValue`, gán `errorText = null` và cập nhật lại giao diện.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Bố Cục Chống Tràn Bàn Phím Chuẩn Quy Cách

Kết hợp `SingleChildScrollView` với thuộc tính `keyboardDismissBehavior` để tạo trải nghiệm cuộn tự nhiên:

```dart
import 'package:flutter/material.dart';

class KeyboardAvoidanceScreen extends StatelessWidget {
  const KeyboardAvoidanceScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Xử Lý Bàn Phím')),
      // Mặc định là true, Scaffold tự co chiều cao body
      resizeToAvoidBottomInset: true,
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.symmetric(horizontal: 24.0, vertical: 16.0),
          // Tự động đóng bàn phím khi người dùng thao tác cuộn
          keyboardDismissBehavior: ScrollViewKeyboardDismissBehavior.onDrag,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              const FlutterLogo(size: 100),
              const SizedBox(height: 32),
              const TextField(
                decoration: InputDecoration(
                  labelText: 'Họ và tên',
                  border: OutlineInputBorder(),
                ),
              ),
              const SizedBox(height: 16),
              const TextField(
                decoration: InputDecoration(
                  labelText: 'Địa chỉ nơi ở',
                  border: OutlineInputBorder(),
                ),
              ),
              const SizedBox(height: 16),
              const TextField(
                maxLines: 4,
                decoration: InputDecoration(
                  labelText: 'Ghi chú thêm',
                  border: OutlineInputBorder(),
                ),
              ),
              const SizedBox(height: 32),
              ElevatedButton(
                onPressed: () => FocusManager.instance.primaryFocus?.unfocus(),
                child: const Text('Hoàn Tất'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

### 3.2 — Xử Lý Bàn Phím Ảo Trong Modal Bottom Sheet

Khi mở một `showModalBottomSheet` có chứa `TextField`, nội dung thường bị bàn phím che khuất hoàn toàn nếu không được cấu hình đúng:

```dart
import 'package:flutter/material.dart';

void showCommentBottomSheet(BuildContext context) {
  showModalBottomSheet(
    context: context,
    // Cho phép BottomSheet mở rộng chiều cao vượt quá 50% màn hình
    isScrollControlled: true,
    shape: const RoundedRectangleBorder(
      borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
    ),
    builder: (BuildContext ctx) {
      return Padding(
        // Bù trừ khoảng đệm bằng đúng chiều cao bàn phím
        padding: EdgeInsets.only(
          bottom: MediaQuery.of(ctx).viewInsets.bottom,
          left: 20,
          right: 20,
          top: 20,
        ),
        child: SingleChildScrollView(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Text(
                'Thêm Bình Luận',
                style: Theme.of(ctx).textTheme.titleLarge,
              ),
              const SizedBox(height: 16),
              const TextField(
                autofocus: true,
                decoration: InputDecoration(
                  hintText: 'Nhập nội dung phản hồi...',
                  border: OutlineInputBorder(),
                ),
              ),
              const SizedBox(height: 16),
              ElevatedButton(
                onPressed: () => Navigator.pop(ctx),
                child: const Text('Gửi Bình Luận'),
              ),
              const SizedBox(height: 16),
            ],
          ),
        ),
      );
    },
  );
}
```

---

### 3.3 — Xây Dựng Custom `FormField<int>`: Đánh Giá Số Sao (Star Rating)

Tạo một trường nhập liệu tùy biến đánh giá từ 1 đến 5 sao, tích hợp đầy đủ vào hệ thống `Form`:

```dart
import 'package:flutter/material.dart';

class StarRatingFormField extends FormField<int> {
  StarRatingFormField({
    super.key,
    super.initialValue = 0,
    super.onSaved,
    super.validator,
    super.autovalidateMode,
  }) : super(
          builder: (FormFieldState<int> state) {
            final int currentRating = state.value ?? 0;
            final bool hasError = state.hasError;

            return Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  mainAxisSize: MainAxisSize.min,
                  children: List.generate(5, (index) {
                    final starIndex = index + 1;
                    return IconButton(
                      icon: Icon(
                        starIndex <= currentRating ? Icons.star : Icons.star_border,
                        color: starIndex <= currentRating ? Colors.amber : Colors.grey,
                        size: 32,
                      ),
                      onPressed: () {
                        // Gọi didChange để cập nhật giá trị và kích hoạt validator
                        state.didChange(starIndex);
                      },
                    );
                  }),
                ),
                // Hiển thị dòng thông báo lỗi chuẩn nếu không đạt điều kiện
                if (hasError)
                  Padding(
                    padding: const EdgeInsets.only(left: 12.0, top: 4.0),
                    child: Text(
                      state.errorText ?? '',
                      style: TextStyle(
                        color: ThemeData().colorScheme.error,
                        fontSize: 12.0,
                      ),
                    ),
                  ),
              ],
            );
          },
        );
}
```

Sử dụng trong một `Form`:
```dart
Form(
  key: _formKey,
  child: Column(
    children: [
      StarRatingFormField(
        validator: (rating) {
          if (rating == null || rating == 0) {
            return 'Vui lòng chọn số sao đánh giá.';
          }
          return null;
        },
        onSaved: (rating) {
          debugPrint('Đánh giá đã lưu: $rating sao');
        },
      ),
      ElevatedButton(
        onPressed: () {
          if (_formKey.currentState?.validate() ?? false) {
            _formKey.currentState!.save();
          }
        },
        child: const Text('Gửi Đánh Giá'),
      ),
    ],
  ),
)
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Lỗi tràn đáy giao diện (A RenderFlex overflowed on the bottom)

#### Mô tả vấn đề:
Khi bàn phím xuất hiện, màn hình xuất hiện các dải sọc màu vàng-đen ở góc dưới báo tràn từ $100\text{px} - 250\text{px}$.

#### Nguyên nhân kỹ thuật:
Màn hình sử dụng `Column` với các kích thước cố định. Khi `Scaffold` co chiều cao vùng `body` để nhường chỗ cho bàn phím, không gian còn lại không đủ chứa các widget con.

#### Biện pháp khắc phục:
Bọc `Column` bằng `SingleChildScrollView` để nội dung tự động chuyển sang chế độ có thể cuộn khi bị co hẹp.

---

### 4.2 — Tắt `resizeToAvoidBottomInset: false` mà không xử lý thủ công

#### Mô tả vấn đề:
Cài đặt `resizeToAvoidBottomInset = false` để ngăn lỗi tràn khung, nhưng dẫn đến việc bàn phím trượt lên đè hoàn toàn lên ô nhập mật khẩu hoặc nút bấm "Xác nhận".

#### Nguyên nhân kỹ thuật:
Khi tắt tính năng tự co giãn, `Scaffold` giữ nguyên không gian rendering. Bàn phím trượt đè lên tầng trên cùng, làm người dùng không thể nhìn thấy nội dung mình đang gõ.

#### Biện pháp khắc phục:
Chỉ đặt `false` khi màn hình có các thành phần nền (background art/video) không muốn bị co rúm, đồng thời phải bọc riêng vùng nhập liệu trong `SingleChildScrollView` với khoảng đệm `EdgeInsets.only(bottom: MediaQuery.of(context).viewInsets.bottom)`.

---

### 4.3 — Gọi `setState()` trong Custom FormField thay vì `state.didChange()`

#### Mô tả vấn đề:
Tự tạo một biến `_rating` trong StatefulWidget và gọi `setState()` để cập nhật biểu tượng sao, nhưng `formKey.currentState!.validate()` không nhận biết được giá trị mới.

#### Nguyên nhân kỹ thuật:
`FormState` chỉ quản lý giá trị được lưu giữ bên trong đối tượng `FormFieldState`. Nếu chỉ cập nhật biến cục bộ bên ngoài mà không thông báo qua `state.didChange(newValue)`, hệ thống Form vẫn giữ giá trị cũ (hoặc `null`) và tiếp tục báo lỗi.

#### Biện pháp khắc phục:
Luôn gọi `state.didChange(newValue)` bên trong callback tương tác của widget tùy biến.

---

### 4.4 — Cố định cứng kích thước bàn phím (Hardcoded Keyboard Height)

#### Mô tả vấn đề:
Đặt khoảng đệm cố định: `padding: EdgeInsets.only(bottom: 300)`.

#### Nguyên nhân kỹ thuật:
Kích thước bàn phím biến động rất lớn giữa các thiết bị (iPhone SE nhỏ hơn iPad, bàn phím gõ số thấp hơn bàn phím chữ cái, bàn phím có thanh gợi ý cao hơn bàn phím thường). Cố định $300\text{px}$ sẽ gây khoảng trống thừa trên thiết bị nhỏ và tiếp tục bị che khuất trên thiết bị lớn.

#### Biện pháp khắc phục:
Luôn lấy thông số động thông qua `MediaQuery.of(context).viewInsets.bottom`.

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Tại sao `MediaQuery.of(context).viewInsets.bottom` kích hoạt rebuild nhiều lần khi bàn phím đang trượt lên?
*Phân tích:*
Bàn phím ảo hệ điều hành không xuất hiện tức thì trong một khung hình duy nhất mà trượt lên thông qua một hiệu ứng chuyển động có thời lượng khoảng $250\text{ms} - 300\text{ms}$. Trong suốt quá trình trượt này, hệ điều hành liên tục gửi các thông số chiều cao trung gian cập nhật lên Flutter Engine ở mỗi frame ($50\text{px} \to 120\text{px} \to 210\text{px} \to 280\text{px}$). Do đó, `MediaQueryData` thay đổi liên tục qua từng khung hình, khiến tất cả các widget phụ thuộc vào `MediaQuery.of(context)` đều được rebuild tương ứng để tạo hiệu ứng co giãn mượt mà.

---

#### Câu hỏi 2: Sự khác biệt bản chất giữa `FormField<T>.initialValue` và việc gán giá trị khởi tạo cho `TextEditingController`?
*Phân tích:*
- `FormField.initialValue`: Là giá trị dùng để khôi phục khi phương thức `FormState.reset()` được gọi.
- Nếu một `TextFormField` được truyền một `controller`, thuộc tính `initialValue` bắt buộc phải để `null` (nếu khai báo cả hai, framework sẽ ném ngoại lệ assert). Khi có controller, chính `controller.text` sẽ đóng vai trò quản lý giá trị khởi tạo và chuỗi hiển thị.

---

#### Câu hỏi 3: Phương thức `Scrollable.ensureVisible()` hoạt động như thế nào khi bàn phím mở lên?
*Phân tích:*
Khi một trường nhập liệu nhận tiêu điểm (`hasFocus == true`), nó gọi phương thức `Scrollable.ensureVisible(context)`. Framework tính toán tọa độ `Rect` của widget đó so với khung nhìn khả dụng của `Scrollable` cha. Nếu khoảng cách từ mép dưới của widget đến đáy màn hình nhỏ hơn chiều cao của bàn phím (`viewInsets.bottom`), `Scrollable` sẽ tự động kích hoạt một animation cuộn (Scroll Animation) dịch chuyển nội dung lên trên vừa đủ để ô nhập liệu nằm trọn vẹn trong vùng an toàn.

---

#### Câu hỏi 4: Làm thế nào để ngăn chặn hiện tượng mất dữ liệu trong Custom FormField khi nằm trong danh sách dài `ListView`?
*Phân tích:*
Do cơ chế ảo hóa danh sách của Flutter, khi một phần tử bị cuộn ra khỏi màn hình, `Element` và `State` của nó có thể bị hủy bỏ để thu hồi bộ nhớ. Để bảo toàn dữ liệu:
1. Bọc widget tùy biến bằng `AutomaticKeepAliveClientMixin` và ghi đè `wantKeepAlive => true`.
2. Hoặc lưu trữ giá trị nhập liệu vào một lớp quản lý trạng thái bên ngoài (Model / Controller) thay vì chỉ phụ thuộc vào bộ nhớ cục bộ của widget.

---

#### Câu hỏi 5: Tại sao việc lồng nhiều widget `SafeArea` liên tiếp có thể gây sai lệch khoảng đệm?
*Phân tích:*
Widget `SafeArea` đọc `MediaQuery.padding` từ ngữ cảnh cha, áp dụng khoảng đệm đó vào layout con và truyền một `MediaQuery` mới đã được trừ đi padding xuống các con bên dưới (`padding = EdgeInsets.zero`). Nếu một widget con bên dưới tiếp tục bọc thêm một `SafeArea` khác một cách không cần thiết, nó sẽ không gây lỗi nhưng nếu đọc ngược lại từ ngữ cảnh cũ có thể gây hiện tượng cộng dồn khoảng cách hai lần.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho một widget hiển thị cấu trúc sau:

```dart
class InsetDemo extends StatelessWidget {
  const InsetDemo({super.key});

  @override
  Widget build(BuildContext context) {
    final insets = MediaQuery.of(context).viewInsets.bottom;
    final padding = MediaQuery.of(context).padding.bottom;
    debugPrint('Rebuild: insets = $insets, padding = $padding');

    return Scaffold(
      body: Center(child: TextField(autofocus: false)),
    );
  }
}
```

Giả sử ứng dụng đang chạy trên thiết bị có thanh điều hướng Home Indicator cao $34\text{px}$ và bàn phím ảo khi mở lên có chiều cao $300\text{px}$.
1. Khi màn hình vừa khởi chạy (bàn phím chưa mở), dòng log in ra những giá trị nào?
2. Khi người dùng chạm vào `TextField` và bàn phím trượt lên hoàn tất, dòng log cuối cùng in ra những giá trị nào?
3. Giải thích nguyên nhân vì sao giá trị của `padding.bottom` thay đổi khi bàn phím xuất hiện.

---

#### Kết quả phân tích kỹ thuật:

1. **Khi màn hình vừa khởi chạy (Bàn phím chưa mở):**
   - Bàn phím chưa xuất hiện: `viewInsets.bottom = 0.0`.
   - Thanh điều hướng Home Indicator phần cứng: `padding.bottom = 34.0`.
   - Dòng log in ra:
     **`Rebuild: insets = 0.0, padding = 34.0`**

2. **Khi bàn phím trượt lên hoàn tất:**
   - Bàn phím chiếm chiều cao $300\text{px}$: `viewInsets.bottom = 300.0`.
   - Thanh Home Indicator bị bàn phím che khuất hoàn toàn. Do đó vùng an toàn của cửa sổ không còn bị cản bởi phần cứng: `padding.bottom = 0.0`.
   - Dòng log cuối cùng in ra:
     **`Rebuild: insets = 300.0, padding = 0.0`**

3. **Giải thích nguyên nhân:**
   - Thuộc tính `padding.bottom` đại diện cho khoảng cách cần tránh để không bị che bởi phần cứng thiết bị.
   - Khi bàn phím xuất hiện, mép dưới của cửa sổ hiển thị chính là mép trên của bàn phím ảo (vốn đã nằm đè lên thanh Home Indicator). Bàn phím ảo được tính vào `viewInsets`.
   - Do đó, phần diện tích cần né tránh của phần cứng ở đáy màn hình được đưa về `0.0` để tránh việc áp dụng đúp hai lần khoảng cách ($300\text{px} + 34\text{px}$), giúp bố cục hiển thị áp sát tự nhiên ngay trên mép bàn phím.
