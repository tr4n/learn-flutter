# Bài 3.3: Memory Leak Investigation với Flutter DevTools

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~35 phút  
> **Yêu cầu**: Biết Dart lifecycle, Stream, AnimationController cơ bản

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: App mất 400MB RAM sau 30 phút

**News Reader app, iOS production:**

```
Crash report (Firebase Crashlytics):
Exception: Signal 9 (SIGKILL)
Reason: Memory pressure — jetsam killed app
Memory at crash: 1.2GB RSS

Timeline:
  t=0min:    App launch — Memory: 80MB
  t=5min:    User đọc 10 bài — Memory: 140MB  
  t=15min:   User đọc 30 bài — Memory: 320MB
  t=30min:   User đọc 60 bài — Memory: 780MB  ← CRASH IMMINENT
  t=31min:   SIGKILL by iOS jetsam

Rate: ~11MB/phút = Memory leak không được GC
```

**Câu hỏi**: Cái gì giữ 11MB mỗi phút không cho GC thu hồi?

---

## Phần 2 — Low-Level Mechanics

### 2.1. Dart GC — Khi nào object được thu hồi?

```
Dart dùng Generational Garbage Collector:

Nursery (New Space) — objects mới tạo:
  - Minor GC: nhanh (~1ms), chạy thường xuyên
  - Object tồn tại qua 2-3 Minor GC → promote sang Old Space

Old Space — objects sống lâu:
  - Major GC: chậm hơn (~10-50ms), chạy ít thường hơn
  - GC chạy khi Old Space gần đầy

Object bị thu hồi KHI: không còn root reference nào trỏ đến nó

Root references bao gồm:
- Local variables trong stack (function đang chạy)
- Static variables
- Global variables (getIt singletons)
- Event listeners đang active
- Closures capture variable
- Timer callbacks chưa cancel
```

### 2.2. Retaining Path — Tại sao object không được GC?

```
Memory Leak = Object không được GC vì còn ≥1 root trỏ đến

Ví dụ: ArticleBloc leak sau khi user rời màn hình

ROOT: App (tồn tại suốt app)
  └─► BlocProvider (ở widget tree gốc — KHÔNG được dispose)
        └─► ArticleBloc
              └─► StreamSubscription (EventChannel stream)
                    └─► ArticleList [50 Article objects]
                          └─► Article.imageData [~8MB each]

Retaining path:
  App → BlocProvider → ArticleBloc → StreamSubscription → ArticleList

→ ArticleBloc không bao giờ được GC vì BlocProvider gốc vẫn alive
→ StreamSubscription không được cancel vì Bloc không dispose
→ 50 Article (400MB) mắc kẹt theo ArticleBloc
```

### 2.3. 5 Pattern Leak Phổ Biến Nhất trong Flutter

```
LEAK PATTERN 1: Stream subscription không cancel
──────────────────────────────────────────────────
class ArticleScreen extends StatefulWidget { ... }

class _ArticleScreenState extends State<ArticleScreen> {
  // ❌ StreamSubscription không bao giờ cancel
  StreamSubscription? _sub;
  
  @override
  void initState() {
    super.initState();
    _sub = ArticleService.stream.listen((articles) {
      setState(() { this.articles = articles; });
    });
    // dispose() không cancel _sub → stream giữ State alive
  }
}

LEAK PATTERN 2: AnimationController không dispose
──────────────────────────────────────────────────
class _AnimatedHeaderState extends State<AnimatedHeader>
    with TickerProviderStateMixin {
  late AnimationController _controller; // ❌ Không gọi dispose()
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this, duration: 1.seconds);
    _controller.repeat(); // Ticker tick mãi mãi nếu không stop
  }
  // Thiếu @override void dispose() { _controller.dispose(); super.dispose(); }
}
// → Ticker vẫn chạy sau khi widget unmount → giữ State không GC

LEAK PATTERN 3: Timer không cancel
──────────────────────────────────────────────────
class _CountdownState extends State<Countdown> {
  Timer? _timer;
  
  @override
  void initState() {
    super.initState();
    _timer = Timer.periodic(
      const Duration(seconds: 1),
      (timer) => setState(() => _seconds--),
    );
    // ❌ Timer.periodic chạy mãi, callback giữ _CountdownState alive
  }
  // Thiếu dispose: _timer?.cancel();
}

LEAK PATTERN 4: GlobalKey giữ Widget alive
──────────────────────────────────────────────────
// ❌ GlobalKey lưu trữ ở static field → Element không bao giờ được GC
class AppKeys {
  static final scaffoldKey = GlobalKey<ScaffoldState>();
  static final formKey = GlobalKey<FormState>();
  // GlobalKey giữ strong reference đến Element và State
}

LEAK PATTERN 5: Closure capture context
──────────────────────────────────────────────────
class _ImageViewerState extends State<ImageViewer> {
  @override
  void initState() {
    super.initState();
    // ❌ Closure capture `this` (State object)
    // imageLoadingService giữ callback → giữ State alive
    ImageLoadingService.instance.addListener(
      (ImageEvent event) {
        if (event.url == widget.url) {
          setState(() { /* update */ }); // 'this' captured
        }
      }
    );
    // Không có removeListener khi dispose → State không GC
  }
}
```

