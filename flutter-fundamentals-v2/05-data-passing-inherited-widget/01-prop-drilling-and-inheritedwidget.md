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

### Câu Hỏi Phỏng Vấn

> **[Junior]** — nắm khái niệm | **[Middle]** — hiểu cơ chế | **[Senior]** — hiểu Flutter internals | **[Trace Code]** — đọc code và dự đoán output

---

#### Q1 [Junior] — "Prop Drilling là gì và tại sao nó là vấn đề?"

**Trả lời chuẩn:**

**Prop Drilling** xảy ra khi phải truyền data qua nhiều tầng widget trung gian không cần data đó — chỉ để chuyển data xuống cho widget sâu hơn.

```dart
// Prop drilling: UserProfile chỉ "trung chuyển" userId, không dùng
class HomePage extends StatelessWidget {
  final String userId;
  @override Widget build(context) => Column(children: [
    UserProfile(userId: userId),   // chỉ pass xuống
    const OtherContent(),
  ]);
}

class UserProfile extends StatelessWidget {
  final String userId;             // nhận nhưng không dùng — chỉ truyền tiếp
  @override Widget build(context) => UserAvatar(userId: userId);
}

class UserAvatar extends StatelessWidget {
  final String userId;             // widget thực sự cần data
  @override Widget build(context) => Image.network('api/avatar/$userId');
}
```

**Vấn đề:**
- **Verbose:** Mỗi tầng phải khai báo field và truyền prop
- **Brittle:** Thêm field mới → phải sửa tất cả các tầng trung gian
- **Coupling cao:** Widget trung gian biết về data structure mà nó không cần
- **Refactor khó:** Move widget sang vị trí khác → phải rewire prop chain

---

#### Q2 [Junior] — "`InheritedWidget` hoạt động như thế nào? Tại sao chỉ rebuild `registered dependents`?"

**Trả lời chuẩn:**

`InheritedWidget` là ancestor widget đặc biệt — descendants có thể "đăng ký" để nhận notification khi data thay đổi:

```dart
// 1. Define InheritedWidget
class UserData extends InheritedWidget {
  final String userId;
  const UserData({required this.userId, required super.child, super.key});

  static UserData of(BuildContext context) =>
      context.dependOnInheritedWidgetOfExactType<UserData>()!;

  @override
  bool updateShouldNotify(UserData old) => userId != old.userId;
}

// 2. Đặt trong ancestor tree
UserData(userId: 'u123', child: const App())

// 3. Descendant đọc và đăng ký dependency — không cần truyền qua trung gian
class UserAvatar extends StatelessWidget {
  @override Widget build(context) {
    final data = UserData.of(context); // đăng ký dependency
    return Image.network('api/avatar/${data.userId}');
  }
}
```

**Cơ chế "chỉ rebuild dependents":** Khi `UserData` được replace (e.g., userId thay đổi), Flutter gọi `updateShouldNotify()`. Nếu true → chỉ các Elements đã gọi `dependOnInheritedWidgetOfExactType<UserData>()` được mark dirty. Các widget khác không đăng ký → không rebuild.

---

#### Q3 [Middle] — "`updateShouldNotify` trả về false có nghĩa gì? Khi nào xảy ra?"

**Trả lời chuẩn:**

```dart
@override
bool updateShouldNotify(UserData old) => userId != old.userId;
```

Khi `false`: Flutter **bỏ qua** việc notify dependents — ngay cả khi InheritedWidget object mới được tạo.

**Khi nào xảy ra:** Parent của InheritedWidget rebuild → tạo `InheritedWidget` instance mới → Flutter gọi `updateShouldNotify(oldWidget)`. Nếu data không thực sự thay đổi (e.g., `userId` vẫn như cũ) → `false` → descendants không rebuild.

```dart
// Scenario: Parent rebuild nhưng userId không đổi
setState(() => _count++); // parent counter thay đổi, không liên quan userId

// Flutter:
// 1. Parent.build() → tạo UserData(userId: 'u123') MỚI (object mới)
// 2. Flutter gọi updateShouldNotify(oldUserData)
// 3. 'u123' != 'u123' → false → KHÔNG notify dependents
// → UserAvatar KHÔNG rebuild dù parent rebuild
```

**Tầm quan trọng:** `updateShouldNotify` là optimization cốt lõi — tránh cascade rebuild không cần thiết khi parent rebuild nhưng data không thay đổi.

---

#### Q4 [Senior] — "`InheritedWidget.updateShouldNotify()` được gọi khi nào chính xác? Dependency registration hoạt động thế nào?"

**Trả lời chuẩn:**

**Khi `updateShouldNotify` được gọi:**
1. `InheritedElement.update(newWidget)` được gọi khi parent rebuild
2. Trước khi update, Flutter gọi `updateShouldNotify(oldWidget)`
3. Nếu true → `InheritedElement.notifyClients()` → iterate `_dependents` map và mark chúng dirty

