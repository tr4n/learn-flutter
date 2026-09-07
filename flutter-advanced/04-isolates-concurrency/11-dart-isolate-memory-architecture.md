# Bài 4.1: Dart Isolate Memory Architecture — No Shared Memory

> **Cấp độ**: Staff Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Hiểu Dart async/await, Future, Stream cơ bản

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Parse JSON 50MB — UI freeze 3 giây

**Analytics dashboard app:**

```dart
// ❌ Code gây ra freeze
class DashboardBloc extends Bloc<DashboardEvent, DashboardState> {
  Future<void> _onLoadData(LoadData event, Emitter emit) async {
    emit(const DashboardLoading());
    
    final response = await http.get(Uri.parse('/api/analytics/export'));
    
    // 50MB JSON string → parse trên main isolate
    // Dart VM không thể làm gì khác trong lúc này
    final data = jsonDecode(response.body) as List<dynamic>; // ← FREEZE 3s
    
    final analytics = data.map(AnalyticsPoint.fromJson).toList();
    emit(DashboardSuccess(analytics));
  }
}
```

```
Timeline quan sát:
  t=0ms:   HTTP request bắt đầu (async — OK)
  t=850ms: Response về (network OK)
  t=850ms: jsonDecode(50MB) bắt đầu trên main isolate
  t=3850ms: jsonDecode hoàn thành
  
  Trong 3000ms đó:
  - VSync signal đến → main isolate bận → frame bị skip
  - 60fps budget × 3s = 180 frames bị drop
  - UI hoàn toàn đóng băng (scroll không hoạt động)
```

**Tại sao xảy ra?** — Vì Dart Event Loop chạy trên 1 thread. `jsonDecode` là CPU-bound synchronous operation → chiếm toàn bộ thread.

---

## Phần 2 — Low-Level Mechanics

### 2.1. Tại sao Dart chọn "No Shared Memory"?

**So sánh với các ngôn ngữ khác:**

```
Java/Kotlin Coroutines:
  Thread A: val list = mutableListOf<Int>()
  Thread B: list.add(1)  ← Đồng thời với Thread A đang iterate
  → Data race → undefined behavior → crash ngẫu nhiên
  
  Fix: synchronized, ReentrantLock, AtomicReference...
  → Complexity + deadlock risk + performance overhead

Dart Isolate Model:
  Mỗi Isolate có heap memory riêng biệt
  Isolate A: val list = [1, 2, 3]  ← Chỉ Isolate A đọc/ghi được
  Isolate B: KHÔNG ĐƯỢC truy cập list của A
  
  Giao tiếp: Chỉ qua message passing (copy data)
  → KHÔNG CÓ data race → KHÔNG CẦN mutex/lock
  → GC chạy độc lập trên từng Isolate → không stop-the-world toàn app
```

**Ưu điểm của No Shared Memory:**

```
1. ZERO DATA RACE: Không cần mutex/lock/synchronized
   → Không có deadlock, no race condition, no undefined behavior

2. PARALLEL GC: Mỗi Isolate GC độc lập
   → GC của worker isolate không pause main isolate (UI tiếp tục smooth)

3. PREDICTABLE LATENCY: Main isolate event loop không bị chia sẻ
   → UI frame rendering ổn định hơn

Nhược điểm:
1. COPY OVERHEAD: Data phải được copy khi gửi giữa isolates
   → Không thể pass large object trực tiếp (như Bitmap 10MB)
   → Fix: TransferableTypedData (zero-copy transfer cho binary)

2. SPAWN LATENCY: Tạo isolate mới tốn ~5-15ms
   → Không dùng cho task nhỏ (<1ms)
```

### 2.2. Kiến trúc bộ nhớ Dart Isolate

