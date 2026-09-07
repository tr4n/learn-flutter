# Bài 7.3: Integration Testing E2E với integration_test

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Đã biết widget testing; hiểu CI/CD pipeline

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Critical flow cần E2E test

**E-commerce app — Bug production:**

```
Bug report (P0): User checkout xong nhưng:
  - Order không tạo được (server timeout)
  - Tiền đã bị charge
  - Cart không được clear
  - User không nhận được error message

Root cause: Race condition giữa payment callback và order creation
→ Không có E2E test → không phát hiện trước production

Sau khi thêm E2E test:
  test flow: login → add to cart → checkout → verify order created
  Run trên Android emulator + iOS simulator trong CI
  → Phát hiện ngay khi developer commit code gây ra race condition
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. integration_test vs flutter_test

```
flutter_test (unit + widget test):
  - Chạy trong isolated test environment
  - Không có real platform (Android/iOS)
  - Mock network, mock platform APIs
  - Fast: <5ms per test
  - Không test real device behavior

integration_test:
  - Chạy ON real device hoặc emulator
  - Real Flutter engine + real platform
  - Real camera, real network (hoặc mock)
  - Real gestures (pointer events)
  - Slow: 30-120s per flow
  - Test real user behavior end-to-end

Khi nào dùng integration_test:
  ✓ Critical user flows (login, checkout, payment)
  ✓ Complex multi-screen flows
  ✓ Platform-specific behavior (camera, biometric, notifications)
  ✗ Business logic (dùng unit test)
  ✗ UI component (dùng widget test)
```

### 2.2. App Launch trong integration_test

```dart
// integration_test app setup phải gọi integration_test binding
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  //                                  ↑
  // Replace FlutterTest binding bằng binding cho on-device test
  // Cho phép screenshot, drive từ external test runner
  
  testWidgets('checkout flow', (tester) async {
    // App phải được start từ đầu — real app instance
    app.main(); // gọi main() của app thật
    await tester.pumpAndSettle(); // Đợi app fully loaded
    
    // Sau đó interact như user thật
  });
}
```

---

## Phần 3 — Production Code Implementation

### 3.1. E2E Test: Login → Add to Cart → Checkout Flow

```dart
// integration_test/checkout_flow_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

