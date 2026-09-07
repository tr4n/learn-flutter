# Bài 7.2: Widget Testing & Golden Tests

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Đã biết Flutter widget testing cơ bản

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Visual regression khi upgrade Flutter

**Design System library của E-commerce app:**

```
Sprint 25: Upgrade Flutter 3.24 → 3.27
  
Sau upgrade:
- Button padding thay đổi nhỏ (Material 3 specification update)
- Icon size thay đổi trong NavigationBar
- Text baseline alignment thay đổi trong Card

3 Bug reports từ QA team sau 2 ngày test thủ công:
  "Button trông nhỏ hơn trước"
  "Navigation icons lệch vị trí"
  "Text trong card bị cắt"

Golden Tests phát hiện ngay khi merge PR:
  ❌ test/goldens/button_primary.png — pixel diff: 823 pixels changed
  ❌ test/goldens/navigation_bar.png — pixel diff: 1,240 pixels changed
  ❌ test/goldens/product_card.png — pixel diff: 45 pixels changed

→ Phát hiện visual regression trong CI, trước khi merge, không cần QA manual
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. WidgetTester — Cơ chế hoạt động

```
testWidgets('...', (tester) async {
  await tester.pumpWidget(MyWidget());
  //                 ↑
  // Build widget tree (1 frame)
  // Không xử lý timer, animation
  
  await tester.pump();
  // Thêm 1 frame — tick pending futures/timers
  
  await tester.pump(Duration(milliseconds: 300));
  // Advance time 300ms + process frames
  
  await tester.pumpAndSettle();
  // Pump frames cho đến khi không còn pending animation
  // MAX: 100 frames (throw nếu vẫn còn animation sau đó)
  
  await tester.tap(find.byType(ElevatedButton));
  // Simulate tap gesture (giả lập touch event)
  await tester.pump(); // Process tap result
});
```

### 2.2. Golden Test — matchesGoldenFile

```
First run (create golden):
  await expectLater(
    find.byType(ProductCard),
    matchesGoldenFile('goldens/product_card.png'),
  );
  → File không tồn tại → CREATE file với pixel content hiện tại
  → Commit file vào git

Subsequent runs (compare):
  → Load golden file
  → Capture current widget pixels
  → Compare pixel-by-pixel với threshold
  → FAIL nếu diff > threshold
  
Update golden (khi UI thay đổi intentionally):
  flutter test --update-goldens
  → Re-generate all golden files
  → Review diff trong git → approve changes
```

---

## Phần 3 — Production Code Implementation

### 3.1. Widget Test Production Pattern

```dart
// test/features/catalog/presentation/widgets/product_card_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class MockCartBloc extends MockBloc<CartEvent, CartState> implements CartBloc {}
class MockCartEvent extends Fake implements CartEvent {}

