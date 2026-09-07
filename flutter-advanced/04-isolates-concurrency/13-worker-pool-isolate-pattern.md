# Bài 4.3: Worker Pool Isolate Pattern

> **Cấp độ**: Staff Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Đã đọc Bài 4.1 và 4.2

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: E2E Encryption trong Chat App

**Encrypted messaging app, 50k DAU:**

```
Yêu cầu:
- Mỗi message gửi/nhận phải được mã hóa/giải mã AES-256-GCM
- Peak load: 200 messages/giây
- Latency yêu cầu: <50ms per message (encrypt + send)

Vấn đề với single isolate:
  AES-256 encrypt 1KB text: ~5ms
  200 messages/s × 5ms = 1000ms/s → CPU 100% trên 1 isolate
  → Tắc nghẽn: queue build up → latency tăng vô hạn

Giải pháp: Worker Pool với N isolates xử lý song song
  4 isolates × 5ms = 25ms total for 4 concurrent messages
  → Throughput: 4 × 200ms/s = 800 ops/s — đủ dư margin
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. Worker Pool Architecture

```
Main Isolate:
  Task Queue (FIFO):  [Task1, Task2, Task3, Task4, Task5, Task6...]
                            │
                    Pool Manager
                   /     |     |     \
               Worker1 Worker2 Worker3 Worker4
               [busy]  [idle]  [busy]  [idle]
                            │           │
                      ←Result3    ←Result4
                      
Điều phối: 
  - Khi worker idle → nhận task tiếp theo từ queue
  - Nếu tất cả busy → task wait trong queue
  - Khi worker crash → restart worker + retry task
  - Backpressure: nếu queue > threshold → reject new task / apply pressure

Load Balancing: Work-stealing hoặc Round-robin
  Simple: Round-robin (gửi lần lượt cho từng worker)
  Smart: Work-stealing (worker idle tự lấy task từ queue)
```

### 2.2. Backpressure — Tránh OOM khi quá tải

```
Không có backpressure:
  Main: gửi 10,000 tasks trong 1 giây
  Workers (4): xử lý 800 tasks/giây
  Queue: accumulate 9,200 tasks
  Memory: 9,200 × (task size + Completer) ≈ OOM crash

Có backpressure:
  Queue limit: 100 tasks
  Khi queue > 100: throw QueueFullException
  Client: back-off và retry sau 500ms
  → Queue stable, memory stable
```

---

## Phần 3 — Production Code Implementation

### 3.1. Generic Worker Pool — Production-grade

```dart
// lib/core/isolate/worker_pool.dart
import 'dart:async';
import 'dart:isolate';

/// Task được gửi đến Worker
final class PoolTask<I, O> {
  PoolTask({
    required this.id,
    required this.input,
    required this.completer,
  });

  final String id;
  final I input;
  final Completer<O> completer;
}

/// Message từ Main → Worker
sealed class WorkerInbound<I> {}

final class TaskMessage<I> extends WorkerInbound<I> {
  const TaskMessage({required this.taskId, required this.input});
  final String taskId;
  final I input;
}

final class ShutdownWorker<I> extends WorkerInbound<I> {
  const ShutdownWorker();
}

/// Message từ Worker → Main
sealed class WorkerOutbound {}

final class ReadySignal extends WorkerOutbound {
  const ReadySignal({required this.sendPort});
  final SendPort sendPort;
}

final class TaskResult<O> extends WorkerOutbound {
  const TaskResult({required this.taskId, required this.result});
  final String taskId;
  final O result;
}

final class TaskError extends WorkerOutbound {
  const TaskError({
    required this.taskId,
    required this.error,
    required this.stackTrace,
  });
  final String taskId;
  final Object error;
  final String stackTrace;
}

final class WorkerDied extends WorkerOutbound {
  const WorkerDied({required this.workerId});
  final int workerId;
}

