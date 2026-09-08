# Bài 7.3 — JSON Serialization & Models

## Phần 1 — Khái Niệm & Mục Tiêu Bài Học

### Tại sao bài này quan trọng?

JSON là ngôn ngữ chung giữa Flutter app và REST API. Nhưng từ JSON raw đến Dart model type-safe, cần:

```dart
// Raw JSON từ API:
// {"id": "abc", "name": "Flutter Book", "price": 99.0, "tags": ["dart", "flutter"]}

// Dart model:
final product = Product.fromJson(jsonDecode(response.body));
print(product.name.toUpperCase()); // Type-safe!
```

Bài này dùng **manual serialization** (không codegen) để bạn hiểu cơ chế. Production app thường dùng `json_serializable` để tự động generate code.

### Bạn sẽ hiểu được sau bài này:
- `jsonDecode`, `jsonEncode` — parse và encode JSON
- `fromJson` factory constructor pattern
- `toJson()` method
- Immutable model với `const` constructor và `copyWith`
- Parse nested JSON một cách an toàn

---

## Phần 2 — Cơ Chế Hoạt Động (Under the Hood)

### JSON → Dart Type Mapping

```
JSON          → Dart
null          → null
true/false    → bool
number (int)  → int
number (float) → double
"string"      → String
[...]         → List<dynamic>
{...}         → Map<String, dynamic>
```

```mermaid
flowchart LR
    API["API Response\nString (JSON)"]
    Decode["jsonDecode()\nString → dynamic\n(thực ra là\nMap<String, dynamic>)"]
    Cast["Cast an toàn\nas Map<String, dynamic>"]
    fromJson["Model.fromJson()\nMap<String, dynamic> → Model"]
    Model["Dart Model\n(type-safe)"]

    API --> Decode --> Cast --> fromJson --> Model
```

---

## Phần 3 — Code Mẫu Chuẩn Google

### 3.1 — Model cơ bản với fromJson/toJson

```dart
import 'dart:convert';

// Immutable model (best practice)
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

  // Factory constructor fromJson — parse từ Map
  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as String,
      name: json['name'] as String,
      price: (json['price'] as num).toDouble(), // num vì có thể là int hoặc double
      imageUrl: json['image_url'] as String?,   // Nullable — safe cast
      tags: (json['tags'] as List<dynamic>?)
              ?.map((e) => e as String)
              .toList() ?? [], // Default empty list nếu null
      category: ProductCategory.fromJson(
        json['category'] as Map<String, dynamic>,
      ),
      createdAt: DateTime.parse(json['created_at'] as String),
    );
  }

  // toJson — serialize về Map
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
      'price': price,
      if (imageUrl != null) 'image_url': imageUrl, // Chỉ include nếu không null
      'tags': tags,
      'category': category.toJson(),
      'created_at': createdAt.toIso8601String(),
    };
  }

  // copyWith — immutable update pattern
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

  // Equality để detect changes (dùng trong updateShouldNotify, ==)
  @override
  bool operator==(Object other) =>
      identical(this, other) ||
      other is Product &&
          runtimeType == other.runtimeType &&
          id == other.id &&
          name == other.name &&
          price == other.price;

  @override
  int get hashCode => Object.hash(id, name, price);

  @override
  String toString() => 'Product(id: $id, name: $name, price: $price)';
}

class ProductCategory {
  final String id;
  final String name;

  const ProductCategory({required this.id, required this.name});

  factory ProductCategory.fromJson(Map<String, dynamic> json) {
    return ProductCategory(
      id: json['id'] as String,
      name: json['name'] as String,
    );
  }

  Map<String, dynamic> toJson() => {'id': id, 'name': name};
}
```

### 3.2 — Parse List và Nested Objects an toàn

