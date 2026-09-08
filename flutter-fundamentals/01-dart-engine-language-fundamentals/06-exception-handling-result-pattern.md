# Bài 1.6 — Exception Handling & Result Pattern

## Phần 1 — Khái Niệm & Mục Tiêu Bài Học

### Tại sao bài này quan trọng?

Exception trong Dart là invisible — hàm không khai báo exception nó có thể throw (không như Java với `throws`). Kết quả:

```dart
// Caller không biết hàm này có thể throw gì
Future<User> fetchUser(String id) async {
  // Có thể throw: SocketException, HttpException, FormatException...
  // Caller không nhìn vào signature mà biết được!
}

// Caller thường bỏ qua exception hoặc catch quá rộng
try {
  final user = await fetchUser(id);
} catch (e) {
  print('Có lỗi: $e'); // Catch tất cả — không biết cụ thể là gì
}
```

**Result Pattern** là giải pháp: biến lỗi thành *giá trị trả về*, không phải exception vô hình.

### Bạn sẽ hiểu được sau bài này:
- `try/catch/finally` đúng cách trong Dart
- Typed Exception — tạo hierarchy lỗi rõ ràng
- `rethrow` vs `throw` — khi nào dùng cái nào
- `sealed class Result<T, E>` — lỗi là giá trị, không phải exception
- Khi nào dùng Exception, khi nào dùng Result type

---

## Phần 2 — Cơ Chế Hoạt Động (Under the Hood)

### Exception Propagation trong Dart

```mermaid
flowchart TD
    A["fetchUser()"] -->|throw| B["SocketException"]
    B -->|propagate up| C["UserRepository.findById()"]
    C -->|propagate up| D["UserViewModel.loadUser()"]
    D -->|"catch (e)"| E["Handle error"]
    D -->|"không catch"| F["Stack unwinding tiếp tục"]
    F --> G["Flutter Framework catches\nFlutterError.onError"]
    G --> H["Red screen in debug\nBlack screen in release"]

    style H fill:#ff4444,color:#fff
```

### Dart Error vs Exception

```
Error (không nên catch):
├── StateError        — Gọi hàm trong state không hợp lệ
├── AssertionError    — assert() fail
├── TypeError         — Type mismatch
└── ArgumentError     — Argument không hợp lệ

Exception (có thể catch):
├── IOException
│   ├── SocketException    — Network không có
│   └── FileSystemException
├── FormatException        — Parse lỗi
├── TimeoutException       — Timeout
└── Custom exceptions      — App-specific
```

### Result Pattern Flow

```
Thay vì exception:
   Repository ──[throw]──► ViewModel ──[crash]──► UI (red screen)

Dùng Result type:
   Repository ──[Result.failure]──► ViewModel ──[handle]──► UI (error state)
```

---

### Bản Chất Kỹ Thuật (Dart Compiler Level)

#### Tại Sao Dart Không Có Checked Exceptions (Design Decision)

Java buộc bạn phải khai báo exception trong signature (`throws IOException`) và caller phải handle — gọi là **checked exceptions**. Dart (và Kotlin) chọn **unchecked exceptions**:

```java
// Java — checked exception: compiler enforce
public User fetchUser(String id) throws IOException, SQLException {
  // Caller BẮT BUỘC phải catch hoặc khai báo throws tiếp
}
```

```dart
// Dart — unchecked: không có throws declaration
Future<User> fetchUser(String id) async {
  // Có thể throw SocketException, FormatException... caller không biết từ signature
}
```

