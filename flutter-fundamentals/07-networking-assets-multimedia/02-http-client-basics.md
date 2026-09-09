# Bài 7.2 — Giao Tiếp Mạng Trong Flutter: package:http & package:dio

## Tài Liệu Tham Khảo Chính Thức
- [Flutter Documentation: Fetch data from the internet](https://docs.flutter.dev/cookbook/networking/fetch-data)
- [Dart package:http Documentation](https://pub.dev/packages/http)
- [Dart package:dio Documentation](https://pub.dev/packages/dio)

---

## Phần 1 — Khái Niệm & Kiến Trúc Tầng Mạng (Networking Layer Architecture)

### 1.1 — Mô Hình Phân Tầng Kiến Trúc Mạng

Trong kiến trúc phần mềm Flutter, tầng giao tiếp mạng (Networking / Data Layer) chịu trách nhiệm gửi các yêu cầu (HTTP Requests) và tiếp nhận dữ liệu phản hồi (HTTP Responses) từ các dịch vụ máy chủ.

Theo tài liệu hướng dẫn kiến trúc của Flutter và nguyên lý phân tách mối quan tâm (Separation of Concerns), luồng dữ liệu được tổ chức phân tầng như sau:

```
┌────────────────────────────────────────────────────────────────────────┐
│ MÔ HÌNH PHÂN TẦNG KIẾN TRÚC MẠNG                                       │
│                                                                        │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ PRESENTATION LAYER (Widgets, Screens)                              │ │
│ │  • Hiển thị dữ liệu, tiếp nhận sự kiện từ người dùng               │ │
│ └──────────────────────────────────┬─────────────────────────────────┘ │
│                                    │ Lắng nghe State & Kích hoạt tác vụ│
│                                    ▼                                   │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ BUSINESS LOGIC LAYER (Bloc, Cubit, ChangeNotifier, AsyncNotifier)   │ │
│ │  • Điều phối luồng nghiệp vụ, quản lý trạng thái tải               │ │
│ └──────────────────────────────────┬─────────────────────────────────┘ │
│                                    │ Gọi phương thức qua Interface     │
│                                    ▼                                   │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ REPOSITORY LAYER (Data Abstraction)                                │ │
│ │  • Ánh xạ DTO sang Domain Model, kết hợp Cache và Remote Data      │ │
│ └──────────────────────────────────┬─────────────────────────────────┘ │
│                                    │ Truy xuất dữ liệu thô             │
│                                    ▼                                   │
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ NETWORKING INFRASTRUCTURE (http.Client / Dio)                      │ │
│ │  • Quản lý HTTP, Interceptor Chain, Xác thực Token, Timeouts       │ │
│ └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

- **Presentation Layer**: Không tham chiếu trực tiếp đến các lớp mạng hoặc URL endpoint.
- **Repository Layer**: Đóng vai trò là lớp trừu tượng hóa nguồn dữ liệu (Data Source Abstraction), giúp tầng nghiệp vụ không phụ thuộc vào thư viện HTTP cụ thể nào.

---

### 1.2 — So Sánh Kỹ Thuật: `package:http` vs `package:dio`

Hệ sinh thái Flutter và Dart cung cấp hai thư viện phổ biến để thực hiện các cuộc gọi HTTP:

#### 1. `package:http`
- **Nguồn gốc**: Được phát triển và bảo trì trực tiếp bởi nhóm phát triển Dart (Dart Team).
- **Đặc tính**: Cung cấp API cấp cao, tối giản trên nền tảng `dart:io` `HttpClient` (trên di động/desktop) và `HttpRequest` (trên trình duyệt web).
- **Khả năng**:
  - Hỗ trợ các phương thức HTTP cơ bản (`get`, `post`, `put`, `delete`, `head`, `patch`).
  - Lớp `http.Client` hỗ trợ duy trì kết nối liên tục (persistent connection / Keep-Alive) giữa nhiều yêu cầu gửi tới cùng một máy chủ.
  - Cho phép kiểm thử với `http/testing.dart` (`MockClient`).
- **Phù hợp với**: Các ứng dụng có quy mô nhỏ, các tác vụ mạng cơ bản, hoặc khi cần tối ưu kích thước gói cài đặt.

#### 2. `package:dio`
- **Nguồn gốc**: Dự án mã nguồn mở do cộng đồng phát triển (CFUG - Flutter China User Group).
- **Đặc tính**: Thư viện HTTP Client đa năng, mở rộng nhiều tính năng nâng cao.
- **Khả năng**:
  - Hỗ trợ chuỗi chặn trung gian (Interceptor Pipeline: `onRequest`, `onResponse`, `onError`).
  - Hỗ trợ hàng đợi xử lý tuần tự (`QueuedInterceptor`) phù hợp cho bài toán làm mới Token đồng thời.
  - Hỗ trợ hủy yêu cầu đang thực thi thông qua `CancelToken`.
  - Tách bạch cấu hình thời gian chờ: `connectTimeout`, `sendTimeout`, `receiveTimeout`.
  - Hỗ trợ theo dõi tiến trình tải lên / tải xuống (`onSendProgress`, `onReceiveProgress`).
  - Hỗ trợ định dạng `FormData` và tải lên tệp nhị phân (`MultipartFile`).
- **Phù hợp với**: Các ứng dụng cần quản lý phiên đăng nhập (Auth Token refresh), xử lý lỗi tập trung, tải tệp tin lớn, hoặc yêu cầu kiểm soát kết nối chi tiết.

#### Bảng so sánh tính năng kỹ thuật:

| Tiêu Chí Kỹ Thuật | `package:http` | `package:dio` |
| :--- | :--- | :--- |
| **Đơn vị bảo trì** | Dart Team | Cộng đồng Flutter (CFUG) |
| **Kiến trúc Interceptors** | Không hỗ trợ sẵn (cần tự viết BaseClient) | Hỗ trợ sẵn chuỗi Interceptor |
| **Khóa hàng đợi làm mới Token** | Cần tự triển khai thủ công | Hỗ trợ qua `QueuedInterceptor` |
| **Cấu hình Timeout chi tiết** | Chỉ hỗ trợ timeout toàn cục | Phân tách: Connect, Send, Receive Timeout |
| **Cơ chế Hủy Yêu Cầu (Cancel)** | Không hỗ trợ trực tiếp trên request | Hỗ trợ thông qua `CancelToken` |
| **Theo dõi tiến trình truyền tệp** | Không hỗ trợ trực tiếp | Hỗ trợ qua `onSendProgress`, `onReceiveProgress` |
| **Tự động giải mã JSON Body** | Phải dùng `jsonDecode()` thủ công | Tự động giải mã sang `Map/List` |
| **Hỗ trợ tải lên FormData / File** | Qua `http.MultipartRequest` | Qua `FormData` và `MultipartFile` |

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Vòng Đời Interceptor Pipeline Trong Dio

Theo tài liệu của `dio`, một Interceptor đóng vai trò như một middleware cho các yêu cầu HTTP. Khi một request được gửi đi, nó đi qua các phương thức theo thứ tự:

```
┌────────────────────────────────────────────────────────────────────────┐
│ DIO INTERCEPTOR PIPELINE                                               │
│                                                                        │
│ [GIAI ĐOẠN REQUEST]                                                    │
│ RequestOptions ──► Interceptor 1 ──► Interceptor 2 ──► Network Layer   │
│                    (Gắn Token)       (Ghi Log)                         │
│                                                                        │
│ [GIAI ĐOẠN RESPONSE]                                                   │
│ Response       ◄── Interceptor 1 ◄── Interceptor 2 ◄── Network Layer   │
│                    (Xử lý Data)      (Ghi Log)                         │
│                                                                        │
│ [GIAI ĐOẠN ERROR]                                                      │
│ AppException   ◄── Interceptor 1 ◄── Interceptor 2 ◄── Network Error   │
│                    (Ánh xạ lỗi)      (Refresh Token)                   │
└────────────────────────────────────────────────────────────────────────┘
```

Mỗi Interceptor bao gồm 3 phương thức xử lý chính:
1. **`onRequest(RequestOptions options, RequestInterceptorHandler handler)`**:
   - Được thực thi trước khi request được chuyển xuống tầng socket.
   - Thường dùng để: Gắn header xác thực (`Authorization: Bearer <token>`), thêm tham số truy vấn chung (`locale`, `app_version`), hoặc kiểm tra điều kiện tiên quyết.
   - Kết thúc bằng `handler.next(options)` để chuyển tiếp, hoặc `handler.resolve(response)` để trả về kết quả ngay mà không gửi request, hoặc `handler.reject(error)` để dừng request với lỗi.
2. **`onResponse(Response response, ResponseInterceptorHandler handler)`**:
   - Được thực thi khi nhận phản hồi thành công từ máy chủ (mã trạng thái $2xx$).
   - Thường dùng để: Đọc thông tin từ Response Header, lưu trữ dữ liệu phiên, hoặc biến đổi cấu trúc phản hồi.
   - Kết thúc bằng `handler.next(response)` để tiếp tục.
3. **`onError(DioException err, ErrorInterceptorHandler handler)`**:
   - Được thực thi khi xảy ra lỗi mạng hoặc máy chủ trả về mã trạng thái lỗi ($4xx, 5xx$).
   - Thường dùng để: Phục hồi phiên đăng nhập khi gặp mã $401$, thử lại request (retry), hoặc chuẩn hóa cấu trúc lỗi trước khi chuyển ra tầng ngoài.
   - Kết thúc bằng:
     - `handler.next(err)`: Tiếp tục chuyển lỗi cho interceptor tiếp theo.
     - `handler.resolve(response)`: Biến lỗi thành phản hồi thành công (sau khi retry thành công).
     - `handler.reject(err)`: Dừng luồng với lỗi đã được biến đổi.

---

### 2.2 — Cơ Chế Hủy Yêu Cầu Bằng `CancelToken`

Trong ứng dụng di động, các yêu cầu mạng thường gắn liền với vòng đời của màn hình hoặc thao tác nhập liệu của người dùng (ví dụ: tìm kiếm dạng autocomplete hoặc người dùng thoát khỏi màn hình trước khi API trả về kết quả).

Nếu không hủy các yêu cầu không còn cần thiết:
- Thiết bị tiêu tốn băng thông và năng lượng để tiếp tục nhận dữ liệu không còn sử dụng.
- Dữ liệu trả về muộn có thể kích hoạt các xử lý giao diện không mong muốn nếu widget đã bị dỡ bỏ.

```
┌────────────────────────────────────────────────────────────────────────┐
│ CƠ CHẾ CANCEL TOKEN                                                    │
│                                                                        │
│  Tầng Gọi: cancelToken.cancel('User navigated away')                   │
│      │                                                                 │
│      ▼                                                                 │
│  Dio Adapter                                                           │
│      │                                                                 │
│      ▼                                                                 │
│  dart:io HttpClientRequest.abort()                                     │
│      │                                                                 │
│      ▼                                                                 │
│  Đóng Socket TCP ──► Giải phóng tài nguyên I/O tức thì                 │
└────────────────────────────────────────────────────────────────────────┘
```

Theo thiết kế của `dio`:
- Mỗi `CancelToken` giữ một `Completer<DioException>`.
- Khi gọi `cancelToken.cancel([reason])`, token phát tín hiệu hủy tới adapter mạng bên dưới (`IOHttpClientAdapter`).
- Adapter gọi hàm `HttpClientRequest.abort()` trên kết nối của Dart VM, gửi tín hiệu ngắt socket xuống hệ điều hành, giải phóng luồng I/O mà không cần đọc hết dữ liệu còn lại từ máy chủ.
- Yêu cầu sẽ ném ra ngoại lệ `DioException` với kiểu `DioExceptionType.cancel`.

---

### 2.3 — Quản Lý Kết Nối & Tái Sử Dụng Socket (Connection Pooling / Keep-Alive)

Khi thực hiện yêu cầu qua HTTPS, mỗi kết nối mới cần trải qua các bước:
1. Phân giải địa chỉ DNS (DNS Lookup).
2. Bắt tay thiết lập kết nối TCP (3-way TCP Handshake: SYN, SYN-ACK, ACK).
3. Bắt tay mã hóa bảo mật (TLS/SSL Handshake: trao đổi chứng chỉ và tạo khóa mã hóa).

Tổng thời gian cho quá trình này thường mất từ $100\text{ms} - 300\text{ms}$ tùy thuộc vào độ trễ mạng.

```
┌────────────────────────────────────────────────────────────────────────┐
│ TÁI SỬ DỤNG KẾT NỐI (HTTP KEEP-ALIVE)                                  │
│                                                                        │
│  Request 1 ──► [TCP Handshake] ──► [TLS Handshake] ──► [Truyền Dữ Liệu]│
│                                                              │         │
│                                                    Socket mở │ (Reuse) │
│                                                              ▼         │
│  Request 2 ──────────────────────────────────────────► [Truyền Dữ Liệu]│
│                        (Không cần bắt tay lại)                         │
└────────────────────────────────────────────────────────────────────────┘
```

- **Nguyên tắc**: Cả `http.Client` và `Dio` đều quản lý một đối tượng `HttpClient` ngầm định, hỗ trợ cơ chế HTTP Keep-Alive. Nếu giữ nguyên cùng một instance Client, các yêu cầu tiếp theo gửi tới cùng một máy chủ sẽ tái sử dụng kết nối TCP đã được thiết lập, loại bỏ thời gian bắt tay TCP/TLS cho các yêu cầu sau.
- **Lưu ý**: Khởi tạo đối tượng Client mới ở mỗi lần gọi hàm sẽ làm mất khả năng tái sử dụng socket, khiến mỗi yêu cầu đều phải thiết lập lại kết nối từ đầu, làm tăng độ trễ và tiêu hao tài nguyên hệ thống.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Sử Dụng `package:http` Theo Hướng Dẫn Chính Thức Của Flutter

Theo tài liệu chính thức của Flutter ([Fetch data from the internet](https://docs.flutter.dev/cookbook/networking/fetch-data)), khi thực hiện nhiều yêu cầu hoặc cần kiểm thử, nên sử dụng `http.Client`:

```dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

class AlbumService {
  final http.Client _client;

  AlbumService({http.Client? client}) : _client = client ?? http.Client();

  Future<Album> fetchAlbum(int id) async {
    final uri = Uri.https('jsonplaceholder.typicode.com', '/albums/$id');
    final response = await _client.get(
      uri,
      headers: {
        HttpHeaders.contentTypeHeader: 'application/json; charset=UTF-8',
        HttpHeaders.acceptHeader: 'application/json',
      },
    );

    if (response.statusCode == 200) {
      final Map<String, dynamic> json = jsonDecode(response.body) as Map<String, dynamic>;
      return Album.fromJson(json);
    } else {
      throw HttpException('Failed to load album: ${response.statusCode}');
    }
  }

  void dispose() {
    _client.close();
  }
}

class Album {
  final int userId;
  final int id;
  final String title;

  const Album({required this.userId, required this.id, required this.title});

  factory Album.fromJson(Map<String, dynamic> json) {
    return Album(
      userId: json['userId'] as int,
      id: json['id'] as int,
      title: json['title'] as String,
    );
  }
}
```

---

### 3.2 — Cấu Hình `DioClient` Tập Trung

Để quản lý cấu hình chung (URL cơ sở, thời gian chờ, bộ Interceptors), tạo một lớp `DioClient` dạng Singleton:

```dart
import 'package:dio/dio.dart';
import 'package:flutter/foundation.dart';

class DioClient {
  static final DioClient _instance = DioClient._internal();
  factory DioClient() => _instance;

  late final Dio _dio;
  Dio get dio => _dio;

  DioClient._internal() {
    _dio = Dio(
      BaseOptions(
        baseUrl: 'https://api.example.com/api/v1',
        // Thời gian chờ tối đa khi thiết lập kết nối TCP
        connectTimeout: const Duration(seconds: 10),
        // Thời gian chờ tối đa giữa hai gói tin nhận về
        receiveTimeout: const Duration(seconds: 20),
        // Thời gian chờ tối đa để gửi toàn bộ dữ liệu đi
        sendTimeout: const Duration(seconds: 10),
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
          'X-Platform': defaultTargetPlatform.name,
        },
        responseType: ResponseType.json,
      ),
    );

    _dio.interceptors.addAll([
      AuthInterceptor(),
      if (kDebugMode) LoggingInterceptor(),
      ErrorMappingInterceptor(),
    ]);
  }
}
```

---

### 3.3 — Triển Khai Các Interceptors (Auth, Logging, Error Mapping)

#### 1. `AuthInterceptor`: Tự động gắn Token xác thực
Tự động bổ sung `Authorization: Bearer <token>` vào tiêu đề của các yêu cầu cần xác thực:

```dart
import 'package:dio/dio.dart';

class AuthInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final bool isPublicEndpoint = options.extra['isPublic'] == true;

    if (!isPublicEndpoint) {
      final String? token = await TokenStorage.getAccessToken();
      if (token != null && token.isNotEmpty) {
        options.headers['Authorization'] = 'Bearer $token';
      }
    }

    return handler.next(options);
  }
}