```dart
// Parse từ API response
class ProductApiResponse {
  final List<Product> items;
  final int total;
  final int page;
  final int pageSize;
  final bool hasMore;

  const ProductApiResponse({
    required this.items,
    required this.total,
    required this.page,
    required this.pageSize,
    required this.hasMore,
  });

  factory ProductApiResponse.fromJson(Map<String, dynamic> json) {
    // Parse nested list — quan trọng: cast đúng type
    final rawItems = json['items'];
    final List<Product> items;

    if (rawItems is List) {
      items = rawItems
          .whereType<Map<String, dynamic>>() // Filter ra items không phải Map
          .map(Product.fromJson)
          .toList();
    } else {
      items = [];
    }

    return ProductApiResponse(
      items: items,
      total: json['total'] as int? ?? 0,
      page: json['page'] as int? ?? 0,
      pageSize: json['page_size'] as int? ?? 20,
      hasMore: json['has_more'] as bool? ?? false,
    );
  }
}

// Sử dụng trong repository
Future<ProductApiResponse> fetchProducts({int page = 0}) async {
  final response = await _client.get('/v1/products?page=$page');
  final json = response.data as Map<String, dynamic>;
  return ProductApiResponse.fromJson(json);
}
```

### 3.3 — Type-safe JSON parsing utilities

```dart
// Extension để parse JSON an toàn, không throw
extension JsonExt on Map<String, dynamic> {
  String getString(String key, {String defaultValue = ''}) {
    return (this[key] as String?) ?? defaultValue;
  }

  int getInt(String key, {int defaultValue = 0}) {
    return (this[key] as int?) ?? defaultValue;
  }

  double getDouble(String key, {double defaultValue = 0}) {
    return (this[key] as num?)?.toDouble() ?? defaultValue;
  }

  bool getBool(String key, {bool defaultValue = false}) {
    return (this[key] as bool?) ?? defaultValue;
  }

  DateTime? getDateTime(String key) {
    final raw = this[key] as String?;
    if (raw == null) return null;
    return DateTime.tryParse(raw);
  }

  List<T> getList<T>(String key, T Function(dynamic) transform) {
    final raw = this[key];
    if (raw is! List) return [];
    return raw.map(transform).toList();
  }
}

// Model dùng extension — much cleaner
class UserProfile {
  final String id;
  final String name;
  final String email;
  final int age;
  final bool isPremium;
  final DateTime? lastSeen;
  final List<String> roles;

  const UserProfile({
    required this.id,
    required this.name,
    required this.email,
    required this.age,
    required this.isPremium,
    this.lastSeen,
    required this.roles,
  });

  factory UserProfile.fromJson(Map<String, dynamic> json) {
    return UserProfile(
      id: json.getString('id'),
      name: json.getString('name', defaultValue: 'Unknown'),
      email: json.getString('email'),
      age: json.getInt('age'),
      isPremium: json.getBool('is_premium'),
      lastSeen: json.getDateTime('last_seen'),
      roles: json.getList('roles', (r) => r as String),
    );
  }
}
```

### 3.4 — encode/decode và string conversion

```dart
class JsonHelper {
  // Encode model → JSON string
  static String encode(Object model) {
    if (model is Product) {
      return jsonEncode(model.toJson());
    }
    throw ArgumentError('Unknown model type: ${model.runtimeType}');
  }

  // Decode JSON string → Map
  static Map<String, dynamic> decode(String jsonString) {
    final dynamic decoded = jsonDecode(jsonString);
    if (decoded is! Map<String, dynamic>) {
      throw FormatException('Expected JSON object, got ${decoded.runtimeType}');
    }
    return decoded;
  }

  // Safe decode — không throw
  static Map<String, dynamic>? tryDecode(String jsonString) {
    try {
      return decode(jsonString);
    } on FormatException {
      return null;
    }
  }
}

// Lưu model vào SharedPreferences
Future<void> saveProduct(Product product) async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.setString('last_viewed_product', jsonEncode(product.toJson()));
}

// Load model từ SharedPreferences
Future<Product?> loadLastProduct() async {
  final prefs = await SharedPreferences.getInstance();
  final jsonStr = prefs.getString('last_viewed_product');
  if (jsonStr == null) return null;

  final json = JsonHelper.tryDecode(jsonStr);
  if (json == null) return null;

  return Product.fromJson(json);
}
```

---

## Phần 4 — Lỗi Sai Phổ Biến & Best Practices

### ❌ Anti-pattern 1: Không cast type — dùng dynamic

