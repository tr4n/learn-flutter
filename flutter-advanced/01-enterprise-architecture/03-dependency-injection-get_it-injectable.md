# Bài 1.3: Dependency Injection — get_it + injectable Production

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Đã đọc Bài 1.1 và 1.2; hiểu khái niệm Inversion of Control

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: DI Initialization Race Condition

**Dự án thực tế — E-commerce app, team 8 người:**

```
[FATAL] Bad state: GetIt: Object/factory for type AuthRepository
        not registered inside GetIt. (Only registered if app 
        is not in test mode)
        
Xảy ra: 3% lần khởi động app trên thiết bị Android cũ (RAM < 3GB)
```

**Nguyên nhân**:

```dart
// ❌ Anti-pattern: Đăng ký DI không đảm bảo thứ tự
void main() {
  runApp(const MyApp()); // Flutter UI chạy ngay
}

class MyApp extends StatefulWidget {
  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  @override
  void initState() {
    super.initState();
    configureDependencies(); // async — nhưng Widget đã build!
    //         ↑ Race: Router có thể resolve AuthBloc trước khi 
    //           AuthRepository được đăng ký
  }
}
```

**Giải pháp đúng**: DI phải được khởi tạo **đồng bộ trước** khi `runApp()`.

---

## Phần 2 — Low-Level Mechanics

### 2.1. get_it — Service Locator vs DI Container

```
Service Locator (get_it):
┌─────────────────────────────────────────────────────────┐
│                   GetIt Registry                         │
│  Map<Type, Object/Factory>                              │
│  ┌──────────────────┬────────────────────────────────┐  │
│  │ AuthRepository   │ → AuthRepositoryImpl (singleton)│  │
│  │ CartBloc         │ → () => CartBloc(getIt())       │  │
│  │ NetworkClient    │ → DioClient (lazy singleton)    │  │
│  │ AnalyticsService │ → FirebaseAnalytics (singleton) │  │
│  └──────────────────┴────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

getIt<AuthRepository>() → lookup bằng Type key → O(1)
```

**Phân biệt 4 lifecycle:**

```
Singleton:      Tạo NGAY khi đăng ký, tồn tại suốt app
                → Network client, Analytics

LazySingleton:  Tạo lần ĐẦU tiên khi được resolve, tồn tại suốt app
                → Repository, DataSource (lazy = không tốn resource khi không dùng)

Factory:        Tạo instance MỚI mỗi lần resolve
                → BLoC, Cubit (mỗi màn hình cần instance riêng)

FactoryParam:   Factory nhưng nhận parameter
                → ProductDetailBloc(productId: 'abc')

                Time ──────────────────────────────────────►
Singleton       ▓ (created)─────────────────────────────────
LazySingleton   ░░░░░░░░░ (created on first call)───────────
Factory         ░░░░░░░░░░░░░░░░░░░░ ▓(call1) ▓(call2) ▓...
FactoryParam    ░░░░░░░░░░░░░░░░░░░░ ▓(p1) ▓(p2) ▓(p3) ▓...
```

### 2.2. Scope Management — Quan trọng cho Multi-Screen App

```
App Scope (Default):
  AuthRepository, NetworkClient, ThemeService
  → Tồn tại toàn bộ vòng đời app
  
Session Scope (pushNewScope khi user login):
  UserProfileCache, CartSyncService, NotificationManager
  → Tự động giải phóng khi user logout (popScope)
  
Screen Scope (pushNewScope khi vào màn hình):
  FormValidationService, DraftSaver
  → Tự động giải phóng khi pop màn hình
  
Flow Scope (checkout flow):
  CheckoutCoordinator, PaymentSession
  → Giải phóng sau khi checkout hoàn tất hoặc user thoát
```

### 2.3. injectable Code Generation — Cách hoạt động