/// Worker Pool generic — hoạt động với bất kỳ I/O type nào
final class WorkerPool<I, O> {
  WorkerPool({
    required int workerCount,
    required void Function(SendPort, ReceivePort) entryPoint,
    int maxQueueSize = 500,
  })  : _workerCount = workerCount,
        _entryPoint = entryPoint,
        _maxQueueSize = maxQueueSize;

  final int _workerCount;
  final void Function(SendPort, ReceivePort) _entryPoint;
  final int _maxQueueSize;

  final _workers = <int, _WorkerState>{}; // workerId → state
  final _taskQueue = <PoolTask<I, O>>[];
  final _pendingTasks = <String, PoolTask<I, O>>{};
  
  ReceivePort? _mainReceivePort;
  bool _initialized = false;
  bool _disposed = false;
  int _taskCounter = 0;

  /// Khởi tạo pool — gọi 1 lần trước khi sử dụng
  Future<void> initialize() async {
    if (_initialized) return;
    _initialized = true;
    
    _mainReceivePort = ReceivePort();
    
    // Lắng nghe tất cả messages từ mọi worker
    _mainReceivePort!.listen(_handleWorkerMessage);
    
    // Spawn N workers
    await Future.wait(
      List.generate(_workerCount, (i) => _spawnWorker(i)),
    );
  }

  Future<void> _spawnWorker(int workerId) async {
    final workerCompleter = Completer<void>();
    _workers[workerId] = _WorkerState(
      id: workerId,
      isReady: false,
      readyCompleter: workerCompleter,
    );

    final isolate = await Isolate.spawn(
      _workerWrapper,
      _WorkerInitMessage(
        mainSendPort: _mainReceivePort!.sendPort,
        workerId: workerId,
        entryPoint: _entryPoint,
      ),
      debugName: 'Worker_$workerId',
      errorsAreFatal: false,
    );

    _workers[workerId]!.isolate = isolate;
    await workerCompleter.future; // Đợi worker sẵn sàng
  }

  void _handleWorkerMessage(dynamic message) {
    if (message is! WorkerOutbound) return;
    
    switch (message) {
      case ReadySignal(:final sendPort):
        final workerId = _findWorkerBySendPort(sendPort);
        if (workerId != null) {
          _workers[workerId]!
            ..sendPort = sendPort
            ..isReady = true
            ..readyCompleter.complete();
          _dispatchNextTask(workerId); // Dispatch ngay nếu có task chờ
        }
        
      case TaskResult<O>(:final taskId, :final result):
        final task = _pendingTasks.remove(taskId);
        task?.completer.complete(result);
        
        // Worker idle → dispatch task tiếp
        final workerId = _findWorkerByTask(taskId);
        if (workerId != null) {
          _workers[workerId]!.currentTaskId = null;
          _dispatchNextTask(workerId);
        }
        
      case TaskError(:final taskId, :final error, :final stackTrace):
        final task = _pendingTasks.remove(taskId);
        task?.completer.completeError(
          error,
          StackTrace.fromString(stackTrace),
        );
        
        final workerId = _findWorkerByTask(taskId);
        if (workerId != null) {
          _workers[workerId]!.currentTaskId = null;
          _dispatchNextTask(workerId);
        }
        
      case WorkerDied(:final workerId):
        // Worker crash — restart và retry pending task
        final deadWorker = _workers[workerId];
        if (deadWorker?.currentTaskId != null) {
          final failedTask = _pendingTasks[deadWorker!.currentTaskId!];
          if (failedTask != null) {
            _taskQueue.insert(0, failedTask); // Re-queue failed task
            _pendingTasks.remove(deadWorker.currentTaskId);
          }
        }
        // Restart worker
        _restartWorker(workerId);
    }
  }

  /// Submit task — returns Future<O>
  Future<O> submit(I input) {
    if (_disposed) throw StateError('WorkerPool đã bị dispose');
    if (_taskQueue.length >= _maxQueueSize) {
      throw QueueFullException('Worker Pool queue đầy: $_maxQueueSize tasks');
    }
    
    final taskId = 'task_${_taskCounter++}';
    final completer = Completer<O>();
    final task = PoolTask<I, O>(
      id: taskId,
      input: input,
      completer: completer,
    );
    
    // Tìm idle worker
    final idleWorkerId = _findIdleWorker();
    if (idleWorkerId != null) {
      _sendTaskToWorker(idleWorkerId, task);
    } else {
      // Tất cả busy → queue task
      _taskQueue.add(task);
    }
    
    return completer.future;
  }