```dart
// ❌ Nguy hiểm: Không có type check → runtime crash
factory Product.fromJson(dynamic json) {
  return Product(
    id: json['id'],           // dynamic → có thể null, có thể wrong type
    name: json['name'],       // Không có compile-time check
    price: json['price'],     // int? double? String? → crash khi dùng
  );
}

// ✅ Đúng: Cast tường minh
factory Product.fromJson(Map<String, dynamic> json) {
  return Product(
    id: json['id'] as String,       // Explicit cast → rõ ràng
    name: json['name'] as String,
    price: (json['price'] as num).toDouble(), // num → double an toàn
  );
}
```

### ❌ Anti-pattern 2: Mutable model — fields không final

```dart
// ❌ Sai: Mutable model → unexpected mutation, khó debug
class Product {
  String id;   // Non-final → có thể thay đổi bất kỳ lúc nào!
  String name;
  double price;
}

// ✅ Đúng: Immutable model với copyWith
class Product {
  final String id;
  final String name;
  final double price;

  const Product({required this.id, required this.name, required this.price});

  Product copyWith({String? id, String? name, double? price}) => Product(
    id: id ?? this.id,
    name: name ?? this.name,
    price: price ?? this.price,
  );
}
```

### ❌ Anti-pattern 3: Parse trong Widget build()

```dart
// ❌ Sai: Parse JSON trong build() → expensive CPU mỗi rebuild
Widget build(BuildContext context) {
  final product = Product.fromJson(jsonDecode(rawJson)); // Parse mỗi build!
  return Text(product.name);
}

// ✅ Đúng: Parse một lần trong initState hoặc repository
// Truyền model object (đã parse) vào widget
```

---

## Phần 5 — Bài Tập Củng Cố Tư Duy

### Challenge: Parse Nested JSON API Response

**API Response (JSONPlaceholder):**
```json
{
  "userId": 1,
  "id": 1,
  "title": "sunt aut facere repellat...",
  "body": "quia et suscipit..."
}
```

**Nhiệm vụ:**
1. Tạo `Post` model với `fromJson/toJson/copyWith`
2. Fetch 100 posts từ `jsonplaceholder.typicode.com/posts`
3. Parse tất cả thành `List<Post>`
4. Tạo `PostListResponse` với total count

**Mở rộng:**
- Thêm nested `User` object (fetch `/users/:id`)
- Cache posts vào SharedPreferences
- Implement offline-first: hiển thị cache trước, load fresh sau

### Thử Thách Tư Duy & Thẩm Định Chuyên Sâu (Conceptual & Deep-Dive Check)

> **[Junior]** — nắm khái niệm | **[Middle]** — hiểu cơ chế | **[Senior]** — hiểu Flutter internals | **[Trace Code]** — đọc code và dự đoán output

---

#### Q1 [Junior] — "Sự khác biệt giữa manual serialization và `json_serializable`?"

**Trả lời chuẩn:**

| | Manual | `json_serializable` | `freezed` |
|---|---|---|---|
| **Boilerplate** | Nhiều | Ít (generated) | Ít nhất |
| **build_runner** | Không | Cần | Cần |
| **copyWith** | Tự viết | Tự viết | Generated |
| **Equality** | Tự viết | Tự viết | Generated |
| **Union types** | Không | Không | Có |
| **Learning curve** | Thấp | Trung bình | Cao |

```dart
// Manual — dễ hiểu, nhiều code
class User {
  final String id;
  final String name;
  
  factory User.fromJson(Map<String, dynamic> json) => User(
    id: json['id'] as String,
    name: json['name'] as String,
  );
  
  Map<String, dynamic> toJson() => {'id': id, 'name': name};
}

// json_serializable — generate fromJson/toJson
@JsonSerializable()
class User {
  final String id;
  final String name;
  
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  Map<String, dynamic> toJson() => _$UserToJson(this);
}
// Chạy: dart run build_runner build
```

---

#### Q2 [Junior] — "Tại sao model nên immutable? `copyWith` giúp gì?"

**Trả lời chuẩn:**

Immutable model có **3 lợi ích** chính:

**1. Dễ track changes:** Mỗi thay đổi state tạo object mới → so sánh `old != new` dễ dàng → Flutter/Provider biết khi nào cần rebuild.