void main() {
  late MockCartBloc mockCartBloc;
  
  setUp(() {
    mockCartBloc = MockCartBloc();
    registerFallbackValue(MockCartEvent());
  });

  Widget buildSubject({
    Product? product,
    bool showWishlistButton = true,
  }) {
    return MaterialApp(
      theme: AppTheme.light, // Dùng theme thật để test đúng màu sắc
      home: Scaffold(
        body: BlocProvider<CartBloc>.value(
          value: mockCartBloc,
          child: ProductCard(
            product: product ?? Product.fixture(),
            showWishlistButton: showWishlistButton,
          ),
        ),
      ),
    );
  }

  group('ProductCard', () {
    testWidgets(
      'displays product name, price, and image',
      (tester) async {
        // Arrange
        final product = Product.fixture(
          name: 'iPhone 15 Pro',
          price: Money.of(29_990_000, 'VND'),
        );

        // Act
        await tester.pumpWidget(buildSubject(product: product));

        // Assert
        expect(find.text('iPhone 15 Pro'), findsOneWidget);
        expect(find.text('29.990.000 ₫'), findsOneWidget);
        expect(find.byType(CachedNetworkImage), findsOneWidget);
      },
    );

    testWidgets(
      'when add to cart tapped, dispatches AddToCartEvent to CartBloc',
      (tester) async {
        // Arrange
        when(() => mockCartBloc.state).thenReturn(const CartInitial());
        final product = Product.fixture(id: 'PROD-123');

        // Act
        await tester.pumpWidget(buildSubject(product: product));
        
        // Find button bằng key (tốt hơn dùng text — tránh localization issues)
        await tester.tap(find.byKey(const Key('add_to_cart_button')));
        await tester.pump();

        // Assert
        verify(() => mockCartBloc.add(
          AddToCartEvent(productId: 'PROD-123', quantity: 1),
        )).called(1);
      },
    );

    testWidgets(
      'when product is out of stock, add to cart button is disabled',
      (tester) async {
        // Arrange
        final outOfStockProduct = Product.fixture(stockStatus: StockStatus.outOfStock);

        // Act
        await tester.pumpWidget(buildSubject(product: outOfStockProduct));

        // Assert
        final button = tester.widget<ElevatedButton>(
          find.byKey(const Key('add_to_cart_button')),
        );
        expect(button.onPressed, isNull); // disabled = onPressed null
      },
    );

    testWidgets(
      'shows discount badge when product has active discount',
      (tester) async {
        // Arrange
        final discountedProduct = Product.fixture(
          originalPrice: Money.of(30_000_000, 'VND'),
          currentPrice: Money.of(24_000_000, 'VND'), // 20% discount
        );

        // Act
        await tester.pumpWidget(buildSubject(product: discountedProduct));

        // Assert
        expect(find.byKey(const Key('discount_badge')), findsOneWidget);
        expect(find.text('-20%'), findsOneWidget);
      },
    );
  });

  group('ProductCard — Gesture Tests', () {
    testWidgets(
      'double tap on wishlist button triggers wishlist toggle twice',
      (tester) async {
        // Arrange
        final product = Product.fixture(isWishlisted: false);
        final wishlistBloc = MockWishlistBloc();
        when(() => wishlistBloc.state).thenReturn(const WishlistInitial());

        // Act
        await tester.pumpWidget(buildSubject(product: product));
        
        // Double tap
        await tester.tap(find.byKey(const Key('wishlist_button')));
        await tester.pump();
        await tester.tap(find.byKey(const Key('wishlist_button')));
        await tester.pump();

        // Assert: toggle gọi 2 lần (đây là intended behavior hay bug?)
        // Đây là place để document expected behavior
      },
    );
  });
}
```

### 3.2. Golden Test Setup chuẩn cho CI

```dart
// test/helpers/golden_test_helper.dart

/// Helper để test golden với font loading đúng
Future<void> loadFonts() async {
  // Load font từ package để golden test có text đúng
  // Flutter test env không auto-load fonts từ pubspec.yaml
  final fontLoader = FontLoader('Roboto');
  fontLoader.addFont(
    rootBundle.load('assets/fonts/Roboto-Regular.ttf'),
  );
  await fontLoader.load();
}

