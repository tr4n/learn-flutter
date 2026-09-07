# Bài 3.2 — Vòng Đời Đầy Đủ của StatefulWidget

## Phần 1 — Khái Niệm & Mục Tiêu Bài Học

### Tại sao bài này quan trọng?

Lỗi memory leak phổ biến nhất trong Flutter:

```dart
class _MyState extends State<MyWidget> {
  StreamSubscription? _sub;

  @override
  void initState() {
    super.initState();
    _sub = someStream.listen((_) { /* ... */ }); // Subscribe
  }
  // QUÊN dispose() → Stream vẫn chạy sau khi widget unmount!
  // → Memory leak + setState after dispose
}
```

Hiểu đầy đủ lifecycle giúp bạn:
- Biết *đúng chỗ* để khởi tạo và giải phóng tài nguyên
- Tránh "calling setState after dispose" error
- Handle widget config change đúng cách qua `didUpdateWidget`
- Dùng InheritedWidget correctly qua `didChangeDependencies`

### Bạn sẽ hiểu được sau bài này:
- 7 phase lifecycle theo thứ tự đúng
- `super.initState()` phải là dòng đầu tiên, `super.dispose()` là dòng cuối
- `didUpdateWidget` vs `didChangeDependencies` — khác nhau như thế nào
- Implement stream subscription lifecycle hoàn chỉnh

---

## Phần 2 — Cơ Chế Hoạt Động (Under the Hood)

### Full State Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Mounted: Widget được insert vào tree

    state Mounted {
        [*] --> initState: StatefulElement mount State
        initState --> didChangeDependencies: InheritedWidget setup
        didChangeDependencies --> build: Render UI
        build --> Alive: Trạng thái bình thường

        state Alive {
            [*] --> Ready
            Ready --> setState: User interaction / Timer
            setState --> build: mark dirty → rebuild
            build --> Ready

            Ready --> didUpdateWidget: Parent truyền config mới
            didUpdateWidget --> build

            Ready --> didChangeDependencies: InheritedWidget thay đổi
            didChangeDependencies --> build
        }
    }

    Mounted --> Deactivated: Widget bị remove tạm thời
    Deactivated --> Mounted: Widget được insert lại (GlobalKey reparent)
    Deactivated --> Disposed: Element unmount

    state Disposed {
        dispose: Giải phóng tài nguyên
        [*] --> dispose
    }

    Disposed --> [*]
```

### Thứ tự gọi super()

```
initState():
  super.initState()  ← PHẢI là dòng ĐẦU TIÊN
  _myInit()

dispose():
  _myCleanup()
  super.dispose()    ← PHẢI là dòng CUỐI CÙNG

Lý do: Flutter framework cần setup/teardown trước khi code của bạn chạy
```

---

## Phần 3 — Code Mẫu Chuẩn Google

### 3.1 — Template đầy đủ lifecycle

```dart
class FullLifecycleWidget extends StatefulWidget {
  final String userId;
  final bool isActive;

  const FullLifecycleWidget({
    super.key,
    required this.userId,
    required this.isActive,
  });

  @override
  State<FullLifecycleWidget> createState() => _FullLifecycleWidgetState();
}

class _FullLifecycleWidgetState extends State<FullLifecycleWidget> {
  // Tài nguyên cần cleanup
  StreamSubscription<UserData>? _userSub;
  Timer? _refreshTimer;
  late final TextEditingController _searchController;

  // State data
  UserData? _userData;
  String? _errorMessage;

  // ─── PHASE 1: initState ───────────────────────────────────────────────────
  @override
  void initState() {
    super.initState(); // BẮT BUỘC là dòng đầu tiên

    // ✅ Setup tài nguyên không cần context/InheritedWidget
    _searchController = TextEditingController();

    // ✅ Subscribe stream (với widget.userId có sẵn)
    _subscribeToUser(widget.userId);

    // ✅ Timer
    if (widget.isActive) _startRefreshTimer();

    // ❌ KHÔNG dùng Theme.of(context), Navigator.of(context)
    // ❌ KHÔNG gọi setState
    debugPrint('initState: userId=${widget.userId}');
  }

