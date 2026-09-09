# Bài 1.2 — Hệ Thống Kiểu Chữ (Typography): TextTheme, M3 Type Scale & Tối Ưu Phông Chữ

## Tài Liệu Tham Khảo Chính Thức
- [Material Design 3: Typography system](https://m3.material.io/styles/typography/overview)
- [Flutter Documentation: Use a custom font](https://docs.flutter.dev/cookbook/design/fonts)
- [Flutter Documentation: Use Google Fonts](https://docs.flutter.dev/cookbook/design/package-fonts)
- [Flutter API: TextTheme class](https://api.flutter.dev/flutter/material/TextTheme-class.html)
- [Flutter API: TextScaler class](https://api.flutter.dev/flutter/painting/TextScaler-class.html)

---

## Phần 1 — Khái Niệm & Vai Trò Của Typography Ngữ Nghĩa (Semantic Typography)

### 1.1 — Vấn Đề Khi Sử Dụng Cỡ Chữ Cố Định (Magic Numbers)

Trong các dự án phần mềm không có hệ thống thiết kế nhất quán, lập trình viên thường tự do gán các giá trị kích thước chữ tùy ý:
```dart
// Không khuyến nghị: Khai báo kích thước tùy ý rải rác
Text('Tiêu đề sản phẩm', style: TextStyle(fontSize: 19, fontWeight: FontWeight.w600))
Text('Mô tả ngắn', style: TextStyle(fontSize: 13, color: Colors.grey))
```

Hạn chế của cách làm này:
1. **Phân mảnh giao diện**: Ứng dụng có hàng chục cỡ chữ khác nhau không theo quy chuẩn, gây mất tính chuyên nghiệp.
2. **Không thích ứng với Dark Mode**: Màu chữ hardcode cố định không thể tự đổi màu khi người dùng bật chế độ tối.
3. **Phá vỡ khả năng tiếp cận (Accessibility)**: Không hỗ trợ tự động co giãn theo cài đặt cỡ chữ hệ thống của người dùng lớn tuổi hoặc thị lực kém.

---

### 1.2 — Khái Niệm Material 3 Type Scale

Material 3 giải quyết triệt để vấn đề trên bằng cách định nghĩa **M3 Type Scale** — một hệ thống phân cấp kiểu chữ gồm **15 vai trò ngữ nghĩa** rõ ràng. Thay vì hỏi *"Chữ này cỡ bao nhiêu pixel?"*, câu hỏi đúng theo tư duy kiến trúc là: *"Đoạn văn bản này đóng vai trò chức năng gì trong giao diện?"*.

```
┌────────────────────────────────────────────────────────────────────────┐
│ PHÂN CẤP 5 NHÓM VAI TRÒ TYPOGRAPHY TRONG MATERIAL 3                    │
│                                                                        │
│ 1. Display (Large / Medium / Small):                                   │
│    • Dùng cho các con số nổi bật, màn hình Onboarding, banner lớn      │
│                                                                        │
│ 2. Headline (Large / Medium / Small):                                  │
│    • Dùng cho tiêu đề màn hình chính, tiêu đề phân đoạn quan trọng     │
│                                                                        │
│ 3. Title (Large / Medium / Small):                                     │
│    • Dùng cho tiêu đề thẻ Card, thanh tiêu đề AppBar, tiêu đề Dialog   │
│                                                                        │
│ 4. Body (Large / Medium / Small):                                      │
│    • Dùng cho nội dung văn bản đọc chính, đoạn văn mô tả chi tiết      │
│                                                                        │
│ 5. Label (Large / Medium / Small):                                     │
│    • Dùng cho chữ trên nút bấm, nhãn Chip, phụ đề và chú thích        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Bảng Thông Số Kỹ Thuật M3 Type Scale

Dưới đây là bảng thông số tiêu chuẩn của 15 vai trò kiểu chữ trong `TextTheme` của Flutter theo quy chuẩn Material 3:

| Text Style Role | Cỡ Chữ (`fontSize`) | Chiều Cao Dòng (`lineHeight`) | Độ Đậm (`fontWeight`) | Khoảng Cách Ký Tự (`letterSpacing`) | Ngữ Cảnh Áp Dụng |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **`displayLarge`** | 57 sp | 64 dp | Regular (w400) | -0.25 sp | Con số đo lường lớn, Splash screen |
| **`displayMedium`** | 45 sp | 52 dp | Regular (w400) | 0.0 sp | Tiêu đề chào mừng lớn |
| **`displaySmall`** | 36 sp | 44 dp | Regular (w400) | 0.0 sp | Tiêu đề phần giới thiệu nổi bật |
| **`headlineLarge`** | 32 sp | 40 dp | Regular (w400) | 0.0 sp | Tiêu đề màn hình chính |
| **`headlineMedium`**| 28 sp | 36 dp | Regular (w400) | 0.0 sp | Tiêu đề nhóm nội dung lớn |
| **`headlineSmall`** | 24 sp | 32 dp | Regular (w400) | 0.0 sp | Tiêu đề phân mục vừa |
| **`titleLarge`** | 22 sp | 28 dp | Regular (w400) | 0.0 sp | Tiêu đề `AppBar`, hộp thoại `Dialog` |
| **`titleMedium`** | 16 sp | 24 dp | Medium (w500) | +0.15 sp | Tiêu đề thẻ `Card`, mục danh sách `ListTile` |
| **`titleSmall`** | 14 sp | 20 dp | Medium (w500) | +0.1 sp | Tiêu đề phụ, nhãn nhóm nhỏ |
| **`bodyLarge`** | 16 sp | 24 dp | Regular (w400) | +0.5 sp | Đoạn văn đọc chính (Khuyên dùng đọc dài) |
| **`bodyMedium`** | 14 sp | 20 dp | Regular (w400) | +0.25 sp | Nội dung mặc định cho hầu hết widget |
| **`bodySmall`** | 12 sp | 16 dp | Regular (w400) | +0.4 sp | Đoạn mô tả phụ, thông tin bản quyền |
| **`labelLarge`** | 14 sp | 20 dp | Medium (w500) | +0.1 sp | Chữ trên nút bấm (`FilledButton`, `ElevatedButton`) |
| **`labelMedium`** | 12 sp | 16 dp | Medium (w500) | +0.5 sp | Nhãn `Chip`, Tab bar item |
| **`labelSmall`** | 11 sp | 16 dp | Medium (w500) | +0.5 sp | Chú thích ảnh (Caption), thông báo số lượng |

---

### 2.2 — Cơ Chế Co Giãn Chữ Động Với `TextScaler` (Flutter 3.16+)

Trước phiên bản Flutter 3.16, hệ thống sử dụng thuộc tính `textScaleFactor` (một số thực tuyến tính như $1.0, 1.5, 2.0$). 

**Hạn chế của phương pháp cũ**: Nếu người dùng tăng cỡ chữ lên $200\%$, tất cả các chữ đều bị nhân đôi tuyến tính:
- Chữ nhỏ ($14\text{sp}$) tăng lên $28\text{sp}$ $\to$ Đọc tốt.
- Chữ tiêu đề lớn ($32\text{sp}$) tăng lên $64\text{sp}$ $\to$ Làm vỡ toàn bộ bố cục ứng dụng, chữ bị cắt cụt (clipping) hoặc tràn khung.

Từ Flutter 3.16 trở đi, Flutter giới thiệu lớp **`TextScaler`** hỗ trợ co giãn phi tuyến tính (Non-linear Text Scaling tương thích với Android 14 và iOS Dynamic Type):
- Chữ nhỏ được phóng to với tỷ lệ cao hơn để đảm bảo khả năng đọc.
- Chữ lớn được phóng to với tỷ lệ thấp hơn để bảo toàn bố cục giao diện.

```dart
// Đọc TextScaler từ MediaQuery hiện tại:
final TextScaler textScaler = MediaQuery.textScalerOf(context);

// Tính toán cỡ chữ thực tế được render ra màn hình:
final double renderedSize = textScaler.scale(16.0);
```

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Cấu Hình Phông Chữ Nội Bộ Qua Local Assets

Đây là giải pháp ưu tiên cho môi trường sản xuất có yêu cầu hoạt động Offline hoàn toàn, không phụ thuộc vào kết nối mạng bên ngoài:

#### 1. Đặt tệp phông chữ vào thư mục `assets/fonts/`:
```
assets/fonts/
  ├── Inter-Regular.ttf
  ├── Inter-Medium.ttf
  ├── Inter-SemiBold.ttf
  └── Inter-Bold.ttf
```

#### 2. Khai báo ánh xạ trong `pubspec.yaml`:
```yaml
flutter:
  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf
          weight: 400
        - asset: assets/fonts/Inter-Medium.ttf
          weight: 500
        - asset: assets/fonts/Inter-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Inter-Bold.ttf
          weight: 700
```

#### 3. Gán phông chữ toàn cục vào `ThemeData`:
```dart
ThemeData(
  useMaterial3: true,
  fontFamily: 'Inter', // Áp dụng cho toàn bộ TextTheme
  colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
)
```

---

### 3.2 — Tích Hợp Với Thư Viện `google_fonts`

Sử dụng thư viện chính thức `google_fonts` để tích hợp phông chữ mà không cần tải thủ công:

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';

ThemeData buildAppTheme() {
  final baseTheme = ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
  );

  return baseTheme.copyWith(
    // Áp dụng font Roboto Flex hoặc Plus Jakarta Sans cho toàn bộ TextTheme
    textTheme: GoogleFonts.plusJakartaSansTextTheme(baseTheme.textTheme),
  );
}
```

---

### 3.3 — Tùy Biến `TextTheme` Kế Thừa Bằng `.copyWith()`

Khi cần điều chỉnh một số kiểu chữ đặc thù mà vẫn giữ nguyên các thông số chuẩn khác của M3:

```dart
import 'package:flutter/material.dart';

TextTheme configureCustomTypography(TextTheme baseTextTheme, ColorScheme colorScheme) {
  return baseTextTheme.copyWith(
    // Tùy biến tiêu đề lớn của màn hình
    headlineLarge: baseTextTheme.headlineLarge?.copyWith(
      fontWeight: FontWeight.bold,
      letterSpacing: -0.5,
      color: colorScheme.onSurface,
    ),
    // Tùy biến tiêu đề thẻ
    titleMedium: baseTextTheme.titleMedium?.copyWith(
      fontWeight: FontWeight.w600,
      color: colorScheme.onSurface,
    ),
    // Tùy biến chữ nút bấm
    labelLarge: baseTextTheme.labelLarge?.copyWith(
      fontWeight: FontWeight.bold,
      letterSpacing: 0.5,
    ),
  );
}
```

---

### 3.4 — Sử Dụng Typography Chuẩn Trong Widget

Sử dụng trực tiếp các vai trò ngữ nghĩa từ `Theme.of(context).textTheme`:

```dart
import 'package:flutter/material.dart';

class ArticleListItem extends StatelessWidget {
  final String category;
  final String title;
  final String summary;
  final String readTime;

  const ArticleListItem({
    super.key,
    required this.category,
    required this.title,
    required this.summary,
    required this.readTime,
  });

  @override
  Widget build(BuildContext context) {
    final textTheme = Theme.of(context).textTheme;
    final colorScheme = Theme.of(context).colorScheme;

    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 12.0, horizontal: 16.0),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // 1. Nhãn thể loại: labelSmall kết hợp màu primary
          Text(
            category.toUpperCase(),
            style: textTheme.labelSmall?.copyWith(
              color: colorScheme.primary,
              fontWeight: FontWeight.bold,
            ),
          ),
          const SizedBox(height: 4),

          // 2. Tiêu đề bài viết: titleLarge kết hợp onSurface
          Text(
            title,
            style: textTheme.titleLarge?.copyWith(
              color: colorScheme.onSurface,
            ),
            maxLines: 2,
            overflow: TextOverflow.ellipsis,
          ),
          const SizedBox(height: 8),

          // 3. Đoạn tóm tắt: bodyMedium kết hợp onSurfaceVariant (màu phụ nhẹ hơn)
          Text(
            summary,
            style: textTheme.bodyMedium?.copyWith(
              color: colorScheme.onSurfaceVariant,
            ),
            maxLines: 3,
            overflow: TextOverflow.ellipsis,
          ),
          const SizedBox(height: 8),

          // 4. Thời gian đọc: labelSmall
          Text(
            readTime,
            style: textTheme.labelSmall?.copyWith(
              color: colorScheme.outline,
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Tràn khung chữ (Text Overflow) khi người dùng bật cỡ chữ lớn

#### Mô tả vấn đề:
Giao diện hiển thị bình thường trên máy nhà phát triển, nhưng bị lỗi tràn sọc vàng đen hoặc chữ bị mất góc trên máy người dùng thực tế có bật trợ năng tăng cỡ chữ.

#### Nguyên nhân kỹ thuật:
Các container chứa văn bản được cài đặt chiều cao cố định (`height: 50.0`) hoặc đặt trong `Row` mà không bọc `Expanded`. Khi `TextScaler` phóng to kích thước phông chữ, văn bản vượt quá không gian bao bọc.

#### Biện pháp khắc phục:
1. Tránh gán cứng chiều cao (`height`) cho các container chứa văn bản; sử dụng khoảng đệm (`padding`) để chiều cao co giãn tự nhiên.
2. Khi đặt văn bản cạnh hình ảnh trong `Row`, luôn bọc văn bản bằng `Expanded` hoặc `Flexible` kết hợp thuộc tính `overflow: TextOverflow.ellipsis`.

---

### 4.2 — Thiếu khai báo biến thể `FontWeight` trong `pubspec.yaml`

#### Mô tả vấn đề:
Khai báo phông chữ tùy biến nhưng khi áp dụng `FontWeight.bold`, chữ chỉ hơi đậm nhẹ hoặc bị biến dạng góc cạnh (Faux Bold).

#### Nguyên nhân kỹ thuật:
Nếu `pubspec.yaml` chỉ khai báo duy nhất tệp tin Regular (`weight: 400`), khi Flutter Engine cần vẽ chữ `FontWeight.w700`, hệ thống đồ họa sẽ dùng giải thuật tô đậm giả lập (Faux Bold) bằng cách vẽ chồng nhiều nét viền lên phông Regular, làm mất đi các đường nét tinh xảo ban đầu của nhà thiết kế phông chữ.

#### Biện pháp khắc phục:
Khai báo đầy đủ các tệp font tương ứng với từng mức độ đậm: `weight: 400`, `weight: 500`, `weight: 700`.

---

### 4.3 — Hiện tượng chữ đổi kiểu đột ngột (FOUT - Flash of Unstyled Text)

#### Mô tả vấn đề:
Khi khởi động ứng dụng sử dụng `google_fonts`, trong vài giây đầu tiên văn bản hiển thị bằng phông chữ hệ thống thô, sau đó đột ngột giật chuyển sang phông Google Fonts sau khi tệp tin tải xong từ Internet.

#### Biện pháp khắc phục:
Đóng gói sẵn các tệp phông chữ phổ biến trực tiếp vào thư mục assets của ứng dụng và thông báo cho thư viện thông qua phương thức `GoogleFonts.config.allowRuntimeFetching = false;` để ép ứng dụng luôn nạp từ bộ nhớ cục bộ mà không tải qua mạng.

---

### 4.4 — Gán cứng thuộc tính `color` trong `TextStyle` toàn cục

#### Mô tả vấn đề:
Khai báo `textTheme: TextTheme(bodyMedium: TextStyle(color: Colors.black87))` trong `ThemeData`.

#### Nguyên nhân kỹ thuật:
Màu chữ bị cố định thành màu đen cho cả Light Theme và Dark Theme. Khi ứng dụng chuyển sang Dark Mode, nền biến thành màu đen và chữ cũng màu đen, làm biến mất toàn bộ nội dung hiển thị.

#### Biện pháp khắc phục:
Không gán màu cứng vào `TextTheme` gốc. Để framework tự động áp dụng màu từ `ColorScheme` (`onSurface` hoặc `onSurfaceVariant`), hoặc chỉ gán màu thông qua `.copyWith(color: colorScheme.onSurface)` tại vị trí sử dụng.

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Thuộc tính `height` trong `TextStyle` của Flutter có ý nghĩa kỹ thuật gì và cách tính toán chiều cao dòng thực tế?
*Phân tích:*
Trong Flutter, thuộc tính `height` của `TextStyle` không phải là một kích thước đo bằng pixel tuyệt đối, mà là một **hệ số tỷ lệ nhân (Multiplier)**:
$$\text{Line Height (dp)} = \text{fontSize} \times \text{height}$$
Ví dụ: Một đoạn văn bản có `fontSize: 16.0` và `height: 1.5` sẽ có chiều cao mỗi dòng thực tế trên màn hình là:
$$16.0 \times 1.5 = 24.0\text{ dp}$$
Nếu không chỉ định `height` (giá trị `null`), chiều cao dòng sẽ lấy theo thông số tự nhiên được định nghĩa bên trong bảng số liệu đo lường (Font Metrics Table) của chính tệp tin phông chữ đó.

---

#### Câu hỏi 2: Sự khác biệt bản chất giữa `TextScaler` (Flutter 3.16+) và `textScaleFactor` cũ là gì?
*Phân tích:*
- **`textScaleFactor`**: Là một số thực đơn lẻ (Scalar value, ví dụ `1.5`). Mọi cỡ chữ từ $10\text{sp}$ đến $60\text{sp}$ đều bị nhân với cùng một hệ số $1.5$, gây ra hiện tượng chữ tiêu đề lớn bị phóng đại quá mức làm phá vỡ layout.
- **`TextScaler`**: Là một đối tượng trừu tượng có phương thức `double scale(double fontSize)`. Nó cho phép triển khai giải thuật co giãn phi tuyến tính (Non-linear font scaling): Đối với các cỡ chữ nhỏ, tỷ lệ co giãn có thể là $1.3$, nhưng đối với các cỡ chữ lớn, tỷ lệ co giãn chỉ là $1.1$, bảo đảm tính đọc được mà vẫn giữ được tính toàn vẹn của cấu trúc không gian giao diện.

---

#### Câu hỏi 3: Cơ chế Font Fallback hoạt động như thế nào khi phông chữ chính không hỗ trợ ký tự tiếng Việt hoặc Emoji?
*Phân tích:*
Khi Flutter Engine render một ký tự, nó tìm kiếm bảng mã ký tự (Glyph Table) của phông chữ đang được chỉ định. Nếu phông chữ chính (ví dụ một phông chữ tiếng Anh cổ điển) không có glyph cho ký tự tiếng Việt có dấu (như `ệ`, `ở`, `đ`), Engine sẽ tự động duyệt qua danh sách dự phòng trong mảng `fontFamilyFallback`. Nếu danh sách này cũng không chứa glyph, Engine sẽ fallback về phông chữ mặc định của hệ điều hành (Roboto trên Android hoặc San Francisco / Apple Color Emoji trên iOS) để bảo đảm không xuất hiện ký tự ô vuông lỗi (Tofu box).

---

#### Câu hỏi 4: Tại sao các vai trò `label...` trong M3 thường có `FontWeight.w500` (Medium) trong khi `body...` là `FontWeight.w400` (Regular)?
*Phân tích:*
Các vai trò `label` chủ yếu xuất hiện trên các thành phần tương tác có diện tích nhỏ gọn như nút bấm (`Button`), thẻ nhãn (`Chip`) hoặc thanh điều hướng (`NavigationBar`). Ở kích thước nhỏ ($11\text{sp} - 14\text{sp}$), nét chữ `Regular` sẽ rất mảnh và dễ bị nuốt mất độ nét khi hiển thị trên các màn hình có mật độ điểm ảnh thấp hoặc nền có màu sắc tương phản cao. Việc sử dụng độ đậm `Medium (w500)` giúp tăng cường độ đậm quang học, bảo đảm khả năng nhận diện chức năng tương tác tức thì của người dùng.

---

#### Câu hỏi 5: Thuộc tính `leadingDistribution` trong `TextStyle` giải quyết vấn đề gì trong căn chỉnh giao diện?
*Phân tích:*
Mỗi dòng chữ đều có khoảng đệm phía trên và phía dưới (gọi là Leading). Mặc định, khoảng đệm này có thể phân bổ không đều tùy thuộc vào từng font chữ, khiến văn bản khó căn giữa chính xác theo trục dọc bên trong một nút bấm hoặc icon container. Bằng cách thiết lập:
`leadingDistribution: TextLeadingDistribution.even`,
Flutter Engine sẽ chia đều khoảng không gian thừa còn lại lên cả hai nửa trên và dưới của dòng chữ, giúp việc căn giữa đồng đều theo phương đứng.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho một đoạn văn bản được cấu hình kiểu chữ như sau:

```dart
Text(
  'Tiêu đề bài viết',
  style: TextStyle(
    fontSize: 20.0,
    height: 1.4,
  ),
)
```

Giả sử đoạn văn bản này được render trên hai thiết bị khác nhau:
1. **Thiết bị A**: Người dùng đặt cỡ chữ hệ thống mặc định (`TextScaler.noScaling`).
2. **Thiết bị B**: Người dùng bật tùy chọn trợ năng với tỷ lệ phóng to chữ $130\%$ (`TextScaler.linear(1.3)`).

#### Yêu cầu phân tích:
1. Hãy tính toán chiều cao của một dòng văn bản (Line Height tính bằng dp) khi hiển thị trên **Thiết bị A**.
2. Hãy tính toán cỡ chữ hiển thị (`renderedFontSize`) và chiều cao của một dòng văn bản khi hiển thị trên **Thiết bị B**.
3. Nếu đặt đoạn văn bản trên vào một `SizedBox(height: 25.0)`, điều gì sẽ xảy ra trên Thiết bị A và Thiết bị B?

---

#### Kết quả phân tích kỹ thuật:

1. **Trên Thiết bị A (Không co giãn):**
   - Cỡ chữ: $\text{fontSize} = 20.0\text{ sp}$.
   - Chiều cao dòng:
     $$\text{Line Height}_A = 20.0 \times 1.4 = \mathbf{28.0\text{ dp}}$$

2. **Trên Thiết bị B (Tỷ lệ 1.3):**
   - Cỡ chữ hiển thị thực tế:
     $$\text{renderedFontSize}_B = 20.0 \times 1.3 = \mathbf{26.0\text{ dp}}$$
   - Do hệ số `height: 1.4` là tỷ lệ nhân trực tiếp trên cỡ chữ thực tế, chiều cao dòng tương ứng:
     $$\text{Line Height}_B = 26.0 \times 1.4 = \mathbf{36.4\text{ dp}}$$

3. **Hiện tượng khi đặt vào `SizedBox(height: 25.0)`:**
   - Chiều cao khả dụng cố định của hộp chứa là $25.0\text{ dp}$.
   - **Trên Thiết bị A**: Chiều cao dòng thực tế ($28.0\text{ dp}$) lớn hơn chiều cao hộp chứa ($25.0\text{ dp}$) $\to$ Văn bản bị tràn nhẹ $3.0\text{ dp}$, phần đuôi của các ký tự có dấu rơi xuống dưới (như ký tự `g`, `y`, `p` hoặc dấu nặng `.` trong tiếng Việt) có thể bị cắt cụt một phần.
   - **Trên Thiết bị B**: Chiều cao dòng thực tế ($36.4\text{ dp}$) vượt xa giới hạn $25.0\text{ dp}$ $\to$ Văn bản bị cắt cụt nghiêm trọng hơn $11.4\text{ dp}$, làm mất hoàn toàn khả năng đọc của người dùng.
   - **Kết luận kỹ thuật**: Không bao giờ bọc các phần tử văn bản bằng `SizedBox` có chiều cao cố định nhỏ hơn hoặc xấp xỉ chiều cao dòng lý thuyết. Luôn để chiều cao co giãn tự nhiên theo nội dung hoặc cung cấp khoảng đệm đàn hồi.
