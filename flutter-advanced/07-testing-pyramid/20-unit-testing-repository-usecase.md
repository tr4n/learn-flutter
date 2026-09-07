# Bài 7.1: Unit Testing — Repository & UseCase với Mocktail

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Đã biết testing cơ bản; đã đọc Module 1 (Architecture)

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Unit test UseCase với 7 edge cases

```
PlaceOrderUseCase cần test:
1. ✓ Happy path: tất cả bước thành công
2. ✓ Cart empty: validate fail ngay, không gọi API
3. ✓ Inventory insufficient: gọi inventory check, fail → trả InsufficientStockFailure
4. ✓ Payment fail: charge fail → không tạo order → không rollback gì
5. ✓ Order create fail AFTER payment: charge thành công → tạo order fail → phải refund!
6. ✓ Cart clear fail AFTER order: không ảnh hưởng kết quả (silent fail)
7. ✓ Network timeout: trả TimeoutFailure

Nếu không có test → case 5 (refund logic) sẽ bị miss trong review
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. Mocktail vs Mockito — Tại sao ưu tiên Mocktail?

```
Mockito:
  Yêu cầu: Code generation (build_runner)
  @GenerateMocks([AuthRepository])
  void main() {}
  → Chạy: dart run build_runner build
  → Tạo: mock_auth_repository.mocks.dart
  
  Nhược điểm:
  - Setup phức tạp
  - Generated file cần commit hoặc tạo lại mỗi khi thay đổi
  - Dart null safety ép cần overrideWhen thêm

Mocktail (khuyến nghị):
  KHÔNG cần code generation
  class MockAuthRepository extends Mock implements AuthRepository {}
  → Xong! Không cần build_runner
  
  Ưu điểm:
  - Zero codegen setup
  - Cùng API với Mockito (when, verify, any)
  - Dart 3 null safety ready
  - Fast — test run không cần build step

Fake vs Mock vs Stub:
  Mock: Verify interactions (gọi bao nhiêu lần, với args gì)
  Stub: Định nghĩa return value (when().thenReturn())
  Fake: Lightweight implementation thật (in-memory storage)
  
  Dùng Mock/Stub: Repository, DataSource, Service (external dependencies)
  Dùng Fake: Simple implementations thay thế real (FakeCartRepository)
```

### 2.2. AAA Pattern — Arrange-Act-Assert

```
Arrange: Thiết lập test data và mock behavior
  final mockRepo = MockOrderRepository();
  when(() => mockRepo.create(...)).thenAnswer((_) async => Success(order));

Act: Thực hiện action cần test
  final result = await useCase.execute(params);

Assert: Kiểm tra kết quả
  expect(result, isA<Success<PlaceOrderResult>>());
  verify(() => mockPaymentRepo.refund(paymentId: any(named: 'paymentId'))).called(1);
  verifyNever(() => mockCartRepo.clearCart());
```

---

## Phần 3 — Production Code Implementation

### 3.1. Setup test dependencies

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.4
  bloc_test: ^9.1.7      # Cho BLoC testing
  fake_async: ^1.3.1     # Test time-dependent code
```

### 3.2. PlaceOrderUseCase — 7 test cases