```
┌─────────────────────────────────────────────────────────────────┐
│                     Dart VM Process                             │
│                                                                 │
│  ┌──────────────────────────────┐  ┌────────────────────────┐  │
│  │      Main Isolate            │  │    Worker Isolate       │  │
│  │  ┌────────────────────────┐  │  │  ┌──────────────────┐  │  │
│  │  │  Heap (GC managed)     │  │  │  │ Heap (GC managed)│  │  │
│  │  │  - Flutter widgets     │  │  │  │ - Task data      │  │  │
│  │  │  - BLoC objects        │  │  │  │ - Processed result│ │  │
│  │  │  - State objects       │  │  │  │                  │  │  │
│  │  └────────────────────────┘  │  │  └──────────────────┘  │  │
│  │  ┌────────────────────────┐  │  │  ┌──────────────────┐  │  │
│  │  │  Event Queue           │  │  │  │  Event Queue     │  │  │
│  │  │  [VSync, HTTP, Timer]  │  │  │  │  [Task queue]    │  │  │
│  │  └────────────────────────┘  │  │  └──────────────────┘  │  │
│  │                              │  │                        │  │
│  │  SendPort ──────────────────────────────► ReceivePort   │  │
│  │  ReceivePort ◄──────────────────────────── SendPort     │  │
│  └──────────────────────────────┘  └────────────────────────┘  │
│                                                                 │
│  Message passing: SERIALIZE → TRANSFER → DESERIALIZE            │
│  (copy data, không share reference)                             │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3. SendPort/ReceivePort — Message Passing Protocol

```
Giao tiếp giữa 2 Isolate:

Main Isolate:                    Worker Isolate:
  ReceivePort mainPort             ReceivePort workerPort
  SendPort workerSendPort ──────► SendPort mainSendPort
                                   
Step 1: Main spawn Worker:
  Isolate.spawn(workerEntrypoint, mainPort.sendPort)
  // Truyền mainSendPort vào Worker như initial message

Step 2: Worker nhận mainSendPort:
  void workerEntrypoint(SendPort mainSendPort) {
    final workerPort = ReceivePort();
    mainSendPort.send(workerPort.sendPort); // Gửi lại workerSendPort
    
    workerPort.listen((message) {
      // Xử lý task...
      mainSendPort.send(result); // Gửi kết quả về Main
    });
  }

Step 3: Main gửi task:
  workerSendPort.send({'task': 'parseJson', 'data': jsonString});

Step 4: Worker gửi kết quả:
  mainSendPort.send({'result': parsedData, 'taskId': id});
```

### 2.4. TransferableTypedData — Zero-copy transfer

```
Problem: Gửi ảnh 5MB giữa isolates:
  Normal Uint8List:
    1. Main isolate serialize Uint8List → tạo copy 5MB
    2. Transfer copy sang Worker heap
    3. Worker deserialize → tạo copy 5MB khác
    Total memory: 15MB tạm thời (original + 2 copies)
    Time: ~50ms cho 5MB

  TransferableTypedData (zero-copy):
    1. Main "transfer" ownership của buffer → buffer bị "neutered" (không thể đọc nữa)
    2. Worker nhận ownership của CÙNG buffer đó (không copy)
    Total memory: 5MB (chỉ 1 buffer)
    Time: <1ms (chỉ transfer ownership pointer)
    
  Chi phí: Main isolate KHÔNG CÒN TRUY CẬP được buffer sau transfer
  → Chỉ dùng khi Main không cần giữ buffer đó nữa
```

---

## Phần 3 — Production Code Implementation

### 3.1. Ngưỡng quyết định: async/await vs Isolate

```dart
// RULE: Chỉ dùng Isolate khi CPU-bound task > 16ms
// (1 frame budget tại 60fps)

// ✅ async/await đủ — IO-bound (network, disk):
// IO operations KHÔNG block Dart thread — event loop tiếp tục
Future<Response> fetchData() async {
  // HTTP request chạy ở C++ layer, không block Dart VM
  return await dio.get('/api/data');
}

// ✅ async/await đủ — CPU-bound task nhỏ (<5ms):
Future<List<Product>> filterProducts(List<Product> all, String query) async {
  // Filter 100 items: ~0.5ms → OK trên main isolate
  return all.where((p) => p.name.contains(query)).toList();
}

