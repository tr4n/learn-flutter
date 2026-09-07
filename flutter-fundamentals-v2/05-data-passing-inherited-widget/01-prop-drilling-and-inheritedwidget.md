# Bài 5.1 — Prop Drilling & InheritedWidget

## Phần 1 — Khái Niệm & Mục Tiêu Bài Học

### Tại sao bài này quan trọng?

Mọi app thực tế đều có data cần chia sẻ: user profile, theme, cart, locale... Cách "đơn giản" — truyền qua constructor — nhanh chóng trở nên không bền vững:

```dart
// Prop Drilling: truyền theme qua 5 tầng widget
class App → HomePage(theme) → BodySection(theme) → ContentRow(theme) → ProductCard(theme) → PriceTag(theme)
// Thay đổi theme → sửa 5 chỗ
// Thêm tầng widget mới → thêm parameter ở giữa chain
```

`InheritedWidget` giải quyết vấn đề này bằng cách "broadcast" data xuống toàn bộ subtree, cho phép any descendant tham chiếu trực tiếp không cần qua intermediaries.

### Bạn sẽ hiểu được sau bài này:
- Prop Drilling — vấn đề và hạn chế
- `InheritedWidget` — broadcast data pattern
- `updateShouldNotify()` — selective rebuild
- Xây `ThemeProvider` đơn giản từ InheritedWidget

---

## Phần 2 — Cơ Chế Hoạt Động (Under the Hood)

### InheritedWidget Broadcast

```mermaid
graph TB
    IW["InheritedWidget\n(data: theme)"]
    A["Widget A"]
    B["Widget B"]
    C["Widget C\n★ subscribed"]
    D["Widget D"]
    E["Widget E\n★ subscribed"]

    IW --> A --> B --> C
    A --> D --> E

    IW -->|"Khi data thay đổi\nchỉ C và E rebuild"| C
    IW -->|""| E

    style C fill:#90EE90
    style E fill:#90EE90
    style A fill:#f5f5f5
    style B fill:#f5f5f5
    style D fill:#f5f5f5
```

### `dependOnInheritedWidgetOfExactType` — Đăng ký dependency

```dart
// Khi widget gọi:
final theme = context.dependOnInheritedWidgetOfExactType<MyTheme>();
// Flutter ghi nhớ: widget này phụ thuộc vào MyTheme

// Khi MyTheme.updateShouldNotify() → true:
// Flutter tìm tất cả widget đã register dependency
// → Rebuild chỉ những widget đó
```

---

## Phần 3 — Code Mẫu Chuẩn Google

### 3.1 — Vấn đề Prop Drilling

```dart
// ❌ Prop Drilling — verbose và brittle
class AppSettings {
  final String locale;
  final bool isDarkMode;
  const AppSettings({required this.locale, required this.isDarkMode});
}

// Phải truyền settings qua TẤT CẢ các tầng
class HomePage extends StatelessWidget {
  final AppSettings settings; // Nhận để truyền xuống
  const HomePage({super.key, required this.settings});

  @override
  Widget build(BuildContext context) => BodySection(settings: settings); // Truyền tiếp
}

class BodySection extends StatelessWidget {
  final AppSettings settings;
  const BodySection({super.key, required this.settings});

  @override
  Widget build(BuildContext context) => ContentRow(settings: settings); // Truyền tiếp
}

class ContentRow extends StatelessWidget {
  final AppSettings settings;
  const ContentRow({super.key, required this.settings});

  @override
  Widget build(BuildContext context) => ProductCard(settings: settings); // Truyền tiếp
}

class ProductCard extends StatelessWidget {
  final AppSettings settings;
  const ProductCard({super.key, required this.settings});

  @override
  Widget build(BuildContext context) {
    // Chỉ đây mới THỰC SỰ dùng settings!
    return Text(
      'Price',
      style: TextStyle(color: settings.isDarkMode ? Colors.white : Colors.black),
    );
  }
}
```

### 3.2 — InheritedWidget — Giải pháp

```dart
// InheritedWidget: broadcast data xuống toàn bộ subtree
class AppSettingsData extends InheritedWidget {
  final AppSettings settings;

  const AppSettingsData({
    super.key,
    required this.settings,
    required super.child, // Subtree nhận data này
  });

  // Convenience accessor: gọi từ bất kỳ descendant nào
  static AppSettingsData of(BuildContext context) {
    // dependOn: đăng ký dependency → widget này sẽ rebuild khi data thay đổi
    final result = context.dependOnInheritedWidgetOfExactType<AppSettingsData>();
    assert(result != null, 'AppSettingsData không được tìm thấy trong ancestor tree!');
    return result!;
  }

  // maybeOf: trả về null nếu không tìm thấy (optional dependency)
  static AppSettingsData? maybeOf(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<AppSettingsData>();
  }

  // updateShouldNotify: Flutter gọi khi InheritedWidget mới được tạo
  // return true → rebuild tất cả subscribers
  // return false → không rebuild (data không thay đổi)
  @override
  bool updateShouldNotify(AppSettingsData oldWidget) {
    // Chỉ rebuild nếu settings thực sự thay đổi
    return settings != oldWidget.settings;
  }
}

// Sử dụng: Đặt InheritedWidget ở ancestor chung
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return AppSettingsData(
      settings: const AppSettings(locale: 'vi', isDarkMode: false),
      child: MaterialApp(home: HomePage()),
    );
  }
}

// Bất kỳ descendant nào đều có thể access settings
class ProductCard extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Không cần truyền settings qua constructor!
    final settings = AppSettingsData.of(context).settings;
    return Text(
      'Giá',
      style: TextStyle(color: settings.isDarkMode ? Colors.white : Colors.black),
    );
  }
}
```

