# Bài 3.1: Flutter Rendering Pipeline — VSync đến Raster

> **Cấp độ**: Staff Engineer  
> **Thời gian đọc**: ~35 phút  
> **Yêu cầu**: Hiểu Widget/Element/RenderObject tree; đã dùng DevTools cơ bản

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Frame 32ms — Tìm thủ phạm ở đâu?

**E-commerce app production, Android Pixel 6:**

```
Triệu chứng: Animation sản phẩm "flip card" giật khi scroll nhanh
DevTools Timeline hiển thị:
  Frame 1: 8ms  (normal)
  Frame 2: 8ms  (normal) 
  Frame 3: 32ms (JANK — vượt 16.6ms budget)
  Frame 4: 9ms  (normal)

Câu hỏi: 32ms đó nằm ở giai đoạn nào? UI thread hay Raster thread?
         → Câu trả lời quyết định hoàn toàn hướng sửa lỗi.
```

Để trả lời câu hỏi này, cần hiểu toàn bộ pipeline.

---

## Phần 2 — Low-Level Mechanics

### 2.1. 6 Giai đoạn Rendering Pipeline

```
VSync Signal (từ Display Hardware)
         │
         │ 16.6ms budget bắt đầu
         ▼
┌────────────────────────────────────────────────────────────────┐
│ Phase 1: ANIMATE                                               │
│  - SchedulerBinding.handleBeginFrame()                         │
│  - Tick tất cả AnimationController (tính giá trị Tween mới)   │
│  - Tick tất cả registered FrameCallback                        │
│  - Chi phí: O(n) animations                                    │
└─────────────────────────────┬──────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────────┐
│ Phase 2: BUILD                                                 │
│  - SchedulerBinding.handleDrawFrame()                          │
│  - BuildOwner.buildScope() → gọi build() của dirty Elements   │
│  - Tạo/update Widget tree                                      │
│  - Diffing: so sánh Widget cũ vs mới → cập nhật Element       │
│  - Chi phí: O(dirty widgets) — mục tiêu <2ms                  │
└─────────────────────────────┬──────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────────┐
│ Phase 3: LAYOUT                                                │
│  - PipelineOwner.flushLayout()                                 │
│  - RenderObject.performLayout() từ root xuống leaf             │
│  - Tính toán size và position (BoxConstraints)                 │
│  - Chi phí: O(n) render objects — single pass                  │
└─────────────────────────────┬──────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────────┐
│ Phase 4: PAINT                                                 │
│  - PipelineOwner.flushPaint()                                  │
│  - RenderObject.paint() → vẽ vào PictureRecorder              │
│  - Tạo DisplayList (danh sách lệnh vẽ — chưa rasterize)       │
│  - Chi phí: O(dirty render objects)                            │
└─────────────────────────────┬──────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────────┐
│ Phase 5: COMPOSITE                                             │
│  - RenderView.compositeFrame()                                 │
│  - Gộp các Layer thành LayerTree                               │
│  - Quyết định layer nào cần rasterize lại                      │
│  - Chi phí: O(layers changed)                                  │
└─────────────────────────────┬──────────────────────────────────┘
                              │ Gửi LayerTree sang Raster Thread
┌─────────────────────────────▼──────────────────────────────────┐
│ Phase 6: RASTER (Raster Thread — SONG SONG với UI Thread)     │
│  - Impeller/Skia nhận LayerTree                                │
│  - Compile DisplayList → GPU commands (Metal/Vulkan)           │
│  - GPU rasterize → Pixel buffer                                │
│  - Swap buffer → Màn hình hiển thị                             │
│  - Chi phí: phụ thuộc vào độ phức tạp GPU operations           │
└────────────────────────────────────────────────────────────────┘
```

### 2.2. UI Thread vs Raster Thread — Chạy song song

```
VSync ──► UI Thread (Dart VM):
           Animate + Build + Layout + Paint + Composite
           ████████████████████│
                               └──► Gửi LayerTree

           Raster Thread (C++ Engine, GPU):
                                ████████████████████│
                                Nhận LayerTree → Rasterize → Hiển thị

JANK xảy ra khi:
1. UI Thread > 16.6ms: Raster thread phải chờ → frame missed
2. Raster Thread > 16.6ms: Dù UI xong → GPU chưa xong → frame missed
```

### 2.3. Impeller vs Skia — Tại sao Impeller loại bỏ Shader Jank

