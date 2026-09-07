# Live Coding Challenges: Các Bài Toán Thực Chiến Phỏng Vấn Senior

> **Cấp độ**: Senior / Lead Mobile Engineer  
> **Chủ đề**: 5 bài toán Live Coding kinh điển thường gặp trong các vòng phỏng vấn kỹ thuật trực tiếp (Live Coding 45-60 phút), kèm mã nguồn chuẩn Dart 3 và phân tích độ phức tạp thuật toán.

---

## BÀI 1: Tự Cài Đặt Bộ Nhớ Đệm LRU Cache $O(1)$ Với Hạn Sử Dụng (TTL)

### 1.1. Đề Bài
Hãy viết một class `LruCache<K, V>` thuần Dart đáp ứng:
1. `get(K key)`: Lấy giá trị trong thời gian **$O(1)$**. Nếu đã hết hạn (TTL) thì xóa và trả về `null`.
2. `put(K key, V value, {Duration? ttl})`: Thêm hoặc cập nhật trong thời gian **$O(1)$**.
3. Nếu số lượng phần tử vượt quá `capacity`, tự động đẩy phần tử **ít được sử dụng nhất (Least Recently Used)** ra khỏi cache.
4. **Không dùng thư viện bên ngoài**.

### 1.2. Giải Pháp Chuẩn: HashMap Kết Hợp Doubly Linked List (Danh Sách Liên Kết Kép)

```dart
class _CacheNode<K, V> {
  final K key;
  V value;
  final DateTime? expiresAt;
  _CacheNode<K, V>? prev;
  _CacheNode<K, V>? next;

  _CacheNode(this.key, this.value, this.expiresAt);

  bool get isExpired => expiresAt != null && DateTime.now().isAfter(expiresAt!);
}

class LruCache<K, V> {
  final int capacity;
  final Map<K, _CacheNode<K, V>> _map = {};
  
  _CacheNode<K, V>? _head; // Phần tử mới dùng gần nhất (Most Recently Used)
  _CacheNode<K, V>? _tail; // Phần tử lâu nhất chưa dùng (Least Recently Used)

  LruCache(this.capacity) : assert(capacity > 0, 'Capacity must be greater than 0');

  V? get(K key) {
    final node = _map[key];
    if (node == null) return null;

    // Kiểm tra hết hạn TTL
    if (node.isExpired) {
      _removeNode(node);
      _map.remove(key);
      return null;
    }

    // Đưa node lên đầu danh sách (đánh dấu vừa được sử dụng)
    _moveToHead(node);
    return node.value;
  }

  void put(K key, V value, {Duration? ttl}) {
    final expiresAt = ttl != null ? DateTime.now().add(ttl) : null;

    if (_map.containsKey(key)) {
      final node = _map[key]!;
      node.value = value;
      _moveToHead(node);
      return;
    }

    // Nếu đã chạm trần dung lượng, xóa phần tử ở đuôi (tail - LRU)
    if (_map.length >= capacity) {
      if (_tail != null) {
        _map.remove(_tail!.key);
        _removeNode(_tail!);
      }
    }

    final newNode = _CacheNode(key, value, expiresAt);
    _addToHead(newNode);
    _map[key] = newNode;
  }

  void _addToHead(_CacheNode<K, V> node) {
    node.next = _head;
    node.prev = null;
    if (_head != null) _head!.prev = node;
    _head = node;
    _tail ??= node;
  }

  void _removeNode(_CacheNode<K, V> node) {
    if (node.prev != null) {
      node.prev!.next = node.next;
    } else {
      _head = node.next;
    }

    if (node.next != null) {
      node.next!.prev = node.prev;
    } else {
      _tail = node.prev;
    }
  }

  void _moveToHead(_CacheNode<K, V> node) {
    _removeNode(node);
    _addToHead(node);
  }
}
```

---

## BÀI 2: Tự Viết Custom `RenderBox` (LeafRenderObjectWidget)

### 2.1. Đề Bài
Thay vì dùng `CustomPaint` thông thường, hãy tạo một Widget vẽ **Vòng Tròn Tiến Trình (Circular Progress Ring)** bằng cách kế thừa trực tiếp từ **`LeafRenderObjectWidget`** và **`RenderBox`**, tối ưu hóa việc vẽ và tính toán kích thước theo đúng chuẩn Flutter Engine.

### 2.2. Giải Pháp Chuẩn