**2. Thread-safe:** Immutable objects không cần synchronization — multiple Isolates có thể đọc cùng object mà không có race condition.

**3. Predictable state:** Không ai có thể "accidentally" thay đổi model từ chỗ khác.

```dart
// Immutable model với copyWith
class User {
  final String id;
  final String name;
  final String email;
  
  const User({required this.id, required this.name, required this.email});
  
  // Tạo copy với một số field thay đổi
  User copyWith({String? id, String? name, String? email}) => User(
    id: id ?? this.id,
    name: name ?? this.name,
    email: email ?? this.email,
  );
}

// Dùng copyWith — không modify trực tiếp
final updatedUser = currentUser.copyWith(name: 'New Name');
// currentUser không đổi → ChangeNotifier/Provider detect sự khác biệt
setState(() => _user = updatedUser);
```

---

#### Q3 [Middle] — "Tại sao `(json['price'] as num).toDouble()` thay vì `json['price'] as double`?"

**Trả lời chuẩn:**

**Vấn đề:** JSON specification không có int/double distinction cho numbers. Dart's JSON decoder (`dart:convert`) maps:
- `99` (no decimal) → `int`
- `99.0` (with decimal) → `double`

```dart
import 'dart:convert';

final json = jsonDecode('{"price": 99}');
print(json['price'].runtimeType); // int

final json2 = jsonDecode('{"price": 99.0}');
print(json2['price'].runtimeType); // double

// ❌ Crash khi server gửi 99 (integer)
double price = json['price'] as double; // TypeError: int is not double

// ✅ An toàn — num accept cả int và double
double price = (json['price'] as num).toDouble(); // OK

// ✅ Alternative
double price = (json['price'] as num?)?.toDouble() ?? 0.0; // null safe
```

**Server behavior không nhất quán:** Backend có thể gửi `99` hoặc `99.0` tùy implementation. Ruby on Rails thường gửi `99` cho integer. Python `json.dumps(99.0)` gửi `99.0`. Dart side phải handle cả hai.

---

#### Q4 [Senior] — "`json_serializable` generate code ra file `.g.dart`: tại sao cần `build_runner`? Cơ chế code generation?"

**Trả lời chuẩn:**

**`build_runner`** là build system của Dart cho **source code generation**. Nó scan files, tìm annotations (`@JsonSerializable`, `@freezed`), và gọi các `Builder` tương ứng để generate code:

```
dart run build_runner build
  ↓
build_runner scan tất cả .dart files trong project
  ↓
Tìm class có @JsonSerializable annotation
  ↓
json_serializable_generator.Builder được gọi
  ↓ analyze class (fields, types, annotations)
  ↓ generate _$UserFromJson() và _$UserToJson()
  ↓
Ghi output vào user.g.dart
```

**Tại sao không dùng reflection (dart:mirrors)?**

Dart reflection (mirrors) không work trong ahead-of-time (AOT) compilation — Flutter app được compile AOT. Code generation tạo normal Dart code tại **build time** → không cần reflection at runtime → performance tốt hơn.

```dart
// Generated file: user.g.dart
User _$UserFromJson(Map<String, dynamic> json) => User(
  id: json['id'] as String,
  name: json['name'] as String,
  createdAt: DateTime.parse(json['createdAt'] as String),
);

Map<String, dynamic> _$UserToJson(User instance) => <String, dynamic>{
  'id': instance.id,
  'name': instance.name,
  'createdAt': instance.createdAt.toIso8601String(),
};
```

**Watch mode cho dev:** `dart run build_runner watch` — tự động regenerate khi file thay đổi.

---

#### Q5 [Middle] — "`freezed` vs `json_serializable` — khi nào chọn freezed?"

**Trả lời chuẩn:**

| Feature | `json_serializable` | `freezed` |
|---|---|---|
| **JSON serialize** | Có | Có (tích hợp) |
| **copyWith** | Phải tự viết | Generated |
| **Equality (`==`)** | Phải tự viết | Generated (deep equality) |
| **Union types** | Không | Có (sealed-like) |
| **Immutability** | Phải tự enforce | Generated (unmodifiable) |
| **Pattern matching** | Không | Có (`when`, `map`) |