```dart
// Bạn viết:
@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  AuthRepositoryImpl(this._remoteSource, this._localSource);
  final AuthRemoteDataSource _remoteSource;
  final AuthLocalDataSource _localSource;
}

// injectable generate ra:
// injection.config.dart (KHÔNG cần viết tay)
extension GetItInjectableX on GetIt {
  GetIt init() {
    getIt
      ..registerLazySingleton<AuthRemoteDataSource>(
        () => AuthRemoteDataSourceImpl(getIt<DioClient>()),
      )
      ..registerLazySingleton<AuthLocalDataSource>(
        () => AuthLocalDataSourceImpl(getIt<HiveBox>()),
      )
      ..registerLazySingleton<AuthRepository>(
        () => AuthRepositoryImpl(
          getIt<AuthRemoteDataSource>(),
          getIt<AuthLocalDataSource>(),
        ),
      );
    return this;
  }
}
```

---

## Phần 3 — Production Code Implementation

### 3.1. Setup get_it + injectable đúng chuẩn

```yaml
# pubspec.yaml
dependencies:
  get_it: ^8.0.3
  injectable: ^2.5.0

dev_dependencies:
  injectable_generator: ^2.7.0
  build_runner: ^2.4.13
```

```dart
// lib/core/di/injection.dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';

import 'injection.config.dart'; // Generated file

// Global accessor — chỉ dùng ở entry point và test setup
// KHÔNG inject getIt vào Widget trực tiếp
final getIt = GetIt.instance;

@InjectableInit(
  initializerName: 'init', // Tên method được generate
  preferRelativeImports: true,
  asExtension: true,
)
Future<void> configureDependencies({String? environment}) async {
  // environment: 'production', 'development', 'test'
  await getIt.init(environment: environment);
}
```

```dart
// lib/main.dart — ĐÚNG CÁCH: DI trước runApp
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Khởi tạo các platform service cần thiết trước
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  // DI phải complete TRƯỚC khi runApp
  await configureDependencies(environment: kReleaseMode ? 'production' : 'development');
  
  runApp(const App());
}
```

### 3.2. Module Pattern — Mỗi Feature tự đăng ký DI

```dart
// features/auth/di/auth_module.dart
// @module đánh dấu class này là DI module
@module
abstract class AuthModule {
  // External dependency không có constructor inject được
  // → Dùng @singleton + getter để đăng ký thủ công
  
  @singleton
  Dio get dio => DioFactory.create(
        baseUrl: Env.apiBaseUrl,
        connectTimeout: const Duration(seconds: 30),
      );

  @lazySingleton
  HiveInterface get hive => Hive;
}

// features/auth/data/datasources/auth_remote_datasource.dart
@LazySingleton(as: AuthRemoteDataSource)
final class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  // get_it tự inject Dio vì đã đăng ký ở AuthModule
  const AuthRemoteDataSourceImpl(this._dio);
  final Dio _dio;
  
  @override
  Future<UserDto> signIn({
    required String email, 
    required String password,
  }) async {
    final response = await _dio.post('/auth/signin', data: {
      'email': email,
      'password': password,
    });
    return UserDto.fromJson(response.data as Map<String, dynamic>);
  }
}

// features/auth/domain/repositories/auth_repository.dart
// Interface — không có @injectable annotation
abstract interface class AuthRepository {
  Future<Result<User, Failure>> signIn({
    required String email,
    required String password,
  });
  
  Future<Result<void, Failure>> signOut();
  Stream<User?> watchCurrentUser();
}

// features/auth/data/repositories/auth_repository_impl.dart
@LazySingleton(as: AuthRepository) // Bind interface → implementation
final class AuthRepositoryImpl implements AuthRepository {
  const AuthRepositoryImpl(
    this._remoteSource,
    this._localSource,
    this._tokenManager, // Injectable dependency
  );
  
  final AuthRemoteDataSource _remoteSource;
  final AuthLocalDataSource _localSource;
  final TokenManager _tokenManager;
  
  @override
  Future<Result<User, Failure>> signIn({
    required String email,
    required String password,
  }) async {
    try {
      final dto = await _remoteSource.signIn(email: email, password: password);
      await _tokenManager.saveTokens(
        accessToken: dto.accessToken,
        refreshToken: dto.refreshToken,
      );
      final user = dto.toEntity();
      await _localSource.cacheUser(dto);
      return Success(user);
    } on DioException catch (e) {
      return Failure_(_mapDioError(e));
    }
  }
  
  Failure _mapDioError(DioException e) {
    return switch (e.type) {
      DioExceptionType.connectionTimeout ||
      DioExceptionType.receiveTimeout =>
        const TimeoutFailure(),
      DioExceptionType.connectionError =>
        const NetworkFailure(),
      DioExceptionType.badResponse when e.response?.statusCode == 401 =>
        const UnauthorizedFailure(),
      DioExceptionType.badResponse =>
        ServerFailure(
          message: e.response?.data?['message'] as String? ?? 'Lỗi server',
          code: e.response?.statusCode?.toString(),
        ),
      _ => const NetworkFailure(),
    };
  }
}
```