  void _dispatchNextTask(int workerId) {
    if (_taskQueue.isEmpty) return;
    final nextTask = _taskQueue.removeAt(0);
    _sendTaskToWorker(workerId, nextTask);
  }

  void _sendTaskToWorker(int workerId, PoolTask<I, O> task) {
    final worker = _workers[workerId]!;
    worker.currentTaskId = task.id;
    _pendingTasks[task.id] = task;
    worker.sendPort!.send(TaskMessage<I>(taskId: task.id, input: task.input));
  }

  int? _findIdleWorker() {
    for (final entry in _workers.entries) {
      if (entry.value.isReady && entry.value.currentTaskId == null) {
        return entry.key;
      }
    }
    return null;
  }

  int? _findWorkerBySendPort(SendPort port) {
    for (final entry in _workers.entries) {
      if (entry.value.sendPort == port) return entry.key;
    }
    return null;
  }

  int? _findWorkerByTask(String taskId) {
    for (final entry in _workers.entries) {
      if (entry.value.currentTaskId == taskId) return entry.key;
    }
    return null;
  }

  Future<void> _restartWorker(int workerId) async {
    _workers[workerId]?.isolate?.kill();
    await _spawnWorker(workerId);
  }

  Future<void> dispose() async {
    _disposed = true;
    // Fail tất cả pending tasks
    for (final task in _pendingTasks.values) {
      task.completer.completeError(
        StateError('Worker pool disposed'),
      );
    }
    _pendingTasks.clear();
    // Shutdown workers
    for (final worker in _workers.values) {
      worker.sendPort?.send(const ShutdownWorker());
    }
    await Future.delayed(const Duration(milliseconds: 100));
    for (final worker in _workers.values) {
      worker.isolate?.kill(priority: Isolate.immediate);
    }
    _mainReceivePort?.close();
  }
}

class _WorkerState {
  _WorkerState({required this.id, required bool isReady, required this.readyCompleter})
      : _isReady = isReady;
  
  final int id;
  bool _isReady;
  bool get isReady => _isReady;
  set isReady(bool v) => _isReady = v;
  
  Isolate? isolate;
  SendPort? sendPort;
  String? currentTaskId;
  final Completer<void> readyCompleter;
}

// Worker wrapper — chạy trong isolate
class _WorkerInitMessage {
  const _WorkerInitMessage({
    required this.mainSendPort,
    required this.workerId,
    required this.entryPoint,
  });
  final SendPort mainSendPort;
  final int workerId;
  final void Function(SendPort, ReceivePort) entryPoint;
}

void _workerWrapper(_WorkerInitMessage init) {
  final receivePort = ReceivePort();
  init.mainSendPort.send(ReadySignal(sendPort: receivePort.sendPort));
  
  // Delegate actual processing logic
  init.entryPoint(init.mainSendPort, receivePort);
}

class QueueFullException implements Exception {
  const QueueFullException(this.message);
  final String message;
  @override
  String toString() => 'QueueFullException: $message';
}
```

### 3.2. Ứng dụng: Encryption Worker Pool

```dart
// features/chat/infrastructure/encryption_pool.dart

// Worker entry point cho AES-256-GCM encryption
void _encryptionWorkerEntryPoint(SendPort mainPort, ReceivePort receivePort) {
  receivePort.listen((dynamic message) {
    if (message is ShutdownWorker) {
      receivePort.close();
      return;
    }
    
    if (message is! TaskMessage<EncryptionTask>) return;
    
    try {
      final result = switch (message.input.operation) {
        EncryptionOperation.encrypt => _encrypt(
            message.input.data,
            message.input.key,
          ),
        EncryptionOperation.decrypt => _decrypt(
            message.input.data,
            message.input.key,
          ),
      };
      
      mainPort.send(TaskResult<Uint8List>(
        taskId: message.taskId,
        result: result,
      ));
    } catch (e, st) {
      mainPort.send(TaskError(
        taskId: message.taskId,
        error: e,
        stackTrace: st.toString(),
      ));
    }
  });
}