**Dependency registration mechanism:**
```dart
// Khi UserAvatar.build() gọi UserData.of(context):
T? dependOnInheritedWidgetOfExactType<T extends InheritedWidget>() {
  // 1. Walk up element tree để tìm InheritedElement
  final InheritedElement? ancestor = _inheritedElements[T];
  
  if (ancestor != null) {
    // 2. Đăng ký: thêm current element vào ancestor._dependents
    return dependOnInheritedElement(ancestor) as T;
  }
  return null;
}

// InheritedElement.updateDependencies()
void updateDependencies(Element dependent, Object? aspect) {
  _dependents[dependent] = null; // key = dependent element
}
```

**`_inheritedElements` cache:** Mỗi Element có một `Map<Type, InheritedElement>` được populate khi walk up tree — O(1) lookup sau lần đầu, không phải O(depth) mỗi lần.

---

#### Q5 [Middle] — "Tại sao `InheritedWidget` chỉ rebuild registered dependents, không phải toàn bộ subtree?"

**Trả lời chuẩn:**

`InheritedElement` duy trì `Map<Element, Object?> _dependents` — chỉ chứa elements đã gọi `dependOnInheritedWidgetOfExactType`. Khi notify, chỉ các elements trong map này được mark dirty:

```dart
// InheritedElement.notifyClients() — gọi khi updateShouldNotify = true
@override
void notifyClients(InheritedWidget oldWidget) {
  for (final Element dependent in _dependents.keys) {
    // Chỉ notify từng dependent đã đăng ký
    notifyDependent(oldWidget, dependent);
    // → dependent.didChangeDependencies()
    // → dependent.markNeedsBuild()
  }
  // Các widget khác trong subtree KHÔNG được notify
}
```

**Điểm quan trọng:** Nếu widget con trong subtree không gọi `dependOnInheritedWidgetOfExactType<UserData>()` (e.g., chỉ dùng `context.read()` hoặc không dùng UserData), nó **không nằm trong `_dependents`** → không rebuild dù UserData thay đổi.

---

#### Q6 [Middle] — "`InheritedWidget` với mutable state: vấn đề gì nếu mutate trực tiếp?"

**Trả lời chuẩn:**

`InheritedWidget` phải **immutable** (fields là `final`). Nếu mutate trực tiếp:

```dart
// ❌ Sai — mutate trực tiếp (giả sử không có final)
class BadData extends InheritedWidget {
  List<String> items; // không final
  
  void addItem(String item) {
    items.add(item); // mutate directly
    // updateShouldNotify KHÔNG được gọi — Flutter không biết có thay đổi
    // → Dependents KHÔNG rebuild → UI không update!
  }
}
```

**Tại sao không hoạt động:** `updateShouldNotify(oldWidget)` chỉ được gọi khi `InheritedWidget` được **replace** (parent rebuild tạo widget mới). Nếu mutate field của widget hiện tại, không có replacement → không có notify.

**Pattern đúng:** Kết hợp `StatefulWidget` (tạo widget mới) + `InheritedWidget` (expose data):

```dart
class DataProvider extends StatefulWidget {
  // Giữ State mutable, thay thế InheritedWidget khi state thay đổi
  @override State<DataProvider> createState() => _DataProviderState();
}

class _DataProviderState extends State<DataProvider> {
  List<String> _items = [];
  
  void addItem(String item) {
    setState(() {
      _items = [..._items, item]; // tạo list MỚI (immutable pattern)
    }); // → parent rebuild → InheritedWidget MỚI → updateShouldNotify → notify
  }
  
  @override Widget build(context) => DataInherited(items: _items, child: widget.child);
}
```

---

#### Q7 [Trace Code] — "Xác định widget nào rebuild khi `InheritedWidget` thay đổi"

```dart
class CounterData extends InheritedWidget {
  final int count;
  const CounterData({required this.count, required super.child, super.key});
  
  static CounterData of(BuildContext context) =>
      context.dependOnInheritedWidgetOfExactType<CounterData>()!;
  
  static CounterData read(BuildContext context) =>
      context.getInheritedWidgetOfExactType<CounterData>()!;

  @override
  bool updateShouldNotify(CounterData old) => count != old.count;
}

class WidgetA extends StatelessWidget {
  @override Widget build(context) {
    final count = CounterData.of(context).count; // dùng dependOn
    print('A build: $count');
    return Text('$count');
  }
}

class WidgetB extends StatelessWidget {
  @override Widget build(context) {
    print('B build');
    return const Text('Static B');
  }
}

class WidgetC extends StatelessWidget {
  @override Widget build(context) {
    CounterData.read(context); // dùng getInherited (không đăng ký dependency)
    print('C build');
    return const Text('C reads but not watches');
  }
}
```

**Khi count thay đổi từ 0 → 1:**

```
A build: 1
```

**Giải thích:**
- **WidgetA:** gọi `dependOnInheritedWidgetOfExactType` → registered trong `_dependents` → **rebuild** → "A build: 1"
- **WidgetB:** không tương tác với `CounterData` → **không rebuild** → không in gì
- **WidgetC:** gọi `getInheritedWidgetOfExactType` (không đăng ký dependency) → **không rebuild** → không in gì
- **Nếu có widget D dùng `const`:** Cũng không rebuild (same reason as B)