---

## Phần 3 — Production Code Implementation

### 3.1. Sử dụng DevTools Memory Tab step-by-step

```
Bước 1: Kết nối DevTools
  flutter run --debug
  → Mở DevTools → Memory tab
  
Bước 2: Thiết lập baseline
  - App mới launch, chưa navigate
  - Click "GC" button (thủ công trigger GC)
  - Click "Snapshot" → lưu snapshot A
  
Bước 3: Reproduce scenario
  - Navigate đến ArticleList screen
  - Đọc 10 bài (scroll + tap)
  - Pop về màn hình trước (unmount ArticleScreen)
  - Click "GC" button
  - Đợi 5 giây (GC settle)
  - Click "Snapshot" → lưu snapshot B

Bước 4: So sánh
  - Snapshot B - Snapshot A = leaked objects
  - Filter: tìm class liên quan đến Article, Bloc, Stream

Bước 5: Retaining Path
  - Click vào object suspect (ví dụ: ArticleBloc)
  - "Retaining path" tab → thấy đường đi từ root đến object
  - Đây là nguyên nhân: ai đang giữ ArticleBloc alive?
```

### 3.2. Tích hợp `leak_tracker` vào CI

```yaml
# pubspec.yaml
dev_dependencies:
  leak_tracker_flutter_testing: ^3.0.8
```

```dart
// test/helpers/leak_testing_setup.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:leak_tracker_flutter_testing/leak_tracker_flutter_testing.dart';

void setupLeakTracking() {
  LeakTracking.warnForNotGCed = false; // Chỉ fail cho leaked objects
  LeakTracking.failForNotGCed = true;  // CI fail nếu có leak
  
  LeakTracking.ignore = LeakTracking.ignore.copyWith(
    // Ignore known false-positives (third-party packages)
    ignoredLeaks: IgnoredLeaks(
      experimentalNotGCed: {
        'NetworkImageCache', // Hive cache — intentionally kept alive
      },
    ),
  );
}

// test/widget_test.dart
void main() {
  setupLeakTracking();
  
  testWidgets(
    'ArticleScreen disposes all resources correctly',
    (tester) async {
      // LeakTracking tự động track objects trong test này
      await tester.pumpWidget(
        const MaterialApp(home: ArticleScreen()),
      );
      
      // Simulate user interaction
      await tester.tap(find.byType(ArticleCard).first);
      await tester.pumpAndSettle();
      
      // Pop về screen trước — trigger dispose
      await tester.pageBack();
      await tester.pumpAndSettle();
      
      // LeakTracking sẽ fail test nếu ArticleBloc hoặc Stream không được GC
    },
    // timeout 60 giây để GC có thời gian chạy
  );
}
```

### 3.3. Allocation Tracing — Tìm object được tạo nhiều nhất

```dart
// Bật Allocation Tracing trong DevTools:
// Memory tab → "Allocation Tracing" → Toggle ON
// Thực hiện scenario → Toggle OFF
// Xem: class nào được instantiate nhiều nhất?

// Ví dụ phát hiện: ArticleDto được tạo 10,000 lần trong 5 phút
// → Investigate: tại sao?

// Code gây ra: JSON decode trong build() → mỗi rebuild → new ArticleDto
class ArticleList extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: articles.length,
      itemBuilder: (context, index) {
        // ❌ Decode JSON mỗi lần build() gọi
        final dto = ArticleDto.fromJson(articles[index]); // Allocation!
        return ArticleCard(article: dto.toEntity());
      },
    );
  }
}

// ✅ Fix: Pre-decode trong State hoặc UseCase, không trong build()
class _ArticleListState extends State<ArticleList> {
  late List<Article> _articles;
  
  @override
  void initState() {
    super.initState();
    // Decode một lần khi mount
    _articles = widget.rawArticles
        .map((json) => ArticleDto.fromJson(json).toEntity())
        .toList();
  }
  
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: _articles.length,
      itemBuilder: (context, index) => ArticleCard(article: _articles[index]),
    );
  }
}
```

### 3.4. Fix Pattern hoàn chỉnh cho tất cả 5 leak types