abstract class TokenStorage {
  static Future<String?> getAccessToken() async => 'sample_jwt_token';
}
```

#### 2. `LoggingInterceptor`: Ghi nhật ký yêu cầu và phản hồi
Hỗ trợ kiểm tra thông tin yêu cầu trong quá trình phát triển (Debug):

```dart
import 'package:dio/dio.dart';
import 'package:flutter/foundation.dart';

class LoggingInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    debugPrint('--> HTTP ${options.method} ${options.uri}');
    debugPrint('Headers: ${options.headers}');
    if (options.data != null) {
      debugPrint('Body: ${options.data}');
    }
    return handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    debugPrint('<-- HTTP ${response.statusCode} ${response.requestOptions.uri}');
    return handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    debugPrint('<-- HTTP ERROR [${err.response?.statusCode}] ${err.requestOptions.uri}');
    debugPrint('Message: ${err.message}');
    return handler.next(err);
  }
}
```

#### 3. `ErrorMappingInterceptor`: Chuẩn hóa ngoại lệ mạng
Chuyển đổi các mã lỗi HTTP thô thành các lớp ngoại lệ nghiệp vụ độc lập với thư viện:

```dart
import 'package:dio/dio.dart';

class ErrorMappingInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final AppException appException = _mapDioExceptionToAppException(err);

    return handler.reject(
      DioException(
        requestOptions: err.requestOptions,
        response: err.response,
        type: err.type,
        error: appException,
        message: appException.message,
      ),
    );
  }

  AppException _mapDioExceptionToAppException(DioException err) {
    switch (err.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        return const TimeoutException('Kết nối quá thời gian chờ quy định.');

      case DioExceptionType.connectionError:
        return const NoInternetException('Không thể kết nối đến máy chủ.');

      case DioExceptionType.badResponse:
        final int statusCode = err.response?.statusCode ?? 0;
        final dynamic data = err.response?.data;
        final String serverMessage = (data is Map && data['message'] != null)
            ? data['message'].toString()
            : 'Yêu cầu không thể thực hiện (Mã lỗi: $statusCode).';

        if (statusCode == 400) return BadRequestException(serverMessage);
        if (statusCode == 401) return UnauthorizedException(serverMessage);
        if (statusCode == 403) return ForbiddenException(serverMessage);
        if (statusCode == 404) return NotFoundException(serverMessage);
        if (statusCode >= 500) return ServerException('Lỗi máy chủ ($statusCode).');
        return UnknownNetworkException(serverMessage);

      case DioExceptionType.cancel:
        return const RequestCancelledException('Yêu cầu đã bị hủy.');

      default:
        return UnknownNetworkException(err.message ?? 'Đã xảy ra lỗi không xác định.');
    }
  }
}