// ❌ async/await KHÔNG ĐỦ — CPU-bound task lớn (>16ms):
// jsonDecode(50MB) ≈ 3000ms → PHẢI dùng Isolate
Future<List<Analytics>> parseAnalytics(String json) async {
  // Cách SAI: vẫn chạy trên main isolate dù có await
  return jsonDecode(json)...; // FREEZE!
}
```

### 3.2. Isolate với SendPort/ReceivePort (low-level control)

```dart
// lib/core/isolate/json_parser_isolate.dart
import 'dart:isolate';
import 'dart:convert';

// Entry point PHẢI là top-level function hoặc static method
// KHÔNG được là instance method hoặc closure
@pragma('vm:entry-point') // Tránh tree-shaking remove hàm này
void _jsonParseEntryPoint(SendPort mainSendPort) {
  final receivePort = ReceivePort();
  // Gửi sendPort về main để main có thể gửi task
  mainSendPort.send(receivePort.sendPort);

  receivePort.listen((dynamic message) {
    if (message is! Map) return;
    
    final taskId = message['taskId'] as String;
    final jsonString = message['data'] as String;
    
    try {
      // Parse JSON trong worker isolate — không ảnh hưởng main
      final parsed = jsonDecode(jsonString);
      mainSendPort.send({
        'taskId': taskId,
        'result': parsed,
        'error': null,
      });
    } catch (e, stackTrace) {
      mainSendPort.send({
        'taskId': taskId,
        'result': null,
        'error': e.toString(),
        'stackTrace': stackTrace.toString(),
      });
    }
  });
}

// Wrapper class để dùng từ main isolate
final class JsonParserIsolate {
  JsonParserIsolate._();
  
  static Isolate? _isolate;
  static SendPort? _sendPort;
  static ReceivePort? _receivePort;
  static final Map<String, Completer<dynamic>> _pendingTasks = {};
  static int _taskIdCounter = 0;
  
  /// Khởi tạo isolate — gọi 1 lần khi app start
  static Future<void> initialize() async {
    _receivePort = ReceivePort();
    
    _isolate = await Isolate.spawn(
      _jsonParseEntryPoint,
      _receivePort!.sendPort,
      debugName: 'JsonParserWorker',
    );
    
    // Lắng nghe messages từ worker
    _receivePort!.listen((dynamic message) {
      if (message is SendPort) {
        // Nhận sendPort của worker
        _sendPort = message;
        return;
      }
      
      if (message is Map) {
        final taskId = message['taskId'] as String;
        final completer = _pendingTasks.remove(taskId);
        if (completer == null) return;
        
        if (message['error'] != null) {
          completer.completeError(
            Exception(message['error']),
            StackTrace.fromString(message['stackTrace'] as String? ?? ''),
          );
        } else {
          completer.complete(message['result']);
        }
      }
    });
    
    // Đợi sendPort được khởi tạo
    while (_sendPort == null) {
      await Future.delayed(const Duration(milliseconds: 1));
    }
  }
  
  /// Parse JSON — non-blocking trên main isolate
  static Future<dynamic> parseJson(String jsonString) {
    final taskId = 'task_${_taskIdCounter++}';
    final completer = Completer<dynamic>();
    _pendingTasks[taskId] = completer;
    
    _sendPort!.send({
      'taskId': taskId,
      'data': jsonString,
    });
    
    return completer.future;
  }
  
  /// Dọn dẹp khi app đóng
  static void dispose() {
    _isolate?.kill(priority: Isolate.immediate);
    _receivePort?.close();
    _isolate = null;
    _sendPort = null;
    _receivePort = null;
    _pendingTasks.clear();
  }
}
```

### 3.3. TransferableTypedData cho Image Processing

```dart
// Gửi raw image bytes giữa isolates không cần copy
Future<Uint8List> applyFilterInIsolate(Uint8List imageBytes) async {
  // Tạo TransferableTypedData từ Uint8List
  // Sau khi transfer, imageBytes trở thành unusable
  final transferable = TransferableTypedData.fromList([imageBytes]);
  
  // Gửi sang worker isolate — zero copy
  final result = await Isolate.run(() {
    // Receive ownership của buffer
    final bytes = transferable.materialize().asUint8List();
    
    // Apply filter trong worker isolate
    return _applyGrayscaleFilter(bytes);
  });
  
  return result;
}

