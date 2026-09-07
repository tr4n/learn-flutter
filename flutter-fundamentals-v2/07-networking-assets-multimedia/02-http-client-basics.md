# Bài 7.2 — HTTP Client Basics

## Phần 1 — Khái Niệm & Mục Tiêu Bài Học

### Tại sao bài này quan trọng?

Hầu hết Flutter app đều kết nối API. Có hai lựa chọn chính:
- `http` package: lightweight, official-ish, đủ cho simple use case
- `Dio`: full-featured, interceptors, cancellation, form data, progress tracking

Biết khi nào dùng cái nào và cách sử dụng đúng là nền tảng cho mọi networked app.

### Bạn sẽ hiểu được sau bài này:
- `http` package: GET, POST, PUT, DELETE với error handling
- Headers, status codes, response body parsing
- `Dio`: interceptors, BaseOptions, timeouts
- Repository pattern để abstract HTTP calls

---

## Phần 2 — Cơ Chế Hoạt Động (Under the Hood)

### HTTP Client Architecture

```mermaid
graph TB
    UI["Widget / ViewModel"]
    Repo["Repository\n(abstract interface)"]
    HTTP["HTTP Client\n(http / Dio)"]
    API["REST API"]

    UI -->|"repository.getProducts()"| Repo
    Repo -->|"GET /api/products"| HTTP
    HTTP <-->|"HTTP Request/Response"| API
    HTTP -->|"raw Response"| Repo
    Repo -->|"List<Product>"| UI

    Note["Repository pattern: UI không biết\ncụ thể là http hay Dio"]
```

---

## Phần 3 — Code Mẫu Chuẩn Google

### 3.1 — `http` package cơ bản

```yaml
# pubspec.yaml
dependencies:
  http: ^1.2.0
```

```dart
import 'package:http/http.dart' as http;

class SimpleApiClient {
  static const _baseUrl = 'https://jsonplaceholder.typicode.com';

  // GET request
  Future<List<Map<String, dynamic>>> getUsers() async {
    final uri = Uri.parse('$_baseUrl/users');
    final response = await http.get(
      uri,
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    );

    if (response.statusCode == 200) {
      final List<dynamic> json = jsonDecode(response.body);
      return json.cast<Map<String, dynamic>>();
    }

    throw HttpException('GET /users failed: ${response.statusCode}');
  }

  // POST request với body
  Future<Map<String, dynamic>> createPost({
    required String title,
    required String body,
    required int userId,
  }) async {
    final uri = Uri.parse('$_baseUrl/posts');
    final response = await http.post(
      uri,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer $authToken',
      },
      body: jsonEncode({
        'title': title,
        'body': body,
        'userId': userId,
      }),
    );

    if (response.statusCode == 201) {
      return jsonDecode(response.body) as Map<String, dynamic>;
    }

    throw HttpException('POST /posts failed: ${response.statusCode}');
  }

  String get authToken => 'your-auth-token'; // From secure storage
}
```

### 3.2 — Dio — Production-ready client

```yaml
# pubspec.yaml
dependencies:
  dio: ^5.0.0
```

```dart
import 'package:dio/dio.dart';

// Singleton Dio client với config
class ApiClient {
  static final ApiClient _instance = ApiClient._();
  factory ApiClient() => _instance;
  ApiClient._();

  late final Dio _dio = Dio(
    BaseOptions(
      baseUrl: 'https://api.example.com',
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'X-App-Version': '1.0.0',
      },
    ),
  )
    ..interceptors.add(_AuthInterceptor())
    ..interceptors.add(_LoggingInterceptor())
    ..interceptors.add(_ErrorInterceptor());

  // GET
  Future<Response<T>> get<T>(
    String path, {
    Map<String, dynamic>? queryParameters,
    Options? options,
  }) async {
    return _dio.get<T>(
      path,
      queryParameters: queryParameters,
      options: options,
    );
  }

  // POST
  Future<Response<T>> post<T>(
    String path, {
    dynamic data,
    Options? options,
  }) async {
    return _dio.post<T>(path, data: data, options: options);
  }

  // PUT
  Future<Response<T>> put<T>(String path, {dynamic data}) async {
    return _dio.put<T>(path, data: data);
  }

  // DELETE
  Future<Response<void>> delete(String path) async {
    return _dio.delete(path);
  }
}

// Auth Interceptor: tự động thêm token
class _AuthInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final token = AuthStorage.getToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options); // Tiếp tục request
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      // Token expired → refresh token
      try {
        await AuthService().refreshToken();
        // Retry original request
        final retryResponse = await ApiClient()._dio.fetch(err.requestOptions);
        handler.resolve(retryResponse);
        return;
      } catch (_) {
        AuthService().logout();
      }
    }
    handler.next(err);
  }
}

// Logging Interceptor (debug only)
class _LoggingInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    if (kDebugMode) {
      debugPrint('→ ${options.method} ${options.path}');
    }
    handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    if (kDebugMode) {
      debugPrint('← ${response.statusCode} ${response.requestOptions.path}');
    }
    handler.next(response);
  }
}

// Error Interceptor: transform DioException → app exception
class _ErrorInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final appError = switch (err.type) {
      DioExceptionType.connectionTimeout ||
      DioExceptionType.receiveTimeout => const NetworkException(
          endpoint: '',
          message: 'Kết nối quá chậm',
        ),
      DioExceptionType.connectionError => const NetworkException(
          endpoint: '',
          message: 'Không có kết nối mạng',
        ),
      _ => NetworkException(
          endpoint: err.requestOptions.path,
          statusCode: err.response?.statusCode,
          message: 'Lỗi không xác định',
        ),
    };
    // Reject với app exception
    handler.reject(DioException(
      requestOptions: err.requestOptions,
      error: appError,
    ));
  }
}
```