**Lý do Dart (và Kotlin, C#) từ bỏ checked exceptions:**
1. **API fragility**: thêm exception mới vào deep layer → buộc thay đổi signature tất cả layers phía trên
2. **Verbose boilerplate**: `catch (e) { throw e; }` để "re-wrap" exception phổ biến đến mức vô nghĩa
3. **Bypass dễ dàng**: `catch (Exception e) {}` — developer hay viết catch-all để cho xong, mất hẳn ý nghĩa
4. **Kotlin/Dart solution**: dùng **typed exception hierarchy** + **Result type** để explicit về lỗi mà không cần checked mechanism

---

#### Stack Unwinding — Cơ Chế Lan Truyền Exception

Khi `throw` xảy ra, Dart VM thực hiện **stack unwinding**: tua ngược call stack, tìm `catch` handler phù hợp:

```
Call Stack khi exception xảy ra trong fetchUser():

Frame 4: jsonDecode()           ← throw FormatException tại đây
Frame 3: UserRepository.findById()
Frame 2: UserViewModel.loadUser()  ← catch (FormatException) → MATCH! dừng unwind
Frame 1: Widget.onTap()
Frame 0: Flutter Framework

Stack Unwinding:
  jsonDecode() → pop frame 4
  findById()   → không có catch FormatException → pop frame 3
  loadUser()   → CÓ catch FormatException → HANDLE ở đây
```

```dart
// Dart VM runtime behavior:
try {
  final data = jsonDecode(badJson);        // throw FormatException
} on SocketException catch (e) {           // ← type check: FormatException is SocketException? NO
  handle(e);
} on FormatException catch (e, stackTrace) { // ← type check: FormatException is FormatException? YES
  log(e, stackTrace);                      // ← MATCH — stop unwinding, execute this block
} finally {
  cleanup();  // ← LUÔN chạy bất kể exception hay không, TRƯỚC KHI throw tiếp tục
}
```

**`finally` block chạy trong mọi trường hợp:**
- Không có exception → chạy sau `try` block
- Exception được catch → chạy sau `catch` block
- Exception không được catch → chạy rồi tiếp tục unwind lên frame trên

---

#### `rethrow` vs `throw e` — Stacktrace Preservation

```dart
// ❌ throw e — tạo stacktrace MỚI từ điểm này
} catch (e) {
  logError(e);
  throw e;  // Stacktrace trỏ về dòng này, mất context gốc
}

// ✅ rethrow — giữ NGUYÊN stacktrace gốc
} catch (e) {
  logError(e);
  rethrow;  // Stacktrace vẫn trỏ về nơi exception xảy ra ban đầu
}
```

```
// Dart VM — sự khác biệt trong error report:

// throw e:
FormatException: ...
  at UserRepository.findById (repo.dart:42)  ← trỏ vào catch block, mất gốc

// rethrow:
FormatException: ...
  at jsonDecode (convert.dart:301)           ← trỏ đúng nơi xảy ra
  at UserRepository.findById (repo.dart:38)
  at UserViewModel.loadUser (vm.dart:25)
```

**Rule**: Trong `catch` block, dùng `rethrow` nếu chỉ muốn log rồi re-throw. Chỉ dùng `throw e` hoặc `throw NewException(e)` khi muốn **wrap** exception thành type mới.

---

## Phần 3 — Code Mẫu Chuẩn Google

### 3.1 — Exception Hierarchy tốt

```dart
// Base exception cho domain của app
sealed class AppException implements Exception {
  const AppException();
  String get userMessage; // Hiển thị cho user
  String get technicalMessage; // Log cho developer
}

// Network-related exceptions
final class NetworkException extends AppException {
  final int? statusCode;
  final String endpoint;

  const NetworkException({
    required this.endpoint,
    this.statusCode,
  });

  @override
  String get userMessage => statusCode == null
      ? 'Không có kết nối mạng. Kiểm tra WiFi và thử lại.'
      : 'Lỗi server (${statusCode}). Thử lại sau.';

  @override
  String get technicalMessage =>
      'NetworkException: $endpoint returned $statusCode';
}

// Parsing exceptions
final class ParseException extends AppException {
  final String fieldName;
  final dynamic receivedValue;

  const ParseException({
    required this.fieldName,
    required this.receivedValue,
  });

  @override
  String get userMessage => 'Dữ liệu không hợp lệ. Thử lại.';

  @override
  String get technicalMessage =>
      'ParseException: Cannot parse field "$fieldName" from $receivedValue';
}

// Auth exceptions
final class AuthException extends AppException {
  final AuthErrorCode code;
  const AuthException(this.code);

  @override
  String get userMessage => switch (code) {
    AuthErrorCode.invalidCredentials => 'Email hoặc mật khẩu không đúng.',
    AuthErrorCode.sessionExpired => 'Phiên đăng nhập hết hạn.',
    AuthErrorCode.unauthorized => 'Bạn không có quyền truy cập.',
  };

  @override
  String get technicalMessage => 'AuthException: $code';
}

enum AuthErrorCode { invalidCredentials, sessionExpired, unauthorized }
```

### 3.2 — `try/catch/finally` đúng cách

```dart
class UserRepository {
  Future<User> findById(String id) async {
    try {
      final response = await _httpClient.get('/api/users/$id');

      // Check HTTP status trước khi parse
      if (response.statusCode == 401) {
        throw const AuthException(AuthErrorCode.unauthorized);
      }
      if (response.statusCode != 200) {
        throw NetworkException(
          endpoint: '/api/users/$id',
          statusCode: response.statusCode,
        );
      }

      // Parse — có thể throw FormatException
      return User.fromJson(jsonDecode(response.body));

    } on SocketException {
      // Network không có — wrap thành app-specific exception
      throw const NetworkException(endpoint: '/api/users/$id');

    } on FormatException catch (e) {
      // JSON parse lỗi — wrap với context
      throw ParseException(fieldName: 'user_response', receivedValue: e.source);

    } on AppException {
      // App exception đã type-safe — rethrow không modify
      rethrow;

    } catch (e, stackTrace) {
      // Unexpected error — log và wrap
      _logger.error('Unexpected error in findById', e, stackTrace);
      throw NetworkException(endpoint: '/api/users/$id');

    } finally {
      // Luôn chạy — dùng cho cleanup
      _metrics.recordApiCall('/api/users/$id');
    }
  }
}
```

### 3.3 — Result Pattern — Lỗi là giá trị

```dart
// Result type (không dùng package nào)
sealed class Result<T, E extends Object> {
  const Result();

  bool get isOk => this is Ok<T, E>;
  bool get isErr => this is Err<T, E>;

  T get value => (this as Ok<T, E>).value;
  E get error => (this as Err<T, E>).error;

  // Map: transform success value
  Result<R, E> map<R>(R Function(T) transform) => switch (this) {
    Ok(:final value) => Ok(transform(value)),
    Err(:final error) => Err(error),
  };

  // flatMap: chain operations that might fail
  Result<R, E> flatMap<R>(Result<R, E> Function(T) transform) => switch (this) {
    Ok(:final value) => transform(value),
    Err(:final error) => Err(error),
  };

  // Fold: xử lý cả hai case
  R fold<R>({
    required R Function(T value) onOk,
    required R Function(E error) onErr,
  }) => switch (this) {
    Ok(:final value) => onOk(value),
    Err(:final error) => onErr(error),
  };

  // getOrElse: giá trị hoặc fallback
  T getOrElse(T Function(E error) fallback) => switch (this) {
    Ok(:final value) => value,
    Err(:final error) => fallback(error),
  };
}

final class Ok<T, E extends Object> extends Result<T, E> {
  final T value;
  const Ok(this.value);
}

final class Err<T, E extends Object> extends Result<T, E> {
  final E error;
  const Err(this.error);
}
```

### 3.4 — Repository với Result type

```dart
// Repository trả về Result thay vì throw
class UserRepository {
  Future<Result<User, AppException>> findById(String id) async {
    try {
      final response = await _httpClient.get('/api/users/$id');
      if (response.statusCode != 200) {
        return Err(NetworkException(
          endpoint: '/api/users/$id',
          statusCode: response.statusCode,
        ));
      }
      final user = User.fromJson(jsonDecode(response.body));
      return Ok(user);
    } on SocketException {
      return const Err(NetworkException(endpoint: '/api/users'));
    } on FormatException catch (e) {
      return Err(ParseException(fieldName: 'user', receivedValue: e.source));
    }
  }
}

// ViewModel dùng Result
class UserViewModel {
  Future<void> loadUser(String id) async {
    _state = const Loading();
    notifyListeners();

    final result = await _repository.findById(id);

    // Pattern matching exhaustive — compiler check
    _state = result.fold(
      onOk: (user) => Success(user),
      onErr: (error) => Failure(message: error.userMessage),
    );

    notifyListeners();
  }
}

// Widget — không cần biết gì về exception
Widget buildError(AppException error) {
  return Column(
    children: [
      Text(error.userMessage),
      // Log technical message cho developer
      // (trong debug mode)
      if (kDebugMode) Text(error.technicalMessage, style: TextStyle(color: Colors.grey)),
    ],
  );
}
```

### 3.5 — Refactor từ Exception sang Result

```dart
// TRƯỚC: Exception-based — caller không biết phải catch gì
Future<Map<String, dynamic>> parseJson(String jsonString) async {
  // Có thể throw FormatException — không rõ ràng từ signature
  return jsonDecode(jsonString) as Map<String, dynamic>;
}

// SAU: Result-based — signature tường minh về khả năng lỗi
Result<Map<String, dynamic>, String> parseJsonSafe(String jsonString) {
  try {
    final decoded = jsonDecode(jsonString);
    if (decoded is! Map<String, dynamic>) {
      return const Err('JSON không phải object');
    }
    return Ok(decoded);
  } on FormatException catch (e) {
    return Err('JSON không hợp lệ: ${e.message}');
  }
}

// Cách dùng — clear và explicit
void processConfig(String rawJson) {
  final result = parseJsonSafe(rawJson);
  switch (result) {
    case Ok(:final value):
      _applyConfig(value);
    case Err(:final error):
      _logger.warning('Config lỗi: $error');
      _applyDefaultConfig();
  }
}
```

---

## Phần 4 — Lỗi Sai Phổ Biến & Best Practices

### ❌ Anti-pattern 1: Catch quá rộng, bỏ qua exception

```dart
// ❌ Sai: Catch mọi thứ và bỏ qua
try {
  final data = await fetchData();
  process(data);
} catch (_) {
  // Silent fail — developer không biết có lỗi!
}

// ✅ Đúng: Log và handle cụ thể
try {
  final data = await fetchData();
  process(data);
} on NetworkException catch (e) {
  _logger.warning('Network error', e);
  showRetryDialog();
} on ParseException catch (e) {
  _logger.error('Parse error — check API contract', e);
  showErrorAndReport(e);
} catch (e, stackTrace) {
  // Unexpected — log đầy đủ để debug
  _logger.error('Unexpected error', e, stackTrace);
  showGenericError();
}
```

### ❌ Anti-pattern 2: Throw Error thay vì Exception

```dart
// ❌ Sai: Throw Error (không nên catch ở application level)
Future<User> getUser(String? id) async {
  if (id == null) throw ArgumentError('ID không được null'); // Là Error, không phải Exception
}

// ✅ Đúng: Dùng Exception cho recoverable errors
Future<Result<User, String>> getUser(String? id) async {
  if (id == null) return const Err('ID không được để trống');
  // hoặc validate bằng assert nếu là programming error:
  assert(id.isNotEmpty, 'ID không được rỗng'); // Chỉ trong debug
}
```

### ❌ Anti-pattern 3: Dùng Exception cho flow control

```dart
// ❌ Sai: Dùng exception để control flow bình thường
bool isUserLoggedIn() {
  try {
    getCurrentUser(); // Throw nếu chưa đăng nhập
    return true;
  } catch (_) {
    return false;
  }
}
// Exception rất expensive (tạo stacktrace) — không dùng cho control flow

// ✅ Đúng: Return nullable hoặc Result
User? getCurrentUser() { ... } // Trả null nếu chưa login

bool isUserLoggedIn() => getCurrentUser() != null;
```

### ❌ Anti-pattern 4: `rethrow` vs `throw e`

```dart
void processData() {
  try {
    _parse();
  } catch (e, stackTrace) {
    logError(e);

    // ❌ Sai: throw e — mất original stacktrace!
    throw e;

    // ✅ Đúng: rethrow — giữ nguyên stacktrace gốc
    rethrow;
  }
}
```

---

## Phần 5 — Bài Tập Củng Cố Tư Duy

### Challenge: Refactor JSON Parsing sang Result Type

**Code hiện tại (exception-based):**
```dart
class ApiClient {
  Future<List<Product>> getProducts() async {
    final response = await http.get(Uri.parse('/api/products'));
    // Throws trên mọi lỗi — caller không rõ phải handle gì
    final json = jsonDecode(response.body) as List;
    return json.map((e) => Product.fromJson(e)).toList();
  }
}
```

**Nhiệm vụ:**
1. Tạo sealed class `ApiError` với các variant: `NetworkError`, `ParseError`, `ServerError(int code)`
2. Refactor `getProducts()` trả về `Future<Result<List<Product>, ApiError>>`
3. Viết ViewModel sử dụng Result này
4. Trong Widget: dùng `result.fold()` để render UI

**Gợi ý hướng giải:**
- Catch từng loại exception cụ thể, map sang `ApiError` tương ứng
- `SocketException` → `ApiError.networkError`
- `FormatException` → `ApiError.parseError`
- Status code >= 400 → `ApiError.serverError(statusCode)`

### Thử Thách Tư Duy & Thẩm Định Chuyên Sâu (Conceptual & Deep-Dive Check)

> **[Junior]** — nắm khái niệm | **[Middle]** — hiểu cơ chế | **[Senior]** — hiểu compiler/VM level | **[Trace Code]** — đọc code và dự đoán output

---

#### Q1 [Junior] — "Khi nào dùng Exception (throw/catch), khi nào dùng Result type? Tiêu chí phân biệt?"

**Trả lời chuẩn:**

**Dùng Exception (throw/catch) khi:**
- Lỗi thực sự **không mong đợi** — điều kiện ngoài tầm kiểm soát của caller
- **Programming errors** — vi phạm contract (assert, ArgumentError)
- Lỗi ở tầng rất thấp mà caller không thể reasonable handle

**Dùng Result type khi:**
- Lỗi **có thể xảy ra trong flow bình thường** — caller phải handle như một trường hợp bình thường
- **Network failure, validation fail, not found** — những thứ caller có thể và nên react
- Muốn signature của hàm **tường minh về khả năng lỗi** — caller nhìn vào `Future<Result<User, AppError>>` biết ngay phải handle error

```dart
// Exception: caller không mong đợi, khó recover
void connect(String url) {
  if (url.isEmpty) throw ArgumentError('URL cannot be empty'); // programming error
}

// Result: caller biết mạng có thể fail, phải handle
Future<Result<User, NetworkError>> fetchUser(String id) async {
  try { ... }
  on SocketException { return const Err(NetworkError.noConnection); }
}
```

**Rule of thumb:** Nếu caller hợp lý có thể *hỏi "nếu thất bại thì sao?"* → dùng Result. Nếu thất bại là bug hoặc catastrophic → dùng Exception.

---

#### Q2 [Junior] — "Sự khác biệt giữa `Error` và `Exception` trong Dart? Tại sao không nên catch `Error`?"

**Trả lời chuẩn:**

Dart phân chia rõ ràng thành 2 class gốc:

**`Error`** — programming bugs, **không nên catch ở application level:**
- `AssertionError` — `assert()` fail
- `TypeError` — type mismatch (wrong type passed)
- `StateError` — gọi method trong state không hợp lệ
- `RangeError` — index out of bounds
- `LateInitializationError` — `late` field chưa được gán

Những lỗi này chỉ xảy ra khi code sai — catch chúng sẽ che giấu bug thay vì fix. Dart convention: để chúng crash, đọc stack trace, fix code.

**`Exception`** — runtime conditions có thể recover, **nên catch và handle:**
- `IOException` / `SocketException` — network issues
- `FormatException` — JSON/format parse fail
- `TimeoutException` — request quá lâu
- Custom app exceptions — domain-specific errors

```dart
// ✅ Đúng: catch Exception, không catch Error
try {
  final data = await fetchData();
} on NetworkException catch (e) { // ← Exception (recoverable)
  showRetryDialog();
} catch (e, stackTrace) { // ← unknown — log, report
  FirebaseCrashlytics.instance.recordError(e, stackTrace);
}
// Không catch TypeError, AssertionError, v.v. — để crash để fix bug
```

---

#### Q3 [Middle] — "`rethrow` vs `throw e` vs `throw NewException(e)` — khi nào dùng cái nào?"

**Trả lời chuẩn:**

| | `rethrow` | `throw e` | `throw NewException(e)` |
|---|---|---|---|
| Stack trace | **Giữ nguyên** gốc | **Mất** — tạo mới từ đây | Mới, nhưng có thể set `cause` |
| Use case | Log rồi propagate | **Tránh dùng** | Wrap exception thành type mới |

```dart
// ✅ rethrow: log rồi để exception tiếp tục với stack trace gốc
} catch (e, stackTrace) {
  _logger.error('Unexpected error', e, stackTrace);
  rethrow; // Stack trace trỏ về nơi exception thực sự xảy ra

// ❌ throw e: mất stack trace gốc — debug khó hơn
} catch (e) {
  throw e; // Stack trace bắt đầu từ ĐÂY — mất context gốc

// ✅ throw NewException(e): wrap sang type khác khi cần
} on SocketException catch (e) {
  throw NetworkException(
    endpoint: url,
    cause: e,           // giữ original exception làm cause
  );
}
```

**Minh họa sự khác biệt trong error report:**
```
// rethrow → stacktrace đầy đủ:
FormatException: ...
  at jsonDecode (convert.dart:301)       ← nơi thực sự xảy ra
  at UserRepository.findById (repo.dart:38)
  at UserViewModel.loadUser (vm.dart:25)

// throw e → stacktrace bị cắt:
FormatException: ...
  at UserRepository.findById (repo.dart:45)  ← chỉ thấy catch block, mất gốc
```

---

#### Q4 [Senior] — "Tại sao Dart không có checked exceptions như Java? Kotlin và C# cũng bỏ — lý do chung là gì?"

**Trả lời chuẩn:**

**Checked exceptions trong Java** buộc developer khai báo `throws IOException` trong signature và caller phải `try/catch` hoặc khai báo `throws` tiếp. Nghe có vẻ tốt nhưng thực tế gây nhiều vấn đề:

**1. API fragility:** Thêm exception mới vào deep layer → phải thay đổi signature *tất cả* layers phía trên → breaking change lan rộng:
```java
// Java: thêm NetworkException vào Repository → sửa tất cả layers
interface UserRepository { User findById(String id) throws IOException; }
// → Service phải thêm throws IOException
// → Controller phải thêm throws IOException hoặc catch
// → N layers phải sửa
```

**2. Verbose boilerplate:** Developer thường viết catch-all để "cho xong" — phá vỡ toàn bộ ý nghĩa:
```java
try {
  result = riskyOperation();
} catch (Exception e) { } // silence mọi thứ — tệ hơn không có gì
```

**3. Dễ bypass:** Wrap trong `RuntimeException` để thoát checked mechanism — anti-pattern phổ biến trong Java code.

**Giải pháp Dart/Kotlin/C#:** Unchecked exceptions + **typed exception hierarchy** (developer tự tổ chức) + **Result type** (explicit error handling trong signature). Đây là explicit-by-design, không phải laissez-faire.

---

#### Q5 [Senior] — "Stack unwinding hoạt động thế nào? `finally` block chạy khi nào, theo thứ tự nào?"

**Trả lời chuẩn:**

Khi `throw` xảy ra, Dart VM thực hiện **stack unwinding**: pop từng stack frame, tìm `catch` handler phù hợp:

```
Call Stack tại thời điểm throw FormatException:

Frame 4: jsonDecode()         ← throw FormatException tại đây
Frame 3: UserRepository.findById()
Frame 2: UserViewModel.loadUser() ← catch (on FormatException) → MATCH!
Frame 1: Widget.onTap()
Frame 0: Flutter Framework

Stack Unwinding:
  Pop frame 4 (jsonDecode)   → không có catch → tiếp tục unwind
  Pop frame 3 (findById)     → có try/catch nhưng không match FormatException → unwind
  Pop frame 2 (loadUser)     → có catch (FormatException) → MATCH → dừng unwind, execute catch
```

**`finally` block chạy theo quy tắc:**
1. Không có exception → sau `try` block
2. Exception được catch → sau `catch` block *trước khi* exit try/catch
3. Exception **không được catch** → `finally` chạy rồi tiếp tục unwind lên frame trên

```dart
void demo() {
  try {
    throw Exception('error');
  } catch (e) {
    print('catch');
    throw e;          // re-throw
  } finally {
    print('finally'); // chạy TRƯỚC KHI re-throw propagate
  }
}
// Output: catch, finally → rồi exception propagate lên caller
```

**`finally` luôn là nơi cleanup an toàn:** close file, cancel timer, release lock — đảm bảo chạy bất kể exception hay không.

---

#### Q6 [Middle] — "`on SocketException catch (e)` — đây là compile-time hay runtime type check? Khác `is` ở điểm nào?"

**Trả lời chuẩn:**

`on X catch (e)` là **runtime type check** — Dart VM kiểm tra type của exception object tại runtime để tìm handler phù hợp. Đây không phải compile-time proof.

```dart
try {
  riskyOperation();
} on SocketException catch (e) {     // runtime: e.runtimeType is SocketException?
  handleNetworkError(e);
} on FormatException catch (e) {     // runtime: e.runtimeType is FormatException?
  handleParseError(e);
} catch (e) {                        // catch-all: match mọi thứ
  handleUnknown(e);
}
```

**Dart VM check theo thứ tự từ trên xuống** — trường hợp đầu tiên match → execute, bỏ qua các case còn lại.

**Khác `is` operator:**
- `on X catch (e)`: dùng trong `try/catch`, check type của *exception đang được propagate*
- `x is T`: dùng ở bất kỳ đâu trong code, check type của bất kỳ object nào

```dart
// on: trong try/catch, check exception type
try { ... } on SocketException catch (e) { ... }

// is: bất kỳ đâu, kết hợp với type promotion
if (error is AppException) {
  print(error.userMessage); // type promoted to AppException
}
```

**Lưu ý quan trọng:** `catch (e)` không có `on` thì catch *mọi thứ* — kể cả `Error`. Tốt nhất là luôn specify type: `on Exception catch (e)` để tránh catch `Error`.

---

#### Q7 [Trace Code] — "Xác định thứ tự output và exception nào propagate ra ngoài:"

```dart
Future<void> demo() async {
  try {
    print('try');
    throw Exception('original');
  } catch (e) {
    print('catch: $e');
    throw Exception('from catch'); // throw mới trong catch block
  } finally {
    print('finally');
    // Không throw ở đây
  }
}

// Caller:
try {
  await demo();
} catch (e) {
  print('outer catch: $e');
}
```

**Đáp án:**
```
try
catch: Exception: original
finally
outer catch: Exception: from catch
```

**Giải thích từng bước:**
1. `print('try')` → in `try`
2. `throw Exception('original')` → vào catch block
3. `print('catch: $e')` → in `catch: Exception: original`
4. `throw Exception('from catch')` — re-throw mới nhưng `finally` PHẢI chạy trước
5. `print('finally')` → in `finally`
6. Exception `'from catch'` tiếp tục propagate → outer caller catch → in `outer catch: Exception: from catch`

**Key insight:** `finally` luôn chạy trước exception propagate ra ngoài — kể cả khi `catch` block throw exception mới. Exception trong `finally` sẽ *override* exception từ `catch` (đây là anti-pattern cần tránh trong `finally` block).