```dart
// test/features/orders/domain/usecases/place_order_usecase_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

// Mock classes — không cần codegen
class MockOrderRepository extends Mock implements OrderRepository {}
class MockCartRepository extends Mock implements CartRepository {}
class MockPaymentRepository extends Mock implements PaymentRepository {}
class MockInventoryRepository extends Mock implements InventoryRepository {}
class MockPriceCalculator extends Mock implements PriceCalculator {}

void main() {
  late PlaceOrderUseCase sut; // System Under Test
  late MockOrderRepository mockOrderRepo;
  late MockCartRepository mockCartRepo;
  late MockPaymentRepository mockPaymentRepo;
  late MockInventoryRepository mockInventoryRepo;
  late MockPriceCalculator mockCalculator;

  // Test data
  const testPaymentId = 'PAY-001';
  final testItems = [
    CartItem(
      productId: 'PROD-1',
      name: 'iPhone 15',
      price: 25_000_000,
      quantity: 1,
    ),
  ];
  
  final testParams = PlaceOrderParams(
    cartItems: testItems,
    deliveryAddress: DeliveryAddress.test(),
    paymentMethod: PaymentMethod.creditCard(last4: '4242'),
    userId: 'USER-001',
  );
  
  final testOrder = Order(
    id: 'ORD-001',
    estimatedDelivery: DateTime.now().add(const Duration(days: 3)),
    totalAmount: Money.of(25_000_000, 'VND'),
  );

  setUp(() {
    mockOrderRepo = MockOrderRepository();
    mockCartRepo = MockCartRepository();
    mockPaymentRepo = MockPaymentRepository();
    mockInventoryRepo = MockInventoryRepository();
    mockCalculator = MockPriceCalculator();

    sut = PlaceOrderUseCase(
      orderRepository: mockOrderRepo,
      cartRepository: mockCartRepo,
      paymentRepository: mockPaymentRepo,
      inventoryRepository: mockInventoryRepo,
      priceCalculator: mockCalculator,
    );
    
    // Register fallback values cho Mocktail (cần cho named params)
    registerFallbackValue(const InventoryCheckParams(items: []));
    registerFallbackValue(const PaymentChargeParams(amount: 0));
  });

  group('PlaceOrderUseCase', () {
    
    // Test Case 1: Happy path
    test(
      'given valid params and all services succeed, '
      'when execute called, '
      'then returns Success with orderId and clears cart',
      () async {
        // Arrange
        _setupInventorySuccess(mockInventoryRepo);
        _setupPriceCalculation(mockCalculator, Money.of(25_000_000, 'VND'));
        _setupPaymentSuccess(mockPaymentRepo, testPaymentId);
        _setupOrderCreateSuccess(mockOrderRepo, testOrder);
        when(() => mockCartRepo.clearCart()).thenAnswer((_) async {});

        // Act
        final result = await sut.execute(testParams);

        // Assert
        expect(result, isA<Success<PlaceOrderResult>>());
        final success = result as Success<PlaceOrderResult>;
        expect(success.value.orderId, equals('ORD-001'));
        
        // Verify cart WAS cleared
        verify(() => mockCartRepo.clearCart()).called(1);
        // Verify payment WAS charged
        verify(() => mockPaymentRepo.charge(
          amount: any(named: 'amount'),
          method: any(named: 'method'),
          userId: any(named: 'userId'),
        )).called(1);
        // Verify NO refund was made
        verifyNever(() => mockPaymentRepo.refund(paymentId: any(named: 'paymentId')));
      },
    );

    // Test Case 2: Empty cart
    test(
      'given empty cart, '
      'when execute called, '
      'then returns ValidationFailure without calling any repository',
      () async {
        // Arrange
        final emptyCartParams = PlaceOrderParams(
          cartItems: [], // Empty!
          deliveryAddress: DeliveryAddress.test(),
          paymentMethod: PaymentMethod.creditCard(last4: '4242'),
          userId: 'USER-001',
        );

        // Act
        final result = await sut.execute(emptyCartParams);

        // Assert
        expect(result, isA<Failure_<PlaceOrderResult, Failure>>());
        expect((result as Failure_).failure, isA<ValidationFailure>());
        
        // Verify NOTHING was called (short-circuit)
        verifyNever(() => mockInventoryRepo.checkAvailability(
          items: any(named: 'items'),
        ));
        verifyNever(() => mockPaymentRepo.charge(
          amount: any(named: 'amount'),
          method: any(named: 'method'),
          userId: any(named: 'userId'),
        ));
      },
    );

    // Test Case 3: Insufficient inventory
    test(
      'given insufficient stock, '
      'when execute called, '
      'then returns failure without calling payment',
      () async {
        // Arrange
        when(() => mockInventoryRepo.checkAvailability(
          items: any(named: 'items'),
        )).thenAnswer(
          (_) async => const Failure_(InsufficientStockFailure(
            productId: 'PROD-1',
            available: 0,
            requested: 1,
          )),
        );
        _setupPriceCalculation(mockCalculator, Money.of(25_000_000, 'VND'));

        // Act
        final result = await sut.execute(testParams);

        // Assert
        expect(result, isA<Failure_>());
        expect((result as Failure_).failure, isA<InsufficientStockFailure>());
        
        // Payment KHÔNG được gọi khi inventory check fail
        verifyNever(() => mockPaymentRepo.charge(
          amount: any(named: 'amount'),
          method: any(named: 'method'),
          userId: any(named: 'userId'),
        ));
      },
    );

    // Test Case 5: CRITICAL — Order create fail AFTER payment success
    test(
      'given payment succeeds but order creation fails, '
      'when execute called, '
      'then refunds payment and returns Failure',
      () async {
        // Arrange
        _setupInventorySuccess(mockInventoryRepo);
        _setupPriceCalculation(mockCalculator, Money.of(25_000_000, 'VND'));
        _setupPaymentSuccess(mockPaymentRepo, testPaymentId);
        
        // Order creation FAILS
        when(() => mockOrderRepo.create(
          items: any(named: 'items'),
          address: any(named: 'address'),
          paymentId: any(named: 'paymentId'),
          totalAmount: any(named: 'totalAmount'),
          userId: any(named: 'userId'),
        )).thenAnswer(
          (_) async => const Failure_(ServerFailure(message: 'DB timeout')),
        );
        
        // Refund should succeed
        when(() => mockPaymentRepo.refund(
          paymentId: any(named: 'paymentId'),
        )).thenAnswer((_) async => const Success(null));

        // Act
        final result = await sut.execute(testParams);

        // Assert
        expect(result, isA<Failure_>());
        
        // CRITICAL: Refund PHẢI được gọi
        verify(() => mockPaymentRepo.refund(paymentId: testPaymentId))
            .called(1);
        
        // Cart KHÔNG được clear (order không thành công)
        verifyNever(() => mockCartRepo.clearCart());
      },
    );

    // Test Case 6: Cart clear fail AFTER successful order
    test(
      'given order succeeds but cart clear fails, '
      'when execute called, '
      'then returns Success (cart clear is non-critical)',
      () async {
        // Arrange
        _setupInventorySuccess(mockInventoryRepo);
        _setupPriceCalculation(mockCalculator, Money.of(25_000_000, 'VND'));
        _setupPaymentSuccess(mockPaymentRepo, testPaymentId);
        _setupOrderCreateSuccess(mockOrderRepo, testOrder);
        
        // Cart clear fails
        when(() => mockCartRepo.clearCart()).thenThrow(
          const NetworkFailure(message: 'Network error'),
        );

        // Act
        final result = await sut.execute(testParams);

        // Assert: Vẫn Success dù cart clear fail
        expect(result, isA<Success<PlaceOrderResult>>());
        
        // Cart clear WAS attempted
        verify(() => mockCartRepo.clearCart()).called(1);
        // No refund (order succeeded)
        verifyNever(() => mockPaymentRepo.refund(paymentId: any(named: 'paymentId')));
      },
    );
  });
}

// Helper setup methods
void _setupInventorySuccess(MockInventoryRepository repo) {
  when(() => repo.checkAvailability(items: any(named: 'items')))
      .thenAnswer((_) async => const Success(InventoryCheckResult(available: true)));
}

void _setupPriceCalculation(MockPriceCalculator calc, Money total) {
  when(() => calc.calculate(
    items: any(named: 'items'),
    address: any(named: 'address'),
  )).thenReturn(PriceResult(totalAmount: total, breakdown: []));
}

void _setupPaymentSuccess(MockPaymentRepository repo, String paymentId) {
  when(() => repo.charge(
    amount: any(named: 'amount'),
    method: any(named: 'method'),
    userId: any(named: 'userId'),
  )).thenAnswer((_) async => Success(paymentId));
}

void _setupOrderCreateSuccess(MockOrderRepository repo, Order order) {
  when(() => repo.create(
    items: any(named: 'items'),
    address: any(named: 'address'),
    paymentId: any(named: 'paymentId'),
    totalAmount: any(named: 'totalAmount'),
    userId: any(named: 'userId'),
  )).thenAnswer((_) async => Success(order));
}
```