// Hệ thống ngoại lệ độc lập với UI
abstract class AppException implements Exception {
  final String message;
  const AppException(this.message);
  @override
  String toString() => message;
}

class TimeoutException extends AppException { const TimeoutException(super.message); }
class NoInternetException extends AppException { const NoInternetException(super.message); }
class BadRequestException extends AppException { const BadRequestException(super.message); }
class UnauthorizedException extends AppException { const UnauthorizedException(super.message); }
class ForbiddenException extends AppException { const ForbiddenException(super.message); }
class NotFoundException extends AppException { const NotFoundException(super.message); }
class ServerException extends AppException { const ServerException(super.message); }
class RequestCancelledException extends AppException { const RequestCancelledException(super.message); }
class UnknownNetworkException extends AppException { const UnknownNetworkException(super.message); }
```

---

### 3.4 — Xử Lý Làm Mới Token Đồng Thời Bằng `QueuedInterceptor`

Khi người dùng mở một màn hình kích hoạt đồng thời nhiều yêu cầu API (ví dụ: thông tin người dùng, thông báo, danh mục) và Token xác thực đã hết hạn:
- Cả 3-4 yêu cầu đều nhận phản hồi `401 Unauthorized` gần như cùng lúc.
- Nếu sử dụng `Interceptor` thông thường, mỗi yêu cầu sẽ độc lập gọi API làm mới Token. Kết quả là có nhiều yêu cầu refresh token được gửi lên đồng thời, gây xung đột phiên xác thực (Race Condition) trên máy chủ.

#### Giải pháp: Sử dụng `QueuedInterceptor`
Theo tài liệu của `dio`, `QueuedInterceptor` tích hợp hàng đợi tuần tự. Khi một yêu cầu đang thực hiện tác vụ bất đồng bộ bên trong `onError`, các yêu cầu khác cùng đi vào interceptor sẽ được giữ lại trong hàng đợi cho đến khi yêu cầu đầu tiên hoàn tất:

```dart
import 'package:dio/dio.dart';

