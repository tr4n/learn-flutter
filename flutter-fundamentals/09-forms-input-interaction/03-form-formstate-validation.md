# Bài 9.3 — Kiến Trúc Biểu Mẫu: Form, FormState & Cơ Chế Xác Thực Dữ Liệu

## Tài Liệu Tham Khảo Chính Thức
- [Flutter Documentation: Build a form with validation](https://docs.flutter.dev/cookbook/forms/validation)
- [Flutter API: Form class](https://api.flutter.dev/flutter/widgets/Form-class.html)
- [Flutter API: FormState class](https://api.flutter.dev/flutter/widgets/FormState-class.html)
- [Flutter API: FormField class](https://api.flutter.dev/flutter/widgets/FormField-class.html)
- [Flutter API: TextFormField class](https://api.flutter.dev/flutter/material/TextFormField-class.html)
- [Flutter API: AutovalidateMode enum](https://api.flutter.dev/flutter/widgets/AutovalidateMode.html)

---

## Phần 1 — Khái Niệm & Vai Trò Của Form Trong Flutter

### 1.1 — Kiến Trúc Biểu Mẫu (Form Architecture)

Khi một màn hình chứa nhiều trường nhập liệu có mối liên kết logic (ví dụ biểu mẫu đăng ký gồm: Họ tên, Email, Mật khẩu, Xác nhận mật khẩu), việc quản lý thủ công từng `TextEditingController` độc lập sẽ làm mã nguồn trở nên cồng kềnh.

Flutter cung cấp bộ ba thành phần để giải quyết bài toán quản trị biểu mẫu:

```
┌────────────────────────────────────────────────────────────────────────┐
│ MÔ HÌNH KIẾN TRÚC FORM TRONG FLUTTER                                   │
│                                                                        │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ GlobalKey<FormState>                                               │ │
│ │  • Cung cấp điểm truy cập điều khiển: validate(), save(), reset()  │ │
│ └──────────────────────────────────┬─────────────────────────────────┘ │
│                                    │ Liên kết và điều phối             │
│                                    ▼                                   │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ Form (Container Widget)                                            │ │
│ │  • Cung cấp _FormScope (InheritedWidget) cho các con               │ │
│ └──────────────┬──────────────────────────────────────┬──────────────┘ │
│                │ Quản lý tập hợp các trường           │                │
│                ▼                                      ▼                │
│ ┌─────────────────────────────┐      ┌─────────────────────────────┐   │
│ │ TextFormField (Email)       │      │ TextFormField (Password)    │   │
│ │  • validator()              │      │  • validator()              │   │
│ │  • onSaved()                │      │  • onSaved()                │   │
│ └─────────────────────────────┘      └─────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

1. **`Form`**: Widget đóng vai trò là thùng chứa (Container), cung cấp ngữ cảnh chung cho toàn bộ các trường nhập liệu bên trong.
2. **`FormField<T>` / `TextFormField`**: Widget đại diện cho một trường nhập liệu đơn lẻ có khả năng duy trì trạng thái (`FormFieldState<T>`), thực thi logic kiểm tra tính hợp lệ (`validator`) và lưu trữ dữ liệu (`onSaved`).
3. **`FormState`**: Lớp quản lý trạng thái nội bộ của `Form`, được truy xuất thông qua `GlobalKey<FormState>`.

---

### 1.2 — Các Chế Độ Tự Động Xác Thực (`AutovalidateMode`)

Thuộc tính `autovalidateMode` quyết định thời điểm logic kiểm tra lỗi được kích hoạt:

| Giá Trị | Cơ Chế Hoạt Động | Đánh Giá UX |
| :--- | :--- | :--- |
| **`disabled` (Mặc định)** | Không bao giờ tự động validate. Chỉ kiểm tra khi gọi `formKey.currentState!.validate()` tường minh. | Phù hợp khi chỉ muốn báo lỗi sau khi người dùng đã nhấn nút "Gửi" (Submit). |
| **`always`** | Tự động chạy validator ở mỗi frame ngay từ khi màn hình vừa khởi tạo. | **Không khuyến nghị**: Giao diện hiển thị lỗi đỏ ngay khi người dùng chưa kịp tương tác, gây ức chế tâm lý. |
| **`onUserInteraction`** | Chỉ bắt đầu tự động validate sau khi người dùng đã chạm hoặc gõ ký tự đầu tiên vào trường đó. | Phản hồi thời gian thực mượt mà: Sau khi người dùng gõ sai, thông báo lỗi tự động biến mất ngay khi họ sửa đúng. |

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Cơ Chế Đăng Ký Tự Động Giữa Form Và FormField

Làm thế nào mà `Form` biết được có bao nhiêu trường con bên trong nó, ngay cả khi các trường đó nằm sâu bên trong các cấu trúc layout phức tạp (`Column`, `Padding`, `Card`)?

```
┌────────────────────────────────────────────────────────────────────────┐
│ CƠ CHẾ TỰ ĐĂNG KÝ VÀ QUẢN LÝ DANH SÁCH FIELD                          │
│                                                                        │
│ 1. Form widget tạo ra một _FormScope (kế thừa InheritedWidget)         │
│      │                                                                 │
│      ▼                                                                 │
│ 2. FormFieldState.initState() được gọi:                                │
│    • Tìm kiếm FormState gần nhất qua: Form._maybeOf(context)           │
│    • Nếu tìm thấy FormState:                                           │
│      -> Gọi FormState._register(this)                                  │
│      -> Lưu trữ tham chiếu vào Set<FormFieldState<dynamic>> _fields    │
│      │                                                                 │
│      ▼                                                                 │
│ 3. FormFieldState.dispose() được gọi:                                  │
│    • Tự động gọi FormState._unregister(this) để giải phóng tham chiếu │
└────────────────────────────────────────────────────────────────────────┘
```

Nhờ mô hình này:
- Lập trình viên không cần truyền danh sách controller vào `Form`.
- Cấu trúc cây widget hoàn toàn linh hoạt: Có thể thêm hoặc bớt các trường nhập liệu một cách động (Dynamic Forms).

---

### 2.2 — Chu Trình Thực Thi Của Phương Thức `validate()`

Khi gọi `formKey.currentState!.validate()`, một quy trình tuần tự được kích hoạt:

```
┌────────────────────────────────────────────────────────────────────────┐
│ QUY TRÌNH THỰC THI CỦA FORMSTATE.VALIDATE()                            │
│                                                                        │
│ bool hasError = false;                                                 │
│ Duyệt qua từng field trong Set<FormFieldState> _fields:                │
│    │                                                                   │
│    ├──► Gọi hàm validator(field.value)                                 │
│    │      │                                                            │
│    │      ├── Trả về String (Có lỗi):                                  │
│    │      │     • field.setState(() => errorText = errorMessage)       │
│    │      │     • hasError = true;                                     │
│    │      │                                                            │
│    │      └── Trả về null (Hợp lệ):                                    │
│    │            • field.setState(() => errorText = null)               │
│    │                                                                   │
│ Kết quả cuối cùng: return !hasError;                                   │
└────────────────────────────────────────────────────────────────────────┘
```

- Nếu hàm trả về `true`: Tất cả các trường đều đạt tiêu chuẩn, an toàn để tiếp tục bước gửi dữ liệu.
- Nếu hàm trả về `false`: Các trường vi phạm sẽ tự động hiển thị dòng chữ thông báo lỗi màu đỏ ngay dưới ô nhập, và chu trình submit bị chặn lại.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Biểu Mẫu Đăng Ký Tài Khoản Hoàn Chỉnh

Biểu mẫu áp dụng kiểm tra định dạng email bằng biểu thức chính quy (Regex), kiểm tra độ dài mật khẩu và xác nhận tính trùng khớp:

```dart
import 'package:flutter/material.dart';

// DTO lưu trữ kết quả đầu ra
class RegistrationData {
  String email = '';
  String password = '';
}

class RegisterFormScreen extends StatefulWidget {
  const RegisterFormScreen({super.key});

  @override
  State<RegisterFormScreen> createState() => _RegisterFormScreenState();
}

class _RegisterFormScreenState extends State<RegisterFormScreen> {
  // 1. Khởi tạo GlobalKey duy nhất để quản lý FormState
  final _formKey = GlobalKey<FormState>();
  final _formData = RegistrationData();
  final _passwordController = TextEditingController();

  @override
  void dispose() {
    _passwordController.dispose();
    super.dispose();
  }

  void _submitForm() {
    // 2. Kích hoạt kiểm tra tính hợp lệ toàn bộ biểu mẫu
    if (_formKey.currentState?.validate() ?? false) {
      // 3. Kích hoạt onSaved() trên từng field để ghi dữ liệu vào DTO
      _formKey.currentState!.save();

      // Đóng bàn phím
      FocusManager.instance.primaryFocus?.unfocus();

      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Đăng ký thành công: ${_formData.email}')),
      );
    }
  }

  void _resetForm() {
    // 4. Xóa toàn bộ dữ liệu đã nhập và đưa các thông báo lỗi về trạng thái ban đầu
    _formKey.currentState?.reset();
    _passwordController.clear();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Đăng Ký Tài Khoản')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(24.0),
        child: Form(
          key: _formKey,
          // Tự động kiểm tra lỗi khi người dùng bắt đầu nhập
          autovalidateMode: AutovalidateMode.onUserInteraction,
          child: Column(
            children: [
              // Trường Email
              TextFormField(
                keyboardType: TextInputType.emailAddress,
                decoration: const InputDecoration(
                  labelText: 'Email',
                  prefixIcon: Icon(Icons.email_outlined),
                  border: OutlineInputBorder(),
                ),
                validator: FormValidators.validateEmail,
                onSaved: (value) => _formData.email = value?.trim() ?? '',
              ),
              const SizedBox(height: 16),

              // Trường Mật Khẩu
              TextFormField(
                controller: _passwordController,
                obscureText: true,
                decoration: const InputDecoration(
                  labelText: 'Mật khẩu',
                  prefixIcon: Icon(Icons.lock_outline),
                  border: OutlineInputBorder(),
                ),
                validator: FormValidators.validatePassword,
                onSaved: (value) => _formData.password = value ?? '',
              ),
              const SizedBox(height: 16),

              // Trường Xác Nhận Mật Khẩu
              TextFormField(
                obscureText: true,
                decoration: const InputDecoration(
                  labelText: 'Xác nhận mật khẩu',
                  prefixIcon: Icon(Icons.lock_reset),
                  border: OutlineInputBorder(),
                ),
                validator: (value) => FormValidators.validateConfirmPassword(
                  value,
                  _passwordController.text,
                ),
              ),
              const SizedBox(height: 24),

              Row(
                children: [
                  Expanded(
                    child: OutlinedButton(
                      onPressed: _resetForm,
                      child: const Text('Nhập Lại'),
                    ),
                  ),
                  const SizedBox(width: 16),
                  Expanded(
                    child: ElevatedButton(
                      onPressed: _submitForm,
                      child: const Text('Đăng Ký'),
                    ),
                  ),
                ],
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

### 3.2 — Tách Biệt Bộ Quy Tắc Xác Thực (Decoupled Validation Rules)

Tách các hàm `validator` thành các hàm thuần túy (Pure Functions) không phụ thuộc vào widget, giúp dễ dàng viết Unit Test:

```dart
abstract class FormValidators {
  static final _emailRegex = RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$');

  static String? validateEmail(String? value) {
    if (value == null || value.trim().isEmpty) {
      return 'Vui lòng nhập địa chỉ email.';
    }
    if (!_emailRegex.hasMatch(value.trim())) {
      return 'Địa chỉ email không đúng định dạng.';
    }
    return null; // Hợp lệ
  }

  static String? validatePassword(String? value) {
    if (value == null || value.isEmpty) {
      return 'Vui lòng nhập mật khẩu.';
    }
    if (value.length < 8) {
      return 'Mật khẩu phải chứa ít nhất 8 ký tự.';
    }
    return null;
  }

  static String? validateConfirmPassword(String? value, String originalPassword) {
    if (value == null || value.isEmpty) {
      return 'Vui lòng xác nhận mật khẩu.';
    }
    if (value != originalPassword) {
      return 'Mật khẩu xác nhận không khớp.';
    }
    return null;
  }
}
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Bỏ qua kiểm tra `null` trong hàm `validator`

#### Mô tả vấn đề:
Truy xuất trực tiếp các phương thức trên biến `value` mà không kiểm tra null:
```dart
// Lỗi runtime: Null check operator used on a null value
validator: (value) {
  if (value!.isEmpty) return 'Lỗi'; // Ném TypeError khi value là null!
}
```

#### Biện pháp khắc phục:
Luôn kiểm tra an toàn theo cú pháp null-safe:
```dart
validator: (value) {
  if (value == null || value.trim().isEmpty) {
    return 'Vui lòng điền thông tin.';
  }
  return null;
}
```

---

### 4.2 — Quên gọi `form.save()` trước khi sử dụng biến kết quả

#### Mô tả vấn đề:
Lập trình viên kiểm tra `if (_formKey.currentState!.validate())` thành công, nhưng lập tức gửi object dữ liệu lên API mà không gọi `_formKey.currentState!.save()`.

#### Nguyên nhân kỹ thuật:
Phương thức `validate()` chỉ kiểm tra lỗi và trả về boolean. Callbacks `onSaved: (val) { ... }` trên từng `TextFormField` chỉ thực sự được kích hoạt khi phương thức `save()` được gọi tường minh. Nếu bỏ qua bước này, các trường trong DTO vẫn giữ giá trị rỗng ban đầu.

#### Biện pháp khắc phục:
Luôn gọi `save()` ngay sau khi `validate()` trả về `true`:
```dart
if (_formKey.currentState?.validate() ?? false) {
  _formKey.currentState!.save();
  _sendApiRequest(_formData);
}
```

---

### 4.3 — Đặt logic bất đồng bộ (Gọi API) trực tiếp vào trong `validator`

#### Mô tả vấn đề:
Cố gắng biến hàm `validator` thành hàm bất đồng bộ (`async`) để kiểm tra email đã tồn tại trên server hay chưa:
```dart
// SAI LẦM: validator trong Flutter chỉ chấp nhận hàm đồng bộ
validator: (value) async {
  final exists = await api.checkEmail(value); // Không hợp lệ về kiểu dữ liệu!
  return exists ? 'Email đã tồn tại' : null;
}
```

#### Nguyên nhân kỹ thuật:
Chữ ký của hàm `FormFieldValidator<T>` là:
`String? Function(T? value)` (hàm đồng bộ). Framework không hỗ trợ `Future<String?>` trong chu trình render vì giao diện cần hiển thị kết quả ngay trong khung hình hiện tại mà không thể chờ phản hồi mạng.

#### Biện pháp khắc phục:
1. `validator` chỉ chịu trách nhiệm kiểm tra định dạng cú pháp cục bộ (Format, độ dài).
2. Khi người dùng nhấn nút Submit, gọi API kiểm tra bất đồng bộ. Nếu máy chủ báo trùng, cập nhật lỗi thông qua một biến state hoặc cơ chế State Management.

---

### 4.4 — Khởi tạo `GlobalKey<FormState>` bên trong phương thức `build()`

#### Mô tả vấn đề:
Khai báo `final formKey = GlobalKey<FormState>();` trực tiếp trong hàm `build()`.

#### Nguyên nhân kỹ thuật:
Mỗi lần widget rebuild, một `GlobalKey` mới được cấp phát trong bộ nhớ. Toàn bộ trạng thái lỗi và dữ liệu nhập trước đó của `FormState` cũ bị hủy bỏ hoàn toàn.

#### Biện pháp khắc phục:
Luôn khai báo `GlobalKey` là biến thành viên của lớp `State` trong `StatefulWidget`.

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Lớp `FormState` xử lý như thế nào khi một `FormField` nằm trong một Widget con bị ẩn đi do điều kiện `if`?
*Phân tích:*
Khi điều kiện `if` đổi sang `false`, `FormField` bị tháo gỡ khỏi cây Element. Phương thức `dispose()` của `FormFieldState` được framework tự động gọi. Tại đây, nó kích hoạt hàm `FormState._unregister(this)`, loại bỏ tham chiếu của chính nó khỏi tập hợp `_fields` của Form cha. Do đó, khi `formKey.currentState!.validate()` được gọi ở các lần tiếp theo, trường bị ẩn hoàn toàn không còn tham gia vào quá trình kiểm tra lỗi.

---

#### Câu hỏi 2: Sự khác biệt bản chất giữa `FormState.reset()` và việc gán rỗng các `TextEditingController`?
*Phân tích:*
- Gán `controller.clear()`: Chỉ đơn thuần đưa chuỗi văn bản về rỗng `""`. Các dòng chữ thông báo lỗi màu đỏ (nếu đang hiển thị) vẫn giữ nguyên trên màn hình.
- Gọi `formKey.currentState!.reset()`: Đưa toàn bộ các `FormField` con về lại trạng thái khởi tạo ban đầu (`initialValue`), đồng thời **xóa sạch toàn bộ các thông báo lỗi (`errorText = null`)** và gán lại cờ trạng thái hợp lệ.

---

#### Câu hỏi 3: Tại sao phương thức `validate()` duyệt qua toàn bộ các trường thay vì dừng lại ngay khi gặp lỗi đầu tiên (Short-circuit)?
*Phân tích:*
Mục tiêu cốt lõi của giao diện người dùng (UI) là cung cấp bức tranh phản hồi toàn diện. Nếu áp dụng cơ chế dừng sớm (Short-circuit evaluation), người dùng sẽ chỉ nhìn thấy lỗi ở ô đầu tiên. Sau khi sửa xong và bấm Submit lần hai, họ lại thấy lỗi ở ô thứ hai, gây ức chế và làm gián đoạn trải nghiệm nhập liệu. Việc duyệt qua toàn bộ danh sách cho phép hiển thị đồng loạt tất cả các trường chưa đạt yêu cầu trong một lần kiểm tra duy nhất.

---

#### Câu hỏi 4: Làm thế nào để tự động cuộn màn hình đến vị trí của ô nhập liệu đầu tiên bị lỗi?
*Phân tích:*
1. Gán một `FocusNode` riêng biệt cho từng trường nhập liệu.
2. Trong hàm `validator`, khi phát hiện lỗi đầu tiên, lưu tham chiếu của `FocusNode` đó vào một biến tạm.
3. Nếu `validate()` trả về `false`, gọi `focusNode.requestFocus()`.
4. Khi node nhận tiêu điểm, framework (thông qua `Scrollable.ensureVisible`) sẽ tự động cuộn khung nhìn đưa ô nhập liệu bị lỗi vào giữa màn hình hiển thị.

---

#### Câu hỏi 5: Ưu điểm của việc sử dụng `TextFormField` kết hợp `onSaved` so với việc tạo hàng chục `TextEditingController`?
*Phân tích:*
Khi biểu mẫu có từ 10 đến 20 trường nhập liệu (như form khai báo y tế, đăng ký hồ sơ):
- Nếu dùng `TextEditingController`: Phải khởi tạo 20 instance, quản lý 20 biến thành viên, và viết 20 dòng lệnh `dispose()` thủ công để tránh rò rỉ bộ nhớ.
- Nếu dùng `TextFormField` với `onSaved`: Không cần bất kỳ `TextEditingController` nào. Dữ liệu được trích xuất trực tiếp từ chuỗi thô của từng `FormFieldState` và ánh xạ thẳng vào đối tượng Model chỉ bằng một lệnh gọi `formKey.currentState!.save()`, giúp mã nguồn gọn gàng và an toàn tài nguyên.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho một biểu mẫu với 3 trường nhập liệu có hàm `validator` và `onSaved` như sau:

```dart
String field1 = 'initial';
String field2 = 'initial';
String field3 = 'initial';

// Trường 1:
validator: (v) => (v == null || v.isEmpty) ? 'Lỗi 1' : null,
onSaved: (v) => field1 = 'saved_1',

// Trường 2:
validator: (v) => (v == null || v.length < 5) ? 'Lỗi 2' : null,
onSaved: (v) => field2 = 'saved_2',

// Trường 3:
validator: (v) => null, // Luôn hợp lệ
onSaved: (v) => field3 = 'saved_3',
```

Người dùng nhập vào các giá trị:
- Ô 1: `'Flutter'` (Hợp lệ)
- Ô 2: `'abc'` (Độ dài bằng 3, không hợp lệ)
- Ô 3: `'Dart'` (Hợp lệ)

Đoạn mã xử lý khi bấm nút Submit:
```dart
final isValid = formKey.currentState?.validate() ?? false;
if (isValid) {
  formKey.currentState?.save();
}
```

#### Yêu cầu phân tích:
1. Biến `isValid` sẽ có giá trị bằng bao nhiêu?
2. Có bao nhiêu hàm `validator` được thực thi?
3. Giá trị của 3 biến `field1`, `field2`, `field3` sau khi đoạn mã trên chạy xong là gì?

---

#### Kết quả phân tích kỹ thuật:

1. **Giá trị của `isValid`:**
   - Trường 1: Hợp lệ (`null`).
   - Trường 2: Độ dài 3 < 5 $\to$ Trả về chuỗi `'Lỗi 2'`.
   - Trường 3: Hợp lệ (`null`).
   - Do có ít nhất 1 trường trả về chuỗi lỗi, biến `isValid` nhận giá trị **`false`**.

2. **Số lượng hàm `validator` được thực thi:**
   - `FormState.validate()` không dừng sớm mà duyệt qua toàn bộ các phần tử trong danh sách.
   - Do đó, cả **3 hàm `validator`** đều được thực thi đầy đủ.

3. **Giá trị của 3 biến sau khi kết thúc:**
   - Do `isValid == false`, khối lệnh `if (isValid)` không được thực thi $\to$ Phương thức `formKey.currentState?.save()` **không bao giờ được gọi**.
   - Do đó, các callback `onSaved` không hoạt động.
   - Cả 3 biến vẫn giữ nguyên giá trị ban đầu:
     - `field1`: **`'initial'`**
     - `field2`: **`'initial'`**
     - `field3`: **`'initial'`**
