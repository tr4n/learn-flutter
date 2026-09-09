# Bài 7.4 — Xây Dựng Giao Diện Bất Đồng Bộ: FutureBuilder & StreamBuilder

## Tài Liệu Tham Khảo Chính Thức
- [Flutter Documentation: FutureBuilder class](https://api.flutter.dev/flutter/widgets/FutureBuilder-class.html)
- [Flutter Documentation: StreamBuilder class](https://api.flutter.dev/flutter/widgets/StreamBuilder-class.html)
- [Dart Documentation: Asynchronous programming (Futures and Streams)](https://dart.dev/libraries/async/async-await)

---

## Phần 1 — Khái Niệm & Kiến Trúc Giao Diện Bất Đồng Bộ (Async UI Architecture)

### 1.1 — Vai Trò Của `FutureBuilder` và `StreamBuilder`

Trong Flutter, giao diện người dùng (UI) là một hàm thuần túy phản ánh trạng thái hiện tại (`UI = f(state)`). Khi tương tác với các nguồn dữ liệu bên ngoài (mạng Internet, cơ sở dữ liệu cục bộ, cảm biến thiết bị), dữ liệu được trả về dưới dạng bất đồng bộ:

1. **`FutureBuilder<T>`**:
   - Dùng cho các tác vụ bất đồng bộ diễn ra **một lần duy nhất (One-shot operation)**.
   - Khi tác vụ hoàn tất, nó trả về một giá trị duy nhất (`T`) hoặc ném ra một lỗi ngoại lệ (`Object`).
   - Ví dụ: Gửi HTTP GET request lấy danh sách bài viết, đọc tệp tin từ bộ nhớ, truy vấn cơ sở dữ liệu SQLite.

2. **`StreamBuilder<T>`**:
   - Dùng cho các luồng dữ liệu **liên tục phát ra nhiều giá trị theo thời gian (Ongoing event stream)**.
   - Giao diện người dùng sẽ tự động vẽ lại mỗi khi có một sự kiện dữ liệu mới được đẩy vào Stream.
   - Ví dụ: Kết nối WebSocket nhận tin nhắn chat thời gian thực, lắng nghe thay đổi vị trí từ GPS, lắng nghe sự kiện từ Firebase Firestore realtime database.

---

### 1.2 — Máy Trạng Thái `ConnectionState` và Đối Tượng `AsyncSnapshot`

Cả `FutureBuilder` và `StreamBuilder` đều chuyển đổi trạng thái của luồng bất đồng bộ thành một đối tượng `AsyncSnapshot<T>` thông qua máy trạng thái (State Machine):

```
┌────────────────────────────────────────────────────────────────────────┐
│ MÁY TRẠNG THÁI CONNECTIONSTATE                                         │
│                                                                        │
│                 ┌───────────────────────────┐                          │
│                 │   ConnectionState.none    │                          │
│                 └─────────────┬─────────────┘                          │
│                               │ Khởi tạo Future / Stream               │
│                               ▼                                        │
│                 ┌───────────────────────────┐                          │
│                 │  ConnectionState.waiting  │                          │
│                 └──────┬─────────────┬──────┘                          │
│                        │             │                                 │
│        (Chỉ với Stream)│             │ (Future hoàn thành /            │
│        Nhận event đầu  │             │  Stream đóng lại)               │
│                        ▼             ▼                                 │
│         ┌──────────────────────┐  ┌───────────────────────┐            │
│         │ ConnectionState.     │  │ ConnectionState.done  │            │
│         │ active               ├──┤                       │            │
│         └──────────────────────┘  └───────────────────────┘            │
│          (Tiếp tục nhận events)    (Có Data hoặc Error)                │
└────────────────────────────────────────────────────────────────────────┘
```

#### Các giá trị của `ConnectionState`:
- **`none`**: Chưa liên kết với Future hoặc Stream nào (khi thuộc tính `future` hoặc `stream` nhận giá trị `null`).
- **`waiting`**: Đang chờ kết quả từ Future hoặc đang chờ sự kiện dữ liệu đầu tiên được phát ra từ Stream.
- **`active`**: Đang tích cực kết nối với Stream và đã nhận được ít nhất một sự kiện dữ liệu (trạng thái này không bao giờ xuất hiện với `FutureBuilder`).
- **`done`**: Future đã hoàn tất (trả về kết quả hoặc lỗi), hoặc Stream đã được đóng lại (`Stream.close()`).

#### Cấu trúc của đối tượng `AsyncSnapshot<T>`:
```dart
class AsyncSnapshot<T> {
  final ConnectionState connectionState;
  final T? data;
  final Object? error;
  final StackTrace? stackTrace;

  bool get hasData => data != null;
  bool get hasError => error != null;
  T get requireData => data!; // Ném lỗi nếu data == null
}
```

---

## Phần 2 — Cơ Chế Hoạt Động Cốt Lõi (Under the Hood)

### 2.1 — Cơ Chế Hoạt Động Nội Bộ Của `FutureBuilder`

`FutureBuilder` là một `StatefulWidget`. Logic xử lý được quản lý bên trong lớp `_FutureBuilderState`:

```
┌────────────────────────────────────────────────────────────────────────┐
│ VÒNG ĐỜI NỘI BỘ CỦA _FUTUREBUILDERSTATE                                │
│                                                                        │
│ 1. initState()                                                         │
│    _subscribe(): Đăng ký lắng nghe Future được truyền vào              │
│    snapshot = AsyncSnapshot.waiting()                                  │
│      │                                                                 │
│      ▼                                                                 │
│ 2. didUpdateWidget(oldWidget)                                          │
│    if (oldWidget.future != widget.future) {                            │
│       _unsubscribe(); // Bỏ đăng ký future cũ                          │
│       _subscribe();   // Đăng ký future mới                            │
│       snapshot = snapshot.inState(ConnectionState.waiting);            │
│    }                                                                   │
│      │                                                                 │
│      ▼                                                                 │
│ 3. Future hoàn tất (then / catchError)                                 │
│    if (_activeCallbackIdentity == identity) {                          │
│       setState(() {                                                    │
│          snapshot = AsyncSnapshot.withData(ConnectionState.done, data);│
│       });                                                              │
│    }                                                                   │
└────────────────────────────────────────────────────────────────────────┘
```

> **Nguyên nhân gốc rễ của lỗi Rebuild liên tục**: Nếu `widget.future` được truyền vào từ một hàm gọi trực tiếp trong `build()` (ví dụ `future: api.fetchData()`), mỗi khi widget cha rebuild, một instance `Future` mới được tạo ra. Phương thức `didUpdateWidget()` nhận thấy `oldWidget.future != widget.future`, ngay lập tức hủy kết nối cũ, tạo kết nối mới và đưa snapshot trở về `ConnectionState.waiting`. Kết quả là giao diện rơi vào vòng lặp tải dữ liệu vô tận.

---

### 2.2 — Cơ Chế Hoạt Động Nội Bộ Của `StreamBuilder`

Khác với `Future`, một Stream cần một đối tượng quản lý đăng ký là `StreamSubscription<T>`. 

1. **Khi widget được gắn vào cây (`initState`)**:
   `_StreamBuilderBaseState` gọi hàm `widget.stream.listen()`, tạo ra một `StreamSubscription` và lưu tham chiếu nội bộ.
2. **Khi có dữ liệu mới phát ra (`onData`)**:
   Callback kích hoạt hàm `setState()` nội bộ, cập nhật `snapshot` sang trạng thái `ConnectionState.active` cùng với giá trị dữ liệu mới.
3. **Khi widget bị tháo khỏi cây (`dispose`)**:
   Hàm `dispose()` của state tự động gọi `_subscription?.cancel()`. Thao tác này giúp ngăn ngừa hiện tượng rò rỉ bộ nhớ (Memory Leak) mà lập trình viên không cần phải tự tay quản lý việc hủy subscription.

---

## Phần 3 — Triển Khai Thực Tế

### 3.1 — Triển Khai `FutureBuilder` Đúng Quy Cách

Quy tắc: Luôn đảm bảo Future có một tham chiếu ổn định (Stable Reference) bằng cách khởi tạo trong `initState()` hoặc sử dụng biến `late final Future`:

```dart
import 'package:flutter/material.dart';

class UserListScreen extends StatefulWidget {
  const UserListScreen({super.key});

  @override
  State<UserListScreen> createState() => _UserListScreenState();
}

class _UserListScreenState extends State<UserListScreen> {
  // 1. Khởi tạo một tham chiếu ổn định, không tạo lại ở mỗi lần build()
  late Future<List<String>> _usersFuture;

  @override
  void initState() {
    super.initState();
    _usersFuture = _fetchUsers();
  }

  Future<List<String>> _fetchUsers() async {
    // Giả lập cuộc gọi API bất đồng bộ
    await Future.delayed(const Duration(seconds: 1));
    return ['Nguyễn Văn A', 'Trần Thị B', 'Lê Văn C'];
  }

  void _reload() {
    // 2. Khi cần làm mới dữ liệu (Refresh), tạo Future mới bên trong setState()
    setState(() {
      _usersFuture = _fetchUsers();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Danh Sách Người Dùng'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: _reload,
          ),
        ],
      ),
      body: FutureBuilder<List<String>>(
        future: _usersFuture, // Tham chiếu ổn định
        builder: (context, snapshot) {
          // 3. Xử lý các trạng thái bằng Dart 3 Pattern Matching
          return switch (snapshot.connectionState) {
            ConnectionState.none => const Center(
                child: Text('Không có tác vụ nào được khởi tạo.'),
              ),
            ConnectionState.waiting => const Center(
                child: CircularProgressIndicator(),
              ),
            ConnectionState.active => const SizedBox.shrink(),
            ConnectionState.done => _buildDoneState(snapshot),
          };
        },
      ),
    );
  }

  Widget _buildDoneState(AsyncSnapshot<List<String>> snapshot) {
    // Luôn kiểm tra lỗi trước khi đọc dữ liệu
    if (snapshot.hasError) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.error_outline, size: 48, color: Colors.red),
            const SizedBox(height: 12),
            Text('Đã xảy ra lỗi: ${snapshot.error}'),
            const SizedBox(height: 12),
            ElevatedButton(
              onPressed: _reload,
              child: const Text('Thử lại'),
            ),
          ],
        ),
      );
    }

    final users = snapshot.data ?? [];
    if (users.isEmpty) {
      return const Center(child: Text('Danh sách trống.'));
    }

    return ListView.builder(
      itemCount: users.length,
      itemBuilder: (context, index) => ListTile(
        leading: CircleAvatar(child: Text('${index + 1}')),
        title: Text(users[index]),
      ),
    );
  }
}
```

---

### 3.2 — Triển Khai `StreamBuilder` Với Luồng Dữ Liệu Thời Gian Thực

Sử dụng `StreamBuilder` kết hợp với thuộc tính `initialData` để tránh hiện tượng màn hình trắng hoặc nhấp nháy giao diện khi chờ đợi sự kiện đầu tiên:

```dart
import 'dart:async';
import 'package:flutter/material.dart';

class TickerScreen extends StatefulWidget {
  const TickerScreen({super.key});

  @override
  State<TickerScreen> createState() => _TickerScreenState();
}

class _TickerScreenState extends State<TickerScreen> {
  late final StreamController<int> _counterController;
  late final Stream<int> _counterStream;
  Timer? _timer;
  int _count = 0;

  @override
  void initState() {
    super.initState();
    _counterController = StreamController<int>();
    _counterStream = _counterController.stream.asBroadcastStream();

    // Phát sự kiện mỗi giây một lần
    _timer = Timer.periodic(const Duration(seconds: 1), (_) {
      _count++;
      _counterController.add(_count);
    });
  }

  @override
  void dispose() {
    _timer?.cancel();
    _counterController.close();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Bộ Đếm Thời Gian Thực')),
      body: Center(
        child: StreamBuilder<int>(
          stream: _counterStream,
          initialData: 0, // Cung cấp giá trị khởi tạo để hiển thị ngay lập tức
          builder: (context, snapshot) {
            if (snapshot.hasError) {
              return Text('Lỗi: ${snapshot.error}');
            }

            final value = snapshot.requireData;
            return Text(
              'Giá trị hiện tại: $value',
              style: Theme.of(context).textTheme.headlineMedium,
            );
          },
        ),
      ),
    );
  }
}
```

---

### 3.3 — So Sánh Kiến Trúc: `FutureBuilder` vs Quản Lý Trạng Thái Phân Tầng

| Tiêu Chí | `FutureBuilder` / `StreamBuilder` | State Management (BLoC / Riverpod / ChangeNotifier) |
| :--- | :--- | :--- |
| **Phạm vi trạng thái (Scope)** | Cục bộ bên trong một Widget (Local State) | Toàn cục hoặc chia sẻ giữa nhiều màn hình (Shared State) |
| **Lưu trữ dữ liệu đệm (Cache)** | Không có cache — mất dữ liệu khi widget unmount | Lưu trữ trong Controller / Store độc lập với vòng đời UI |
| **Tách biệt logic (Separation)** | Logic gọi API gắn liền với Widget | Tách biệt hoàn toàn tầng UI và tầng Business Logic |
| **Khả năng kiểm thử (Unit Test)** | Khó kiểm thử độc lập (cần Widget Test) | Rất dễ viết Unit Test cho tầng Logic thuần khiết |
| **Khuyến nghị sử dụng** | Các tác vụ đơn giản, dữ liệu tĩnh, không chia sẻ | Nghiệp vụ chính của ứng dụng, giỏ hàng, xác thực người dùng |

---

## Phần 4 — Lỗi Kỹ Thuật Thường Gặp & Biện Pháp Khắc Phục (Pitfalls & Solutions)

### 4.1 — Khởi tạo trực tiếp `Future` bên trong phương thức `build()`

#### Mô tả vấn đề:
Truyền lời gọi hàm bất đồng bộ trực tiếp vào tham số `future`:

```dart
// Lỗi nghiêm trọng: Hàm fetchUser() bị gọi lại ở mỗi lần widget rebuild
@override
Widget build(BuildContext context) {
  return FutureBuilder<User>(
    future: apiService.fetchUser(), // Tạo Future mới liên tục!
    builder: (context, snapshot) => ...,
  );
}
```

#### Nguyên nhân kỹ thuật:
Mỗi khi Widget cha gọi `setState()`, bàn phím xuất hiện, hoặc màn hình xoay hướng, phương thức `build()` sẽ được thực thi lại. Việc tạo một `Future` mới làm cho `didUpdateWidget()` của `FutureBuilder` kích hoạt, hủy kết quả trước đó và đưa giao diện quay trở lại trạng thái `ConnectionState.waiting`. Điều này gây ra hiện tượng màn hình chớp nháy và gửi hàng loạt request dư thừa lên máy chủ.

#### Biện pháp khắc phục:
Luôn lưu trữ tham chiếu `Future` vào một biến trong `State` thông qua `initState()`.

---

### 4.2 — Đọc trực tiếp `snapshot.data!` mà không kiểm tra `hasError`

#### Mô tả vấn đề:
Kiểm tra `ConnectionState.done` và lập tức ép kiểu không null:

```dart
// Lỗi tiềm ẩn: Gây crash nếu Future hoàn thành với lỗi
if (snapshot.connectionState == ConnectionState.done) {
  return Text(snapshot.data!.name); // Ném Null Check Operator Used on a Null Value
}
```

#### Nguyên nhân kỹ thuật:
Khi Future kết thúc với một ngoại lệ (ví dụ lỗi mạng), `connectionState` vẫn chuyển sang `ConnectionState.done`, nhưng thuộc tính `snapshot.data` sẽ nhận giá trị `null`, trong khi `snapshot.error` chứa ngoại lệ.

#### Biện pháp khắc phục:
Luôn kiểm tra `snapshot.hasError` trước khi truy xuất `snapshot.data`:
```dart
if (snapshot.hasError) {
  return Text('Lỗi: ${snapshot.error}');
}
if (snapshot.hasData) {
  return Text(snapshot.data!.name);
}
```

---

### 4.3 — Nhầm lẫn rằng `FutureBuilder` có thể tự động ngắt kết nối mạng khi Widget bị hủy

#### Mô tả vấn đề:
Kỳ vọng rằng khi người dùng nhấn Back thoát khỏi trang, `FutureBuilder` sẽ tự động dừng tác vụ mạng đang chạy.

#### Nguyên nhân kỹ thuật:
Theo thiết kế của ngôn ngữ Dart, một `Future` không thể bị hủy ngang (non-cancellable). Khi `FutureBuilder` bị dispose, nó chỉ đơn giản là ngừng lắng nghe kết quả từ Future đó, nhưng tác vụ bất đồng bộ bên dưới vẫn tiếp tục thực thi cho đến khi hoàn tất, tiêu tốn băng thông và năng lượng của thiết bị.

#### Biện pháp khắc phục:
Sử dụng các cơ chế hỗ trợ hủy request ở tầng mạng như `CancelToken` (của `Dio`) kết hợp với hàm `dispose()` của `StatefulWidget`.

---

### 4.4 — Lắng nghe Stream đơn (Single-subscription Stream) nhiều lần

#### Mô tả vấn đề:
Truyền cùng một `Stream` thông thường vào nhiều `StreamBuilder` khác nhau:
```dart
final Stream<int> stream = controller.stream; // Mặc định là single-subscription

StreamBuilder(stream: stream, ...);
StreamBuilder(stream: stream, ...); // Ném lỗi: Bad state: Stream has already been listened to
```

#### Nguyên nhân kỹ thuật:
Mặc định, `Stream` trong Dart chỉ cho phép duy nhất một listener đăng ký tại một thời điểm.

#### Biện pháp khắc phục:
Chuyển đổi Stream thành broadcast stream thông qua phương thức `.asBroadcastStream()` trước khi truyền vào nhiều widget.

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Phân Tích Mã Nguồn (Deep-Dive & Code Tracing)

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Điều gì xảy ra ở tầng Element Tree khi một Future mới được truyền vào thuộc tính `future` của `FutureBuilder`?
*Phân tích:*
Khi `FutureBuilder` nhận một `future` mới có tham chiếu ô nhớ khác với `oldWidget.future`, phương thức `didUpdateWidget()` được framework gọi:
1. `_FutureBuilderState` so sánh tham chiếu danh tính.
2. Bộ lắng nghe kết quả của Future cũ bị hủy bỏ (`_unsubscribe()`).
3. Một đối tượng theo dõi mới được khởi tạo (`_subscribe()`).
4. `snapshot` hiện tại bị gán lại trạng thái `ConnectionState.waiting`, làm kích hoạt một chu kỳ vẽ lại (rebuild) của `Element` tương ứng, hiển thị lại widget chờ tải.

---

#### Câu hỏi 2: Sự khác biệt về các trạng thái `ConnectionState` giữa `FutureBuilder` và `StreamBuilder` là gì?
*Phân tích:*
- Đối với `FutureBuilder`: Do `Future` chỉ trả về một giá trị duy nhất hoặc lỗi, trạng thái chỉ có thể chuyển từ `none` $\to$ `waiting` $\to$ `done`. Trạng thái `ConnectionState.active` **không bao giờ xuất hiện**.
- Đối với `StreamBuilder`: Do `Stream` là chuỗi các sự kiện theo thời gian, sau khi nhận được sự kiện đầu tiên, `ConnectionState` chuyển sang `active` và duy trì ở trạng thái này cho toàn bộ các sự kiện tiếp theo. Trạng thái `ConnectionState.done` chỉ đạt được khi Stream nguồn thực sự phát tín hiệu hoàn tất và đóng luồng.

---

#### Câu hỏi 3: Tại sao `AsyncSnapshot` có thể đồng thời thỏa mãn `hasData == true` và `connectionState == ConnectionState.waiting`?
*Phân tích:*
Hiện tượng này xảy ra trong hai trường hợp:
1. Khi lập trình viên cung cấp thuộc tính `initialData` cho `FutureBuilder` hoặc `StreamBuilder`: Trong thời gian tác vụ bất đồng bộ đang chờ phản hồi (`waiting`), snapshot vẫn chứa dữ liệu khởi tạo (`initialData != null`), giúp `hasData` trả về `true`.
2. Khi `StreamBuilder` nhận một stream mới trong `didUpdateWidget`: Snapshot có thể giữ lại dữ liệu cuối cùng của stream trước đó trong khi đang chờ sự kiện đầu tiên từ stream mới.

---

#### Câu hỏi 4: Hạn chế của `FutureBuilder` so với mô hình Quản lý trạng thái phân tầng là gì?
*Phân tích:*
1. **Ràng buộc vòng đời (Lifecycle Coupling)**: Dữ liệu bị phụ thuộc vào sự tồn tại của Widget. Khi Widget bị unmounted (chuyển trang), dữ liệu bị giải phóng và phải tải lại từ đầu khi người dùng quay lại.
2. **Không có khả năng chia sẻ (State Sharing)**: Dữ liệu trong `snapshot` bị đóng gói bên trong hàm `builder` của widget đó, các widget anh em hoặc màn hình khác không thể truy cập trực tiếp.
3. **Khó khăn trong việc viết Unit Test**: Không thể kiểm thử logic tải và biến đổi dữ liệu một cách độc lập mà bắt buộc phải dựng toàn bộ Widget Test kèm môi trường rendering của Flutter.

---

#### Câu hỏi 5: Làm thế nào để ngăn ngừa rò rỉ bộ nhớ khi sử dụng `StreamBuilder`?
*Phân tích:*
Bản thân `StreamBuilder` đã tự động gọi `_subscription.cancel()` trong hàm `dispose()` của nó. Tuy nhiên, rò rỉ bộ nhớ vẫn có thể xảy ra nếu:
- Đối tượng nguồn (`StreamController`) được tạo ra bên trong `State` nhưng không được đóng (`controller.close()`) trong hàm `dispose()` của widget cha.
- Stream liên kết với các tài nguyên hệ thống dài hạn (như lắng nghe vị trí GPS, socket mạng) mà không được giải phóng khi màn hình kết thúc.

---

### 5.2 — Bài Tập Phân Tích Luồng Thực Thi (Code Tracing)

#### Đề bài:
Cho hai đoạn mã nguồn sau đây:

**Trường hợp A (Khởi tạo trong build):**
```dart
class ScreenA extends StatefulWidget {
  const ScreenA({super.key});
  @override State<ScreenA> createState() => _ScreenAState();
}
class _ScreenAState extends State<ScreenA> {
  int counter = 0;
  Future<String> fetchData() async {
    await Future.delayed(const Duration(milliseconds: 500));
    return 'Dữ liệu';
  }
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        ElevatedButton(
          onPressed: () => setState(() => counter++), 
          child: Text('Tăng: $counter'),
        ),
        FutureBuilder<String>(
          future: fetchData(), // Gọi trực tiếp
          builder: (context, snapshot) {
            if (snapshot.connectionState == ConnectionState.waiting) {
              return const CircularProgressIndicator();
            }
            return Text(snapshot.data ?? '');
          },
        ),
      ],
    );
  }
}
```

**Trường hợp B (Khởi tạo trong initState):**
```dart
class ScreenB extends StatefulWidget {
  const ScreenB({super.key});
  @override State<ScreenB> createState() => _ScreenBState();
}
class _ScreenBState extends State<ScreenB> {
  int counter = 0;
  late Future<String> dataFuture;

  @override
  void initState() {
    super.initState();
    dataFuture = fetchData();
  }

  Future<String> fetchData() async {
    await Future.delayed(const Duration(milliseconds: 500));
    return 'Dữ liệu';
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        ElevatedButton(
          onPressed: () => setState(() => counter++), 
          child: Text('Tăng: $counter'),
        ),
        FutureBuilder<String>(
          future: dataFuture, // Tham chiếu ổn định
          builder: (context, snapshot) {
            if (snapshot.connectionState == ConnectionState.waiting) {
              return const CircularProgressIndicator();
            }
            return Text(snapshot.data ?? '');
          },
        ),
      ],
    );
  }
}
```

Giả sử sau khi màn hình hiển thị được 2 giây (dữ liệu đã tải xong và hiển thị chữ `'Dữ liệu'`), người dùng nhấn vào nút `ElevatedButton`.

#### Yêu cầu phân tích:
1. Mô tả chi tiết chu kỳ thực thi và trạng thái của `FutureBuilder` trong **Trường hợp A**.
2. Mô tả chi tiết chu kỳ thực thi và trạng thái của `FutureBuilder` trong **Trường hợp B**.

---

#### Kết quả phân tích kỹ thuật:

**1. Đối với Trường hợp A:**
- Khi người dùng nhấn nút, `setState()` của `_ScreenAState` được kích hoạt.
- Hàm `build()` chạy lại. Biểu thức `future: fetchData()` được đánh giá, tạo ra một thể hiện `Future` hoàn toàn mới trong bộ nhớ heap.
- `FutureBuilder` nhận `future` mới trong `didUpdateWidget`. Do tham chiếu thay đổi, `FutureBuilder` hủy bỏ kết quả cũ và chuyển `snapshot.connectionState` về lại `ConnectionState.waiting`.
- Giao diện ngay lập tức bị chớp: Dòng chữ `'Dữ liệu'` biến mất và thay thế bằng `CircularProgressIndicator`.
- Sau 500ms, Future mới hoàn tất, giao diện lại chuyển về hiển thị `'Dữ liệu'`. Mỗi lần nhấn nút là một lần gửi request mới và làm nhấp nháy giao diện.

**2. Đối với Trường hợp B:**
- Khi người dùng nhấn nút, `setState()` của `_ScreenBState` được kích hoạt.
- Hàm `build()` chạy lại. Thuộc tính `future: dataFuture` vẫn giữ nguyên tham chiếu đến đối tượng `Future` ban đầu đã hoàn thành từ trước.
- Phương thức `didUpdateWidget` của `FutureBuilder` kiểm tra điều kiện `oldWidget.future == widget.future` (trả về `true`).
- `FutureBuilder` **không hủy bỏ kết quả và không chuyển về trạng thái waiting**. Snapshot vẫn giữ nguyên giá trị `ConnectionState.done` cùng dữ liệu đã có.
- Chỉ có nút bấm `ElevatedButton` được cập nhật lại nhãn văn bản `Tăng: 1`, phần hiển thị dữ liệu của `FutureBuilder` giữ nguyên trạng thái tĩnh mà không hề bị chớp nháy hay phát sinh bất kỳ yêu cầu mạng nào.