class TokenRefreshInterceptor extends QueuedInterceptor {
  final Dio _dioClient;

  TokenRefreshInterceptor(this._dioClient);

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    // Kiểm tra mã lỗi 401 và loại trừ chính endpoint refresh token
    if (err.response?.statusCode == 401 && !err.requestOptions.path.contains('/auth/refresh')) {
      try {
        // Thực hiện làm mới token (các request khác được giữ trong hàng đợi)
        final String? newAccessToken = await _performRefreshToken();

        if (newAccessToken != null) {
          // Cập nhật token mới vào header của yêu cầu ban đầu
          final RequestOptions requestOptions = err.requestOptions;
          requestOptions.headers['Authorization'] = 'Bearer $newAccessToken';

          // Gửi lại (retry) yêu cầu gốc với token mới
          final Response retryResponse = await _dioClient.fetch(requestOptions);

          // Trả kết quả thành công và mở khóa hàng đợi cho các yêu cầu tiếp theo
          return handler.resolve(retryResponse);
        }
      } catch (refreshError) {
        // Nếu refresh thất bại, chuyển người dùng về trạng thái đăng xuất
        await _forceLogout();
        return handler.reject(err);
      }
    }

    return handler.next(err);
  }

  Future<String?> _performRefreshToken() async {
    // Sử dụng instance Dio độc lập để tránh vòng lặp interceptor
    final Dio refreshDio = Dio(BaseOptions(baseUrl: 'https://api.example.com/api/v1'));
    final response = await refreshDio.post('/auth/refresh', data: {
      'refreshToken': 'stored_refresh_token',
    });

    if (response.statusCode == 200) {
      return response.data['accessToken'] as String;
    }
    return null;
  }

  Future<void> _forceLogout() async {
    // Xóa phiên làm việc lưu trong máy
  }
}
```

---

### 3.5 — Tải Lên Tệp Tin (Multipart Upload) & Theo Dõi Tiến Trình

Tài liệu `dio` hỗ trợ gửi dữ liệu nhị phân thông qua `FormData` và `MultipartFile`:

```dart
import 'package:dio/dio.dart';