```
Skia (Flutter < 3.10 trên iOS, < 3.27 trên Android):
  Frame đầu tiên animation:
    1. Skia gặp shader mới → COMPILE shader tại runtime
    2. Shader compile: 30-500ms (tùy GPU)
    3. Frame bị dropped trong khi compile
    → "First-frame jank" — nổi tiếng khó fix

  Workaround cũ: SkSL Shader warm-up
    - Thu thập shader lúc test
    - Bundle vào app
    - Warm-up khi app launch
    - Vẫn không hoàn hảo — shader thay đổi theo device

Impeller (Flutter 3.10+ iOS, 3.27+ Android mặc định):
  Pre-compile ALL shaders khi build app:
    1. Mọi shader được compile thành Metal PSO / Vulkan pipeline object
    2. Lưu vào IPA/APK
    3. Runtime: không bao giờ compile shader → zero shader jank
    
  Chi phí:
    - Build time: +30-60 giây
    - App size: +2-5MB
    - Runtime: 0ms shader compile jank
```

### 2.4. LayerTree và RepaintBoundary

```
Widget Tree:
  MaterialApp
  └── Scaffold
      └── ListView
          ├── ProductCard_1 (animation đang chạy)
          │   └── FlipAnimation
          │       └── Image
          └── ProductCard_2 (static)
          └── ProductCard_3 (static)

LayerTree SỐ khi KHÔNG có RepaintBoundary:
  RootLayer
  └── TransformLayer (ListView scroll offset)
      └── PictureLayer (toàn bộ viewport — 1 layer duy nhất)
      
→ Khi ProductCard_1 animate → paint() lại TOÀN BỘ PictureLayer
→ Bao gồm cả ProductCard_2, _3 dù chúng static
→ Lãng phí GPU: repaint 3x thay vì 1x

LayerTree KHI có RepaintBoundary trên ProductCard_1:
  RootLayer
  └── TransformLayer (ListView scroll offset)
      ├── PictureLayer (ProductCard_1 — riêng biệt)
      │   └── [Cache: GPU texture của Card_1]
      └── PictureLayer (ProductCard_2 + _3 — chung)
          └── [Cache: GPU texture không đổi — KHÔNG repaint]

→ Khi ProductCard_1 animate → chỉ repaint PictureLayer của Card_1
→ ProductCard_2, _3: đọc từ GPU cache (DisplayList cached)
→ Tiết kiệm: 67% GPU paint work
```

---

## Phần 3 — Production Code Implementation

### 3.1. Đọc DevTools Timeline — Phân tích frame 32ms

```
Cách đọc DevTools Performance Timeline:

Frame event bar (top):
├── Xanh dương < 16ms: Normal frame ✓
├── Vàng 16-32ms: Frame over budget (jank)  
└── Đỏ > 32ms: Severe jank (skip frame)

Track "UI": Dart VM thread
  ├── "Animate": thời gian xử lý AnimationController.tick
  ├── "Build": thời gian tất cả Widget.build()
  ├── "Layout": thời gian RenderObject.performLayout()  
  └── "Paint": thời gian RenderObject.paint()
  
Track "Raster": Raster thread
  ├── "Prepare frame": LayerTree processing
  └── "Rasterize": GPU commands execution
  
CHẨN ĐOÁN:
- Cột "UI" cao → bug trong Dart code (build, layout, paint)
- Cột "Raster" cao → bug trong GPU operations (saveLayer, complex path)
```

```dart
// Thêm Timeline markers trong code để trace chính xác
import 'dart:developer' as developer;

class ProductListScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    developer.Timeline.startSync('ProductListScreen.build'); // Bắt đầu marker
    
    final result = _buildContent(context);
    
    developer.Timeline.finishSync(); // Kết thúc marker — visible trong DevTools
    return result;
  }
  
  Widget _buildContent(BuildContext context) {
    // ... actual build logic
    return ListView.builder(
      itemCount: 50,
      itemBuilder: (context, index) {
        developer.Timeline.startSync('ProductCard.build[$index]');
        final card = ProductCard(index: index);
        developer.Timeline.finishSync();
        return card;
      },
    );
  }
}
```

### 3.2. SchedulerBinding — Hook vào rendering pipeline

```dart
// Đo thời gian frame thực tế từ Dart code
class FrameMetricsTracker extends StatefulWidget {
  const FrameMetricsTracker({super.key, required this.child});
  final Widget child;

  @override
  State<FrameMetricsTracker> createState() => _FrameMetricsTrackerState();
}

class _FrameMetricsTrackerState extends State<FrameMetricsTracker>
    with WidgetsBindingObserver {
  final List<FrameTiming> _frameTimings = [];
  
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }
  
  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }
  
  // Callback sau mỗi frame — có timing breakdown chính xác
  @override
  void didChangeMetrics() {}
  
  @override
  void reportTimings(List<FrameTiming> timings) {
    for (final timing in timings) {
      final buildDuration = timing.buildDuration;
      final rasterDuration = timing.rasterDuration;
      final totalDuration = timing.totalSpan;
      
      if (totalDuration.inMilliseconds > 16) {
        debugPrint(
          '[JANK] Frame ${timing.frameNumber}: '
          'Build=${buildDuration.inMilliseconds}ms, '
          'Raster=${rasterDuration.inMilliseconds}ms, '
          'Total=${totalDuration.inMilliseconds}ms',
        );
      }
    }
    
    if (kReleaseMode) {
      // Gửi slow frame metrics lên analytics
      final slowFrames = timings.where(
        (t) => t.totalSpan.inMilliseconds > 16,
      );
      for (final frame in slowFrames) {
        unawaited(AnalyticsService.instance.logSlowFrame(
          buildMs: frame.buildDuration.inMilliseconds,
          rasterMs: frame.rasterDuration.inMilliseconds,
        ));
      }
    }
  }

  @override
  Widget build(BuildContext context) => widget.child;
}
```