Uint8List _encrypt(Uint8List data, Uint8List key) {
  // Pure Dart AES-256-GCM implementation
  // hoặc gọi native crypto qua FFI
  final cipher = AesGcm.with256bits();
  return cipher.encrypt(data, secretKey: SecretKey(key));
}

// Khởi tạo pool
final encryptionPool = WorkerPool<EncryptionTask, Uint8List>(
  workerCount: 4, // Tùy số CPU cores: Platform.numberOfProcessors
  entryPoint: _encryptionWorkerEntryPoint,
  maxQueueSize: 200,
);

// Inject vào DI
@module
abstract class EncryptionModule {
  @singleton
  WorkerPool<EncryptionTask, Uint8List> get encryptionPool =>
      WorkerPool(workerCount: 4, entryPoint: _encryptionWorkerEntryPoint);
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Benchmark: Single Worker vs Pool (Chat App)

```
Test: Encrypt 200 messages/giây, AES-256-GCM, 1KB each
Device: iPhone 14 Pro (6 cores)

1 Worker Isolate:
  Throughput: 200 ops/s (5ms each)
  Queue size at peak: 0 (just keeping up)
  P99 latency: 48ms (task in queue max)
  CPU usage: 1 core at 100%

4 Worker Pool:
  Throughput: 800 ops/s (capacity headroom: 4x)
  Queue size at 200 ops/s: 0 (workers idle most of time)
  P99 latency: 12ms
  CPU usage: 4 cores at 25% each
  Memory overhead: +4 × 15MB = 60MB extra

8 Worker Pool (aggressive):
  Throughput: 1600 ops/s
  P99 latency: 8ms
  CPU usage: 8 cores at 12.5% each
  Memory overhead: +8 × 15MB = 120MB extra
  → Overkill cho 200 ops/s; waste memory
  
NGƯỠNG QUYẾT ĐỊNH số workers:
  workers = min(Platform.numberOfProcessors, ceil(peak_ops / single_worker_capacity) × 1.5)
  Ví dụ: 200 ops/s / 200 = 1, × 1.5 = 1.5 → round up → 2 workers
  Thực tế dùng 4 để có safety margin
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Không có backpressure — queue tăng vô hạn
    ✅ maxQueueSize limit + QueueFullException + client back-off
    Lý do: 10,000 tasks × 5KB each = 50MB memory → OOM

[ ] ❌ Không restart worker khi crash
    ✅ WorkerDied message + auto-restart + re-queue failed task
    Lý do: Worker crash silently → pool degraded, tasks never complete

[ ] ❌ Số workers = Platform.numberOfProcessors luôn
    ✅ Benchmark actual throughput requirement → tính số workers cần thiết
    Lý do: Quá nhiều workers → memory overhead không cần thiết

[ ] ❌ Pool không có dispose() khi app đóng
    ✅ Implement dispose: cancel pending, shutdown workers, close ports
    Lý do: Workers tiếp tục chạy → battery drain, background resource waste

[ ] ❌ Worker entry point là closure (capture outer scope)
    ✅ Top-level hoặc static function — không capture mutable state
    Lý do: Worker isolate có heap riêng — captured object được copy, không shared

[ ] ❌ Dùng Worker Pool cho IO-bound tasks (network calls)
    ✅ Worker Pool chỉ cho CPU-bound: crypto, image processing, JSON parsing
    Lý do: IO không block Dart VM — isolate overhead không có lợi

[ ] ❌ Không có timeout cho task trong pool
    ✅ Completer với timeout: completer.future.timeout(Duration(seconds: 30))
    Lý do: Worker stuck → task never complete → memory leak từ pending Completers
```
