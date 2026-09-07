# Bài 5.1: Platform Channels Internals — BinaryMessenger & JNI/ObjC Bridge

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Biết cơ bản Kotlin (Android) và Swift (iOS)

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: MethodChannel trong animation loop — 2 giây lag

**Navigation app dùng native location:**

```dart
// ❌ Gọi MethodChannel trong Timer callback — mỗi giây 1 lần
Timer.periodic(const Duration(seconds: 1), (_) async {
  final location = await _locationChannel.invokeMethod('getCurrentLocation');
  setState(() => _currentLocation = location);
});
```

```
Kết quả đo lường:
  MethodChannel call latency: 12-28ms mỗi lần
  Với 60fps screen: frame budget = 16.6ms
  12ms cho channel call → 72% frame budget tiêu hết vào bridge
  → Animation trên map bị jank mỗi giây

Root cause: MethodChannel serialization + thread switching cost
```

---

## Phần 2 — Low-Level Mechanics

### 2.1. Kiến trúc BinaryMessenger

```
Flutter App (Dart VM)                 Platform (Android/iOS)
──────────────────────                ──────────────────────

MethodChannel.invokeMethod()
    │
    ▼
BinaryMessenger (Dart)
    │ serialize: StandardMessageCodec
    │ → ByteData (binary)
    │
    ▼
Flutter Engine (C++)
  Platform Task Runner
    │
    │ JNI call (Android) / ObjC message (iOS)
    │
    ▼
Platform Thread (Android: Main Thread!)
    │
    │ execute native code
    │
    ▼
Reply via BinaryReply
    │
    │ JNI callback
    │
    ▼
Flutter Engine (C++)
    │
    ▼
Dart UI Thread (async callback)
    │
    ▼
BinaryMessenger (Dart)
    │ deserialize: StandardMessageCodec
    │ → Dart objects
    │
    ▼
invokeMethod() Future completes

TỔNG CHI PHÍ: serialize + JNI + native code + JNI callback + deserialize
              ≈ 5-30ms tùy platform và payload size
```

### 2.2. StandardMessageCodec — Encoding Types

```
Dart Type         → Binary Encoding
──────────────────────────────────────
null              → 0x00 (1 byte)
bool (true)       → 0x01 (1 byte)
bool (false)      → 0x02 (1 byte)
int (32-bit)      → 0x03 + 4 bytes
int (64-bit)      → 0x04 + 8 bytes
double            → 0x06 + 8 bytes (IEEE 754)
String (UTF-8)    → 0x07 + length varint + bytes
Uint8List         → 0x08 + length varint + bytes
List              → 0x0C + length varint + [elements]
Map               → 0x0D + length varint + [key,value pairs]

Ví dụ: {'lat': 10.7769, 'lng': 106.7009}
  Encoding: 0x0D (Map) + 2 (2 entries)
            + 0x07 (String) + 3 (len) + "lat"
            + 0x06 (Double) + 8 bytes IEEE 754
            + 0x07 (String) + 3 (len) + "lng"  
            + 0x06 (Double) + 8 bytes IEEE 754
  Total: ~40 bytes (overhead ~20% so với raw double)
```

### 2.3. Android: Main Thread Requirement

```
Android Platform Channel handler PHẢI chạy trên Main Thread:

Dart → JNI call → Android Main Thread (UI Thread)
                         │
              ← Platform code PHẢI complete trước khi 
                Android Main Thread free lại
                
VẤN ĐỀ: Nếu native code làm IO (network, DB read):
  Android Main Thread blocked → Android ANR (App Not Responding)
  → System kill app sau 5 giây

GIẢI PHÁP:
  Kotlin coroutine với Dispatchers.IO trong Handler:
  
  MethodChannel(flutterEngine.dartExecutor, "location")
      .setMethodCallHandler { call, result ->
          if (call.method == "getCurrentLocation") {
              // Dispatch sang IO thread, không block Main Thread
              CoroutineScope(Dispatchers.IO).launch {
                  val location = locationManager.getLastKnownLocation()
                  withContext(Dispatchers.Main) {
                      result.success(mapOf("lat" to location.lat, "lng" to location.lng))
                  }
              }
          }
      }
```