class MediaUploadService {
  final Dio _dio;

  MediaUploadService(this._dio);

  Future<String> uploadAvatar({
    required String filePath,
    required void Function(int sentBytes, int totalBytes) onProgress,
    CancelToken? cancelToken,
  }) async {
    final MultipartFile file = await MultipartFile.fromFile(
      filePath,
      filename: 'user_avatar.jpg',
    );

    final FormData formData = FormData.fromMap({
      'file': file,
      'uploadType': 'avatar',
    });

    final Response response = await _dio.post(
      '/users/avatar',
      data: formData,
      cancelToken: cancelToken,
      onSendProgress: (int sent, int total) {
        if (total > 0) {
          onProgress(sent, total);
        }
      },
    );

    return response.data['avatarUrl'] as String;
  }
}
```

---

### 3.6 — Áp Dụng Repository Pattern Với Dio

Tầng Repository đóng vai trò là ranh giới giữa thư viện mạng và các tầng nghiệp vụ bên trong ứng dụng:

```dart
// 1. Giao diện trừu tượng thuộc Domain Layer
abstract interface class UserRepository {
  Future<UserProfile> getUserProfile(String userId, {CancelToken? cancelToken});
  Future<void> updateDisplayName(String userId, String newName);
}

// 2. Model nghiệp vụ bất biến
class UserProfile {
  final String id;
  final String displayName;
  final String email;

