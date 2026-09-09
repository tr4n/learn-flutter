# Bài 7.3 — Chuyển Đổi Dữ Liệu & Mô Hình Hóa JSON Trong Dart/Flutter

## Tài Liệu Tham Khảo Chính Thức
- [Flutter Documentation: JSON and serialization](https://docs.flutter.dev/data-and-backend/serialization/json)
- [Dart Documentation: Decoding and encoding JSON (dart:convert)](https://dart.dev/guides/libraries/library-tour#dartconvert---decoding-and-encoding-json-utf-8-and-more)
- [Dart package:json_serializable Documentation](https://pub.dev/packages/json_serializable)
- [Dart package:freezed Documentation](https://pub.dev/packages/freezed)

---

## Phần 1 — Khái Niệm & Các Phương Pháp Tiếp Cận (Architecture & Approaches)

### 1.1 — Luồng Chuyển Đổi Dữ Liệu JSON Trong Flutter

JSON (JavaScript Object Notation) là định dạng dữ liệu chuẩn trong giao tiếp RESTful API. Tuy nhiên, JSON là định dạng phi cấu trúc kiểu tĩnh (dynamically typed text). Để đảm bảo tính toàn vẹn dữ liệu trong hệ thống kiểu tĩnh của Dart, dữ liệu nhận về cần trải qua quy trình giải mã và ánh xạ:

```
┌────────────────────────────────────────────────────────────────────────┐
│ LUỒNG XỬ LÝ DỮ LIỆU JSON                                              │
│                                                                        │
│ 1. API Response Body (String)                                          │
│    '{"id": "usr_01", "name": "Flutter Developer", "price": 99.0}'     │
│      │                                                                 │
│      ▼ (dart:convert: jsonDecode)                                      │
│ 2. Dynamic Object Structure                                            │
│    Map<String, dynamic>                                                │
│      │                                                                 │
│      ▼ (Model.fromJson factory)                                        │
│ 3. Type-Safe Dart Domain Model                                         │
│    User(id: 'usr_01', name: 'Flutter Developer', price: 99.0)          │
│      │                                                                 │
│      ▼                                                                 │
│ 4. Sử dụng an toàn tại UI & Business Logic                             │
│    Text(user.name) // Trình biên dịch kiểm tra kiểu tĩnh lúc compile   │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 1.2 — So Sánh 3 Phương Pháp Mô Hình Hóa JSON

Theo tài liệu chính thức của Flutter, có 3 phương pháp chính để ánh xạ JSON sang Dart Model:

#### 1. Thủ công (Manual Serialization bằng `dart:convert`)
- **Cách thức**: Tự định nghĩa constructor `factory Model.fromJson(Map<String, dynamic> json)` và phương thức `Map<String, dynamic> toJson()`.
- **Ưu điểm**: Không phụ thuộc vào thư viện bên ngoài, không cần chạy công cụ sinh mã (`build_runner`), tốc độ build nhanh.
- **Hạn chế**: Tốn nhiều dòng code lặp đi lặp lại (boilerplate), dễ xảy ra lỗi chính tả khi truy xuất chuỗi khóa (key strings), bảo trì phức tạp khi cấu trúc JSON thay đổi.
- **Phù hợp với**: Các ứng dụng có quy mô nhỏ hoặc số lượng model hạn chế (dưới 5-10 models).

#### 2. Tự động hóa bằng `package:json_serializable`
- **Cách thức**: Đánh dấu các lớp dữ liệu với annotation `@JsonSerializable()`. Công cụ `build_runner` sẽ phân tích cú pháp tĩnh và tự động sinh ra file `*.g.dart` chứa logic `_$ModelFromJson` và `_$ModelToJson`.
- **Ưu điểm**: Hạn chế sai sót chính tả, hỗ trợ cấu hình ánh xạ tên trường (`@JsonKey(name: '...')`), hỗ trợ chuyển đổi tùy biến (`JsonConverter`).
- **Hạn chế**: Vẫn cần tự viết phương thức `copyWith`, toán tử so sánh `operator ==` và `hashCode` nếu cần so sánh giá trị đối tượng.
- **Phù hợp với**: Ứng dụng quy mô trung bình đến lớn, các dự án có cấu trúc dữ liệu nhiều trường phức tạp.

#### 3. Mô hình hóa lớp dữ liệu bất biến bằng `package:freezed`
- **Cách thức**: Sử dụng bộ sinh mã chuyên biệt cho Data Class, kết hợp `freezed_annotation` và `json_serializable`.
- **Ưu điểm**: Tự động sinh toàn bộ: `fromJson`/`toJson`, `copyWith`, toán tử so sánh sâu (Deep Equality), `toString()`, và hỗ trợ Union Types / Sealed Classes kết hợp với Pattern Matching.
- **Hạn chế**: Phụ thuộc vào công cụ sinh mã của bên thứ ba, thời gian chạy `build_runner` ban đầu lâu hơn.
- **Phù hợp với**: Các ứng dụng lớn, sử dụng kiến trúc phân tầng, quản lý trạng thái bằng BLoC, Riverpod hoặc State Pattern.

#### Bảng so sánh tính năng kỹ thuật:

| Tiêu Chí | Thủ Công (`dart:convert`) | `json_serializable` | `freezed` |
| :--- | :--- | :--- | :--- |
| **Yêu cầu build_runner** | Không | Có | Có |
| **Tự động sinh fromJson/toJson** | Không (tự viết tay) | Có (`.g.dart`) | Có (kết hợp `json_serializable`) |
| **Tự động sinh copyWith** | Không | Không | Có |
| **So sánh giá trị (Deep Equality)** | Không (mặc định tham chiếu) | Không | Có (so sánh toàn bộ các trường) |
| **Hỗ trợ Union Types / Sealed Class** | Không | Không | Có |
| **Tính bất biến (Immutability)** | Cần tự đặt `final` | Cần tự đặt `final` | Tự động tạo cấu trúc immutable |

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Ánh Xạ Kiểu Dữ Liệu Giữa JSON và Dart VM

Thư viện `dart:convert` ánh xạ các kiểu nguyên thủy của JSON sang Dart theo quy tắc:

```
JSON Value Type       ──► Dart VM Runtime Type
───────────────────────────────────────────────
null                  ──► Null
true / false          ──► bool
Số nguyên (vd: 42)    ──► int
Số thực (vd: 3.14)    ──► double
Chuỗi ký tự ("...")   ──► String
Mảng danh sách ([...])──► List<dynamic>
Đối tượng ({...})     ──► Map<String, dynamic>
```

> **Quy tắc quan trọng**: Trong chuỗi JSON, không có định dạng phân biệt giữa `int` và `double` mà chỉ có kiểu số chung (`number`). Khi máy chủ trả về `99`, Dart sẽ giải mã thành kiểu `int`. Nếu trường dữ liệu trong Dart được khai báo là `double` và lập trình viên ép kiểu trực tiếp `json['price'] as double`, Dart VM sẽ ném ra ngoại lệ `TypeError` tại runtime.

---

### 2.2 — Cơ Chế Hoạt Động Của `build_runner` Tại Build-Time

Tại sao Flutter chọn phương pháp sinh mã nguồn tĩnh (Source Code Generation) thay vì sử dụng Reflection (`dart:mirrors`) như trong Java hoặc C#?

```
┌────────────────────────────────────────────────────────────────────────┐
│ CƠ CHẾ SINH MÃ TĨNH BUILD-TIME CỦA BUILD_RUNNER                        │
│                                                                        │
│ 1. Lập trình viên viết mã nguồn: user.dart (@JsonSerializable)         │
│      │                                                                 │
│      ▼ (Chạy lệnh: dart run build_runner build)                         │
│ 2. Analyzer phân tích cú pháp tĩnh (AST - Abstract Syntax Tree)        │
│      │                                                                 │
│      ▼                                                                 │
│ 3. Generator tạo mã Dart thuần: user.g.dart (_$UserFromJson)           │
│      │                                                                 │
│      ▼                                                                 │
│ 4. Flutter AOT Compiler (Ahead-Of-Time) biên dịch ra mã máy            │
│    • Toàn bộ logic đã có sẵn dưới dạng mã nguồn Dart tĩnh              │
│    • Không cần kiểm tra kiểu động tại Runtime                          │
│    • Trình biên dịch tối ưu hóa kích thước (Tree-shaking hiệu quả)     │
└────────────────────────────────────────────────────────────────────────┘
```

- **Lý do loại bỏ Reflection (`dart:mirrors`)**: 
  1. **Tối ưu hóa kích thước nhị phân (Binary Size & Tree-shaking)**: Trình biên dịch AOT của Dart cần biết chính xác những phương thức và trường nào được gọi để loại bỏ các đoạn mã dư thừa. Reflection đòi hỏi giữ lại toàn bộ siêu dữ liệu của chương trình, làm dung lượng ứng dụng tăng đột biến.
  2. **Tốc độ thực thi (Performance)**: Việc đọc metadata qua Reflection tại runtime tiêu tốn CPU và bộ nhớ hơn nhiều so với việc gọi trực tiếp các hàm sinh sẵn tại build-time.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Triển Khai Model Thủ Công (Manual Serialization)

Cấu trúc một model thủ công an toàn, hỗ trợ kiểu số linh hoạt, kiểm tra giá trị null và cấu trúc lồng nhau:

```dart
import 'dart:convert';

class Product {
  final String id;
  final String name;
  final double price;
  final String? imageUrl;
  final List<String> tags;
  final ProductCategory category;
  final DateTime createdAt;

  const Product({
    required this.id,
    required this.name,
    required this.price,
    this.imageUrl,
    required this.tags,
    required this.category,
    required this.createdAt,
  });

  // Factory constructor phân tích từ Map an toàn
  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as String,
      name: json['name'] as String,
      // Ép kiểu qua num trước để tiếp nhận an toàn cả int và double
      price: (json['price'] as num).toDouble(),
      imageUrl: json['image_url'] as String?,
      tags: (json['tags'] as List<dynamic>?)
              ?.map((item) => item as String)
              .toList() ??
          const [],
      category: ProductCategory.fromJson(
        json['category'] as Map<String, dynamic>,
      ),
      createdAt: DateTime.tryParse(json['created_at'] as String? ?? '') ??
          DateTime.now(),
    );
  }

  // Chuyển đổi ngược lại Map để gửi lên API
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
      'price': price,
      if (imageUrl != null) 'image_url': imageUrl,
      'tags': tags,
      'category': category.toJson(),
      'created_at': createdAt.toIso8601String(),
    };
  }

  Product copyWith({
    String? id,
    String? name,
    double? price,
    String? imageUrl,
    List<String>? tags,
    ProductCategory? category,
    DateTime? createdAt,
  }) {
    return Product(
      id: id ?? this.id,
      name: name ?? this.name,
      price: price ?? this.price,
      imageUrl: imageUrl ?? this.imageUrl,
      tags: tags ?? this.tags,
      category: category ?? this.category,
      createdAt: createdAt ?? this.createdAt,
    );
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is Product &&
          runtimeType == other.runtimeType &&
          id == other.id &&
          name == other.name &&
          price == other.price;

  @override
  int get hashCode => Object.hash(id, name, price);
}

class ProductCategory {
  final String id;
  final String title;

  const ProductCategory({required this.id, required this.title});

  factory ProductCategory.fromJson(Map<String, dynamic> json) {
    return ProductCategory(
      id: json['id'] as String,
      title: json['title'] as String,
    );
  }

  Map<String, dynamic> toJson() => {'id': id, 'title': title};
}
```

---

### 3.2 — Triển Khai Với `package:json_serializable`

Cấu hình các annotation tự động sinh mã và xử lý bộ chuyển đổi dữ liệu tùy biến:

#### 1. Khai báo dependencies trong `pubspec.yaml`:
```yaml
dependencies:
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.9
  json_serializable: ^6.8.0
```

#### 2. Định nghĩa Model và Custom Converter:
```dart
import 'package:json_annotation/json_annotation.dart';

part 'user_profile.g.dart';

@JsonSerializable(explicitToJson: true)
class UserProfile {
  final String id;

  @JsonKey(name: 'full_name')
  final String fullName;

  @JsonKey(defaultValue: 'user')
  final String role;

  @JsonKey(name: 'created_at')
  @EpochDateTimeConverter()
  final DateTime createdAt;

  const UserProfile({
    required this.id,
    required this.fullName,
    required this.role,
    required this.createdAt,
  });

  factory UserProfile.fromJson(Map<String, dynamic> json) =>
      _$UserProfileFromJson(json);

  Map<String, dynamic> toJson() => _$UserProfileToJson(this);
}

// Bộ chuyển đổi tùy biến từ Timestamp (milliseconds) sang DateTime
class EpochDateTimeConverter implements JsonConverter<DateTime, int> {
  const EpochDateTimeConverter();

  @override
  DateTime fromJson(int json) => DateTime.fromMillisecondsSinceEpoch(json);

  @override
  int toJson(DateTime object) => object.millisecondsSinceEpoch;
}
```

Lệnh thực thi sinh mã:
```bash
dart run build_runner build --delete-conflicting-outputs
```

---

### 3.3 — Triển Khai Data Class Bất Biến Với `package:freezed`

`freezed` cung cấp giải pháp khai báo model ngắn gọn, tự động tạo `copyWith`, deep equality và hỗ trợ mô hình Union Types (Sealed Classes):

#### 1. Khai báo dependencies trong `pubspec.yaml`:
```yaml
dependencies:
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.9
  freezed: ^2.5.2
  json_serializable: ^6.8.0
```

#### 2. Khai báo Model bất biến:
```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'order.freezed.dart';
part 'order.g.dart';

@freezed
class Order with _$Order {
  const factory Order({
    required String id,
    @JsonKey(name: 'order_number') required String orderNumber,
    required double totalAmount,
    @Default([]) List<String> itemIds,
  }) = _Order;

  factory Order.fromJson(Map<String, dynamic> json) => _$OrderFromJson(json);
}
```

#### 3. Mô hình hóa kết quả mạng dạng Sealed Union:
```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'network_result.freezed.dart';

@freezed
sealed class NetworkResult<T> with _$NetworkResult<T> {
  const factory NetworkResult.success(T data) = Success<T>;
  const factory NetworkResult.failure(String message, int? statusCode) = Failure<T>;
  const factory NetworkResult.loading() = Loading<T>;
}

// Sử dụng với Dart Pattern Matching:
void handleResult(NetworkResult<Order> result) {
  switch (result) {
    case Success(:final data):
      print('Thành công: ${data.orderNumber}');
    case Failure(:final message, :final statusCode):
      print('Thất bại: $message (Mã: $statusCode)');
    case Loading():
      print('Đang xử lý...');
  }
}
```

---

## Phần 4 — Lỗi Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Ép kiểu số học trực tiếp `json['field'] as double`

#### Mô tả vấn đề:
Khai báo trường kiểu `double` và ép kiểu trực tiếp từ `json`:
```dart
// Lỗi tiềm ẩn: Gây crash nếu server trả về số nguyên
double price = json['price'] as double;
```

#### Nguyên nhân kỹ thuật:
Nếu backend trả về `100` thay vì `100.0`, Dart's JSON decoder giải mã giá trị thành kiểu `int`. Khi gọi `as double`, Dart kiểm tra kiểu tại runtime và ném ra ngoại lệ:
`TypeError: type 'int' is not a subtype of type 'double' in type cast`.

#### Biện pháp khắc phục:
Ép kiểu thông qua lớp trừu tượng `num` (là lớp cha của cả `int` và `double`), sau đó gọi `.toDouble()`:
```dart
double price = (json['price'] as num).toDouble();
```

---

### 4.2 — Bỏ qua kiểm tra null khi xử lý cấu trúc lồng nhau

#### Mô tả vấn đề:
Truy xuất trực tiếp các object con hoặc danh sách con mà không kiểm tra giá trị `null`:
```dart
// Gây crash: TypeError: null is not a subtype of type Map<String, dynamic>
Address address = Address.fromJson(json['address'] as Map<String, dynamic>);
```

#### Nguyên nhân kỹ thuật:
Khi người dùng chưa cập nhật địa chỉ hoặc API trả về `address: null`, việc ép kiểu trực tiếp `null` sang `Map<String, dynamic>` sẽ gây lỗi runtime ngay tại thời điểm parse.

#### Biện pháp khắc phục:
Kiểm tra điều kiện `null` an toàn:
```dart
Address? address = json['address'] != null 
    ? Address.fromJson(json['address'] as Map<String, dynamic>) 
    : null;
```

---

### 4.3 — Sử dụng Mutable Model trong State Management

#### Mô tả vấn đề:
Khai báo các thuộc tính trong Model không có từ khóa `final` và thay đổi trực tiếp thuộc tính:
```dart
// Không khuyến nghị: Model có thể bị biến đổi
class User {
  String name;
  User(this.name);
}

user.name = 'New Name'; // Biến đổi tại chỗ
```

#### Nguyên nhân kỹ thuật:
Các công cụ quản lý trạng thái (như `ChangeNotifier`, `Bloc`, `Riverpod`) dựa vào việc so sánh tham chiếu đối tượng cũ và mới (`oldState != newState`). Nếu thay đổi giá trị trực tiếp trên cùng một instance tham chiếu, framework sẽ coi như state không thay đổi và bỏ qua việc kích hoạt vẽ lại giao diện (UI rebuild).

#### Biện pháp khắc phục:
Luôn khai báo các trường là `final`, sử dụng `const` constructor và tạo bản sao mới thông qua phương thức `copyWith`.

---

### 4.4 — Sử dụng `DateTime.parse()` mà không kiểm soát ngoại lệ

#### Mô tả vấn đề:
Gọi trực tiếp `DateTime.parse(json['date'] as String)`.

#### Nguyên nhân kỹ thuật:
Nếu chuỗi ngày tháng trả về không đúng chuẩn ISO 8601 (ví dụ chuỗi rỗng `""` hoặc định dạng sai), `DateTime.parse()` sẽ ném ra ngoại lệ `FormatException` làm gián đoạn toàn bộ quá trình parse dữ liệu của màn hình.

#### Biện pháp khắc phục:
Sử dụng `DateTime.tryParse()` và cung cấp giá trị mặc định nếu cần thiết:
```dart
DateTime createdAt = DateTime.tryParse(json['date'] as String? ?? '') ?? DateTime.now();
```

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Tại sao cần sử dụng `(json['value'] as num).toDouble()` thay vì `as double`?
*Phân tích:*
Trong ngôn ngữ Dart, cả `int` và `double` đều kế thừa từ lớp trừu tượng `num`. Tuy nhiên, `int` không phải là lớp con của `double`. Chuẩn JSON không có sự phân định kiểu nguyên thủy `int` hay `double` mà chỉ có một kiểu số học chung. Khi bộ giải mã `dart:convert` đọc một số không có phần thập phân (ví dụ `100`), nó tự động tạo ra một thể hiện của `int`. Do đó, ép kiểu trực tiếp `as double` sẽ thất bại. Việc ép kiểu sang `num` tương thích với cả hai kiểu số, và phương thức `.toDouble()` sẽ chuyển đổi giá trị nguyên sang số thực một cách an toàn.

---

#### Câu hỏi 2: Tại sao Flutter không sử dụng Reflection (`dart:mirrors`) để parse JSON tại Runtime?
*Phân tích:*
Flutter sử dụng cơ chế biên dịch AOT (Ahead-Of-Time) để chuyển đổi mã nguồn Dart trực tiếp sang mã máy nhị phân (ARM / x86). Để tối ưu hóa hiệu năng khởi động và giảm kích thước tệp cài đặt (APK / IPA), trình biên dịch thực hiện kỹ thuật **Tree-shaking** — loại bỏ toàn bộ các hàm, lớp và trường không được gọi trực tiếp. Cơ chế Reflection đòi hỏi phải lưu trữ toàn bộ bảng ánh xạ kiểu (Type Metadata) tại Runtime, làm vô hiệu hóa khả năng Tree-shaking và tiêu tốn nhiều bộ nhớ RAM khi thiết bị khởi chạy.

---

#### Câu hỏi 3: Lợi thế kiến trúc của Model bất biến (Immutable Data Classes) trong ứng dụng Flutter là gì?
*Phân tích:*
1. **Phát hiện thay đổi hiệu quả (Change Detection)**: Khi cần kiểm tra xem dữ liệu có thay đổi hay không, hệ thống chỉ cần so sánh tham chiếu (`identical(oldModel, newModel)`), đạt độ phức tạp $O(1)$ thay vì phải duyệt sâu qua từng trường dữ liệu.
2. **An toàn đa luồng (Thread-safety)**: Trong ứng dụng xử lý tác vụ nền bằng các `Isolate`, các đối tượng bất biến có thể được truyền hoặc đọc một cách an toàn mà không phát sinh hiện tượng tranh chấp bộ nhớ (Race Condition).
3. **Dự đoán trạng thái (Predictable State)**: Ngăn chặn việc một thành phần giao diện vô tình thay đổi giá trị thuộc tính của model ở nơi khác.

---

#### Câu hỏi 4: Cơ chế hoạt động của `JsonConverter<T, S>` trong thư viện `json_serializable`?
*Phân tích:*
`JsonConverter<T, S>` là một lớp trừu tượng định nghĩa hai phương thức: `T fromJson(S json)` và `S toJson(T object)`, trong đó `T` là kiểu dữ liệu mong muốn trong Dart Model và `S` là kiểu dữ liệu tương thích với JSON (thường là `String`, `int`, `Map`). Trong quá trình sinh mã tĩnh, `build_runner` phát hiện annotation `@JsonConverter` trên một trường dữ liệu và tự động chèn lời gọi hàm `converter.fromJson()` vào mã nguồn được sinh ra tại file `.g.dart`.

---

#### Câu hỏi 5: Sự khác biệt giữa Deep Equality trong `freezed` và Reference Equality mặc định của Dart?
*Phân tích:*
Mặc định trong Dart, toán tử `==` trên các lớp thông thường là **Reference Equality** (so sánh định danh địa chỉ ô nhớ thông qua `identical(a, b)`). Hai đối tượng có cùng các trường dữ liệu nhưng được khởi tạo độc lập sẽ trả về `false`. Thư viện `freezed` ghi đè (override) toán tử `operator ==` và phương thức `hashCode` bằng thuật toán so sánh sâu (**Deep Equality**): Duyệt qua từng trường dữ liệu của đối tượng và so sánh giá trị cụ thể của từng trường (kể cả danh sách các phần tử bên trong), đảm bảo hai instance mang cùng dữ liệu sẽ được coi là tương đương nhau.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho ba cấu hình parse trường `price` như sau:

```dart
class ModelA {
  final double price;
  ModelA.fromJson(Map<String, dynamic> json) : price = json['price'] as double;
}

class ModelB {
  final double price;
  ModelB.fromJson(Map<String, dynamic> json) : price = (json['price'] as num).toDouble();
}

class ModelC {
  final double price;
  ModelC.fromJson(Map<String, dynamic> json) : price = double.tryParse(json['price'].toString()) ?? 0.0;
}
```

Hãy phân tích kết quả khi truyền vào 3 trường hợp dữ liệu JSON:
- Payload 1: `{'price': 100}` (Số nguyên)
- Payload 2: `{'price': 100.5}` (Số thực)
- Payload 3: `{'price': "100.5"}` (Chuỗi ký tự do API cũ gửi)

#### Kết quả phân tích kỹ thuật:

| Payload | ModelA (`as double`) | ModelB (`(as num).toDouble()`) | ModelC (`tryParse(toString())`) |
| :--- | :--- | :--- | :--- |
| **Payload 1: `{'price': 100}`** | ❌ **Crash** (`TypeError: int is not double`) | ✅ **Thành công** (`price = 100.0`) | ✅ **Thành công** (`price = 100.0`) |
| **Payload 2: `{'price': 100.5}`** | ✅ **Thành công** (`price = 100.5`) | ✅ **Thành công** (`price = 100.5`) | ✅ **Thành công** (`price = 100.5`) |
| **Payload 3: `{'price': "100.5"}`** | ❌ **Crash** (`TypeError: String is not double`) | ❌ **Crash** (`TypeError: String is not num`) | ✅ **Thành công** (`price = 100.5`) |

**Nhận định kỹ thuật**:
- `ModelA`: Kém linh hoạt nhất, dễ gây lỗi crash tại runtime khi máy chủ trả về số nguyên chẵn.
- `ModelB`: Giải pháp tối ưu theo chuẩn của Dart/Flutter đối với các API tuân thủ đúng chuẩn REST JSON.
- `ModelC`: Có khả năng chịu lỗi cao nhất khi tích hợp với các hệ thống backend không chuẩn hóa kiểu dữ liệu, tuy nhiên phát sinh chi phí chuyển đổi chuỗi (`toString()`).