import '../lib/main.dart' as app;
import 'helpers/test_helpers.dart';
import 'page_objects/login_page.dart';
import 'page_objects/home_page.dart';
import 'page_objects/cart_page.dart';
import 'page_objects/checkout_page.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Checkout Flow', () {
    testWidgets(
      'user can complete checkout successfully',
      (tester) async {
        // Launch real app với test configuration
        app.main(environment: AppEnvironment.test);
        await tester.pumpAndSettle(const Duration(seconds: 3));

        // Page Object Pattern — không hardcode finders trong test
        final loginPage = LoginPage(tester);
        final homePage = HomePage(tester);
        final cartPage = CartPage(tester);
        final checkoutPage = CheckoutPage(tester);

        // Step 1: Login
        await loginPage.fillEmail('test@example.com');
        await loginPage.fillPassword('Test@123456');
        await loginPage.tapLogin();
        await tester.pumpAndSettle(const Duration(seconds: 3));
        
        // Verify: Đã vào home screen
        expect(loginPage.isVisible, isFalse);
        expect(homePage.isVisible, isTrue);

        // Step 2: Add product to cart
        await homePage.scrollToFirstProduct();
        await homePage.tapAddToCart();
        await tester.pumpAndSettle();
        
        // Verify: Cart badge hiển thị 1
        expect(homePage.cartBadgeCount, equals('1'));

        // Step 3: Go to cart
        await homePage.tapCartTab();
        await tester.pumpAndSettle();
        expect(cartPage.isVisible, isTrue);
        expect(cartPage.itemCount, equals(1));

        // Step 4: Proceed to checkout
        await cartPage.tapCheckout();
        await tester.pumpAndSettle();
        expect(checkoutPage.isVisible, isTrue);

        // Step 5: Fill delivery address
        await checkoutPage.fillDeliveryAddress(
          street: '123 Nguyễn Văn Linh',
          district: 'Quận 7',
          city: 'Hồ Chí Minh',
        );

        // Step 6: Select payment method
        await checkoutPage.selectPaymentMethod(PaymentMethod.creditCard);
        await checkoutPage.fillCardDetails(
          number: '4111111111111111',
          expiry: '12/26',
          cvv: '123',
        );

        // Step 7: Place order
        await checkoutPage.tapPlaceOrder();
        await tester.pumpAndSettle(const Duration(seconds: 5)); // Wait for API

        // Verify: Success screen
        expect(checkoutPage.isSuccessVisible, isTrue);
        expect(checkoutPage.orderIdVisible, isTrue);
        
        // Verify: Cart cleared
        await tester.tap(find.byKey(const Key('cart_tab')));
        await tester.pumpAndSettle();
        expect(cartPage.isEmpty, isTrue);

        // Screenshot for evidence
        await binding.takeScreenshot('checkout_success');
      },
    );

    testWidgets(
      'shows error when payment fails',
      (tester) async {
        // Dùng test user với configured payment failure
        app.main(environment: AppEnvironment.test);
        await tester.pumpAndSettle(const Duration(seconds: 3));
        
        final loginPage = LoginPage(tester);
        final checkoutPage = CheckoutPage(tester);
        
        // Login với user có payment that sẽ fail
        await loginPage.fillEmail('payment-fail@example.com');
        await loginPage.fillPassword('Test@123456');
        await loginPage.tapLogin();
        await tester.pumpAndSettle(const Duration(seconds: 3));
        
        // ... navigate to checkout ...
        
        await checkoutPage.tapPlaceOrder();
        await tester.pumpAndSettle(const Duration(seconds: 5));
        
        // Verify: Error message hiển thị, không phải success
        expect(checkoutPage.isSuccessVisible, isFalse);
        expect(checkoutPage.isErrorVisible, isTrue);
        expect(
          checkoutPage.errorMessage,
          contains('Thanh toán không thành công'),
        );
        
        // Verify: Cart KHÔNG bị clear khi payment fail
        // (Đây là critical business rule)
        // ... navigate to cart and verify ...
      },
    );
  });
}
```

### 3.2. Page Object Pattern — Tái sử dụng navigation logic

```dart
// integration_test/page_objects/login_page.dart
class LoginPage {
  const LoginPage(this.tester);
  final WidgetTester tester;

  // Finders — centralized
  Finder get _emailField => find.byKey(const Key('login_email_field'));
  Finder get _passwordField => find.byKey(const Key('login_password_field'));
  Finder get _loginButton => find.byKey(const Key('login_button'));
  Finder get _loadingIndicator => find.byKey(const Key('login_loading'));

  bool get isVisible => tester.any(_loginButton);

  Future<void> fillEmail(String email) async {
    await tester.tap(_emailField);
    await tester.enterText(_emailField, email);
    await tester.pump();
  }

  Future<void> fillPassword(String password) async {
    await tester.tap(_passwordField);
    await tester.enterText(_passwordField, password);
    await tester.pump();
  }

  Future<void> tapLogin() async {
    await tester.tap(_loginButton);
    // Đợi loading bắt đầu
    await tester.pump();
    // Đợi loading kết thúc (API call)
    await tester.pumpAndSettle(const Duration(seconds: 10));
  }
}

// integration_test/page_objects/checkout_page.dart
class CheckoutPage {
  const CheckoutPage(this.tester);
  final WidgetTester tester;

  bool get isVisible => tester.any(find.byKey(const Key('checkout_screen')));
  bool get isSuccessVisible => tester.any(find.byKey(const Key('order_success_screen')));
  bool get isErrorVisible => tester.any(find.byKey(const Key('payment_error_message')));

  String get errorMessage =>
      tester.widget<Text>(find.byKey(const Key('payment_error_message'))).data ?? '';

  Future<void> tapPlaceOrder() async {
    await tester.tap(find.byKey(const Key('place_order_button')));
    await tester.pump();
  }
  