### 2.4. 3 loại Channel — Chọn đúng loại

```
MethodChannel:
  Request → Response (one-shot)
  Dart gọi → Platform xử lý → Dart nhận kết quả
  Dùng: Lấy device info, mở camera, request permission
  Ví dụ: getBatteryLevel(), shareFile(), vibrate()

EventChannel:
  Platform → Dart (stream, ongoing)
  Platform push events liên tục → Dart subscribe stream
  Dùng: Location updates, sensor data, network state, screen state
  Ví dụ: locationStream, accelerometerStream, connectivityStream
  
BasicMessageChannel<T>:
  Bidirectional messaging (cả 2 chiều init được)
  Custom codec (BinaryCodec, StringCodec, JSONMessageCodec)
  Dùng: Custom protocol, large binary data, simple string passing
  Ví dụ: Raw byte transfer, JSON string passing
```

---

## Phần 3 — Production Code Implementation

### 3.1. MethodChannel Production Pattern

```dart
// features/device/data/datasources/battery_datasource.dart
abstract interface class BatteryDataSource {
  Future<int> getBatteryLevel();
  Future<bool> isCharging();
}

final class BatteryPlatformDataSource implements BatteryDataSource {
  static const _channel = MethodChannel('com.myapp/battery');
  
  @override
  Future<int> getBatteryLevel() async {
    try {
      final level = await _channel.invokeMethod<int>('getBatteryLevel');
      return level ?? -1; // -1 = unknown
    } on PlatformException catch (e) {
      // Map PlatformException sang domain failure
      throw switch (e.code) {
        'UNAVAILABLE' => const DeviceCapabilityException('Battery API không khả dụng'),
        'PERMISSION_DENIED' => const PermissionException('Cần quyền truy cập battery'),
        _ => PlatformBridgeException(message: e.message ?? 'Lỗi platform unknown', code: e.code),
      };
    } on MissingPluginException {
      throw const DeviceCapabilityException('Battery plugin chưa được đăng ký');
    }
  }
  
  @override
  Future<bool> isCharging() async {
    final result = await _channel.invokeMethod<bool>('isCharging');
    return result ?? false;
  }
}
```

```kotlin
// android/app/src/main/kotlin/com/myapp/BatteryPlugin.kt
class BatteryPlugin : FlutterPlugin, MethodCallHandler {
    private lateinit var channel: MethodChannel
    private lateinit var context: Context

    override fun onAttachedToEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        context = binding.applicationContext
        channel = MethodChannel(binding.binaryMessenger, "com.myapp/battery")
        channel.setMethodCallHandler(this)
    }

    override fun onMethodCall(call: MethodCall, result: Result) {
        when (call.method) {
            "getBatteryLevel" -> {
                val batteryManager = context.getSystemService(BATTERY_SERVICE) as BatteryManager
                val level = batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
                if (level != Int.MIN_VALUE) {
                    result.success(level)
                } else {
                    result.error("UNAVAILABLE", "Battery level không đọc được", null)
                }
            }
            "isCharging" -> {
                val batteryManager = context.getSystemService(BATTERY_SERVICE) as BatteryManager
                val status = batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_STATUS)
                result.success(status == BatteryManager.BATTERY_STATUS_CHARGING)
            }
            else -> result.notImplemented()
        }
    }

    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        channel.setMethodCallHandler(null)
    }
}
```

### 3.2. EventChannel — Location Stream

```dart
// features/location/data/datasources/location_stream_datasource.dart
final class LocationStreamDataSource {
  static const _eventChannel = EventChannel('com.myapp/location');
  
  Stream<LocationData> watchLocation() {
    return _eventChannel
        .receiveBroadcastStream()
        .map((dynamic event) {
          final map = event as Map<Object?, Object?>;
          return LocationData(
            lat: (map['lat'] as num).toDouble(),
            lng: (map['lng'] as num).toDouble(),
            accuracy: (map['accuracy'] as num?)?.toDouble() ?? 0.0,
            timestamp: DateTime.fromMillisecondsSinceEpoch(
              (map['timestamp'] as int?) ?? 0,
            ),
          );
        })
        .handleError((Object error) {
          if (error is PlatformException) {
            throw LocationException(error.message ?? 'Location error');
          }
          throw error;
        });
  }
}
```

