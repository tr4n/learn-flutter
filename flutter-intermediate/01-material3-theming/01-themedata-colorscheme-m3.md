# Bài 1.1 — Kiến Trúc Giao Diện Material 3: ThemeData, ColorScheme & Không Gian Màu HCT

## Tài Liệu Tham Khảo Chính Thức
- [Material Design 3: Color system](https://m3.material.io/styles/color/overview)
- [Flutter Documentation: Material 3 theming](https://docs.flutter.dev/ui/design/material)
- [Flutter API: ThemeData class](https://api.flutter.dev/flutter/material/ThemeData-class.html)
- [Flutter API: ColorScheme class](https://api.flutter.dev/flutter/material/ColorScheme-class.html)
- [Material Color Utilities library](https://pub.dev/packages/material_color_utilities)

---

## Phần 1 — Khái Niệm & Sự Tiến Hóa Của Hệ Thống Giao Diện (Design System Evolution)

### 1.1 — Tiến Hóa Từ Material 2 Sang Material 3

Material 3 (M3) là phiên bản hệ thống thiết kế mới nhất của Google, được tích hợp làm giao diện mặc định trong Flutter từ phiên bản 3.16. Sự chuyển đổi từ Material 2 (M2) sang Material 3 đại diện cho một bước chuyển biến lớn về triết lý giao diện:

```
┌────────────────────────────────────────────────────────────────────────┐
│ SO SÁNH TRIẾT LÝ THIẾT KẾ: MATERIAL 2 VS MATERIAL 3                   │
│                                                                        │
│ Tiêu Chí        Material 2 (M2)               Material 3 (M3)          │
│ ────────────────────────────────────────────────────────────────────── │
│ Bảng màu        Cố định (Primary, Accent)     Động (5 Tonal Palettes)  │
│ Không gian màu  RGB / HSL (Phi tiếp cận)      HCT (Tiếp cận khoa học)  │
│ Độ cao vật lý   Đổ bóng xám (Drop Shadow)     Phủ màu (surfaceTintColor)│
│ Nền AppBar      Màu Primary đậm (Áp đảo)      Màu Surface nhẹ nhàng    │
│ Tùy biến theme  Nhiều thuộc tính phân mảnh   Tập trung vào ColorScheme│
└────────────────────────────────────────────────────────────────────────┘
```

Trong Flutter hiện đại, toàn bộ hệ thống màu sắc của ứng dụng được quản lý tập trung thông qua **`ColorScheme`**. Các thuộc tính kế thừa cũ của Material 2 (như `primaryColor`, `accentColor`, `backgroundColor`, `buttonColor`) đã bị đánh dấu lỗi thời (deprecated) để nhường chỗ cho hệ thống vai trò màu sắc (Color Roles) của `ColorScheme`.

---

### 1.2 — Cấu Trúc Thứ Bậc Của `ThemeData`

`ThemeData` là đối tượng cấu hình toàn cục cao nhất, được cung cấp thông qua thuộc tính `theme` và `darkTheme` của `MaterialApp`:

```
MaterialApp
  └── ThemeData
       ├── colorScheme           ← Bảng màu M3 (Color Roles)
       ├── textTheme             ← Hệ thống kiểu chữ (M3 Type Scale)
       ├── appBarTheme           ← Cấu hình thanh tiêu đề
       ├── elevatedButtonTheme   ← Cấu hình nút nổi
       ├── cardTheme             ← Cấu hình thẻ hiển thị (Elevation, Shape)
       └── extensions            ← Các thuộc tính tùy biến mở rộng (ThemeExtension)
```

Khi một widget gọi `Theme.of(context)`, nó tìm kiếm thể hiện `ThemeData` gần nhất trong cây widget thông qua cơ chế `InheritedWidget` (`_InheritedTheme`).

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Thuật Toán Material Color Utilities & Không Gian Màu HCT

Một trong những hạn chế lớn nhất của không gian màu RGB hay HSL truyền thống là chúng không phản ánh đúng **độ sáng mà mắt người cảm nhận (Perceptual Brightness)**. Ví dụ: Màu vàng thuần (`#FFFF00`) và màu xanh lam thuần (`#0000FF`) đều có `Lightness = 50%` trong mô hình HSL, nhưng trên thực tế mắt người cảm nhận màu vàng sáng hơn rất nhiều so với màu xanh lam.

Material 3 giải quyết vấn đề này bằng việc phát minh ra không gian màu **HCT (Hue, Chroma, Tone)**:

```
┌────────────────────────────────────────────────────────────────────────┐
│ CẤU TRÚC KHÔNG GIAN MÀU HCT                                            │
│                                                                        │
│ • Hue (Sắc tướng):        0° -> 360° (Vị trí trên vòng tròn màu)       │
│ • Chroma (Độ thuần sắc):  0 -> 120+  (Cường độ / Độ bão hòa màu)       │
│ • Tone (Độ sáng cảm nhận):0 -> 100   (Độ sáng quang học thực tế)       │
│                                                                        │
│ Tone = 0   ──► Đen tuyệt đối (Black)                                   │
│ Tone = 50  ──► Mức sáng trung tính (Mọi màu có Tone 50 đều sáng như nhau)│
│ Tone = 100 ──► Trắng tuyệt đối (White)                                 │
└────────────────────────────────────────────────────────────────────────┘
```

#### Thuật toán sinh màu `ColorScheme.fromSeed()`:
Khi truyền một màu đơn lẻ vào `ColorScheme.fromSeed(seedColor: color)`, thư viện thuật toán `material_color_utilities` thực hiện các bước sau:
1. Trích xuất giá trị HCT của `seedColor`.
2. Tạo ra **5 bảng màu sắc độ (Tonal Palettes)**:
   - **Primary Palette**: Dựa trên Hue và Chroma của seed color.
   - **Secondary Palette**: Cùng Hue nhưng giảm bớt Chroma để tạo màu phụ êm dịu.
   - **Tertiary Palette**: Dịch chuyển Hue một góc khoảng $60^{\circ}$ để tạo màu tương phản bổ trợ.
   - **Neutral Palette**: Giảm Chroma gần về 0 để tạo màu nền (`surface`).
   - **Neutral Variant Palette**: Chroma thấp để tạo các đường viền (`outline`) và bề mặt phụ.
3. Mỗi Tonal Palette được chia thành 13 bậc Tone tiêu chuẩn: $0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 95, 99, 100$.

#### Bảo đảm độ tương phản tiếp cận (Accessibility WCAG):
Trong chế độ sáng (Light Mode):
- `primary` được lấy từ **Tone 40**.
- `onPrimary` (chữ/icon trên nền primary) được lấy từ **Tone 100** (Trắng).
- Khoảng cách Tone: $\Delta = 100 - 40 = 60$. Khoảng cách này bảo đảm tỷ lệ tương phản luôn đạt chuẩn tối thiểu **4.5:1** theo tiêu chuẩn WCAG AA mà không cần lập trình viên phải tính toán thủ công.

---

### 2.2 — Hệ Thống Vai Trò Màu Sắc (Color Roles) Trong Material 3

Material 3 định nghĩa các vai trò màu sắc theo cặp (Một màu nền luôn đi kèm với một màu tiền cảnh có tiền tố `on...`):

```
┌────────────────────────────────────────────────────────────────────────┐
│ HỆ THỐNG VAI TRÒ MÀU SẮC (COLOR ROLES)                                 │
│                                                                        │
│ [ACCENT ROLES - NHÓM MÀU ĐIỂM NHẤN]                                    │
│ • primary / onPrimary                 -> Nút chính, FAB, Switch bật    │
│ • primaryContainer / onPrimaryContainer -> Chip đang chọn, thanh tiến trình│
│ • secondary / onSecondary             -> Nút phụ, Filter chips         │
│ • secondaryContainer / onSecondaryContainer -> Chỉ mục NavigationBar   │
│ • tertiary / onTertiary               -> Điểm nhấn bổ trợ, lịch, badge │
│ • tertiaryContainer / onTertiaryContainer -> Thẻ cảnh báo nhẹ          │
│                                                                        │
│ [SURFACE ROLES - NHÓM MÀU BỀ MẶT]                                      │
│ • surface / onSurface                 -> Nền Scaffold, Card, Dialog    │
│ • surfaceContainer / surfaceContainerHigh -> Các tầng nền nâng cao     │
│                                                                        │
│ [UTILITY ROLES - NHÓM TIỆN ÍCH]                                        │
│ • error / onError                     -> Thông báo lỗi nghiêm trọng    │
│ • errorContainer / onErrorContainer   -> Nền cảnh báo lỗi Form         │
│ • outline / outlineVariant            -> Viền ô nhập, đường phân cách  │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 2.3 — Cơ Chế Độ Cao Mới: Lớp Phủ Sắc Độ `surfaceTintColor`

Trong Material 2, độ cao (`elevation`) của một thành phần được biểu thị hoàn toàn bằng độ mờ và bán kính đổ bóng xám (Drop Shadow).

Trong Material 3, độ cao được biểu thị kết hợp thông qua:
1. Độ bóng nhẹ nhàng hơn.
2. **Lớp phủ sắc tố màu Primary (`surfaceTintColor`)**:

```
┌────────────────────────────────────────────────────────────────────────┐
│ CƠ CHẾ ELEVATION VÀ SURFACE TINT TRONG MATERIAL 3                      │
│                                                                        │
│ Elevation Level 0 (0dp)  ──► surface thuần (Không phủ màu)             │
│ Elevation Level 1 (1dp)  ──► surface + 5%  màu primary                 │
│ Elevation Level 2 (3dp)  ──► surface + 8%  màu primary                 │
│ Elevation Level 3 (6dp)  ──► surface + 11% màu primary                 │
│ Elevation Level 4 (8dp)  ──► surface + 12% màu primary                 │
│ Elevation Level 5 (12dp) ──► surface + 14% màu primary                 │
└────────────────────────────────────────────────────────────────────────┘
```

Đặc biệt trong Chế độ tối (Dark Mode), khi đổ bóng xám hoàn toàn vô hình trên nền đen, cơ chế phủ màu sắc độ này giúp người dùng dễ dàng phân biệt thứ bậc không gian của các thẻ (`Card`), hộp thoại (`Dialog`) và tấm trượt (`BottomSheet`).

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Xây Dựng Cấu Hình Theme Tập Trung (`AppTheme`)

Đóng gói cấu hình theme thành một lớp riêng biệt với đầy đủ hỗ trợ cho cả Light Mode và Dark Mode:

```dart
import 'package:flutter/material.dart';

abstract final class AppTheme {
  // Màu hạt giống thương hiệu
  static const Color _seedColor = Color(0xFF0061A4);

  // 1. Cấu hình Light Theme
  static ThemeData get lightTheme {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: _seedColor,
      brightness: Brightness.light,
    );

    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      scaffoldBackgroundColor: colorScheme.surface,
      appBarTheme: AppBarTheme(
        centerTitle: true,
        backgroundColor: colorScheme.surface,
        foregroundColor: colorScheme.onSurface,
        elevation: 0,
        scrolledUnderElevation: 3, // Phủ màu nhẹ khi cuộn nội dung bên dưới
      ),
      cardTheme: CardTheme(
        elevation: 1,
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
        surfaceTintColor: colorScheme.surfaceTint,
      ),
      filledButtonTheme: FilledButtonThemeData(
        style: FilledButton.styleFrom(
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
        ),
      ),
    );
  }

  // 2. Cấu hình Dark Theme
  static ThemeData get darkTheme {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: _seedColor,
      brightness: Brightness.dark,
    );

    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      scaffoldBackgroundColor: colorScheme.surface,
      appBarTheme: AppBarTheme(
        centerTitle: true,
        backgroundColor: colorScheme.surface,
        foregroundColor: colorScheme.onSurface,
        elevation: 0,
        scrolledUnderElevation: 3,
      ),
      cardTheme: CardTheme(
        elevation: 2,
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
        surfaceTintColor: colorScheme.surfaceTint,
      ),
      filledButtonTheme: FilledButtonThemeData(
        style: FilledButton.styleFrom(
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
        ),
      ),
    );
  }
}
```

Áp dụng vào `MaterialApp`:
```dart
MaterialApp(
  title: 'Hệ Thống Giao Diện M3',
  theme: AppTheme.lightTheme,
  darkTheme: AppTheme.darkTheme,
  themeMode: ThemeMode.system, // Tự động theo cài đặt hệ điều hành
  home: const HomeScreen(),
);
```

---

### 3.2 — Áp Dụng Color Roles Đúng Chuẩn Trong Widget

Sử dụng trực tiếp các vai trò màu sắc từ `Theme.of(context).colorScheme` thay vì khai báo màu cố định:

```dart
import 'package:flutter/material.dart';

class ProductSummaryCard extends StatelessWidget {
  final String title;
  final String category;
  final double price;

  const ProductSummaryCard({
    super.key,
    required this.title,
    required this.category,
    required this.price,
  });

  @override
  Widget build(BuildContext context) {
    // Truy xuất ColorScheme từ ngữ cảnh hiện tại
    final colorScheme = Theme.of(context).colorScheme;
    final textTheme = Theme.of(context).textTheme;

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                // Nhãn danh mục: Dùng secondaryContainer và onSecondaryContainer
                Container(
                  padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 4),
                  decoration: BoxDecoration(
                    color: colorScheme.secondaryContainer,
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: Text(
                    category,
                    style: textTheme.labelSmall?.copyWith(
                      color: colorScheme.onSecondaryContainer,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                ),
                // Giá sản phẩm: Điểm nhấn dùng primary
                Text(
                  '${price.toStringAsFixed(2)} đ',
                  style: textTheme.titleMedium?.copyWith(
                    color: colorScheme.primary,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ],
            ),
            const SizedBox(height: 12),

            // Tiêu đề: Dùng onSurface
            Text(
              title,
              style: textTheme.titleLarge?.copyWith(
                color: colorScheme.onSurface,
              ),
            ),
            const SizedBox(height: 16),

            // Nút hành động chính: FilledButton mặc định dùng primary và onPrimary
            Row(
              mainAxisAlignment: MainAxisAlignment.end,
              children: [
                OutlinedButton(
                  onPressed: () {},
                  child: const Text('Xem Chi Tiết'),
                ),
                const SizedBox(width: 8),
                FilledButton.icon(
                  onPressed: () {},
                  icon: const Icon(Icons.add_shopping_cart, size: 18),
                  label: const Text('Chọn Mua'),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Khởi tạo `ThemeData` bên trong phương thức `build()`

#### Mô tả vấn đề:
Khai báo `theme: ThemeData(...)` trực tiếp bên trong phương thức `build()` của widget gốc:
```dart
// Lỗi: Tái tạo ThemeData ở mỗi lần App rebuild
@override
Widget build(BuildContext context) {
  return MaterialApp(
    theme: ThemeData(colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue)),
  );
}
```

#### Nguyên nhân kỹ thuật:
`ThemeData` là một đối tượng cấu hình đồ sộ gồm hàng chục bảng màu, kiểu chữ và component themes. Việc tạo mới ở mỗi lần widget rebuild sẽ tiêu tốn CPU, làm vô hiệu hóa bộ nhớ đệm hình ảnh và kích hoạt rebuild toàn bộ cây widget không cần thiết.

#### Biện pháp khắc phục:
Khởi tạo cấu hình theme dưới dạng biến tĩnh (`static final`) hoặc lưu trữ trong các singleton/controller.

---

### 4.2 — Hardcode màu tĩnh thay vì dùng ColorScheme Roles

#### Mô tả vấn đề:
Sử dụng các màu cố định như `Colors.white`, `Colors.black`, hoặc mã Hex trực tiếp trong giao diện:
```dart
// Lỗi: Khi chuyển sang Dark Mode, nền đen chữ vẫn màu đen -> Chữ vô hình!
Text(
  'Thông báo',
  style: TextStyle(color: Colors.black),
)
```

#### Biện pháp khắc phục:
Luôn đọc màu từ `Theme.of(context).colorScheme`:
- Màu chữ thông thường: `colorScheme.onSurface`.
- Màu chữ thứ cấp / phụ đề: `colorScheme.onSurfaceVariant`.
- Màu chữ trên nút bấm chính: `colorScheme.onPrimary`.

---

### 4.3 — Quên đặt `useMaterial3: true` trên các phiên bản Flutter cũ

#### Mô tả vấn đề:
Sử dụng `ColorScheme.fromSeed()` nhưng các thành phần như `AppBar`, `FloatingActionButton`, `NavigationBar` vẫn hiển thị theo phong cách Material 2 (hình vuông, đổ bóng đậm, AppBar màu tím đậm).

#### Biện pháp khắc phục:
Trong các phiên bản Flutter trước 3.16, luôn khai báo tường minh `useMaterial3: true` trong `ThemeData`. (Từ Flutter 3.16 trở lên, cờ này đã mặc định là `true`).

---

### 4.4 — Gán màu nền mà không gán cặp màu chữ tương ứng

#### Mô tả vấn đề:
Tự đặt màu nền là `colorScheme.primaryContainer` nhưng giữ nguyên màu chữ mặc định.

#### Nguyên nhân kỹ thuật:
Trong Dark Mode, `primaryContainer` là một màu tối có sắc độ trầm, trong khi ở Light Mode nó là một màu nhạt. Nếu không sử dụng đúng màu chữ đối ứng (`colorScheme.onPrimaryContainer`), độ tương phản sẽ bị vi phạm, khiến nội dung bị mờ hoặc chìm vào nền.

#### Biện pháp khắc phục:
Luôn áp dụng nguyên tắc đi theo cặp: Nền `[Role]` luôn đi kèm chữ `on[Role]`.

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Tại sao thuật toán HCT có thể sinh ra màu `primary` có mã Hex khác hoàn toàn so với mã Hex của `seedColor` truyền vào?
*Phân tích:*
Khi truyền một `seedColor` vào `ColorScheme.fromSeed(seedColor: seed, brightness: Brightness.light)`, thuật toán chỉ trích xuất **Hue (Sắc tướng)** và **Chroma (Độ thuần sắc)** từ hạt giống đó. Thuộc tính **Tone** ban đầu của `seedColor` bị loại bỏ và thay thế bằng mốc **Tone 40** cố định cho thuộc tính `primary` trong Light Mode. Do đó, nếu bạn truyền vào một màu xanh rất sáng (`Tone 90`), mã Hex của `primary` tạo ra sẽ đậm hơn nhiều so với màu gốc để bảo đảm tỷ lệ tương phản tiếp cận với màu chữ trắng (`Tone 100`).

---

#### Câu hỏi 2: Sự khác biệt bản chất giữa `ColorScheme.fromSeed` và `ColorScheme.fromSwatch` là gì?
*Phân tích:*
- **`ColorScheme.fromSwatch`**: Là giải pháp của Material 2. Nó dựa trên bảng màu cố định `MaterialColor` (các nấc từ 50 đến 900) được pha thủ công bằng cách trộn màu trắng/đen trong không gian RGB. Nó không hỗ trợ các vai trò màu mới của M3 (như `surfaceContainer`, `tertiaryContainer`).
- **`ColorScheme.fromSeed`**: Là giải pháp của Material 3. Sử dụng mô hình toán học HCT để tính toán động 5 Tonal Palettes độc lập, tự động phân bổ chính xác cho hơn 30 vai trò màu sắc và tự động điều chỉnh độ sáng tối ưu cho cả hai chế độ Light và Dark.

---

#### Câu hỏi 3: Lớp `InheritedTheme` hoạt động như thế nào khi ứng dụng hiển thị một Dialog hoặc BottomSheet?
*Phân tích:*
Dialog và Modal BottomSheet được hiển thị thông qua `Navigator` trên một Route hoàn toàn mới, nằm ở một nhánh con độc lập của `OverlayEntry`. Nếu không có cơ chế đặc biệt, Route mới này sẽ không thừa hưởng được các cấu hình theme cục bộ của màn hình hiện tại. Phương thức `showDialog` sử dụng `InheritedTheme.capture(from: context, to: navigatorContext)` để thu thập toàn bộ các `InheritedTheme` tại vị trí gọi và bọc chúng xung quanh widget con của Dialog, bảo đảm tính nhất quán về mặt giao diện.

---

#### Câu hỏi 4: Thuộc tính `scrolledUnderElevation` trong `AppBarTheme` có cơ chế hoạt động như thế nào?
*Phân tích:*
Trong Material 3, khi màn hình ở trạng thái đứng yên tại đỉnh, `AppBar` có màu nền hoàn toàn trùng khớp với `scaffoldBackgroundColor` (`elevation = 0`). Khi người dùng cuộn nội dung bên dưới (được phát hiện thông qua `ScrollNotificationListener`), `AppBar` tự động kích hoạt `scrolledUnderElevation` (mặc định là `3.0`). Tại mức độ cao này, lớp phủ sắc tố `surfaceTintColor` được kích hoạt, làm màu nền AppBar hơi ngả sang tông màu Primary để phân tách rõ ràng với nội dung đang cuộn bên dưới.

---

#### Câu hỏi 5: Làm thế nào để hỗ trợ các màu sắc thương hiệu không nằm trong hệ thống Color Roles của Material 3?
*Phân tích:*
Không nên cố tình gán các màu thương hiệu đặc thù vào các trường không đúng ngữ nghĩa của `ColorScheme` (ví dụ gán màu thành công Success vào trường `tertiary`). Giải pháp kỹ thuật chuẩn của Flutter là tạo một lớp kế thừa từ **`ThemeExtension<T>`** (ví dụ: `AppCustomColors extends ThemeExtension<AppCustomColors>`), đăng ký vào mảng `ThemeData.extensions`, và truy xuất trong widget bằng cú pháp `Theme.of(context).extension<AppCustomColors>()`.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho bảng phân bổ Tone trong thuật toán Material Color Utilities:
- Trong **Light Mode**:
  - `primary` = Tone 40
  - `onPrimary` = Tone 100
  - `primaryContainer` = Tone 90
  - `onPrimaryContainer` = Tone 10
- Trong **Dark Mode**:
  - `primary` = Tone 80
  - `onPrimary` = Tone 20
  - `primaryContainer` = Tone 30
  - `onPrimaryContainer` = Tone 90

Cho một widget hiển thị nhãn văn bản:
```dart
Container(
  color: theme.colorScheme.primaryContainer,
  child: Text('Dữ liệu', style: TextStyle(color: theme.colorScheme.onPrimaryContainer)),
)
```

#### Yêu cầu phân tích:
1. Hãy tính toán độ chênh lệch Tone ($\Delta \text{Tone}$) giữa màu nền và màu chữ trong **Light Mode**.
2. Hãy tính toán độ chênh lệch Tone ($\Delta \text{Tone}$) giữa màu nền và màu chữ trong **Dark Mode**.
3. Điều gì sẽ xảy ra về mặt thị giác nếu lập trình viên hardcode màu chữ là `Colors.white` thay vì sử dụng `onPrimaryContainer` khi ứng dụng đang ở Light Mode?

---

#### Kết quả phân tích kỹ thuật:

1. **Độ chênh lệch Tone trong Light Mode:**
   - Màu nền `primaryContainer` = Tone 90 (Nền rất sáng).
   - Màu chữ `onPrimaryContainer` = Tone 10 (Chữ rất đậm).
   - Độ chênh lệch:
     $$\Delta \text{Tone} = |90 - 10| = \mathbf{80}$$
   *(Độ tương phản vượt xa tiêu chuẩn WCAG AAA, chữ hiển thị sắc nét trên nền sáng).*

2. **Độ chênh lệch Tone trong Dark Mode:**
   - Màu nền `primaryContainer` = Tone 30 (Nền tối).
   - Màu chữ `onPrimaryContainer` = Tone 90 (Chữ sáng).
   - Độ chênh lệch:
     $$\Delta \text{Tone} = |30 - 90| = \mathbf{60}$$
   *(Độ tương phản đạt tiêu chuẩn WCAG AA, bảo đảm khả năng đọc trong môi trường thiếu sáng).*

3. **Hiện tượng khi hardcode `Colors.white` trong Light Mode:**
   - Màu chữ `Colors.white` có giá trị quang học tương đương Tone 100.
   - Màu nền `primaryContainer` có Tone 90.
   - Độ chênh lệch:
     $$\Delta \text{Tone} = |90 - 100| = \mathbf{10}$$
   - **Hậu quả thị giác**: Độ chênh lệch chỉ đạt 10 đơn vị (dưới ngưỡng tối thiểu). Người dùng gần như **không thể đọc được dòng chữ** (chữ trắng gần như biến mất trên nền màu nhạt), vi phạm tiêu chuẩn tiếp cận (Accessibility Failure).