### 3.3. Scope Management Production Pattern

```dart
// lib/core/di/scope_manager.dart
// Quản lý scope tập trung — tránh pushNewScope/popScope rải rác
final class ScopeManager {
  ScopeManager(this._getIt);
  final GetIt _getIt;
  
  // Gọi khi user đăng nhập thành công
  Future<void> openUserSession(String userId) async {
    await _getIt.pushNewScopeAsync(
      scopeName: 'user_session',
      init: (scope) async {
        scope
          ..registerLazySingleton<UserProfileCache>(
            () => UserProfileCache(userId: userId),
          )
          ..registerLazySingleton<NotificationManager>(
            () => NotificationManager(
              userId: userId,
              fcmToken: scope<FcmTokenManager>().currentToken,
            ),
          )
          ..registerLazySingleton<CartSyncService>(
            () => CartSyncService(
              repository: scope<CartRepository>(),
              userId: userId,
            ),
          );
      },
    );
  }
  
  // Gọi khi user đăng xuất — tự động dispose tất cả singleton trong scope
  Future<void> closeUserSession() async {
    if (_getIt.hasScope('user_session')) {
      await _getIt.popScopesTill('user_session');
    }
  }
  
  // Scope cho checkout flow — tự động cleanup khi hoàn tất
  Future<T> withinCheckoutScope<T>(Future<T> Function() action) async {
    await _getIt.pushNewScopeAsync(
      scopeName: 'checkout',
      init: (scope) async {
        scope
          ..registerFactory<CheckoutCoordinator>(CheckoutCoordinator.new)
          ..registerFactory<PaymentSessionManager>(PaymentSessionManager.new);
      },
    );
    
    try {
      return await action();
    } finally {
      await _getIt.popScopesTill('checkout');
    }
  }
}
```

### 3.4. Factory với Parameter — BLoC Production

```dart
// Đăng ký BLoC với parameter
// features/product/di/product_module.dart
@module
abstract class ProductModule {
  // FactoryParam cho BLoC cần productId khi khởi tạo
  @factoryParam // Annotation cho phép inject 1 param
  ProductDetailBloc productDetailBloc(String productId) =>
      ProductDetailBloc(
        productId: productId,
        useCase: getIt<GetProductDetailUseCase>(),
        analyticsService: getIt<AnalyticsService>(),
      );
}

// Sử dụng trong Widget
class ProductDetailScreen extends StatelessWidget {
  const ProductDetailScreen({super.key, required this.productId});
  final String productId;
  
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      // Factory tạo instance mới với productId cụ thể
      create: (_) => getIt<ProductDetailBloc>(param1: productId),
      child: const ProductDetailView(),
    );
  }
}
```

### 3.5. Cấu hình môi trường (Environment)

```dart
// Đánh dấu implementation theo môi trường
@LazySingleton(as: AnalyticsService, env: ['production'])
final class FirebaseAnalyticsService implements AnalyticsService {
  @override
  Future<void> logEvent(String name, Map<String, Object> params) async {
    await FirebaseAnalytics.instance.logEvent(name: name, parameters: params);
  }
}

@LazySingleton(as: AnalyticsService, env: ['development', 'test'])
final class NoopAnalyticsService implements AnalyticsService {
  @override
  Future<void> logEvent(String name, Map<String, Object> params) async {
    debugPrint('[Analytics DEV] $name: $params');
    // Không gửi event thật trong dev/test
  }
}

// Trong main.dart:
await configureDependencies(
  environment: kReleaseMode ? 'production' : 'development',
);
```

### 3.6. Test Setup — Swap implementation dễ dàng

