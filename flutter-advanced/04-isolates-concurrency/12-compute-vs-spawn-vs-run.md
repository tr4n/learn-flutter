# Bài 4.2: compute() vs Isolate.spawn() vs Isolate.run()

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~25 phút  
> **Yêu cầu**: Đã đọc Bài 4.1 (Isolate Memory Architecture)

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Chọn API nào cho image processing pipeline?

**Photo editing app cần:**
1. Apply filter cho 1 ảnh khi user chọn (single call)
2. Process batch 50 ảnh khi export (repeated calls)
3. Watermark ảnh realtime khi preview (frequent calls)

3 tình huống khác nhau → 3 API khác nhau.

---

## Phần 2 — Low-Level Mechanics

### 2.1. So sánh kiến trúc 3 API

```
compute() [Flutter helper, deprecated in favor of Isolate.run()]:
┌────────────────────────────────────────────────────────────┐
│ compute(function, message)                                 │
│   → Gọi Isolate.spawn() bên trong                          │
│   → Tự động serialize/deserialize result                   │
│   → Dispose isolate ngay sau khi task xong                │
│   → API: Function<R, Q> (1 argument type)                  │
│                                                            │
│ Chi phí: Spawn (12ms) + Task + Teardown mỗi lần gọi       │
│ Dùng: Single, one-shot background task (parse JSON 1 lần) │
└────────────────────────────────────────────────────────────┘

Isolate.run() [Dart 2.19+ — khuyến nghị thay compute()]:
┌────────────────────────────────────────────────────────────┐
│ await Isolate.run(() async { ... })                        │
│   → Về bản chất giống compute() nhưng:                    │
│   → Hỗ trợ async function trong isolate                    │
│   → Không giới hạn 1 argument (closure capture OK)         │
│   → Type-safe return value (generic)                       │
│   → Dispose isolate sau khi xong                           │
│                                                            │
│ Chi phí: Spawn (12ms) + Task + Teardown mỗi lần gọi       │
│ Dùng: Single task, cần closure capture hoặc async logic   │
└────────────────────────────────────────────────────────────┘

Isolate.spawn() [Full control, long-lived]:
┌────────────────────────────────────────────────────────────┐
│ await Isolate.spawn(entryPoint, initialMessage)            │
│   → Tạo isolate LONG-LIVED (không tự dispose)              │
│   → Giao tiếp 2 chiều qua SendPort/ReceivePort             │
│   → Xử lý nhiều tasks liên tiếp                            │
│   → Phải tự quản lý lifecycle                              │
│                                                            │
│ Chi phí: Spawn 1 lần (12ms) + Task × N + Manual dispose   │
│ Dùng: Worker pool, frequent tasks, long-running background │
└────────────────────────────────────────────────────────────┘
```

### 2.2. Decision Tree

```
Cần chạy code trong isolate?
         │
         ▼
Sẽ gọi nhiều lần liên tiếp?
         │
    NO ──┤──── YES
         │         │
         ▼         ▼
  Task có async?  Dùng Isolate.spawn()
         │        (Worker Pool Pattern)
    YES  │  NO
         │    │
         ▼    ▼
  Isolate   compute()
  .run()    (deprecated)
            hoặc
            Isolate.run()
```

---

## Phần 3 — Production Code Implementation

### 3.1. `Isolate.run()` — Single task, modern API

```dart
// features/photo/domain/usecases/apply_filter_usecase.dart
final class ApplyFilterUseCase {
  const ApplyFilterUseCase();
  
  /// Apply filter cho 1 ảnh — non-blocking
  Future<Uint8List> execute(Uint8List imageBytes, ImageFilter filter) async {
    // Capture params trong closure — không cần top-level function
    return Isolate.run(() => _applyFilterSync(imageBytes, filter));
    //                        ↑ closure capture OK với Isolate.run()
  }
  
  // Synchronous function — sẽ chạy trong worker isolate
  // PHẢI là pure function: không access global state, không side effects
  static Uint8List _applyFilterSync(Uint8List bytes, ImageFilter filter) {
    return switch (filter) {
      ImageFilter.grayscale => _applyGrayscale(bytes),
      ImageFilter.sepia => _applySepia(bytes),
      ImageFilter.blur => _applyGaussianBlur(bytes),
      ImageFilter.sharpen => _applySharpen(bytes),
    };
  }
  
  static Uint8List _applyGrayscale(Uint8List pixels) {
    final result = Uint8List(pixels.length);
    for (var i = 0; i < pixels.length; i += 4) {
      final gray = (0.299 * pixels[i] + 0.587 * pixels[i + 1] + 0.114 * pixels[i + 2]).round();
      result[i] = result[i + 1] = result[i + 2] = gray;
      result[i + 3] = pixels[i + 3];
    }
    return result;
  }
}

// Sử dụng:
class PhotoEditorBloc extends Bloc<PhotoEvent, PhotoState> {
  Future<void> _onFilterApplied(FilterApplied event, Emitter emit) async {
    emit(const PhotoProcessing());
    
    try {
      // Không block UI — filter chạy trong isolate riêng
      final filtered = await _filterUseCase.execute(
        event.imageBytes, 
        event.filter,
      );
      emit(PhotoSuccess(imageBytes: filtered));
    } catch (e) {
      emit(PhotoError(message: e.toString()));
    }
  }
}
```