### 3.3. Test Stream-based Repository

```dart
// Test watchCartItems() — stream repository
group('CartRepository.watchCartItems', () {
  test('emits updated items when cart changes', () async {
    // Arrange
    final mockStream = Stream.fromIterable([
      [CartItem(productId: 'P1', name: 'Item 1', price: 100, quantity: 1)],
      [
        CartItem(productId: 'P1', name: 'Item 1', price: 100, quantity: 2),
        CartItem(productId: 'P2', name: 'Item 2', price: 200, quantity: 1),
      ],
    ]);
    
    when(() => mockLocalDataSource.watchCachedItems())
        .thenAnswer((_) => mockStream.map((dtos) => dtos));

    // Act & Assert
    expect(
      sut.watchCartItems(),
      emitsInOrder([
        [isA<CartItem>().having((i) => i.productId, 'productId', 'P1')],
        [isA<CartItem>(), isA<CartItem>()],
      ]),
    );
  });
});
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Test execution time — Target

```
Unit test suite mục tiêu (100 tests):
  Với real async: ~10-30 seconds (network delay simulation)
  Với fake_async:  ~0.5-2 seconds
  
  fake_async: Simulate time advances without actual waiting
  test('debounce 300ms', () {
    fakeAsync((fake) {
      searchBloc.add(SearchQueryChanged('flutter'));
      fake.elapse(Duration(milliseconds: 350)); // Advance time instantly
      expect(searchBloc.state, isA<SearchSuccess>());
    });
  });
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Test verify hành vi thay vì trạng thái nội tâm
    ✅ Verify qua public interface: kết quả trả về, method nào được gọi
    Lý do: Test implementation detail → test vỡ khi refactor dù behavior không đổi

[ ] ❌ Một test file test nhiều use case / nhiều class
    ✅ Một file test = một class; describe nhóm = một method
    Lý do: Test fail không biết ngay class nào bị vỡ

[ ] ❌ Dùng real async delay (await Future.delayed()) trong unit test
    ✅ fake_async cho time-dependent test; mock ngay return value cho async calls
    Lý do: Real delay → test chậm; 100 test × 300ms delay = 30s CI time

[ ] ❌ Không test refund logic khi order creation fail
    ✅ Explicit test case cho mỗi compensating transaction
    Lý do: Refund logic critical nhất nhưng path khó reach → test bắt buộc

[ ] ❌ any() cho tất cả arguments trong verify()
    ✅ Specify argument cụ thể khi quan trọng:
       verify(() => mockRepo.refund(paymentId: 'PAY-001')).called(1);
    Lý do: any() quá rộng → không verify đúng argument được truyền

[ ] ❌ Test coverage < 80% trên Domain layer
    ✅ Mục tiêu: Domain layer ≥ 90% coverage
    Lý do: Domain chứa business logic quan trọng nhất — phải được test kỹ

[ ] ❌ setUp() không clear mock state
    ✅ Mỗi test tạo mock mới trong setUp() — không share state
    Lý do: Shared mock state → test fail không nhất quán (flaky tests)
```