### 3.3 — Repository Pattern

```dart
// Abstract interface: tách logic từ implementation
abstract interface class ProductRepository {
  Future<List<Product>> fetchProducts({int page = 0, int pageSize = 20});
  Future<Product> fetchProductById(String id);
  Future<Product> createProduct(CreateProductRequest request);
  Future<Product> updateProduct(String id, UpdateProductRequest request);
  Future<void> deleteProduct(String id);
}

// Concrete implementation với Dio
class ApiProductRepository implements ProductRepository {
  final ApiClient _client;

  const ApiProductRepository({ApiClient? client})
      : _client = client ?? const ApiClient();

  @override
  Future<List<Product>> fetchProducts({int page = 0, int pageSize = 20}) async {
    final response = await _client.get<List<dynamic>>(
      '/v1/products',
      queryParameters: {'page': page, 'size': pageSize},
    );

    final data = response.data;
    if (data == null) return [];

    return data
        .whereType<Map<String, dynamic>>()
        .map(Product.fromJson)
        .toList();
  }

  @override
  Future<Product> fetchProductById(String id) async {
    final response = await _client.get<Map<String, dynamic>>('/v1/products/$id');
    final data = response.data;
    if (data == null) throw const ParseException(fieldName: 'product', receivedValue: null);
    return Product.fromJson(data);
  }

  @override
  Future<Product> createProduct(CreateProductRequest request) async {
    final response = await _client.post<Map<String, dynamic>>(
      '/v1/products',
      data: request.toJson(),
    );
    return Product.fromJson(response.data!);
  }

  @override
  Future<Product> updateProduct(String id, UpdateProductRequest request) async {
    final response = await _client.put<Map<String, dynamic>>(
      '/v1/products/$id',
      data: request.toJson(),
    );
    return Product.fromJson(response.data!);
  }

  @override
  Future<void> deleteProduct(String id) async {
    await _client.delete('/v1/products/$id');
  }
}

// Mock repository cho test
class MockProductRepository implements ProductRepository {
  final List<Product> _products;
  MockProductRepository({List<Product>? products})
      : _products = products ?? _defaultProducts;

  @override
  Future<List<Product>> fetchProducts({int page = 0, int pageSize = 20}) async {
    await Future.delayed(const Duration(milliseconds: 100)); // Simulate network
    return _products.skip(page * pageSize).take(pageSize).toList();
  }

  // ... other methods

  static final List<Product> _defaultProducts = [
    Product(id: '1', name: 'Product A', price: 100000),
    Product(id: '2', name: 'Product B', price: 200000),
  ];
}
```

---

## Phần 4 — Lỗi Sai Phổ Biến & Best Practices

### ❌ Anti-pattern 1: Tạo Dio client mới mỗi lần gọi

```dart
// ❌ Sai: Tạo Dio object mới → mất interceptors, mất connection pool
Future<List<Product>> getProducts() async {
  final dio = Dio(); // Tạo mới mỗi lần!
  return dio.get('/products');
}

// ✅ Đúng: Singleton Dio instance
class ApiClient {
  static final ApiClient _instance = ApiClient._();
  factory ApiClient() => _instance;
  late final Dio _dio = Dio(BaseOptions(...));
}
```

### ❌ Anti-pattern 2: Catch DioException ở mọi nơi

```dart
// ❌ Sai: Xử lý lỗi ở UI layer — tight coupling
try {
  final products = await _repository.fetchProducts();
} on DioException catch (e) {
  // UI biết về Dio?! → coupling với HTTP client
}

// ✅ Đúng: Xử lý ở repository hoặc interceptor → trả về app exception
// Repository throws AppException, không phải DioException
```

### ❌ Anti-pattern 3: Không set timeout

```dart
// ❌ Không có timeout → chờ mãi nếu server không respond
final dio = Dio(BaseOptions(baseUrl: '...'));

// ✅ Luôn set timeout
final dio = Dio(BaseOptions(
  connectTimeout: const Duration(seconds: 10),
  receiveTimeout: const Duration(seconds: 30),
  sendTimeout: const Duration(seconds: 10),
));
```

---

## Phần 5 — Bài Tập Củng Cố Tư Duy

### Challenge: Fetch Danh Sách từ JSONPlaceholder

**Yêu cầu:**
1. Fetch `https://jsonplaceholder.typicode.com/posts` (100 posts)
2. Parse thành `Post(id, title, body, userId)` model
3. Hiển thị trong ListView
4. Handle loading, error, empty states
5. Add `Dio` với logging interceptor

**Bonus:**
- Implement pagination (load 10 posts mỗi lần)
- Pull-to-refresh với `RefreshIndicator`

### Câu hỏi phỏng vấn liên quan:

1. **"Khi nào dùng `http` package, khi nào dùng `Dio`?"**
   - `http`: simple app, ít feature, muốn lightweight
   - `Dio`: interceptors, auth token refresh, file upload, cancellation, request timeout

2. **"Interceptor trong Dio dùng để làm gì?"**
   - Tự động thêm header (auth token)
   - Log requests/responses
   - Handle 401 → refresh token → retry
   - Transform errors → app-specific exceptions

3. **"Repository pattern có lợi ích gì?"**
   - Tách UI khỏi HTTP implementation
   - Dễ swap (http → Dio, mock cho test)
   - Single source of truth cho data fetching logic