/// Wrapper cho golden test với size device chuẩn
testWidgets testGolden(
  String name,
  Widget widget, {
  Size surfaceSize = const Size(390, 844), // iPhone 14 screen size
  ThemeData? theme,
}) {
  testWidgets(name, (tester) async {
    await loadFonts();
    await tester.binding.setSurfaceSize(surfaceSize);
    
    await tester.pumpWidget(
      MaterialApp(
        theme: theme ?? AppTheme.light,
        home: Scaffold(body: widget),
        debugShowCheckedModeBanner: false,
      ),
    );
    await tester.pumpAndSettle();
    
    await expectLater(
      find.byType(MaterialApp),
      matchesGoldenFile('goldens/${name.replaceAll(' ', '_')}.png'),
    );
    
    // Reset surface size sau test
    addTearDown(() => tester.binding.setSurfaceSize(null));
  });
}
```

```dart
// test/design_system/product_card_golden_test.dart
void main() {
  group('ProductCard Golden Tests', () {
    testWidgets('default state', (tester) async {
      await loadFonts();
      await tester.binding.setSurfaceSize(const Size(390, 250));
      
      await tester.pumpWidget(MaterialApp(
        theme: AppTheme.light,
        home: Scaffold(
          body: ProductCard(product: Product.fixture()),
        ),
      ));
      await tester.pumpAndSettle();
      
      await expectLater(
        find.byType(ProductCard),
        matchesGoldenFile('goldens/product_card_default.png'),
      );
    });

    testWidgets('out of stock state', (tester) async {
      await loadFonts();
      await tester.binding.setSurfaceSize(const Size(390, 250));
      
      await tester.pumpWidget(MaterialApp(
        theme: AppTheme.light,
        home: Scaffold(
          body: ProductCard(
            product: Product.fixture(stockStatus: StockStatus.outOfStock),
          ),
        ),
      ));
      await tester.pumpAndSettle();
      
      await expectLater(
        find.byType(ProductCard),
        matchesGoldenFile('goldens/product_card_out_of_stock.png'),
      );
    });

    testWidgets('dark mode', (tester) async {
      await loadFonts();
      await tester.binding.setSurfaceSize(const Size(390, 250));
      
      await tester.pumpWidget(MaterialApp(
        theme: AppTheme.dark,
        home: Scaffold(
          body: ProductCard(product: Product.fixture()),
        ),
      ));
      await tester.pumpAndSettle();
      
      await expectLater(
        find.byType(ProductCard),
        matchesGoldenFile('goldens/product_card_dark.png'),
      );
    });
  });
}
```

### 3.3. CI Golden Test Workflow

```yaml
# .github/workflows/flutter_test.yml
jobs:
  golden-tests:
    runs-on: ubuntu-latest  # LUÔN dùng Linux cho golden test — pixel exact
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.27.0'  # Pin exact version — golden pixel-perfect

      - name: Run Golden Tests
        run: flutter test test/design_system/ --tags golden

      - name: Upload Golden Diffs on Failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: golden-diffs
          path: |
            test/goldens/failures/  # Flutter lưu diff images ở đây khi test fail
            
      # Chỉ update goldens khi PR title có [update-goldens]
      - name: Update Goldens
        if: contains(github.event.pull_request.title, '[update-goldens]')
        run: |
          flutter test --update-goldens test/design_system/
          git config --global user.email "ci@myapp.com"
          git config --global user.name "CI Bot"
          git add test/goldens/
          git commit -m "chore: update golden files"
          git push
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Widget Test vs Golden Test — chi phí

```
Widget Test (functional):
  Không render pixels — chỉ check widget tree structure
  1 test: ~50-200ms
  100 tests: ~10-20 seconds

Golden Test (visual):
  Render đầy đủ + compare pixels
  1 test: ~500ms-1.5s (font loading + render + diff)
  50 tests: ~25-75 seconds
  
Khuyến nghị:
  Widget tests: Mọi logic, interaction, state change
  Golden tests: Chỉ Design System components (Button, Card, Input)
               Không golden test cho mọi screen → CI quá chậm
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Golden test chạy trên macOS CI runner
    ✅ Golden test chỉ chạy trên Linux CI runner (ubuntu-latest)
    Lý do: Font rendering khác nhau giữa OS → false positive fails

[ ] ❌ Flutter version không được pin trong CI golden test
    ✅ flutter-version: '3.27.0' — exact version, không phải 'latest'
    Lý do: Flutter update → Material 3 pixel changes → false fail

[ ] ❌ testWidgets không await tester.pump() sau interaction
    ✅ Luôn await pump() sau tap/type để process kết quả
    Lý do: Không pump → assertion chạy trước widget rebuild → false pass

[ ] ❌ Dùng find.text() cho string từ localization
    ✅ Dùng find.byKey(Key('widget_key')) hoặc find.byType()
    Lý do: Text thay đổi khi i18n → test vỡ dù logic đúng

[ ] ❌ BLoC mock không có when().thenReturn() cho initial state
    ✅ when(() => mockBloc.state).thenReturn(initialState)
    Lý do: Không mock state → BlocBuilder nhận null state → render sai

[ ] ❌ Fake animation trong widget test bằng WidgetsApp.debugAllowBannerTouching
    ✅ Dùng tester.pump(Duration(milliseconds: X)) để advance animation time
    Lý do: pumpAndSettle() timeout nếu animation không end (AnimationController.repeat)

[ ] ❌ Golden files không được commit vào git
    ✅ Commit goldens/ directory — track changes qua git diff
    Lý do: Golden files là "expected output" — phải version controlled
```