```kotlin
// Android EventChannel implementation
class LocationPlugin : FlutterPlugin, EventChannel.StreamHandler {
    private var eventSink: EventChannel.EventSink? = null
    private lateinit var locationManager: LocationManager
    
    override fun onListen(arguments: Any?, events: EventChannel.EventSink) {
        eventSink = events
        // Bắt đầu listen location updates
        locationManager.requestLocationUpdates(
            LocationManager.GPS_PROVIDER,
            1000L, // minimum time: 1 giây
            10f,   // minimum distance: 10 meters
            locationListener
        )
    }
    
    override fun onCancel(arguments: Any?) {
        // QUAN TRỌNG: Phải removeUpdates để tránh battery drain
        locationManager.removeUpdates(locationListener)
        eventSink = null
    }
    
    private val locationListener = LocationListener { location ->
        // Chạy trên background thread → cần post về Main Thread
        Handler(Looper.getMainLooper()).post {
            eventSink?.success(mapOf(
                "lat" to location.latitude,
                "lng" to location.longitude,
                "accuracy" to location.accuracy,
                "timestamp" to location.time,
            ))
        }
    }
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Benchmark: MethodChannel vs EventChannel vs FFI

```
Test: Get GPS coordinates, 100 calls

MethodChannel (invoke per call):
  Latency per call: 12-28ms (serialize + JNI + deserialize)
  100 calls total: 1,200-2,800ms
  Thích hợp: One-shot queries (getBatteryLevel, getDeviceId)

EventChannel (stream):  
  Setup latency: 15ms (1 lần)
  Per update latency: 2-5ms (stream push, không cần round-trip)
  100 updates: ~300-500ms
  Thích hợp: Continuous data (location, sensors, network state)
  
Dart FFI (Direct C call):
  Latency per call: 0.1-0.5ms (direct function call)
  100 calls total: 10-50ms
  Thích hợp: High-frequency, compute-heavy native operations
  KHÔNG thích hợp: API có async/callback (Android system APIs)
```

### Khi nào dùng loại nào?

```
                    Tần suất gọi
              Thấp (1-10/s)    Cao (>30/s)
             ┌───────────────┬────────────────┐
  One-shot   │ MethodChannel │ MethodChannel  │
             │ (simple, safe)│ (OK, monitor   │
             │               │  latency)      │
  ───────────┼───────────────┼────────────────┤
  Continuous │ EventChannel  │ EventChannel   │
  stream     │               │ hoặc FFI       │
             │               │ (nếu critical) │
             └───────────────┴────────────────┘
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ MethodChannel.invokeMethod() trong build() hoặc animation callback
    ✅ Cache kết quả; dùng EventChannel cho data cần update thường xuyên
    Lý do: Channel call 12-28ms → tiêu hết frame budget → jank

[ ] ❌ PlatformException không được xử lý cụ thể
    ✅ Map PlatformException.code sang typed domain exception
    Lý do: e.toString() không đủ thông tin để xử lý từng loại lỗi khác nhau

[ ] ❌ Native code làm IO trực tiếp trên Android Main Thread
    ✅ Dispatch sang IO thread (Coroutines/AsyncTask), kết quả trả về Main Thread
    Lý do: Block Android Main Thread > 5s → ANR → app bị system kill

[ ] ❌ EventChannel.onCancel() không cleanup (removeUpdates, unsubscribe)
    ✅ Luôn cleanup tất cả native subscriptions trong onCancel()
    Lý do: Location update, sensor listener tiếp tục chạy → battery drain

[ ] ❌ Channel name không có namespace (channel: "location")
    ✅ Dùng reverse domain: "com.company.appname/feature_name"
    Lý do: Conflict với Flutter SDK channels hoặc third-party plugins

[ ] ❌ Gửi large object qua MethodChannel (List<1000 items>)
    ✅ Dùng Uint8List + BinaryCodec hoặc chuyển sang FFI + Dart:ffi
    Lý do: Serialization của complex objects tốn nhiều time hơn cần thiết
```