### 3.3 — ThemeProvider với InheritedWidget + StatefulWidget

```dart
// Cần StatefulWidget để quản lý mutable state (theme có thể thay đổi)
// InheritedWidget để broadcast state đó xuống tree

class ThemeProvider extends StatefulWidget {
  final Widget child;
  const ThemeProvider({super.key, required this.child});

  // Convenience: access settings từ descendant
  static ThemeData themeOf(BuildContext context) {
    return _InheritedTheme.of(context).theme;
  }

  static void toggle(BuildContext context) {
    context.findAncestorStateOfType<_ThemeProviderState>()?.toggleTheme();
  }

  @override
  State<ThemeProvider> createState() => _ThemeProviderState();
}

class _ThemeProviderState extends State<ThemeProvider> {
  bool _isDark = false;

  ThemeData get _theme => _isDark ? ThemeData.dark(useMaterial3: true)
                                  : ThemeData.light(useMaterial3: true);

  void toggleTheme() => setState(() => _isDark = !_isDark);

  @override
  Widget build(BuildContext context) {
    // Khi _isDark thay đổi → setState → rebuild → InheritedWidget mới
    // → updateShouldNotify() kiểm tra → rebuild subscribers
    return _InheritedTheme(
      theme: _theme,
      child: widget.child,
    );
  }
}

class _InheritedTheme extends InheritedWidget {
  final ThemeData theme;

  const _InheritedTheme({required this.theme, required super.child});

  static _InheritedTheme of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<_InheritedTheme>()!;
  }

  @override
  bool updateShouldNotify(_InheritedTheme oldWidget) {
    return theme != oldWidget.theme;
  }
}

// Dùng:
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ThemeProvider(
      child: Builder(
        builder: (context) {
          return MaterialApp(
            theme: ThemeProvider.themeOf(context),
            home: const HomeScreen(),
          );
        },
      ),
    );
  }
}

// Toggle theme từ bất kỳ đâu:
ElevatedButton(
  onPressed: () => ThemeProvider.toggle(context),
  child: const Text('Toggle Theme'),
)
```

---

## Phần 4 — Lỗi Sai Phổ Biến & Best Practices

### ❌ Anti-pattern 1: Quên static `of()` accessor

```dart
// ❌ Khó dùng: caller phải biết rõ type và cách gọi
context.dependOnInheritedWidgetOfExactType<MyInheritedWidget>()!.data;

// ✅ Đúng: Cung cấp convenience static method
class MyInheritedWidget extends InheritedWidget {
  // ...
  static MyData of(BuildContext context) =>
      context.dependOnInheritedWidgetOfExactType<MyInheritedWidget>()!.data;
}

// Gọi: final data = MyInheritedWidget.of(context);
```

### ❌ Anti-pattern 2: `updateShouldNotify` luôn `true`

```dart
// ❌ Quá aggressive: rebuild mọi subscriber mỗi khi parent rebuild
@override
bool updateShouldNotify(MyWidget old) => true;
// → Mỗi frame parent rebuild → toàn bộ subscriber rebuild

// ✅ Đúng: Chỉ rebuild khi data thực sự thay đổi
@override
bool updateShouldNotify(AppSettingsData old) =>
    settings != old.settings; // Dùng == operator
```

### ❌ Anti-pattern 3: InheritedWidget cho data thay đổi thường xuyên

```dart
// ❌ Không phù hợp: scroll position thay đổi 60fps → 60 rebuild/s
class ScrollPositionInherited extends InheritedWidget {
  final double scrollOffset; // Thay đổi mỗi frame!
  // → Mọi subscriber rebuild 60fps
}

// ✅ Đúng: InheritedWidget cho data thay đổi ít (theme, locale, user)
// Với data thay đổi nhiều → ValueNotifier + AnimatedBuilder/ListenableBuilder
```

---

## Phần 5 — Bài Tập Củng Cố Tư Duy

### Challenge: Xây ThemeProvider Đơn Giản

**Yêu cầu:**
1. `ThemeProvider` widget bọc ngoài app
2. Có thể toggle dark/light mode từ bất kỳ widget nào
3. Thay đổi theme → chỉ rebuild widget dùng theme, không rebuild toàn bộ app
4. Lưu preference vào `SharedPreferences` (persist qua app restart)

**Gợi ý:**
- InheritedWidget cho broadcast
- StatefulWidget cha cho mutable state
- `updateShouldNotify` compare theme object
- Load pref trong `initState`, save trong `toggleTheme`

### Câu hỏi phỏng vấn liên quan:

1. **"Prop Drilling là gì và tại sao nó là vấn đề?"**
   - Truyền data qua nhiều tầng widget trung gian không cần data đó
   - Vấn đề: verbose, brittle, khó refactor, coupling cao

2. **"InheritedWidget hoạt động như thế nào?"**
   - Đặt widget trong ancestor tree
   - Descendant gọi `dependOnInheritedWidgetOfExactType` để đăng ký dependency
   - Khi `updateShouldNotify` = true → chỉ rebuild registered dependents

3. **"`updateShouldNotify` trả về false có nghĩa là gì?"**
   - Subscribers KHÔNG được rebuild — data không thay đổi
   - Giúp skip rebuild không cần thiết → performance optimization