### 3.3. PipelineOwner — Kiểm soát dirty render objects

```dart
// Hiểu tại sao const Widget giúp giảm Build phase

class ProductCard extends StatelessWidget {
  const ProductCard({          // ← const constructor
    super.key,
    required this.product,
  });
  final Product product;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          // ✅ const widget không bị đưa vào dirty list
          // → Build phase skip hoàn toàn widget này
          const Padding(
            padding: EdgeInsets.all(8),
            child: Icon(Icons.star, color: Colors.amber),
          ),
          
          // Widget phụ thuộc data — không thể const
          // → Phải qua Build phase mỗi frame (nếu parent dirty)
          Text(product.name),
          Text(product.price.toString()),
        ],
      ),
    );
  }
}

// Khi parent rebuild:
// - const Icon(Icons.star) → bị skip bởi Element.updateChild()
//   vì Widget.operator== trả về true (same type + same key + const = identity equal)
// - Text(product.name) → phải qua canUpdate() check → có thể skip nếu name không đổi
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Frame Budget Allocation (16.6ms)

```
Khuyến nghị phân bổ thời gian cho 1 frame (60fps):

Phase          Target    Max      Nếu vượt
─────────────────────────────────────────────────────
Animate        <1ms      2ms      Reduce animation objects
Build          <4ms      6ms      const, RepaintBoundary, keys
Layout         <2ms      3ms      tránh IntrinsicWidth/Height
Paint          <3ms      5ms      tránh saveLayer, opacity widget
Composite      <1ms      2ms      giảm số layer
─────────────────────────────────────────────────────
UI Thread      <8ms      13ms     (Dart execution total)
Raster Thread  <8ms      13ms     (GPU execution total)
─────────────────────────────────────────────────────
TOTAL          <16ms     16.6ms   Margin: ~1ms cho overhead
```

### Impeller Performance trên thiết bị thực

```
Test: Complex animation (ParticleSystem với 200 particles)
Device: iPhone 14 Pro (120Hz ProMotion)
Budget per frame: 8.33ms

              Skia (Flutter 3.0)    Impeller (Flutter 3.16+)
─────────────────────────────────────────────────────────────
First frame   145ms (shader comp)   9ms (pre-compiled)
Steady state  12ms avg              7ms avg
Jank frames   18% (shader comp)     <0.5%
GPU memory    380MB                  290MB (-24%)
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Không biết frame bị jank ở UI thread hay Raster thread
    ✅ Mở DevTools → Performance tab → kiểm tra cột UI vs Raster
    Lý do: Fix sai tầng → tốn thời gian, không giải quyết vấn đề

[ ] ❌ Gọi heavy computation (sort, regex, JSON parse) trong build()
    ✅ Cache kết quả trong State hoặc đẩy sang Isolate
    Lý do: build() nằm trên UI Thread → 60fps budget = 16.6ms total

[ ] ❌ Animation trên toàn screen không có RepaintBoundary
    ✅ Wrap animated widget trong RepaintBoundary
    Lý do: Mọi frame animation → repaint toàn screen → Raster thread overload

[ ] ❌ Dùng Skia shader warm-up thay vì upgrade lên Impeller
    ✅ Upgrade Flutter 3.27+ → Impeller mặc định trên Android
    Lý do: Impeller loại bỏ shader jank hoàn toàn, không phải workaround

[ ] ❌ Benchmark trong Debug mode (flutter run)
    ✅ flutter run --profile hoặc flutter run --release
    Lý do: Debug mode chậm 2-10x do JIT + assertions overhead

[ ] ❌ Không dùng reportTimings() để detect slow frames ở production
    ✅ Implement WidgetsBindingObserver.reportTimings() + gửi lên analytics
    Lý do: Không có visibility khi user gặp jank ở production

[ ] ❌ Ignore "Impeller" warning trên Android do thiếu Vulkan support
    ✅ Test trên thiết bị Android API 29+ để đảm bảo Vulkan support
    Lý do: Android API < 29 fall back về Skia — cần test cả 2 path
```