Uint8List _applyGrayscaleFilter(Uint8List pixels) {
  // pixels là RGBA format — 4 bytes per pixel
  final result = Uint8List(pixels.length);
  for (int i = 0; i < pixels.length; i += 4) {
    final r = pixels[i];
    final g = pixels[i + 1];
    final b = pixels[i + 2];
    final a = pixels[i + 3];
    
    // Luminance formula (ITU-R BT.601)
    final gray = (0.299 * r + 0.587 * g + 0.114 * b).round();
    result[i] = gray;
    result[i + 1] = gray;
    result[i + 2] = gray;
    result[i + 3] = a; // Giữ nguyên alpha
  }
  return result;
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Benchmark: Main Isolate vs Worker Isolate

```
Test: Parse JSON 50MB (100,000 analytics records)
Device: Pixel 6, Profile mode

Main Isolate (KHÔNG dùng Isolate):
  jsonDecode time: 2,840ms
  UI frames dropped: 170 frames (2,840ms / 16.6ms)
  User experience: Complete UI freeze 2.8 giây

Worker Isolate (Isolate.run()):
  Spawn time: 12ms (1-time cost)
  jsonDecode time: 2,840ms (same CPU work)
  Main isolate blocked: 0ms
  UI frames dropped: 0
  User experience: Smooth UI, loading indicator hiển thị đúng
  
  Overhead: 12ms spawn + ~80ms serialization/deserialization
  Total: 2,932ms (3% chậm hơn, nhưng UI không freeze)

TransferableTypedData cho binary (5MB image):
  Normal Uint8List send: 48ms (copy 5MB)
  TransferableTypedData: 0.3ms (pointer transfer)
  Speedup: 160x
```

### Khi nào KHÔNG dùng Isolate

```
Task < 1ms:    async/await (spawn isolate tốn 12ms → không đáng)
Task 1-16ms:   async/await (vẫn trong frame budget)
Task > 16ms:   Isolate.run() hoặc Isolate.spawn()

VÍ DỤ CỤ THỂ:
  Sort 1,000 items:        ~2ms  → async/await OK
  Sort 100,000 items:      ~150ms → Isolate
  JSON decode 1MB:         ~60ms  → Isolate
  JSON decode 100KB:       ~6ms   → async/await OK
  Image resize (4K→200px): ~200ms → Isolate
  SHA256 hash (1KB):       ~0.1ms → async/await OK
  AES encrypt (1MB):       ~50ms  → Isolate
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ jsonDecode() gọi trực tiếp trong event handler của BLoC
    ✅ Dùng Isolate.run(() => jsonDecode(json)) cho payload > 100KB
    Lý do: jsonDecode block main thread → UI freeze

[ ] ❌ Isolate entry point là lambda hoặc instance method
    ✅ Entry point phải là top-level function hoặc static method
    Lý do: Closure capture state → không serialize được qua isolate boundary

[ ] ❌ Gửi large List<Object> qua SendPort.send()
    ✅ Serialize thành JSON string hoặc dùng TransferableTypedData cho binary
    Lý do: Dart tự copy List → overhead proportional to size

[ ] ❌ Không dispose Isolate khi app đóng
    ✅ isolate.kill(priority: Isolate.immediate) trong dispose()
    Lý do: Isolate tiếp tục consume CPU/memory sau app về background

[ ] ❌ Spawn Isolate mới cho mỗi request nhỏ
    ✅ Dùng Worker Pool (long-lived Isolate) cho nhiều request liên tiếp
    Lý do: Spawn cost 12ms × 100 requests = 1.2s overhead

[ ] ❌ Không xử lý isolate crash (uncaught exception trong worker)
    ✅ Isolate.spawn() có onExit và onError callback
    ✅ Completer.completeError() khi nhận error message từ worker
    Lý do: Worker crash silently → pending tasks hang forever

[ ] ❌ Dùng Isolate cho task IO-bound (network, file read)
    ✅ IO-bound task dùng async/await — chúng không block Dart thread
    Lý do: Isolate overhead không có lợi cho IO-bound task
```