  const UserProfile({required this.id, required this.displayName, required this.email});

  factory UserProfile.fromJson(Map<String, dynamic> json) {
    return UserProfile(
      id: json['id'] as String,
      displayName: json['displayName'] as String,
      email: json['email'] as String,
    );
  }
}

// 3. Lớp hiện thực hóa thuộc Data Layer
class RemoteUserRepository implements UserRepository {
  final Dio _dio;

  const RemoteUserRepository(this._dio);

  @override
  Future<UserProfile> getUserProfile(String userId, {CancelToken? cancelToken}) async {
    try {
      final Response response = await _dio.get(
        '/users/$userId',
        cancelToken: cancelToken,
      );
      return UserProfile.fromJson(response.data as Map<String, dynamic>);
    } on DioException catch (e) {
      if (e.error is AppException) {
        throw e.error as AppException;
      }
      throw UnknownNetworkException(e.message ?? 'Lỗi không xác định.');
    }
  }

  @override
  Future<void> updateDisplayName(String userId, String newName) async {
    await _dio.patch('/users/$userId', data: {'displayName': newName});
  }
}
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Khởi tạo instance `Dio` mới ở mỗi lời gọi hàm

#### Mô tả vấn đề:
Khởi tạo `final dio = Dio();` trực tiếp bên trong từng phương thức gửi dữ liệu:

```dart
// Không khuyến nghị: Tạo mới instance ở mỗi yêu cầu
Future<List<Item>> fetchItems() async {
  final dio = Dio(); 
  final res = await dio.get('https://api.example.com/items');
  return parseItems(res.data);
}
```

#### Nguyên nhân kỹ thuật:
- Mất toàn bộ các Interceptors đã cấu hình (thiếu Auth Token, không ghi log, không chuyển đổi lỗi).
- Mỗi đối tượng `Dio` mới sẽ khởi tạo một `HttpClient` riêng biệt, không thể tái sử dụng kết nối TCP (Keep-Alive), làm tăng độ trễ cho mỗi yêu cầu và tiêu hao tài nguyên kết nối socket.

#### Biện pháp khắc phục:
Quản lý `Dio` như một Singleton hoặc thông qua hệ thống tiêm phụ thuộc (Dependency Injection như `get_it`).

---