  // ─── PHASE 2: didChangeDependencies ──────────────────────────────────────
  @override
  void didChangeDependencies() {
    super.didChangeDependencies(); // Gọi trước

    // ✅ Dùng context để lấy InheritedWidget
    // Được gọi sau initState VÀ mỗi khi dependency (InheritedWidget) thay đổi
    final locale = Localizations.localeOf(context);
    debugPrint('didChangeDependencies: locale=$locale');

    // ✅ Có thể gọi setState hoặc update state ở đây
  }

  // ─── PHASE 3: build ───────────────────────────────────────────────────────
  @override
  Widget build(BuildContext context) {
    // Build phải là pure function của state + context
    // Không có side effects trong build!
    return switch ((_userData, _errorMessage)) {
      (final data?, null) => _buildContent(data),
      (null, final err?) => _buildError(err),
      _ => const CircularProgressIndicator(),
    };
  }

  Widget _buildContent(UserData data) => Column(
    children: [Text(data.name), Text(data.email)],
  );

  Widget _buildError(String error) => Text('Error: $error',
      style: const TextStyle(color: Colors.red));

  // ─── PHASE 4: didUpdateWidget ─────────────────────────────────────────────
  @override
  void didUpdateWidget(FullLifecycleWidget oldWidget) {
    super.didUpdateWidget(oldWidget); // Gọi trước

    // Được gọi khi parent rebuild truyền config mới
    // oldWidget = config cũ, widget = config mới (đã update)

    if (oldWidget.userId != widget.userId) {
      // userId thay đổi → cần unsubscribe cũ, subscribe mới
      _userSub?.cancel();
      _subscribeToUser(widget.userId);
    }

    if (oldWidget.isActive != widget.isActive) {
      if (widget.isActive) {
        _startRefreshTimer();
      } else {
        _refreshTimer?.cancel();
        _refreshTimer = null;
      }
    }
  }

  // ─── PHASE 5: deactivate ─────────────────────────────────────────────────
  @override
  void deactivate() {
    // Được gọi khi widget bị remove khỏi tree TẠMTHỜI
    // (ví dụ: GlobalKey reparenting)
    // Ít gặp — không cần override thường xuyên
    debugPrint('deactivate');
    super.deactivate();
  }

  // ─── PHASE 6: dispose ─────────────────────────────────────────────────────
  @override
  void dispose() {
    // Giải phóng tài nguyên TRƯỚC super.dispose()
    _userSub?.cancel();
    _refreshTimer?.cancel();
    _searchController.dispose();
    // Không gọi setState sau đây!

    super.dispose(); // BẮT BUỘC là dòng cuối cùng
    debugPrint('dispose: widget đã bị unmount');
  }

  // ─── Helper methods ───────────────────────────────────────────────────────
  void _subscribeToUser(String userId) {
    _userSub = UserRepository().watchUser(userId).listen(
      (data) {
        if (!mounted) return; // Guard sau async
        setState(() => _userData = data);
      },
      onError: (error) {
        if (!mounted) return;
        setState(() => _errorMessage = error.toString());
      },
    );
  }

  void _startRefreshTimer() {
    _refreshTimer = Timer.periodic(
      const Duration(minutes: 5),
      (_) {
        if (!mounted) return;
        // Refresh logic
      },
    );
  }
}
```

### 3.2 — Stream Subscription Lifecycle đúng cách

```dart
// Template chuẩn cho bất kỳ widget nào cần subscribe stream
class StreamConsumerWidget extends StatefulWidget {
  final Stream<List<Message>> messageStream;
  const StreamConsumerWidget({super.key, required this.messageStream});
  @override State<StreamConsumerWidget> createState() => _StreamConsumerState();
}

class _StreamConsumerState extends State<StreamConsumerWidget> {
  StreamSubscription<List<Message>>? _subscription;
  List<Message> _messages = [];

  @override
  void initState() {
    super.initState();
    _subscribe(widget.messageStream);
  }

  @override
  void didUpdateWidget(StreamConsumerWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    // Stream reference thay đổi → unsubscribe cũ, subscribe mới
    if (oldWidget.messageStream != widget.messageStream) {
      _subscription?.cancel();
      _subscribe(widget.messageStream);
    }
  }

  void _subscribe(Stream<List<Message>> stream) {
    _subscription = stream.listen(
      (messages) {
        if (!mounted) return;
        setState(() => _messages = messages);
      },
      onError: (e) => debugPrint('Stream error: $e'),
      cancelOnError: false, // Không cancel khi có error (reconnect tự động)
    );
  }

