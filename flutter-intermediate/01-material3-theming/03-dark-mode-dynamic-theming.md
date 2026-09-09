# Bài 1.3 — Chế Độ Tối (Dark Mode), Dynamic Color & Mở Rộng Giao Diện Bằng ThemeExtension

## Tài Liệu Tham Khảo Chính Thức
- [Material Design 3: Dark theme](https://m3.material.io/styles/color/advanced/dark-theme)
- [Flutter Documentation: Theming in Flutter](https://docs.flutter.dev/ui/design/material)
- [Flutter API: ThemeMode enum](https://api.flutter.dev/flutter/material/ThemeMode.html)
- [Flutter API: ThemeExtension class](https://api.flutter.dev/flutter/material/ThemeExtension-class.html)
- [Flutter Package: dynamic_color](https://pub.dev/packages/dynamic_color)

---

## Phần 1 — Khái Niệm & Vai Trò Của Hệ Thống Theme Động

### 1.1 — Triết Lý Thiết Kế Chế Độ Tối (Dark Theme)

Chế độ tối (Dark Mode) không đơn thuần là việc đảo ngược màu trắng thành đen (Inversion). Trong Material 3, Dark Theme là một hệ thống thiết kế có chủ đích:
1. **Giảm thiểu mỏi mắt**: Trong môi trường thiếu sáng, độ tương phản chói gắt của nền trắng tinh có thể gây khó chịu thị giác.
2. **Tiết kiệm năng lượng**: Trên các tấm nền hiển thị công nghệ OLED và AMOLED, các pixel màu đen hoặc xám đậm tiêu thụ ít điện năng hơn đáng kể, kéo dài thời lượng pin của thiết bị.
3. **Bảo tồn tính nhận diện thương hiệu**: Sử dụng các mốc Tone màu cao hơn ($70 - 80$) trong bảng màu Tonal Palettes để màu sắc hiển thị rõ nét trên nền tối mà không bị chìm.

---

### 1.2 — Ba Chế Độ Hiển Thị Trong Flutter (`ThemeMode`)

Flutter cung cấp enum `ThemeMode` để quản lý chế độ giao diện:
- **`ThemeMode.system` (Mặc định)**: Tự động phát hiện và chuyển đổi theo cài đặt hệ thống của thiết bị người dùng.
- **`ThemeMode.light`**: Luôn ép buộc ứng dụng chạy ở giao diện sáng, bất kể cài đặt của hệ điều hành.
- **`ThemeMode.dark`**: Luôn ép buộc ứng dụng chạy ở giao diện tối.

---

### 1.3 — Khái Niệm Dynamic Color (Material You)

Từ Android 12 trở lên, Google giới thiệu tính năng **Dynamic Color** (trợ lực bởi Monet Engine). Hệ thống tự động trích xuất các màu sắc chủ đạo từ hình nền (Wallpaper) của người dùng để tạo ra một `ColorScheme` độc bản cho thiết bị. Thư viện `dynamic_color` cho phép ứng dụng Flutter tự động tích hợp bảng màu cá nhân hóa này.

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Luồng Phân Giải Theme Trong `MaterialApp`

Khi một khung hình mới được render, `MaterialApp` thực hiện giải thuật chọn lựa `ThemeData` theo sơ đồ:

```
┌────────────────────────────────────────────────────────────────────────┐
│ LUỒNG PHÂN GIẢI THEMEDATA TRONG MATERIALAPP                            │
│                                                                        │
│                      ┌──────────────────────┐                          │
│                      │ Kiểm tra themeMode   │                          │
│                      └──────────┬───────────┘                          │
│                                 │                                      │
│         ┌───────────────────────┼───────────────────────┐              │
│         ▼                       ▼                       ▼              │
│  [ThemeMode.light]      [ThemeMode.dark]       [ThemeMode.system]      │
│         │                       │                       │              │
│         ▼                       ▼                       ▼              │
│   Dùng 'theme'            Dùng 'darkTheme'       Kiểm tra độ sáng:     │
│                                              PlatformDispatcher.       │
│                                              platformBrightness        │
│                                                         │              │
│                                             ┌───────────┴───────────┐  │
│                                             ▼                       ▼  │
│                                     [Brightness.light]      [Brightness.dark]
│                                             │                       │  │
│                                             ▼                       ▼  │
│                                       Dùng 'theme'            Dùng 'darkTheme'
└────────────────────────────────────────────────────────────────────────┘
```

> **Lưu ý quan trọng**: Nếu hệ điều hành đang ở chế độ tối (`Brightness.dark`) nhưng lập trình viên **không khai báo thuộc tính `darkTheme`** trong `MaterialApp`, framework sẽ tự động fallback và sử dụng thuộc tính `theme` (giao diện sáng).

---

### 2.2 — Cơ Chế Mở Rộng Theme Bằng `ThemeExtension`

Khi ứng dụng có các màu sắc hoặc thông số thiết kế đặc thù không nằm trong các trường mặc định của `ColorScheme` (ví dụ: màu cảnh báo `warning`, màu thành công `success`, khoảng cách `spacing`), Flutter cung cấp lớp trừu tượng `ThemeExtension<T>`.

Hai phương thức bắt buộc phải cài đặt:
1. **`copyWith(...)`**: Tạo một bản sao mới của extension với các trường được ghi đè tùy chọn.
2. **`lerp(ThemeExtension<T>? other, double t)`**: Cung cấp hàm nội suy toán học khi chuyển đổi theme:
   $$\text{currentColor} = \text{Color.lerp}(\text{this.color}, \text{other.color}, t)$$
   Nhờ phương thức `lerp`, khi người dùng bật tắt chế độ tối, các màu mở rộng sẽ biến đổi mượt mà theo tiến độ thời gian $t \in [0.0, 1.0]$ thay vì bị giật đổi màu đột ngột.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Xây Dựng `ThemeController` Lưu Trữ Bền Vững

Quản lý trạng thái theme toàn cục bằng `ChangeNotifier` và lưu trữ cài đặt người dùng vào `SharedPreferences`:

```dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

class ThemeController extends ChangeNotifier {
  static const _themePrefKey = 'user_theme_mode';
  final SharedPreferences _prefs;
  ThemeMode _themeMode;

  ThemeController(this._prefs) : _themeMode = _loadThemeMode(_prefs);

  ThemeMode get themeMode => _themeMode;

  static ThemeMode _loadThemeMode(SharedPreferences prefs) {
    final index = prefs.getInt(_themePrefKey);
    if (index != null && index >= 0 && index < ThemeMode.values.length) {
      return ThemeMode.values[index];
    }
    return ThemeMode.system; // Mặc định theo hệ thống
  }

  Future<void> updateThemeMode(ThemeMode newMode) async {
    if (_themeMode == newMode) return;

    _themeMode = newMode;
    notifyListeners(); // Thông báo cho MaterialApp cập nhật giao diện
    await _prefs.setInt(_themePrefKey, newMode.index);
  }
}
```

---

### 3.2 — Tích Hợp Dynamic Color (Material You)

Sử dụng thư viện `package:dynamic_color` để tự động tích hợp màu hình nền thiết bị, đồng thời cung cấp màu dự phòng (fallback) cho các thiết bị không hỗ trợ:

```dart
import 'package:dynamic_color/dynamic_color.dart';
import 'package:flutter/material.dart';

class DynamicThemeApp extends StatelessWidget {
  final ThemeController themeController;
  static const _defaultSeedColor = Color(0xFF1E88E5);

  const DynamicThemeApp({super.key, required this.themeController});

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: themeController,
      builder: (context, _) {
        return DynamicColorBuilder(
          builder: (ColorScheme? lightDynamic, ColorScheme? darkDynamic) {
            // 1. Tạo Light ColorScheme: Ưu tiên màu từ OS, nếu không dùng màu mặc định
            final lightColorScheme = lightDynamic ??
                ColorScheme.fromSeed(
                  seedColor: _defaultSeedColor,
                  brightness: Brightness.light,
                );

            // 2. Tạo Dark ColorScheme
            final darkColorScheme = darkDynamic ??
                ColorScheme.fromSeed(
                  seedColor: _defaultSeedColor,
                  brightness: Brightness.dark,
                );

            return MaterialApp(
              title: 'Dynamic Theme App',
              themeMode: themeController.themeMode,
              theme: ThemeData(
                useMaterial3: true,
                colorScheme: lightColorScheme,
                extensions: [AppCustomColors.light],
              ),
              darkTheme: ThemeData(
                useMaterial3: true,
                colorScheme: darkColorScheme,
                extensions: [AppCustomColors.dark],
              ),
              home: const SettingsScreen(),
            );
          },
        );
      },
    );
  }
}
```

---

### 3.3 — Triển Khai `ThemeExtension` Cho Màu Sắc Tùy Biến

Tạo mở rộng cho hai trạng thái cảnh báo `warning` và thành công `success`:

```dart
import 'package:flutter/material.dart';

@immutable
class AppCustomColors extends ThemeExtension<AppCustomColors> {
  final Color? success;
  final Color? onSuccess;
  final Color? warning;
  final Color? onWarning;

  const AppCustomColors({
    required this.success,
    required this.onSuccess,
    required this.warning,
    required this.onWarning,
  });

  // Bản phối màu cho Light Theme
  static const light = AppCustomColors(
    success: Color(0xFF2E7D32),
    onSuccess: Colors.white,
    warning: Color(0xFFED6C02),
    onWarning: Colors.white,
  );

  // Bản phối màu cho Dark Theme
  static const dark = AppCustomColors(
    success: Color(0xFF81C784),
    onSuccess: Color(0xFF003300),
    warning: Color(0xFFFFB74D),
    onWarning: Color(0xFF4D2600),
  );

  @override
  AppCustomColors copyWith({
    Color? success,
    Color? onSuccess,
    Color? warning,
    Color? onWarning,
  }) {
    return AppCustomColors(
      success: success ?? this.success,
      onSuccess: onSuccess ?? this.onSuccess,
      warning: warning ?? this.warning,
      onWarning: onWarning ?? this.onWarning,
    );
  }

  @override
  AppCustomColors lerp(ThemeExtension<AppCustomColors>? other, double t) {
    if (other is! AppCustomColors) return this;

    return AppCustomColors(
      success: Color.lerp(success, other.success, t),
      onSuccess: Color.lerp(onSuccess, other.onSuccess, t),
      warning: Color.lerp(warning, other.warning, t),
      onWarning: Color.lerp(onWarning, other.onWarning, t),
    );
  }
}
```

Cách sử dụng trong Widget:
```dart
class StatusBanner extends StatelessWidget {
  final String message;

  const StatusBanner({super.key, required this.message});

  @override
  Widget build(BuildContext context) {
    // Truy xuất trực tiếp extension từ ngữ cảnh hiện tại
    final customColors = Theme.of(context).extension<AppCustomColors>()!;

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
      decoration: BoxDecoration(
        color: customColors.success,
        borderRadius: BorderRadius.circular(8),
      ),
      child: Text(
        message,
        style: TextStyle(color: customColors.onSuccess, fontWeight: FontWeight.bold),
      ),
    );
  }
}
```

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Hiện tượng chớp sáng trắng (Theme Flash / White Flicker) lúc khởi động

#### Mô tả vấn đề:
Người dùng cài đặt ứng dụng chạy ở Chế độ tối. Khi khởi động ứng dụng, màn hình nhấp nháy sáng trắng một frame trước khi chuyển sang màu đen.

#### Nguyên nhân kỹ thuật:
Tác vụ đọc cấu hình từ `SharedPreferences.getInstance()` là một hàm bất đồng bộ (`Future`). Nếu gọi `loadFromPrefs()` bên trong hàm `initState()` của Widget, ứng dụng đã hoàn tất vẽ khung hình đầu tiên bằng cấu hình mặc định (`ThemeMode.light`) trước khi dữ liệu từ bộ nhớ kịp nạp xong.

#### Biện pháp khắc phục:
Đọc trước cấu hình trong hàm `main()` trước khi kích hoạt `runApp()`:
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final prefs = await SharedPreferences.getInstance();
  final themeController = ThemeController(prefs);

  runApp(DynamicThemeApp(themeController: themeController));
}
```

---

### 4.2 — Đảo màu thủ công bằng điều kiện `isDarkMode ? ... : ...`

#### Mô tả vấn đề:
Lập trình viên viết các câu lệnh kiểm tra điều kiện rải rác:
```dart
// Không khuyến nghị: Viết điều kiện thủ công rải rác
final isDark = Theme.of(context).brightness == Brightness.dark;
final textColor = isDark ? Colors.white : Colors.black87;
```

#### Nguyên nhân kỹ thuật:
Phá vỡ nguyên lý đóng gói của Design System. Khi cần thay đổi tông màu (ví dụ từ xám sang xanh thẫm), lập trình viên phải tìm kiếm và sửa đổi hàng trăm vị trí có câu lệnh điều kiện trên toàn bộ dự án.

#### Biện pháp khắc phục:
Luôn quy đổi về các vai trò màu sắc chuẩn của Material 3: `colorScheme.onSurface` hoặc đóng gói vào `ThemeExtension`.

---

### 4.3 — Thiếu cài đặt phương thức `lerp` trong `ThemeExtension`

#### Mô tả vấn đề:
Ghi đè phương thức `lerp` nhưng chỉ trả về `this` mà không thực hiện phép tính nội suy màu:
```dart
// Lỗi: Bỏ qua nội suy chuyển màu
@override
AppCustomColors lerp(ThemeExtension<AppCustomColors>? other, double t) {
  return this; // Gây giật màu tức thì khi đổi theme
}
```

#### Nguyên nhân kỹ thuật:
Flutter sử dụng phương thức `ThemeData.lerp()` để tạo animation chuyển đổi giao diện mượt mà. Nếu extension không cài đặt `Color.lerp()`, toàn bộ giao diện sẽ chuyển màu êm ái ngoại trừ các widget sử dụng extension bị giật đổi màu đột ngột.

#### Biện pháp khắc phục:
Luôn cài đặt đầy đủ phương thức `Color.lerp()` cho từng thuộc tính màu sắc trong extension.

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Phương thức `PlatformDispatcher.instance.onPlatformBrightnessChanged` liên kết với chu trình rebuild của `MaterialApp` như thế nào?
*Phân tích:*
Khi người dùng thay đổi chế độ sáng/tối từ trung tâm điều khiển của hệ điều hành, hệ điều hành gửi tín hiệu ngắt xuống Flutter Engine. `PlatformDispatcher` bắt tín hiệu này và kích hoạt callback `onPlatformBrightnessChanged`. `WidgetsBinding` nhận thông báo và gọi phương thức `didChangePlatformBrightness()` trên tất cả các `WidgetsBindingObserver` đã đăng ký (trong đó có `MaterialApp`). `MaterialApp` kích hoạt một chu kỳ `setState()` nội bộ, đánh giá lại giá trị `platformBrightness` hiện tại và rebuild toàn bộ cây ứng dụng với `ThemeData` mới.

---

#### Câu hỏi 2: Tại sao trong Dark Mode của Material 3, màu đen tuyệt đối (`#000000`) ít khi được sử dụng làm màu `surface` mặc định?
*Phân tích:*
Theo nghiên cứu thị giác của Material Design:
1. **Hiện tượng nhòe chuyển động (OLED Smearing)**: Trên màn hình OLED, khi một pixel chuyển từ trạng thái tắt hoàn toàn (Đen thuần) sang bật sáng, các diode phát quang cần một độ trễ nhỏ để khởi động, tạo ra vệt bóng mờ màu tím/đen khi người dùng cuộn nội dung nhanh.
2. **Khả năng biểu thị độ cao (Elevation)**: Màu đen thuần không thể phản ánh được các lớp phủ sắc độ (`surfaceTintColor`). Việc sử dụng màu xám đậm (Tone 6 đến Tone 10) làm nền bề mặt giúp biểu thị chiều sâu không gian tốt hơn và giảm độ chói tương phản với các văn bản màu sáng.

---

#### Câu hỏi 3: Sự khác biệt bản chất giữa việc sử dụng `ThemeExtension` và việc tạo thêm một `InheritedWidget` riêng lẻ?
*Phân tích:*
- Nếu tạo một `InheritedWidget` riêng lẻ: Cây widget sẽ bị lồng thêm một tầng node mới, và quan trọng nhất là nó **hoàn toàn tách rời khỏi chu trình chuyển đổi theme của Flutter**. Khi gọi `ThemeData.lerp()`, widget riêng này sẽ không tự động đồng bộ thời gian chuyển động với các thành phần Material khác.
- Sử dụng `ThemeExtension`: Được tích hợp trực tiếp vào cấu trúc dữ liệu của `ThemeData`. Nó được tự động nạp qua `Theme.of(context)` và tự động tham gia vào chu trình nội suy màu sắc `ThemeData.lerp()` của framework.

---

#### Câu hỏi 4: Thư viện `dynamic_color` xử lý như thế nào trên các thiết bị không hỗ trợ tính năng Material You?
*Phân tích:*
Widget `DynamicColorBuilder` gọi phương thức platform channel xuống hệ điều hành Android. Nếu thiết bị chạy phiên bản thấp hơn Android 12, hoặc là thiết bị iOS / Desktop / Web, platform channel sẽ trả về giá trị `null` cho cả hai tham số `lightDynamic` và `darkDynamic`. Lập trình viên sử dụng toán tử `??` để fallback về bảng màu được tạo từ `ColorScheme.fromSeed(seedColor: defaultColor)`, bảo đảm ứng dụng hoạt động ổn định và nhất quán trên 100% các nền tảng.

---

#### Câu hỏi 5: Tại sao việc sử dụng `Theme.of(context)` bên trong một widget con nằm sâu có thể gây ảnh hưởng đến hiệu năng render?
*Phân tích:*
`Theme.of(context)` đăng ký widget hiện tại vào danh sách phụ thuộc của `_InheritedTheme`. Khi `ThemeData` thay đổi (ví dụ khi chuyển từ Light sang Dark Mode), tất cả các widget đã từng gọi `Theme.of(context)` đều bị đánh dấu là `dirty` và buộc phải thực thi lại phương thức `build()`. Để tối ưu hóa, nên tách các khối giao diện tiêu tốn nhiều chi phí render thành các widget nhỏ độc lập (Leaf Widgets), giúp giới hạn phạm vi rebuild trong phạm vi hẹp nhất có thể.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho cấu trúc `MaterialApp` được cấu hình như sau:

```dart
MaterialApp(
  themeMode: ThemeMode.system,
  theme: ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(
      seedColor: Colors.blue, 
      brightness: Brightness.light,
    ),
  ),
  // darkTheme KHÔNG ĐƯỢC KHAI BÁO
  home: const Scaffold(body: DemoContent()),
)
```

Giả sử:
1. Ban đầu hệ điều hành đang ở chế độ sáng (`Brightness.light`).
2. Người dùng kéo thanh thông báo của điện thoại xuống và bật tùy chọn **Chế độ tối (Dark Mode)** của hệ thống.

#### Yêu cầu phân tích:
1. Sự kiện gì xảy ra tại tầng `PlatformDispatcher`?
2. Phương thức giải quyết theme của `MaterialApp` sẽ trả về `ThemeData` nào sau khi sự kiện diễn ra?
3. Giao diện của ứng dụng có chuyển sang màu tối hay không? Giải thích nguyên nhân kỹ thuật.

---

#### Kết quả phân tích kỹ thuật:

1. **Sự kiện tại `PlatformDispatcher`:**
   - Hệ điều hành gửi thông báo cập nhật cấu hình hệ thống.
   - `PlatformDispatcher.instance.onPlatformBrightnessChanged` kích hoạt.
   - Thuộc tính `PlatformDispatcher.instance.platformBrightness` chuyển trạng thái từ `Brightness.light` sang **`Brightness.dark`**.

2. **Kết quả giải quyết theme của `MaterialApp`:**
   - `MaterialApp` nhận thấy `themeMode == ThemeMode.system`.
   - Hệ thống kiểm tra: `platformBrightness == Brightness.dark` $\to$ Cần tìm kiếm đối tượng gán cho thuộc tính `darkTheme`.
   - Do thuộc tính `darkTheme` không được khai báo trong mã nguồn (nhận giá trị `null`), phương thức giải quyết theme nội bộ của Flutter sẽ kích hoạt cơ chế dự phòng: **Sử dụng thuộc tính `theme` (Light Theme) làm kết quả mặc định**.

3. **Hiện tượng trên giao diện:**
   - Giao diện của ứng dụng **hoàn toàn KHÔNG chuyển sang màu tối**, mà vẫn giữ nguyên màu nền trắng sáng của `theme`.
   - **Nguyên nhân kỹ thuật**: Lập trình viên quên khai báo thuộc tính `darkTheme`. Dù hệ thống nhận biết chính xác người dùng đang bật Dark Mode, nhưng do thiếu cấu hình bảng màu tối tương ứng, framework buộc phải fallback về theme sáng để tránh sập ứng dụng.