### 3.2. `compute()` — Backward compatibility (Flutter <3.7)

```dart
// Chỉ dùng cho backward compatibility với Flutter < 3.7
// Mới nên dùng Isolate.run() thay thế

// compute() yêu cầu top-level function (không dùng closure)
List<Product> _parseProductsSync(String jsonString) {
  final list = jsonDecode(jsonString) as List<dynamic>;
  return list.map((json) => Product.fromJson(json as Map<String, dynamic>)).toList();
}

// Sử dụng compute():
Future<List<Product>> parseProducts(String json) {
  return compute(_parseProductsSync, json);
  //             ↑ PHẢI là top-level hoặc static function
  //             ↑ Chỉ nhận ĐÚNG 1 argument
}

// Với Isolate.run() (khuyến nghị):
Future<List<Product>> parseProductsModern(String json) {
  return Isolate.run(() {
    final list = jsonDecode(json) as List<dynamic>;
    return list.map((j) => Product.fromJson(j as Map<String, dynamic>)).toList();
    // ↑ Closure capture `json` OK — không cần wrapper function
  });
}
```

### 3.3. `Isolate.spawn()` — Long-lived Worker Pool

```dart
// lib/core/isolate/image_processing_worker.dart
// Worker cho batch image processing

// Messages giữa Main và Worker
sealed class WorkerMessage {}

final class ProcessImageMessage extends WorkerMessage {
  const ProcessImageMessage({
    required this.taskId,
    required this.imageBytes,
    required this.filter,
  });
  final String taskId;
  final Uint8List imageBytes;
  final ImageFilter filter;
}

final class ShutdownMessage extends WorkerMessage {}

sealed class WorkerResponse {}

final class ProcessingResult extends WorkerResponse {
  const ProcessingResult({required this.taskId, required this.result});
  final String taskId;
  final Uint8List result;
}

final class ProcessingError extends WorkerResponse {
  const ProcessingError({required this.taskId, required this.error});
  final String taskId;
  final String error;
}

// Worker entry point
void _imageWorkerEntryPoint(SendPort mainSendPort) {
  final receivePort = ReceivePort();
  mainSendPort.send(receivePort.sendPort); // Handshake

  receivePort.listen((dynamic message) {
    if (message is! WorkerMessage) return;

    switch (message) {
      case ProcessImageMessage(:final taskId, :final imageBytes, :final filter):
        try {
          final result = ApplyFilterUseCase._applyFilterSync(imageBytes, filter);
          mainSendPort.send(ProcessingResult(taskId: taskId, result: result));
        } catch (e) {
          mainSendPort.send(ProcessingError(taskId: taskId, error: e.toString()));
        }
      
      case ShutdownMessage():
        receivePort.close(); // Graceful shutdown
        Isolate.current.kill();
    }
  });
}

// Manager cho long-lived worker
final class ImageProcessingWorker {
  static SendPort? _workerSendPort;
  static ReceivePort? _mainReceivePort;
  static Isolate? _workerIsolate;
  static final _pendingTasks = <String, Completer<Uint8List>>{};
  static int _counter = 0;

  static Future<void> initialize() async {
    _mainReceivePort = ReceivePort();
    
    _workerIsolate = await Isolate.spawn(
      _imageWorkerEntryPoint,
      _mainReceivePort!.sendPort,
      debugName: 'ImageProcessingWorker',
      // Callback khi worker crash unexpectedly
      onExit: _mainReceivePort!.sendPort,
      errorsAreFatal: false, // Worker crash không kill main
    );
    
    _mainReceivePort!.listen((dynamic message) {
      if (message is SendPort) {
        _workerSendPort = message;
        return;
      }
      
      switch (message) {
        case ProcessingResult(:final taskId, :final result):
          _pendingTasks.remove(taskId)?.complete(result);
        case ProcessingError(:final taskId, :final error):
          _pendingTasks.remove(taskId)?.completeError(Exception(error));
        default:
          // Worker exited (onExit message) — restart worker
          _restartWorker();
      }
    });
    
    // Đợi handshake
    while (_workerSendPort == null) {
      await Future.delayed(const Duration(milliseconds: 1));
    }
  }

  static Future<Uint8List> processImage(Uint8List bytes, ImageFilter filter) {
    final taskId = 'img_${_counter++}';
    final completer = Completer<Uint8List>();
    _pendingTasks[taskId] = completer;
    
    _workerSendPort!.send(ProcessImageMessage(
      taskId: taskId,
      imageBytes: bytes,
      filter: filter,
    ));
    
    return completer.future;
  }
  
  static Future<void> _restartWorker() async {
    // Graceful restart sau crash
    _workerSendPort = null;
    await initialize();
    // Fail tất cả pending tasks
    for (final completer in _pendingTasks.values) {
      completer.completeError(Exception('Worker restarted'));
    }
    _pendingTasks.clear();
  }

  static Future<void> dispose() async {
    _workerSendPort?.send(const ShutdownMessage());
    await Future.delayed(const Duration(milliseconds: 100));
    _workerIsolate?.kill(priority: Isolate.immediate);
    _mainReceivePort?.close();
  }
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Benchmark: 50 ảnh, 3 API

```
Test: Process 50 ảnh JPEG, mỗi ảnh ~1MB, apply grayscale filter
Device: iPhone 14, Profile mode