```dart
// test/helpers/test_injection.dart
Future<void> configureTestDependencies() async {
  // Reset GetIt trước mỗi test suite
  await getIt.reset();
  
  getIt
    // Swap real implementation bằng mock — không sửa production code
    ..registerLazySingleton<AuthRepository>(
      () => MockAuthRepository(),
    )
    ..registerLazySingleton<CartRepository>(
      () => FakeCartRepository(), // Fake implementation with in-memory storage
    )
    ..registerFactory<SignInUseCase>(
      () => SignInUseCase(getIt()),
    );
}

// Trong test file:
void main() {
  setUp(() async {
    await configureTestDependencies();
  });
  
  tearDown(() async {
    await getIt.reset();
  });
  
  test('SignInUseCase returns Success when credentials are valid', () async {
    // MockAuthRepository đã được inject — không cần truyền thủ công
    final useCase = getIt<SignInUseCase>();
    // ...
  });
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Đo lường: Singleton vs Factory vs LazySingleton

```
Benchmark: 10,000 lần resolve cùng type

getIt<NetworkClient>() [Singleton]
→ avg: 0.12 μs/op (HashMap lookup chỉ)

getIt<CartRepository>() [LazySingleton — sau lần đầu]
→ avg: 0.14 μs/op (HashMap lookup + null check)

getIt<ProductDetailBloc>() [Factory]  
→ avg: 2.8 μs/op (object instantiation + dependency resolution)
→ Chấp nhận được cho UI interaction (không phải hot path)

KHUYẾN NGHỊ:
- Repository, DataSource, Service: LazySingleton
- BLoC, Cubit (per-screen state): Factory
- Network client, DB: Singleton (khởi tạo sớm, dùng suốt)
```

### Memory Impact của Scope

```
Scenario: User login → sử dụng app 1 giờ → logout → login lại

Không dùng Scope:
  UserProfileCache tồn tại suốt app, kể cả sau logout
  → Memory leak: ~2.4MB per cache instance
  → Dữ liệu user cũ có thể bị dùng bởi user mới

Dùng User Session Scope:
  closeUserSession() → popScope → dispose tất cả singleton trong scope
  → Memory được giải phóng ngay lập tức
  → Zero cross-user data contamination
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ configureDependencies() gọi bên trong StatefulWidget.initState()
    ✅ Gọi trước runApp() trong main() — đảm bảo DI complete trước UI
    Lý do: Race condition → getIt resolve trước khi register → crash 3-5% users

[ ] ❌ getIt<MyBloc>() gọi trực tiếp trong build() method của Widget
    ✅ Dùng BlocProvider(create: (_) => getIt<MyBloc>()) hoặc context.read<>()
    Lý do: Mỗi lần rebuild tạo instance mới nếu là Factory → state bị reset

[ ] ❌ Đăng ký ConcreteClass thay vì Interface:
    getIt.registerLazySingleton<AuthRepositoryImpl>(...)
    ✅ Luôn đăng ký bằng Interface:
    getIt.registerLazySingleton<AuthRepository>(() => AuthRepositoryImpl(...))
    Lý do: Không thể swap implementation trong test

[ ] ❌ getIt.reset() không được gọi trong tearDown() của test
    ✅ Luôn reset GetIt sau mỗi test/test group để tránh state leak giữa tests
    Lý do: Singleton từ test trước ảnh hưởng test sau → kết quả không đáng tin

[ ] ❌ Không có scopeName khi pushNewScope
    ✅ Luôn đặt tên scope: pushNewScope(scopeName: 'user_session')
    Lý do: Không thể popScopesTill() đúng scope khi không có tên

[ ] ❌ Đặt toàn bộ DI registration trong 1 file injection.dart
    ✅ Mỗi feature có module riêng, injection.dart chỉ là assembler
    Lý do: Hot file conflict mỗi sprint khi team lớn

[ ] ❌ @singleton cho BLoC/Cubit
    ✅ @factory cho BLoC/Cubit — mỗi màn hình cần instance riêng biệt
    Lý do: Singleton BLoC chia sẻ state giữa các màn hình → bug không lường trước

[ ] ❌ Chạy build_runner thủ công sau mỗi thay đổi annotation
    ✅ Tích hợp build_runner vào CI: dart run build_runner build --delete-conflicting-outputs
    Lý do: Generated file out-of-date → runtime crash khi deploy
```