  @override
  void dispose() {
    _subscription?.cancel(); // QUAN TRỌNG: cancel trước super.dispose()
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: _messages.length,
      itemBuilder: (_, i) => MessageBubble(message: _messages[i]),
    );
  }
}
```

---

## Phần 4 — Lỗi Sai Phổ Biến & Best Practices

### ❌ Anti-pattern 1: `super.initState()` không phải dòng đầu

```dart
// ❌ Sai: Setup trước super() → framework chưa sẵn sàng
@override
void initState() {
  _controller = AnimationController(vsync: this); // vsync dùng TickerProvider
  super.initState(); // ❌ TickerProvider chưa được setup!
}

// ✅ Đúng
@override
void initState() {
  super.initState(); // Framework setup trước
  _controller = AnimationController(vsync: this); // Sau đó mới dùng
}
```

### ❌ Anti-pattern 2: `super.dispose()` không phải dòng cuối

```dart
// ❌ Sai: cleanup sau super() → framework đã teardown
@override
void dispose() {
  super.dispose();         // Framework teardown xong
  _controller.dispose();  // ❌ Framework state không còn valid
}

// ✅ Đúng
@override
void dispose() {
  _controller.dispose();  // Cleanup trước
  _subscription?.cancel();
  super.dispose();         // Framework teardown cuối
}
```

### ❌ Anti-pattern 3: setState sau dispose

```dart
// ❌ Sai: Callback từ async operation có thể chạy sau dispose
class _BadState extends State<MyWidget> {
  @override
  void initState() {
    super.initState();
    fetchData().then((data) {
      // fetchData mất 5 giây. Trong thời gian đó widget có thể dispose!
      setState(() => _data = data); // 💥 setState after dispose
    });
  }
}

// ✅ Đúng: Check mounted trước setState
Future<void> _load() async {
  final data = await fetchData();
  if (!mounted) return; // Guard!
  setState(() => _data = data);
}
```

### ❌ Anti-pattern 4: Quên cleanup trong dispose

```dart
// ❌ Sai: Không cancel subscription
@override
void initState() {
  super.initState();
  someStream.listen((data) => setState(() => _data = data));
  // Không lưu subscription → không cancel được!
}

// ✅ Đúng
StreamSubscription? _sub;

@override
void initState() {
  super.initState();
  _sub = someStream.listen((data) {
    if (!mounted) return;
    setState(() => _data = data);
  });
}

@override
void dispose() {
  _sub?.cancel();
  super.dispose();
}
```

---

## Phần 5 — Bài Tập Củng Cố Tư Duy

### Challenge: Implement Stream Subscription Lifecycle Đúng Cách

**Yêu cầu:** Xây dựng `ChatScreen` với:
1. Subscribe Firebase Firestore stream của room messages
2. Khi `roomId` thay đổi (user chọn room khác) → unsubscribe cũ, subscribe mới
3. Hiển thị loading trong lúc chờ data đầu tiên
4. Handle error gracefully
5. Cancel subscription khi navigate away

**Template bắt đầu:**
```dart
class ChatScreen extends StatefulWidget {
  final String roomId; // Có thể thay đổi
  const ChatScreen({super.key, required this.roomId});
  @override State<ChatScreen> createState() => _ChatScreenState();
}
```

**Gợi ý:** Dùng `ConnectionState` của `AsyncSnapshot` để track loading state, hoặc quản lý state riêng với field `_isLoading`.

### Câu hỏi phỏng vấn liên quan:

1. **"Thứ tự của các lifecycle methods khi widget được mount lần đầu?"**
   - `createState()` → `initState()` → `didChangeDependencies()` → `build()`

2. **"Sự khác biệt giữa `didUpdateWidget` và `didChangeDependencies`?"**
   - `didUpdateWidget`: parent rebuild với Widget config mới (oldWidget khác widget)
   - `didChangeDependencies`: InheritedWidget mà widget phụ thuộc thay đổi giá trị

3. **"Tại sao `dispose()` cần cleanup trước `super.dispose()`?"**
   - Sau `super.dispose()`, framework teardown TickerProvider, binding, etc.
   - Nếu cleanup sau → đang dùng resources đã bị teardown → crash