compute() / Isolate.run() (new isolate mỗi ảnh):
  Spawn time per call: ~12ms
  Process time per image: ~180ms
  Total overhead (spawn × 50): 600ms
  Total time: 50 × (12ms + 180ms) = 9,600ms
  Peak memory: 1 isolate × 30MB heap = 30MB extra

Isolate.spawn() (1 long-lived worker):
  Spawn time: 12ms (1 lần duy nhất)
  Process time per image: ~180ms
  Total overhead: 12ms
  Total time: 12ms + 50 × 180ms = 9,012ms
  Peak memory: 1 isolate × 15MB heap = 15MB extra
  
  Cải thiện: 7% nhanh hơn + 50% ít memory overhead

Worker Pool (4 workers parallel):
  Spawn time: 4 × 12ms = 48ms
  Process 50 ảnh / 4 parallel: ceil(50/4) × 180ms = 13 × 180ms = 2,340ms
  Total: 48ms + 2,340ms = 2,388ms
  Peak memory: 4 isolate × 15MB = 60MB extra
  
  Cải thuận: 4x nhanh hơn single worker
  Trade-off: 4x memory overhead
```

### Decision Table hoàn chỉnh

```
┌────────────────────┬──────────────┬──────────────┬──────────────┐
│ Criteria           │ compute()    │ Isolate.run()│ Isolate      │
│                    │ (deprecated) │              │ .spawn()     │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ API simplicity     │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐       │
│ Closure support    │ ❌           │ ✅           │ ❌ (manual)  │
│ Async in worker    │ ❌           │ ✅           │ ✅           │
│ Long-lived         │ ❌           │ ❌           │ ✅           │
│ Spawn cost         │ Per call     │ Per call     │ Once         │
│ Memory control     │ Low          │ Low          │ Full         │
│ Error handling     │ Basic        │ Good         │ Full         │
│ Dart version       │ All          │ 2.19+        │ All          │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ BEST FOR           │ Legacy code  │ Single async │ Repeated     │
│                    │              │ tasks        │ heavy tasks  │
└────────────────────┴──────────────┴──────────────┴──────────────┘
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Dùng compute() với Flutter 3.7+ (Dart 2.19+)
    ✅ Migrate sang Isolate.run() — ít boilerplate, hỗ trợ closure
    Lý do: compute() deprecated, Isolate.run() là API chuẩn hiện tại

[ ] ❌ Gọi Isolate.run() trong vòng lặp nhiều lần
    ✅ Dùng Isolate.spawn() với long-lived worker cho batch processing
    Lý do: Spawn overhead (12ms) × 100 lần = 1.2s wasted time

[ ] ❌ Closure capture large objects vào Isolate.run()
    ✅ Chỉ capture primitive types, String, Uint8List
    ✅ Dùng TransferableTypedData cho binary data lớn
    Lý do: Large object capture → serialization cost cao

[ ] ❌ Không handle worker crash trong Isolate.spawn()
    ✅ onError callback + onExit handler + auto-restart logic
    Lý do: Worker crash silently → pending Completers never resolved → memory leak

[ ] ❌ Dùng Isolate cho IO-bound operations (Dio request, file read)
    ✅ IO-bound operations: async/await trên main isolate
    Lý do: IO không block Dart thread — Isolate overhead không có lợi

[ ] ❌ Isolate.run() với mutable global state bên trong
    ✅ Worker function phải pure — không access singleton, không modify global
    Lý do: Worker có heap riêng — global state trong worker KHÔNG affect main
```
