# Bài 9.2 — Quản Lý Nhập Liệu Văn Bản: TextField, Controller & Focus System

## Tài Liệu Tham Khảo Chính Thức
- [Flutter Documentation: Create and style a text field](https://docs.flutter.dev/cookbook/forms/text-input)
- [Flutter Documentation: Focus and text fields](https://docs.flutter.dev/cookbook/forms/focus)
- [Flutter API: TextField class](https://api.flutter.dev/flutter/material/TextField-class.html)
- [Flutter API: TextEditingController class](https://api.flutter.dev/flutter/widgets/TextEditingController-class.html)
- [Flutter API: FocusNode class](https://api.flutter.dev/flutter/widgets/FocusNode-class.html)
- [Flutter API: TextInputFormatter class](https://api.flutter.dev/flutter/services/TextInputFormatter-class.html)

---

## Phần 1 — Khái Niệm & Vai Trò Của Hệ Thống Nhập Liệu

### 1.1 — Phân Loại: Controlled vs Uncontrolled TextField

Trong Flutter, có hai cách tiếp cận để thu thập dữ liệu từ người dùng qua ô nhập liệu `TextField`:

1. **Uncontrolled (Không điều khiển bằng Controller)**:
   - Chỉ sử dụng callback `onChanged: (value) { ... }`.
   - **Đặc điểm**: Nhẹ, không cần khởi tạo và quản lý vòng đời đối tượng.
   - **Hạn chế**: Không thể chủ động thay đổi giá trị của ô nhập từ bên ngoài (ví dụ nút "Xóa hết" hoặc nút "Điền mẫu"), không thể kiểm soát vị trí con trỏ (cursor position).
   - **Ngữ cảnh sử dụng**: Các ô tìm kiếm đơn giản, bộ lọc tức thì.

2. **Controlled (Có điều khiển bằng `TextEditingController`)**:
   - Khởi tạo và gán một instance `TextEditingController` vào thuộc tính `controller`.
   - **Đặc điểm**: Cung cấp toàn quyền kiểm soát văn bản: đọc giá trị tại bất kỳ thời điểm nào (`controller.text`), gán giá trị mới, lắng nghe sự kiện thay đổi (`controller.addListener`), và chọn vùng văn bản (`controller.selection`).
   - **Yêu cầu kỹ thuật**: Bắt buộc phải giải phóng tài nguyên (`dispose()`) khi không còn sử dụng.

---

### 1.2 — Hệ Thống Quản Lý Tiêu Điểm (Focus System)

Hệ thống Focus trong Flutter quản lý thành phần nào đang sẵn sàng tiếp nhận sự kiện bàn phím:
- **`FocusNode`**: Đối tượng đại diện cho tiêu điểm của một widget cụ thể. Quản lý trạng thái có đang được focus hay không (`hasFocus`) và phát tín hiệu khi trạng thái thay đổi.
- **`FocusScope`**: Không gian gom nhóm các `FocusNode` lại với nhau, điều phối thứ tự chuyển tiếp tiêu điểm (như phím Tab trên bàn phím vật lý hoặc nút "Next" trên bàn phím ảo).
- **`FocusManager`**: Lớp quản lý toàn cục cấp cao nhất của framework (`FocusManager.instance`), lưu trữ node đang chiếm tiêu điểm chính (`primaryFocus`).

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Giao Tiếp Với Bàn Phím Hệ Điều Hành (TextInput Platform Channel)

Flutter tự vẽ (render) toàn bộ giao diện bằng Skia/Impeller, nhưng bàn phím ảo (Software Keyboard / IME - Input Method Editor) lại là một thành phần nguyên bản (Native) của hệ điều hành (Android/iOS):

```
┌────────────────────────────────────────────────────────────────────────┐
│ LUỒNG GIAO TIẾP VỚI BÀN PHÍM ẢO HỆ THỐNG                               │
│                                                                        │
│ 1. Người dùng chạm vào TextField                                       │
│      │                                                                 │
│      ▼                                                                 │
│ 2. FocusNode kích hoạt -> TextField yêu cầu mở kết nối:                │
│    SystemChannels.text_input.invokeMethod('TextInput.setClient', ...)  │
│    SystemChannels.text_input.invokeMethod('TextInput.show')            │
│      │                                                                 │
│      ▼ (Platform Channel)                                              │
│ 3. Hệ điều hành kích hoạt InputMethodService (Android) / UIKeyInput(iOS)│
│    Bàn phím ảo trượt lên từ dưới màn hình                              │
│      │                                                                 │
│      ▼                                                                 │
│ 4. Người dùng gõ ký tự trên bàn phím ảo:                               │
│    Hệ điều hành gửi dữ liệu ngược lại qua TextInput.updateEditingState │
│      │                                                                 │
│      ▼                                                                 │
│ 5. Flutter Engine cập nhật TextEditingValue:                           │
│    • Áp dụng chuỗi TextInputFormatter                                 │
│    • Cập nhật chuỗi text và vị trí selection                           │
│    • Thông báo cho TextEditingController và vẽ lại khung nhìn          │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 2.2 — Cấu Trúc Dữ Liệu `TextEditingValue`

`TextEditingController` kế thừa từ `ValueNotifier<TextEditingValue>`. Mỗi khi văn bản thay đổi, một đối tượng `TextEditingValue` mới được phát ra, chứa 3 thông tin cốt lõi:

```dart
class TextEditingValue {
  final String text; // Chuỗi ký tự hiện tại
  final TextSelection selection; // Vị trí con trỏ hoặc vùng đang bôi đen
  final TextRange composing; // Vùng ký tự đang được bộ gõ (IME) xử lý
}
```

- **`TextSelection`**: Xác định con trỏ đang ở đâu. Nếu `baseOffset == extentOffset`, con trỏ đang ở dạng thanh đứng nhấp nháy tại chỉ mục đó. Nếu khác nhau, người dùng đang bôi đen một đoạn văn bản.
- **`TextRange composing`**: Rất quan trọng đối với các bộ gõ tiếng Việt (Telex/VNI) hoặc tiếng Nhật/Hàn. Khi người dùng đang gõ dở một từ (ví dụ `a` + `w` $\to$ `ă`), vùng này được đánh dấu gạch chân tạm thời trước khi chốt thành ký tự chính thức.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Biểu Mẫu Nhập Liệu Với Luồng Chuyển Focus Tự Động

Điều hướng tiêu điểm tuần tự từ ô Email $\to$ Mật khẩu $\to$ Đóng bàn phím và gửi dữ liệu:

```dart
import 'package:flutter/material.dart';

class LoginFormWithFocus extends StatefulWidget {
  const LoginFormWithFocus({super.key});

  @override
  State<LoginFormWithFocus> createState() => _LoginFormWithFocusState();
}

class _LoginFormWithFocusState extends State<LoginFormWithFocus> {
  // 1. Khởi tạo Controllers và FocusNodes
  late final TextEditingController _emailController;
  late final TextEditingController _passwordController;
  late final FocusNode _emailFocusNode;
  late final FocusNode _passwordFocusNode;

  bool _obscurePassword = true;

  @override
  void initState() {
    super.initState();
    _emailController = TextEditingController();
    _passwordController = TextEditingController();
    _emailFocusNode = FocusNode();
    _passwordFocusNode = FocusNode();
  }

  @override
  void dispose() {
    // 2. Bắt buộc giải phóng toàn bộ tài nguyên
    _emailController.dispose();
    _passwordController.dispose();
    _emailFocusNode.dispose();
    _passwordFocusNode.dispose();
    super.dispose();
  }

  void _submit() {
    // Thu hồi bàn phím khi submit
    FocusManager.instance.primaryFocus?.unfocus();
    debugPrint('Đăng nhập: ${_emailController.text}');
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      // Chạm ra ngoài vùng nhập liệu để tự động đóng bàn phím
      onTap: () => FocusScope.of(context).unfocus(),
      child: Scaffold(
        appBar: AppBar(title: const Text('Đăng Nhập')),
        body: Padding(
          padding: const EdgeInsets.all(24.0),
          child: Column(
            children: [
              // Ô nhập Email
              TextField(
                controller: _emailController,
                focusNode: _emailFocusNode,
                keyboardType: TextInputType.emailAddress,
                textInputAction: TextInputAction.next,
                decoration: const InputDecoration(
                  labelText: 'Email',
                  prefixIcon: Icon(Icons.email_outlined),
                  border: OutlineInputBorder(),
                ),
                // Khi bấm Next trên bàn phím, chuyển tiêu điểm sang Password
                onSubmitted: (_) {
                  FocusScope.of(context).requestFocus(_passwordFocusNode);
                },
              ),
              const SizedBox(height: 16),

              // Ô nhập Mật khẩu
              TextField(
                controller: _passwordController,
                focusNode: _passwordFocusNode,
                obscureText: _obscurePassword,
                textInputAction: TextInputAction.done,
                decoration: InputDecoration(
                  labelText: 'Mật khẩu',
                  prefixIcon: const Icon(Icons.lock_outline),
                  border: const OutlineInputBorder(),
                  suffixIcon: IconButton(
                    icon: Icon(
                      _obscurePassword ? Icons.visibility_off : Icons.visibility,
                    ),
                    onPressed: () {
                      setState(() => _obscurePassword = !_obscurePassword);
                    },
                  ),
                ),
                onSubmitted: (_) => _submit(),
              ),
              const SizedBox(height: 24),

              ElevatedButton(
                onPressed: _submit,
                child: const Text('Đăng Nhập'),
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

### 3.2 — Định Dạng Văn Bản Thời Gian Thực Với `TextInputFormatter`

Tự động chèn dấu cách sau mỗi 4 chữ số khi người dùng nhập số thẻ ngân hàng (`XXXX XXXX XXXX XXXX`), đồng thời bảo toàn vị trí con trỏ:

```dart
import 'package:flutter/services.dart';

class CardNumberFormatter extends TextInputFormatter {
  @override
  TextEditingValue formatEditUpdate(
    TextEditingValue oldValue,
    TextEditingValue newValue,
  ) {
    // 1. Chỉ giữ lại các chữ số, loại bỏ ký tự không phải số
    final rawDigits = newValue.text.replaceAll(RegExp(r'\D'), '');

    // Giới hạn tối đa 16 chữ số
    final truncated = rawDigits.length > 16 ? rawDigits.substring(0, 16) : rawDigits;

    // 2. Chèn khoảng trắng sau mỗi 4 ký tự
    final buffer = StringBuffer();
    for (int i = 0; i < truncated.length; i++) {
      if (i > 0 && i % 4 == 0) {
        buffer.write(' ');
      }
      buffer.write(truncated[i]);
    }

    final formattedText = buffer.toString();

    // 3. Tính toán vị trí con trỏ sau khi format
    return TextEditingValue(
      text: formattedText,
      selection: TextSelection.collapsed(offset: formattedText.length),
    );
  }
}
```

Cách áp dụng vào `TextField`:
```dart
TextField(
  keyboardType: TextInputType.number,
  inputFormatters: [
    FilteringTextInputFormatter.digitsOnly,
    CardNumberFormatter(),
  ],
  decoration: const InputDecoration(
    labelText: 'Số Thẻ Thanh Toán',
    hintText: '0000 0000 0000 0000',
  ),
)
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Khởi tạo Controller hoặc FocusNode bên trong phương thức `build()`

#### Mô tả vấn đề:
Khởi tạo instance trực tiếp trong `build()`:
```dart
@override
Widget build(BuildContext context) {
  // Lỗi: Controller mới được sinh ra ở mỗi frame
  final controller = TextEditingController();
  return TextField(controller: controller);
}
```

#### Nguyên nhân kỹ thuật:
Mỗi khi widget cha rebuild (ví dụ bàn phím xuất hiện làm thay đổi kích thước khung nhìn), một đối tượng `TextEditingController` hoàn toàn mới được gán vào `TextField`. Kết quả là văn bản người dùng vừa gõ bị xóa sạch, con trỏ bị mất và bàn phím tự động đóng lại.

#### Biện pháp khắc phục:
Luôn khởi tạo trong `initState()` của `StatefulWidget` hoặc quản lý thông qua State Management.

---

### 4.2 — Mất vị trí con trỏ khi cập nhật `controller.text` thủ công

#### Mô tả vấn đề:
Cập nhật văn bản bằng cách gán `controller.text = newText;`. Khi người dùng đang gõ ở giữa chuỗi, con trỏ lập tức bị nhảy về cuối dòng hoặc đầu dòng.

#### Nguyên nhân kỹ thuật:
Thuộc tính setter của `controller.text` mặc định sẽ đặt lại `selection = TextSelection.collapsed(offset: newText.length)`.

#### Biện pháp khắc phục:
Sử dụng thuộc tính `value` kết hợp với bảo toàn `selection`:
```dart
final currentSelection = _controller.selection;
_controller.value = _controller.value.copyWith(
  text: newText,
  selection: TextSelection.collapsed(
    offset: currentSelection.baseOffset.clamp(0, newText.length),
  ),
);
```

---

### 4.3 — Quên gọi `dispose()` trên Controller và FocusNode

#### Mô tả vấn đề:
Rời khỏi màn hình mà không dọn dẹp các đối tượng nhập liệu.

#### Nguyên nhân kỹ thuật:
`FocusNode` duy trì các listener liên kết với `FocusManager`. Khi không được dispose, cây tiêu điểm toàn cục vẫn tiếp tục giữ các node chết, gây rò rỉ bộ nhớ (Memory Leak) và phát sinh lỗi khi người dùng quay lại màn hình đó.

#### Biện pháp khắc phục:
Gọi `.dispose()` trên từng controller và focusNode trong hàm `dispose()` của `State`.

---

### 4.4 — Bàn phím không tự đóng khi người dùng chạm ra ngoài ô nhập

#### Mô tả vấn đề:
Sau khi gõ xong, người dùng nhấn vào khoảng trắng trên màn hình nhưng bàn phím ảo vẫn không chịu ẩn đi.

#### Biện pháp khắc phục:
Bọc `Scaffold` bên trong một `GestureDetector` với hành vi giải phóng tiêu điểm:
```dart
GestureDetector(
  onTap: () => FocusManager.instance.primaryFocus?.unfocus(),
  behavior: HitTestBehavior.opaque,
  child: Scaffold(...),
)
```

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Lớp `FocusManager` xử lý sự kiện như thế nào khi phương thức `FocusNode.requestFocus()` được gọi?
*Phân tích:*
1. Node hiện tại nhận lệnh gọi `_applyFocusChange()`.
2. `FocusManager` gửi tín hiệu `unfocus()` tới node đang chiếm giữ tiêu điểm cũ (`primaryFocus`).
3. Node cũ phát sự kiện `notifyListeners()`, chuyển cờ `hasFocus` sang `false`.
4. Node mới được gán làm `primaryFocus`, chuyển cờ `hasFocus` sang `true` và phát tín hiệu cho các widget lắng nghe.
5. Nếu node mới liên kết với một ô nhập văn bản, nó sẽ gửi tin nhắn qua `SystemChannels.text_input` để mở bàn phím ảo.

---

#### Câu hỏi 2: Sự khác biệt giữa `FocusScope.of(context).requestFocus(node)` và `node.requestFocus()`?
*Phân tích:*
- `node.requestFocus()`: Tự động tìm kiếm `FocusScope` gần nhất chứa nó và yêu cầu tiêu điểm trực tiếp.
- `FocusScope.of(context).requestFocus(node)`: Sử dụng `BuildContext` để xác định chính xác cây phạm vi (Scope Tree) mong muốn trước khi gán tiêu điểm. Trong hầu hết các trường hợp thông thường, hai cách viết hoạt động tương đương. Tuy nhiên khi có nhiều Scope lồng nhau (ví dụ: trong Dialog hoặc Modal BottomSheet), việc chỉ định Scope tường minh giúp tránh việc tiêu điểm bị nhảy nhầm ra Scope cha bên ngoài.

---

#### Câu hỏi 3: Tại sao `TextInputFormatter.formatEditUpdate` cung cấp cả `oldValue` và `newValue`?
*Phân tích:*
Việc cung cấp cả hai giá trị cho phép thuật toán nhận biết người dùng vừa thực hiện hành động gì:
- Nếu `newValue.text.length > oldValue.text.length`: Người dùng đang **thêm ký tự** (gõ phím hoặc dán văn bản).
- Nếu `newValue.text.length < oldValue.text.length`: Người dùng đang **xóa ký tự** (nhấn Backspace/Delete).
Nhờ đó, formatter có thể xử lý các logic phức tạp như: Không tự động chèn lại dấu phân cách khi người dùng đang cố tình xóa lùi.

---

#### Câu hỏi 4: Thuộc tính `keyboardType: TextInputType.number` có ngăn chặn người dùng nhập chữ cái không?
*Phân tích:*
**Không**. `TextInputType` chỉ là một chỉ dẫn (hint) gửi xuống hệ điều hành để hiển thị layout bàn phím số tương ứng. Trên một số bàn phím của bên thứ ba (như Gboard, SwiftKey) hoặc trên thiết bị có bàn phím cứng, người dùng vẫn có thể dán (paste) hoặc gõ các ký tự chữ cái vào ô. Để ngăn chặn triệt để, lập trình viên bắt buộc phải sử dụng `FilteringTextInputFormatter.digitsOnly` trong danh sách `inputFormatters`.

---

#### Câu hỏi 5: Điều gì xảy ra khi ứng dụng chuyển hướng (Navigator.push) trong lúc bàn phím vẫn đang mở?
*Phân tích:*
Nếu trước khi chuyển trang không gọi lệnh giải phóng tiêu điểm (`unfocus()`), `FocusNode` vẫn ở trạng thái `hasFocus: true`. Màn hình mới được đẩy lên đè lên màn hình cũ, hệ điều hành có thể giữ nguyên bàn phím ảo trên màn hình mới mà không có ô nhập nào tương ứng, gây ra lỗi hiển thị hoặc làm tràn bố cục (`Bottom Overflow`) trên màn hình đích.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho một trường nhập liệu có gắn `TextInputFormatter` tự động chèn dấu gạch ngang sau 3 ký tự:

```dart
class HyphenFormatter extends TextInputFormatter {
  @override
  TextEditingValue formatEditUpdate(TextEditingValue oldValue, TextEditingValue newValue) {
    if (newValue.text.length == 3 && oldValue.text.length == 2) {
      return TextEditingValue(
        text: '${newValue.text}-',
        selection: const TextSelection.collapsed(offset: 4),
      );
    }
    return newValue;
  }
}
```

Giả sử:
- Ban đầu trường dữ liệu đang có nội dung: `'123-'` với con trỏ ở vị trí `offset: 4`.
- Người dùng nhấn phím **Backspace** đúng 1 lần để xóa dấu gạch ngang.

#### Yêu cầu phân tích:
1. Giá trị của `oldValue` và `newValue` được truyền vào hàm `formatEditUpdate` khi phím Backspace được nhấn là gì?
2. Hàm sẽ trả về kết quả gì?
3. Điều gì sẽ xảy ra tiếp theo nếu người dùng tiếp tục nhấn phím `'4'` ngay sau đó?

---

#### Kết quả phân tích kỹ thuật:

1. **Giá trị truyền vào hàm:**
   - `oldValue`: `text = '123-'`, `selection = offset: 4`.
   - `newValue` (do hệ điều hành vừa xóa ký tự cuối cùng `'-'`): `text = '123'`, `selection = offset: 3`.

2. **Kết quả trả về của hàm:**
   - Kiểm tra điều kiện: `newValue.text.length == 3` (Thỏa mãn), nhưng `oldValue.text.length == 2` (Sai, vì `oldValue.text.length` bằng 4).
   - Do đó khối `if` không thực thi.
   - Hàm trả về chính `newValue`: `text = '123'`, `selection = offset: 3`. Dấu gạch ngang được xóa thành công.

3. **Khi người dùng nhấn tiếp phím `'4'`:**
   - `oldValue`: `text = '123'`, `selection = offset: 3`.
   - `newValue`: `text = '1234'`, `selection = offset: 4`.
   - Điều kiện `newValue.text.length == 3` không thỏa mãn.
   - Kết quả hiển thị là `'1234'`. (Nếu muốn hỗ trợ định dạng toàn diện, lập trình viên cần triển khai giải thuật duyệt toàn bộ chuỗi thay vì chỉ kiểm tra độ dài cục bộ).