  // ... các method khác
}
```

### 3.3. Backend Mock cho E2E test

```dart
// lib/core/network/test_interceptor.dart
// Chỉ active khi environment == AppEnvironment.test

class TestNetworkInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final mockResponse = _mockResponses[options.path];
    
    if (mockResponse != null) {
      // Return mock data thay vì gọi real API
      handler.resolve(Response(
        requestOptions: options,
        data: mockResponse,
        statusCode: 200,
      ));
      return;
    }
    
    // Không có mock → gọi real API (test environment)
    handler.next(options);
  }
  
  static final _mockResponses = {
    '/api/auth/login': {
      'token': 'test-token-123',
      'user': {'id': 'USER-001', 'name': 'Test User'},
    },
    '/api/products/featured': {
      'items': [
        {'id': 'PROD-001', 'name': 'Test Product', 'price': 100000},
      ],
    },
  };
}
```

### 3.4. CI: Firebase Test Lab Matrix

```yaml
# .github/workflows/integration_tests.yml
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.27.0'

      - name: Build Android test APK
        run: |
          flutter build apk --debug
          flutter build apk --debug --target=integration_test/checkout_flow_test.dart \
            --flavor staging \
            -t integration_test/checkout_flow_test.dart

      - name: Run on Firebase Test Lab
        run: |
          gcloud firebase test android run \
            --type instrumentation \
            --app build/app/outputs/flutter-apk/app-debug.apk \
            --test build/app/outputs/flutter-apk/app-debug-androidTest.apk \
            --device model=Pixel6,version=33,locale=vi,orientation=portrait \
            --device model=SamsungS21,version=31,locale=vi,orientation=portrait \
            --timeout 5m \
            --results-bucket gs://my-app-test-results \
            --results-dir integration-tests/${{ github.run_id }}

      - name: Download Results
        if: always()
        run: |
          gsutil -m cp -r \
            gs://my-app-test-results/integration-tests/${{ github.run_id }} \
            ./test-results/

      - name: Upload Screenshots
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: e2e-screenshots
          path: test-results/
```

---

## Phần 4 — Profiling & Performance Trade-offs

### E2E test time optimization

```
Không tối ưu (1 test file, serial):
  5 E2E tests × 90s/test = 7.5 minutes

Tối ưu:
  Parallel: 5 tests trên 2 devices = 3.75 minutes (2x)
  Mock backend: -30s/test (không cần real API latency) = 2.5 minutes
  Final: 5 tests × 60s / 2 devices = 2.5 minutes
  
Mục tiêu: Total E2E suite < 5 phút
  → Developer không bỏ qua CI vì chờ quá lâu
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Hardcode finder text trong test ('Đăng nhập', 'Thêm vào giỏ')
    ✅ Dùng Key('login_button') cho tất cả interactive elements
    Lý do: Text thay đổi theo i18n hoặc copywriting → test vỡ dù logic đúng

[ ] ❌ Không có timeout cho pumpAndSettle()
    ✅ pumpAndSettle(const Duration(seconds: 10)) cho API calls
    Lý do: Default timeout 100 frames (~1.6s) → fail sớm khi API chậm

[ ] ❌ Test depend vào real production API
    ✅ Dùng test environment với mock/stub backend
    Lý do: Real API có rate limit, data thay đổi → flaky tests

[ ] ❌ Một test case cover nhiều independent flows
    ✅ Mỗi test = 1 specific flow; setUp tạo fresh state
    Lý do: Test A fail → test B cũng fail (shared state contamination)

[ ] ❌ Không có screenshot capture khi test fail
    ✅ binding.takeScreenshot('failure_state') trong catch block
    Lý do: Không biết UI trông như thế nào khi fail → khó debug

[ ] ❌ E2E test run trên developer machine (CI flakiness)
    ✅ Chỉ run E2E trên CI với dedicated device/emulator pool
    Lý do: Developer machine background processes → non-deterministic results

[ ] ❌ E2E suite > 15 phút
    ✅ Chia thành: smoke tests (<3 phút) + full regression (<10 phút)
    Lý do: Quá chậm → developer skip CI → defeats the purpose
```
