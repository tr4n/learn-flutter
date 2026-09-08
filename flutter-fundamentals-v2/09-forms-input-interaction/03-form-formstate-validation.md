# Bài 9.3 — Form, FormState & Validation

## Phần 1 — Khái Niệm & Mục Tiêu Bài Học

### Tại sao bài này quan trọng?

`Form` + `TextFormField` là Flutter's built-in form validation system. Nó giải quyết vấn đề phức tạp:
- Validate nhiều fields một lúc
- Manage error messages theo field
- Save/reset form state

```dart
// Một lệnh validate toàn bộ form
if (_formKey.currentState!.validate()) {
  _formKey.currentState!.save();
  // Submit!
}
```

### Bạn sẽ hiểu được sau bài này:
- `Form`, `GlobalKey<FormState>`, `TextFormField`
- `validator`: inline validation logic
- `autovalidateMode`: khi nào trigger validation
- `form.save()`, `form.validate()`, `form.reset()`

---

## Phần 2 — Cơ Chế Hoạt Động (Under the Hood)

### Form Validation Flow

```mermaid
flowchart TD
    User([User submit]) --> Validate["form.validate()"]
    Validate --> WalkTree["Walk FormField tree"]
    WalkTree --> Validator1["Field 1: validator(value)"]
    WalkTree --> Validator2["Field 2: validator(value)"]
    WalkTree --> ValidatorN["Field N: validator(value)"]
    Validator1 -->|null| Valid1[Valid ✓]
    Validator1 -->|string| Invalid1[Show error]
    Validator2 -->|null| Valid2[Valid ✓]
    Validator2 -->|string| Invalid2[Show error]
    Valid1 --> AllValid{All valid?}
    Valid2 --> AllValid
    Invalid1 --> AllValid
    Invalid2 --> AllValid
    AllValid -->|Yes| Save["form.save() → onSaved callbacks"]
    AllValid -->|No| Return[Return false]
```

---

## Phần 3 — Code Mẫu Chuẩn Google

### 3.1 — Register Form hoàn chỉnh

```dart
class RegisterScreen extends StatefulWidget {
  const RegisterScreen({super.key});
  @override State<RegisterScreen> createState() => _RegisterScreenState();
}

class _RegisterScreenState extends State<RegisterScreen> {
  // GlobalKey: truy cập FormState từ bên ngoài Form widget
  final _formKey = GlobalKey<FormState>();

  // Lưu giá trị form sau khi validate
  String? _savedEmail;
  String? _savedPassword;
  String? _savedName;
  bool _obscurePassword = true;
  bool _isLoading = false;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Đăng ký')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Form(
          key: _formKey,
          // onChanged: gọi mỗi khi bất kỳ field nào thay đổi
          // Useful để enable/disable submit button
          autovalidateMode: AutovalidateMode.disabled, // Chỉ validate khi submit
          child: Column(
            children: [
              TextFormField(
                decoration: const InputDecoration(
                  labelText: 'Họ tên',
                  prefixIcon: Icon(Icons.person_outline),
                  border: OutlineInputBorder(),
                ),
                textCapitalization: TextCapitalization.words,
                textInputAction: TextInputAction.next,
                // validator: trả về null nếu hợp lệ, chuỗi lỗi nếu không
                validator: (value) {
                  if (value == null || value.trim().isEmpty) {
                    return 'Vui lòng nhập họ tên';
                  }
                  if (value.trim().length < 2) {
                    return 'Họ tên phải có ít nhất 2 ký tự';
                  }
                  return null; // Valid
                },
                // onSaved: gọi sau form.save()
                onSaved: (value) => _savedName = value?.trim(),
              ),
              const SizedBox(height: 16),

              TextFormField(
                decoration: const InputDecoration(
                  labelText: 'Email',
                  prefixIcon: Icon(Icons.email_outlined),
                  border: OutlineInputBorder(),
                ),
                keyboardType: TextInputType.emailAddress,
                textInputAction: TextInputAction.next,
                autocorrect: false,
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'Vui lòng nhập email';
                  }
                  // RFC-compliant email regex
                  final emailRegex = RegExp(
                    r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
                  );
                  if (!emailRegex.hasMatch(value)) {
                    return 'Email không hợp lệ';
                  }
                  return null;
                },
                onSaved: (value) => _savedEmail = value?.toLowerCase().trim(),
              ),
              const SizedBox(height: 16),

              TextFormField(
                decoration: InputDecoration(
                  labelText: 'Mật khẩu',
                  prefixIcon: const Icon(Icons.lock_outlined),
                  border: const OutlineInputBorder(),
                  suffixIcon: IconButton(
                    icon: Icon(_obscurePassword
                        ? Icons.visibility_off
                        : Icons.visibility),
                    onPressed: () => setState(() => _obscurePassword = !_obscurePassword),
                  ),
                ),
                obscureText: _obscurePassword,
                textInputAction: TextInputAction.done,
                onFieldSubmitted: (_) => _submit(),
                validator: (value) {
                  if (value == null || value.isEmpty) return 'Vui lòng nhập mật khẩu';
                  if (value.length < 8) return 'Mật khẩu phải có ít nhất 8 ký tự';
                  if (!value.contains(RegExp(r'[A-Z]'))) {
                    return 'Phải có ít nhất 1 chữ HOA';
                  }
                  if (!value.contains(RegExp(r'[0-9]'))) {
                    return 'Phải có ít nhất 1 chữ số';
                  }
                  return null;
                },
                onSaved: (value) => _savedPassword = value,
              ),
              const SizedBox(height: 24),

              SizedBox(
                width: double.infinity,
                height: 48,
                child: ElevatedButton(
                  onPressed: _isLoading ? null : _submit,
                  child: _isLoading
                      ? const SizedBox(
                          width: 20,
                          height: 20,
                          child: CircularProgressIndicator(strokeWidth: 2),
                        )
                      : const Text('Đăng ký'),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  Future<void> _submit() async {
    // 1. Validate tất cả fields
    if (!_formKey.currentState!.validate()) return;

    // 2. Save — gọi tất cả onSaved callbacks
    _formKey.currentState!.save();

    setState(() => _isLoading = true);

    try {
      // 3. Submit với saved values
      await AuthRepository().register(
        name: _savedName!,
        email: _savedEmail!,
        password: _savedPassword!,
      );

      if (!mounted) return;
      Navigator.pushReplacementNamed(context, '/home');
    } catch (e) {
      if (!mounted) return;
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Lỗi: $e')),
      );
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }
}
```