### 4.2 — Để lọt ngoại lệ thư viện (`DioException`) lên tầng giao diện (UI Layer)

#### Mô tả vấn đề:
Tầng UI bắt trực tiếp ngoại lệ của thư viện `Dio`:

```dart
// Không khuyến nghị: UI phụ thuộc trực tiếp vào package:dio
try {
  await userRepository.getUserProfile(id);
} on DioException catch (e) {
  if (e.response?.statusCode == 404) {
    showSnackbar('Không tìm thấy tài khoản');
  }
}
```

#### Nguyên nhân kỹ thuật:
Gây ghép nối trực tiếp (tight coupling) giữa giao diện người dùng và thư viện bên thứ ba. Khi thay đổi hoặc nâng cấp thư viện mạng, các màn hình UI sẽ phải sửa đổi theo.

#### Biện pháp khắc phục:
Tầng Repository hoặc Interceptor chịu trách nhiệm bắt `DioException` và chuyển đổi thành `AppException` trước khi ném ra ngoài tầng giao diện.

---

### 4.3 — Không cấu hình thời gian chờ (Timeouts)

#### Mô tả vấn đề:
Khởi tạo Client mà không chỉ định thời gian chờ tối đa cho các kết nối.

#### Nguyên nhân kỹ thuật:
Khi thiết bị di chuyển vào vùng phủ sóng yếu hoặc mạng bị mất gói tin ngầm (Packet Loss), socket có thể rơi vào trạng thái chờ kéo dài mà không nhận được tín hiệu ngắt kết nối. Giao diện người dùng sẽ hiển thị trạng thái chờ tải liên tục mà không có thông báo lỗi.

#### Biện pháp khắc phục:
Thiết lập tường minh các giá trị thời gian chờ: `connectTimeout` ($5\text{s} - 10\text{s}$), `sendTimeout` ($10\text{s}$), `receiveTimeout` ($15\text{s} - 30\text{s}$).

---

### 4.4 — Cập nhật trạng thái sau khi Widget đã bị Unmounted

#### Mô tả vấn đề:
Gọi `setState()` sau khi hoàn thành một tác vụ bất đồng bộ mà không kiểm tra xem widget còn nằm trên cây giao diện hay không:

```dart
// Lỗi runtime: setState() called after dispose()
void _loadData() async {
  final data = await _repository.fetchData();
  setState(() => _data = data); 
}
```

#### Biện pháp khắc phục:
1. Thêm điều kiện bảo vệ `if (!mounted) return;` trước khi gọi `setState()`.
2. Sử dụng `CancelToken` để hủy yêu cầu mạng ngay trong hàm `dispose()` của `State`:
   ```dart
   @override
   void dispose() {
     _cancelToken.cancel('Widget disposed');
     super.dispose();
   }
   ```

---

## Phần 5 — Khảo Sát Kỹ Thuật Chuyên Sâu & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Sự khác biệt về cơ chế thực thi giữa `Interceptor` thông thường và `QueuedInterceptor` trong thư viện Dio?
*Phân tích:*
- **`Interceptor` thông thường**: Xử lý bất đồng bộ đồng thời (Concurrent Execution). Khi có nhiều yêu cầu gửi đi cùng lúc, các callback `onRequest`, `onResponse`, `onError` của từng yêu cầu sẽ chạy song song và độc lập.
- **`QueuedInterceptor`**: Quản lý một hàng đợi nội bộ tuần tự (Sequential Queue). Khi một yêu cầu đang xử lý bất đồng bộ trong callback (ví dụ đang chờ API Refresh Token phản hồi), `QueuedInterceptor` sẽ tạm dừng các yêu cầu tiếp theo trong hàng đợi cho đến khi yêu cầu đầu tiên gọi `handler.resolve()` hoặc `handler.next()`.

---

#### Câu hỏi 2: Cơ chế `CancelToken` hoạt động như thế nào ở tầng Socket của Dart VM?
*Phân tích:*
- Một `CancelToken` lưu trữ một `Completer<DioException>`.
- Khi truyền `CancelToken` vào phương thức gửi dữ liệu của Dio, adapter bên dưới (`IOHttpClientAdapter`) đăng ký lắng nghe sự kiện từ token này.
- Khi gọi `cancelToken.cancel()`, listener kích hoạt và gọi trực tiếp `abort()` trên đối tượng `HttpClientRequest` của `dart:io`. Thao tác này gửi tín hiệu ngắt socket xuống hệ điều hành, đóng kết nối ngay lập tức mà không cần chờ đọc nốt các byte dữ liệu còn lại từ máy chủ.

---