```dart
// json_serializable — chỉ serialization
@JsonSerializable()
class ApiError {
  final String message;
  final int code;
  // Phải tự viết copyWith, ==, hashCode
}

// freezed — full-featured model
@freezed
class ApiResult<T> with _$ApiResult<T> {
  const factory ApiResult.success(T data) = Success;
  const factory ApiResult.failure(String message, int code) = Failure;
  const factory ApiResult.loading() = Loading;
}

// Pattern matching như sealed class
result.when(
  success: (data) => showData(data),
  failure: (msg, code) => showError(msg),
  loading: () => showSpinner(),
)
```

**Chọn freezed khi:** Model phức tạp, cần union types (Result/Either pattern), cần generated equality và copyWith.

---

#### Q6 [Middle] — "Nested object serialization: `fromJson` factory cho nested object phải làm gì?"

**Trả lời chuẩn:**

```dart
// API response: {"user": {"id": "1", "address": {"city": "Hanoi", "zip": "10000"}}}

class Address {
  final String city;
  final String zip;
  
  factory Address.fromJson(Map<String, dynamic> json) => Address(
    city: json['city'] as String,
    zip: json['zip'] as String,
  );
}

class User {
  final String id;
  final Address address; // nested object
  
  factory User.fromJson(Map<String, dynamic> json) => User(
    id: json['id'] as String,
    address: Address.fromJson(json['address'] as Map<String, dynamic>), // recursive
  );
}

// List of nested objects
class Order {
  final List<Item> items;
  
  factory Order.fromJson(Map<String, dynamic> json) => Order(
    items: (json['items'] as List<dynamic>)
        .map((item) => Item.fromJson(item as Map<String, dynamic>))
        .toList(),
  );
}

// Nullable nested object
class Profile {
  final Address? address; // optional
  
  factory Profile.fromJson(Map<String, dynamic> json) => Profile(
    address: json['address'] != null
        ? Address.fromJson(json['address'] as Map<String, dynamic>)
        : null,
  );
}
```

---

#### Q7 [Trace Code] — "Server trả `{"price": 99}` vs `{"price": 99.0}`: parse code nào safe, code nào crash?"

```dart
// Model
class Product {
  final double price;
  const Product({required this.price});
  
  factory Product.fromJson(Map<String, dynamic> json) => Product(
    price: json['price'] as double, // ← CASE A
    // price: (json['price'] as num).toDouble(), // ← CASE B
    // price: double.tryParse(json['price'].toString()) ?? 0.0, // ← CASE C
  );
}

// Test 1: Server gửi integer
final json1 = {'price': 99}; // int
final p1 = Product.fromJson(json1);

// Test 2: Server gửi double
final json2 = {'price': 99.0}; // double
final p2 = Product.fromJson(json2);

// Test 3: Server gửi string (typo hoặc legacy API)
final json3 = {'price': '99.5'}; // String!
final p3 = Product.fromJson(json3);
```

**CASE A — `json['price'] as double`:**
- Test 1 (`99` int): **CRASH** — `TypeError: int is not double`
- Test 2 (`99.0` double): ✅ OK
- Test 3 (`'99.5'` string): **CRASH** — `TypeError: String is not double`

**CASE B — `(json['price'] as num).toDouble()`:**
- Test 1 (`99` int): ✅ OK — `int` extends `num` → `99.toDouble()` = 99.0
- Test 2 (`99.0` double): ✅ OK
- Test 3 (`'99.5'` string): **CRASH** — `String` không extends `num`

**CASE C — `double.tryParse(json['price'].toString())`:**
- Test 1 (`99` int): ✅ OK — `99.toString()` = `"99"` → `double.tryParse("99")` = 99.0
- Test 2 (`99.0` double): ✅ OK
- Test 3 (`'99.5'` string): ✅ OK — `'99.5'.toString()` = `"99.5"` → 99.5

**Kết luận:** CASE B là standard pattern (tốt hơn CASE A). CASE C flexible nhất nhưng có overhead của String conversion. Trong production, CASE B là đủ nếu server không bao giờ gửi string prices.