### 3.2 — AutovalidateMode

```dart
// AutovalidateMode: điều khiển khi nào trigger validator

// 1. disabled (default): chỉ validate khi gọi form.validate() thủ công
Form(autovalidateMode: AutovalidateMode.disabled, ...)

// 2. onUserInteraction: validate sau khi user interact với field
// → UX tốt nhất: không annoying khi form mới load
Form(autovalidateMode: AutovalidateMode.onUserInteraction, ...)

// 3. always: validate liên tục mỗi lần rebuild
// → Dùng khi cần validate ngay khi form render
Form(autovalidateMode: AutovalidateMode.always, ...)
```

### 3.3 — Form Reset

```dart
// Sau submit thành công hoặc Cancel button
_formKey.currentState!.reset(); // Xóa text VÀ error messages
// Hoặc set về initial values:
_emailController.clear();
_passwordController.clear();
```

---

## Phần 4 — Lỗi Sai Phổ Biến & Best Practices

### ❌ Anti-pattern 1: Validate không check null

```dart
// ❌ Crash: value có thể null khi field chưa được tap
validator: (value) {
  if (value.isEmpty) return 'Required'; // NullPointerException!
}

// ✅ Luôn check null trước
validator: (value) {
  if (value == null || value.isEmpty) return 'Bắt buộc';
  return null;
}
```

### ❌ Anti-pattern 2: Quên gọi form.save() trước khi dùng saved values

```dart
// ❌ _savedEmail null vì chưa gọi save()
_formKey.currentState!.validate();
submitData(_savedEmail!); // _savedEmail vẫn null!

// ✅ Thứ tự đúng: validate → save → use
if (_formKey.currentState!.validate()) {
  _formKey.currentState!.save(); // Gọi tất cả onSaved
  submitData(_savedEmail!); // Bây giờ _savedEmail có giá trị
}
```

### ❌ Anti-pattern 3: Đặt logic phức tạp trực tiếp trong validator

```dart
// ❌ Không tốt: validator gọi API (async không hỗ trợ)
validator: (value) async {
  final exists = await api.checkEmailExists(value); // ❌ Không work!
  return exists ? 'Email đã tồn tại' : null;
}

// ✅ Async validation: thực hiện trong submit handler
Future<void> _submit() async {
  if (!_formKey.currentState!.validate()) return;
  _formKey.currentState!.save();
  // Async check sau khi synchronous validation pass
  final exists = await api.checkEmailExists(_savedEmail!);
  if (exists) {
    _formKey.currentState!.fields['email']?.invalidate('Email đã tồn tại');
    return;
  }
  // ...
}
```

---

## Phần 5 — Bài Tập Củng Cố Tư Duy

### Challenge: Multi-step Registration Form

**Yêu cầu:**
1. Step 1: Personal info (Name, DOB, Gender)
2. Step 2: Account info (Email, Username, Password, Confirm Password)
3. Step 3: Review & Submit
4. Mỗi step: validate riêng trước khi Next
5. Back button: giữ nguyên data đã nhập