#### Câu hỏi 3: Tại sao việc tái sử dụng một instance Client duy nhất giúp giảm độ trễ mạng?
*Phân tích:*
- Khi giao tiếp qua HTTPS, mỗi kết nối mới yêu cầu thực hiện phân giải DNS, bắt tay 3 bước TCP và bắt tay mã hóa TLS. Quá trình này thường tốn từ $100\text{ms} - 300\text{ms}$.
- Khi tái sử dụng cùng một instance `Client` hoặc `Dio`, cơ chế HTTP Keep-Alive được kích hoạt. Kết nối TCP đã mở sẽ được duy trì trong một khoảng thời gian chờ. Các yêu cầu tiếp theo gửi tới cùng một máy chủ sẽ truyền trực tiếp trên kết nối đã có, loại bỏ hoàn toàn thời gian bắt tay TCP/TLS ban đầu.

---

#### Câu hỏi 4: Cách thiết kế hệ thống xử lý lỗi mạng độc lập với thư viện bên thứ ba?
*Phân tích:*
- Định nghĩa hệ thống lớp lỗi thuần khiết trong tầng Domain (ví dụ: `abstract class AppException`).
- Sử dụng `ErrorInterceptor` của `Dio` hoặc lớp bọc ngoại lệ trong Repository để chuyển đổi các mã HTTP Status Code và `DioExceptionType` thành các thể hiện của `AppException`.
- Tầng UI và tầng Business Logic chỉ bắt và xử lý `AppException`, hoàn toàn không tham chiếu tới `package:dio`.

---

#### Câu hỏi 5: Vai trò của điều kiện `if (!mounted) return;` đối với vòng đời State?
*Phân tích:*
- Cờ `mounted` trong `StatefulWidget` có giá trị `true` sau khi `initState()` hoàn thành và chuyển sang `false` khi `dispose()` được thực thi.
- Một tác vụ mạng bất đồng bộ giữ tham chiếu closure tới instance `State`. Nếu phản hồi trả về sau khi widget đã bị tháo khỏi cây giao diện, việc gọi `setState()` sẽ gây lỗi runtime vì `Element` tương ứng không còn tồn tại trên Render Tree.
- Điều kiện `if (!mounted) return;` kiểm tra xem widget còn hiển thị hay không trước khi thực hiện cập nhật bộ nhớ.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho đoạn mã xử lý với `QueuedInterceptor` như sau:

```dart
class DemoAuthInterceptor extends QueuedInterceptor {
  int refreshCount = 0;

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      debugPrint('Đang xử lý 401 cho: ${err.requestOptions.path}');
      refreshCount++;
      
      // Giả lập thời gian làm mới Token là 200ms
      await Future.delayed(const Duration(milliseconds: 200));
      debugPrint('Đã làm mới token lần thứ $refreshCount');

      err.requestOptions.headers['token'] = 'token_v$refreshCount';
      final response = await fakeRetry(err.requestOptions);
      return handler.resolve(response);
    }
    return handler.next(err);
  }
}
```

Giả sử tại thời điểm $t = 0\text{ms}$, ứng dụng gửi đồng thời 3 yêu cầu:
- Yêu cầu A: `GET /profile`
- Yêu cầu B: `GET /notifications`
- Yêu cầu C: `GET /settings`

Cả 3 yêu cầu đều nhận phản hồi `401 Unauthorized` tại thời điểm $t = 50\text{ms}$.

#### Phân tích luồng thực thi:

1. **Thứ tự thực thi:**
   - Tại $t = 50\text{ms}$, cả 3 yêu cầu gặp lỗi 401. Yêu cầu A đi vào `onError` đầu tiên và khóa hàng đợi.
   - Yêu cầu B và C được giữ lại trong hàng đợi của `QueuedInterceptor`.
   - Yêu cầu A thực thi khối mã: in log `Đang xử lý 401 cho: /profile`, tăng `refreshCount` lên 1, chờ 200ms.
   - Tại $t = 250\text{ms}$, yêu cầu A hoàn tất làm mới token, in log `Đã làm mới token lần thứ 1`, thực hiện retry và gọi `handler.resolve(response)`.
   - Sau khi yêu cầu A hoàn tất, hàng đợi mở khóa và chuyển tiếp yêu cầu B xử lý tuần tự.

2. **So sánh với `Interceptor` thông thường:**
   - Nếu thay bằng `Interceptor` thông thường, không có hàng đợi tuần tự.
   - Cả 3 yêu cầu A, B, C sẽ cùng thực thi hàm `onError` đồng thời tại $t = 50\text{ms}$.
   - Cả 3 yêu cầu sẽ gửi 3 yêu cầu làm mới token riêng biệt lên máy chủ, gây xung đột phiên xác thực và có thể làm vô hiệu hóa token của nhau trên hệ thống backend.