```dart
// lib/core/mixins/disposable_mixin.dart
// Helper mixin để không quên dispose

mixin DisposableMixin<T extends StatefulWidget> on State<T> {
  final List<StreamSubscription> _subscriptions = [];
  final List<AnimationController> _controllers = [];
  final List<Timer> _timers = [];
  final List<ChangeNotifier> _notifiers = [];

  /// Đăng ký subscription — tự động cancel khi dispose
  void registerSubscription(StreamSubscription sub) {
    _subscriptions.add(sub);
  }

  /// Đăng ký controller — tự động dispose khi widget unmount
  void registerController(AnimationController controller) {
    _controllers.add(controller);
  }
  
  void registerTimer(Timer timer) => _timers.add(timer);
  
  void registerNotifier(ChangeNotifier notifier) {
    _notifiers.add(notifier);
  }

  @override
  void dispose() {
    for (final sub in _subscriptions) {
      sub.cancel();
    }
    for (final controller in _controllers) {
      controller.dispose();
    }
    for (final timer in _timers) {
      timer.cancel();
    }
    for (final notifier in _notifiers) {
      notifier.dispose();
    }
    super.dispose();
  }
}

// Sử dụng:
class _ArticleScreenState extends State<ArticleScreen>
    with DisposableMixin, TickerProviderStateMixin {
  
  @override
  void initState() {
    super.initState();
    
    // Đăng ký — tự động cancel khi dispose
    registerSubscription(
      ArticleService.stream.listen(_onArticles),
    );
    
    final controller = AnimationController(
      vsync: this, 
      duration: const Duration(milliseconds: 300),
    );
    registerController(controller);
    
    registerTimer(
      Timer.periodic(const Duration(seconds: 30), _autoRefresh),
    );
  }
  
  // Không cần override dispose() thủ công — DisposableMixin xử lý hết
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Kết quả sau khi fix leak trong News Reader app

```
TRƯỚC FIX:
  t=0min:   80MB
  t=5min:  140MB  (rate: +12MB/min)
  t=30min: 780MB  (CRASH)

SAU FIX (cancel subscriptions, dispose controllers):
  t=0min:   80MB
  t=5min:   95MB  (rate: +3MB/min — từ cache, intentional)
  t=30min: 145MB  (stable — GC hoạt động đúng)
  t=60min: 162MB  (GC giữ stable)

Memory saved: 618MB sau 30 phút sử dụng
Crash rate: từ 8.3% sessions → 0.01% sessions
```

### Chi phí của leak_tracker trong CI

```
Test suite: 150 widget tests + 30 integration tests

Không có leak_tracker:
  CI run time: 4m 12s

Với leak_tracker enabled:
  CI run time: 4m 38s (+26s = +10%)
  
Kết luận: +10% CI time để đổi lấy 0 memory leak ở production
→ Chi phí chấp nhận được
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ StreamSubscription được tạo trong initState() nhưng không cancel trong dispose()
    ✅ Luôn: sub?.cancel() trong dispose() hoặc dùng DisposableMixin
    Lý do: Stream giữ closure → giữ State → giữ toàn bộ widget subtree

[ ] ❌ AnimationController được tạo nhưng không dispose()
    ✅ Luôn: controller.dispose() trong dispose()
    Lý do: Ticker vẫn nhận VSync → giữ TickerProvider → giữ State alive

[ ] ❌ Timer.periodic không có cancel()
    ✅ Lưu timer reference: _timer = Timer.periodic(...)
    ✅ Cancel trong dispose(): _timer?.cancel()
    Lý do: Timer callback capture context → prevent GC

[ ] ❌ Không có leak_tracker trong CI pipeline
    ✅ Tích hợp leak_tracker với threshold = 0 leaked objects
    Lý do: Leak phát triển từ từ — không detect sớm sẽ crash production

[ ] ❌ Dùng GlobalKey static cho màn hình dynamic
    ✅ Chỉ dùng GlobalKey khi thực sự cần (form validation, scaffold drawer)
    ✅ Dispose GlobalKey khi không dùng nữa bằng cách xóa khỏi widget tree
    Lý do: GlobalKey giữ strong reference đến Element + State

[ ] ❌ ChangeNotifier.addListener() không có removeListener() tương ứng
    ✅ Luôn pair: addListener() trong initState() ↔ removeListener() trong dispose()
    Lý do: Listener closure giữ State alive sau widget unmount

[ ] ❌ Investigate leak bằng cách đọc code, không dùng DevTools
    ✅ Dùng Retaining path trong DevTools để xác định chính xác nguyên nhân
    Lý do: Leak ẩn trong third-party package hoặc closure deep nested — khó đọc code

[ ] ❌ Chỉ test memory trên Debug mode
    ✅ Memory profile trên Profile mode: flutter run --profile
    Lý do: Debug build có nhiều object assertion, không phản ánh production
```