**Gợi ý:**
- Mỗi step có `Form` + `GlobalKey<FormState>` riêng
- Dùng `PageController` hoặc `IndexedStack` để switch steps
- Giữ data giữa steps trong parent State
- "Confirm Password" validation: so sánh với Password field

### Câu Hỏi Phỏng Vấn

> **[Junior]** — nắm khái niệm | **[Middle]** — hiểu cơ chế | **[Senior]** — hiểu Flutter internals | **[Trace Code]** — đọc code và dự đoán output

---

#### Q1 [Junior] — "Sự khác biệt giữa `Form + TextFormField` và `TextField` đơn thuần?"

**Trả lời chuẩn:**

| | `Form + TextFormField` | `TextField` standalone |
|---|---|---|
| **Validation** | Built-in (`validator` callback) | Phải tự implement |
| **Grouping** | `Form` quản lý tất cả fields | Riêng lẻ |
| **Save** | `formKey.currentState?.save()` | Phải đọc từng controller |
| **Reset** | `formKey.currentState?.reset()` | Phải set từng controller |
| **`onSaved`** | Có — `FormState.save()` gọi hết | Không |
| **Use case** | Complex forms (login, registration) | Simple single input |

```dart
// Form + TextFormField — recommended cho multi-field forms
final _formKey = GlobalKey<FormState>();
String? _email, _password;

Form(
  key: _formKey,
  autovalidateMode: AutovalidateMode.onUserInteraction,
  child: Column(children: [
    TextFormField(
      decoration: const InputDecoration(labelText: 'Email'),
      validator: (v) => v?.contains('@') == true ? null : 'Invalid email',
      onSaved: (v) => _email = v,
    ),
    TextFormField(
      obscureText: true,
      validator: (v) => (v?.length ?? 0) >= 8 ? null : 'Min 8 chars',
      onSaved: (v) => _password = v,
    ),
    ElevatedButton(
      onPressed: () {
        if (_formKey.currentState?.validate() == true) {
          _formKey.currentState?.save();
          login(_email!, _password!);
        }
      },
      child: const Text('Login'),
    ),
  ]),
)
```

---

#### Q2 [Junior] — "`AutovalidateMode.always` vs `onUserInteraction` vs `disabled`?"

**Trả lời chuẩn:**

| Mode | Validate khi | UX |
|---|---|---|
| `disabled` | Chỉ khi gọi `validate()` thủ công | Không show error cho đến khi submit |
| `onUserInteraction` | Sau khi user bắt đầu interact (type, tap) | Validation ngay lập tức sau touch |
| `always` | Mọi lúc, kể cả khi chưa touch | Error hiện ngay khi load — phiền |

```dart
// disabled (mặc định nếu không set trên Form)
Form(autovalidateMode: AutovalidateMode.disabled)

// onUserInteraction — KHUYẾN NGHỊ cho form UX
Form(autovalidateMode: AutovalidateMode.onUserInteraction)

// always — show error ngay lập tức (kể cả chưa touch)
Form(autovalidateMode: AutovalidateMode.always)
```

**Pattern thường gặp:** `Form(autovalidateMode: disabled)` + gọi `validate()` khi submit. Kết hợp với `onUserInteraction` trên từng `TextFormField` riêng lẻ.

---

#### Q3 [Middle] — "`formKey.currentState?.save()` làm gì? Thứ tự gọi `validate()` và `save()`?"

**Trả lời chuẩn:**

`FormState.save()` **iterate toàn bộ registered `FormField`** trong Form và gọi `onSaved` callback trên mỗi field:

```dart
// FormState.save() — Flutter source (simplified)
void save() {
  for (final FormFieldState<dynamic> field in _fields) {
    field.save(); // gọi onSaved(field.value)
  }
}
```

**Thứ tự bắt buộc: `validate()` TRƯỚC `save()`:**

```dart
// ✅ Đúng thứ tự
void _submit() {
  if (_formKey.currentState!.validate()) {
    // validate() = true → tất cả fields hợp lệ
    _formKey.currentState!.save();
    // save() collect tất cả values vào variables
    doLogin(_email!, _password!);
  }
}

// ❌ Sai — save trước validate → có thể save invalid data
void _badSubmit() {
  _formKey.currentState!.save();
  if (_formKey.currentState!.validate()) {
    doLogin(_email!, _password!); // có thể _email là invalid!
  }
}
```

---

#### Q4 [Senior] — "`Form` và `FormField` communicate thế nào? Registration mechanism?"

**Trả lời chuẩn:**

`Form` widget là `InheritedWidget` ancestor — `FormField` register với `Form` thông qua `FormState`:

```dart
// FormField.initState() — tự động register
@override
void initState() {
  super.initState();
  _register();
}

void _register() {
  final formState = Form.maybeOf(context) as _FormState?;
  formState?._register(this); // thêm vào _fields Set
}

// FormState — giữ danh sách tất cả fields
class _FormState extends State<Form> {
  final Set<FormFieldState<dynamic>> _fields = {};
  
  bool validate() {
    bool isValid = true;
    for (final field in _fields) {
      isValid = field.validate() && isValid; // gọi validator của từng field
    }
    return isValid;
  }
}
```

**Tại sao dùng `InheritedWidget`:** `FormField` không cần truyền `FormState` qua constructor — nó tìm tự động qua `Form.maybeOf(context)`. Điều này cho phép form fields ở bất kỳ depth nào trong tree.

---

#### Q5 [Middle] — "Custom `FormField<T>` cho non-text input (e.g., date picker, rating)?"

**Trả lời chuẩn:**

`FormField<T>` cho phép integrate bất kỳ input widget nào với Form validation system:

```dart
// Custom FormField cho date picker
class DatePickerFormField extends FormField<DateTime> {
  DatePickerFormField({
    super.key,
    DateTime? initialValue,
    FormFieldValidator<DateTime>? validator,
    FormFieldSetter<DateTime>? onSaved,
  }) : super(
    initialValue: initialValue,
    validator: validator,
    onSaved: onSaved,
    builder: (FormFieldState<DateTime> state) {
      return Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          InkWell(
            onTap: () async {
              final picked = await showDatePicker(
                context: state.context,
                initialDate: state.value ?? DateTime.now(),
                firstDate: DateTime(2000),
                lastDate: DateTime(2030),
              );
              if (picked != null) state.didChange(picked);
            },
            child: InputDecorator(
              decoration: InputDecoration(errorText: state.errorText),
              child: Text(state.value?.toString() ?? 'Select date'),
            ),
          ),
        ],
      );
    },
  );
}

// Dùng trong Form
DatePickerFormField(
  validator: (date) => date == null ? 'Please select date' : null,
  onSaved: (date) => _selectedDate = date,
)
```

---

#### Q6 [Middle] — "`FormField.validator` callback: khi nào được gọi? Return null vs non-null?"

**Trả lời chuẩn:**

`validator` được gọi khi:
- `FormState.validate()` được gọi thủ công
- `AutovalidateMode.onUserInteraction` → sau khi user bắt đầu interact
- `AutovalidateMode.always` → mỗi rebuild

**Return value:**
- `null` → **valid** — không hiện error
- `String message` → **invalid** — hiện error message bên dưới field

```dart
TextFormField(
  validator: (value) {
    if (value == null || value.isEmpty) return 'Field cannot be empty';
    if (value.length < 3) return 'Minimum 3 characters';
    if (!RegExp(r'^[a-zA-Z]+$').hasMatch(value)) return 'Letters only';
    return null; // ← valid!
  },
)

// Cross-field validation (e.g., confirm password)
TextFormField(
  validator: (value) {
    if (value != _passwordController.text) {
      return 'Passwords do not match';
    }
    return null;
  },
)
```

**Lưu ý:** `validate()` **luôn gọi tất cả validators** (không short-circuit) để tất cả error messages hiện cùng lúc.

---

#### Q7 [Trace Code] — "`formKey.currentState?.validate()` trả về true/false — điều kiện nào?"

```dart
final _formKey = GlobalKey<FormState>();
String? _name;
String? _email;

Form(
  key: _formKey,
  child: Column(children: [
    TextFormField(
      validator: (v) {
        if (v == null || v.isEmpty) return 'Required';
        return null;
      },
      onSaved: (v) => _name = v,
    ),
    TextFormField(
      validator: (v) {
        if (v == null || v.isEmpty) return 'Required';
        if (!v.contains('@')) return 'Invalid email';
        return null;
      },
      onSaved: (v) => _email = v,
    ),
    ElevatedButton(
      onPressed: () {
        final isValid = _formKey.currentState?.validate() ?? false;
        print('Form valid: $isValid');
        if (isValid) {
          _formKey.currentState?.save();
          print('Name: $_name, Email: $_email');
        }
      },
      child: const Text('Submit'),
    ),
  ]),
)
```

**Scenario A (form rỗng):**
```
Form valid: false
// name: 'Required' error, email: 'Required' error
// validate() gọi cả 2 validators (không short-circuit)
```

**Scenario B (Name='John', Email='notanemail'):**
```
Form valid: false
// name: null (valid), email: 'Invalid email' (invalid)
// validate() = false
```

**Scenario C (Name='John', Email='john@example.com'):**
```
Form valid: true
Name: John, Email: john@example.com
// Cả 2 validators return null → valid
// validate() = true → save() chạy → onSaved callbacks
```