```dart
import 'dart:math' as math;
import 'package:flutter/rendering.dart';
import 'package:flutter/widgets.dart';

class CustomDonutProgress extends LeafRenderObjectWidget {
  final double progress; // 0.0 -> 1.0
  final Color progressColor;
  final Color trackColor;
  final double strokeWidth;

  const CustomDonutProgress({
    super.key,
    required this.progress,
    required this.progressColor,
    this.trackColor = const Color(0xFFEEEEEE),
    this.strokeWidth = 8.0,
  });

  @override
  RenderDonutProgress createRenderObject(BuildContext context) {
    return RenderDonutProgress(
      progress: progress,
      progressColor: progressColor,
      trackColor: trackColor,
      strokeWidth: strokeWidth,
    );
  }

  @override
  void updateRenderObject(BuildContext context, RenderDonutProgress renderObject) {
    renderObject
      ..progress = progress
      ..progressColor = progressColor
      ..trackColor = trackColor
      ..strokeWidth = strokeWidth;
  }
}

class RenderDonutProgress extends RenderBox {
  double _progress;
  Color _progressColor;
  Color _trackColor;
  double _strokeWidth;

  RenderDonutProgress({
    required double progress,
    required Color progressColor,
    required Color trackColor,
    required double strokeWidth,
  })  : _progress = progress,
        _progressColor = progressColor,
        _trackColor = trackColor,
        _strokeWidth = strokeWidth;

  // Setter tối ưu: Chỉ markNeedsPaint() khi đổi màu/tiến trình (không layout lại)
  set progress(double val) {
    if (_progress == val) return;
    _progress = val;
    markNeedsPaint();
  }

  set progressColor(Color val) {
    if (_progressColor == val) return;
    _progressColor = val;
    markNeedsPaint();
  }

  set trackColor(Color val) {
    if (_trackColor == val) return;
    _trackColor = val;
    markNeedsPaint();
  }

  set strokeWidth(double val) {
    if (_strokeWidth == val) return;
    _strokeWidth = val;
    markNeedsLayout(); // Đổi độ dày có thể ảnh hưởng kích thước
  }

  // Layout Phase: Áp dụng quy tắc Constraints go down, Sizes go up
  @override
  void performLayout() {
    // Nhận ràng buộc từ cha và chọn kích thước mong muốn (Default 100x100)
    size = constraints.constrain(const Size(100, 100));
  }

  // Paint Phase: Vẽ DisplayList lên Canvas
  @override
  void paint(PaintingContext context, Offset offset) {
    final Canvas canvas = context.canvas;
    final center = offset + Offset(size.width / 2, size.height / 2);
    final radius = (math.min(size.width, size.height) - _strokeWidth) / 2;

    final trackPaint = Paint()
      ..color = _trackColor
      ..style = PaintingStyle.stroke
      ..strokeWidth = _strokeWidth;

    // 1. Vẽ vòng tròn nền
    canvas.drawCircle(center, radius, trackPaint);

    final progressPaint = Paint()
      ..color = _progressColor
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round
      ..strokeWidth = _strokeWidth;

    // 2. Vẽ cung tiến trình
    final sweepAngle = 2 * math.pi * _progress.clamp(0.0, 1.0);
    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      -math.pi / 2, // Bắt đầu từ đỉnh 12 giờ
      sweepAngle,
      false,
      progressPaint,
    );
  }
}
```

---

## BÀI 3: Tự Viết `StreamTransformer` (Debounce & Distinct)

### 3.1. Đề Bài
Viết một hàm mở rộng `debounceDistinct<T>()` cho `Stream<T>` thuần Dart:
- Bỏ qua các giá trị trùng lặp liên tiếp (`distinct`).
- Chỉ phát ra giá trị khi luồng dữ liệu tạm lắng xuống sau khoảng thời gian `Duration` (`debounce`).
- Phải đảm bảo hủy Timer và đóng Controller an toàn khi nguồn stream kết thúc hoặc listener ngắt kết nối.

### 3.2. Giải Pháp Chuẩn

```dart
import 'dart:async';

extension StreamExtensions<T> on Stream<T> {
  Stream<T> debounceDistinct(Duration duration) {
    late StreamController<T> controller;
    StreamSubscription<T>? subscription;
    Timer? timer;
    T? lastEmittedValue;
    bool hasEmitted = false;

    controller = StreamController<T>(
      sync: true,
      onListen: () {
        subscription = listen(
          (event) {
            // Kiểm tra trùng lặp với giá trị gần nhất đã phát
            if (hasEmitted && lastEmittedValue == event) {
              timer?.cancel();
              return;
            }

            timer?.cancel();
            timer = Timer(duration, () {
              lastEmittedValue = event;
              hasEmitted = true;
              controller.add(event);
            });
          },
          onError: controller.addError,
          onDone: () {
            timer?.cancel();
            controller.close();
          },
        );
      },
      onPause: () => subscription?.pause(),
      onResume: () => subscription?.resume(),
      onCancel: () {
        timer?.cancel();
        return subscription?.cancel();
      },
    );

    return controller.stream;
  }
}
```
