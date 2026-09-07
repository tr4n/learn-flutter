# Bài 3.2: Jank Elimination — RepaintBoundary & GPU Optimization

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Đã đọc Bài 3.1 (Rendering Pipeline)

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Animation 60fps → 18fps

**Social Media app, feed với video preview:**

```
User report: Khi scroll feed, video card có animation "shimmer loading"
bị giật nặng. FPS đo được: 18fps (target: 60fps)

DevTools chẩn đoán:
- UI Thread: 6ms (bình thường)
- Raster Thread: 48ms (CRITICAL — gấp 3x budget)

→ Vấn đề ở Raster Thread: GPU operations quá tốn kém
```

Root cause phân tích: `BackdropFilter(filter: ImageFilter.blur(...))` được dùng cho "frosted glass" effect trên video overlay, đặt bên trong `ListView` → mỗi frame scroll → mỗi item có BackdropFilter rebuild → GPU phải allocate off-screen buffer cho mỗi item.

---

## Phần 2 — Low-Level Mechanics

### 2.1. Tại sao `saveLayer()` tốn kém?

```
Canvas.saveLayer() là gì:
GPU phải allocate một Texture mới ngoài màn hình (off-screen buffer):

Normal rendering (không có saveLayer):
  Widget → DisplayList commands → GPU rasterize trực tiếp lên framebuffer
  Framebuffer: Screen (ví dụ 1440x3120 pixel × 4 bytes/pixel = 17MB)

Với saveLayer (ví dụ Opacity widget bao 1 phần màn hình):
  1. GPU allocate off-screen texture: kích thước = bounding box widget
     Ví dụ: 400x600 pixel × 4 bytes = 960KB VRAM
  2. Render toàn bộ subtree widget vào off-screen texture
  3. Apply effect (blend mode, opacity, color filter)
  4. Composite off-screen texture lên framebuffer chính
  
Chi phí:
  - VRAM allocation: ~1-5ms tùy kích thước
  - 2 render passes: off-screen + composite
  - Nếu có nhiều saveLayer lồng nhau: mỗi cấp thêm 1 off-screen buffer
```

### 2.2. Các Widget gọi `saveLayer()` ẩn

```
Widget          saveLayer?   Chi phí     Thay thế
──────────────────────────────────────────────────────────────────
Opacity         YES          ★★★★☆       FadeTransition, withOpacity
ClipRRect       YES (nếu     ★★☆☆☆       ShaderMask (1 pass),
                antiAlias)               Container(decoration)
ClipPath        YES          ★★★★★       Custom painting
BackdropFilter  YES          ★★★★★       Không có thay thế hoàn hảo
ShaderMask      YES          ★★★☆☆       Không có thay thế
ColorFiltered   YES          ★★☆☆☆       Không có thay thế (nhẹ hơn)
ImageFilter     YES          ★★★★★       Cache image filter result
PhysicalModel   YES (nếu     ★★★☆☆       DecoratedBox + ClipRRect
                elevation>0)
```

### 2.3. RepaintBoundary — Cách hoạt động chi tiết

```
Không có RepaintBoundary:
Widget Tree:            Layer Tree:
ScrollView              TransformLayer (scroll offset)
└─ AnimatedCard_1       └─ PictureLayer (TẤT CẢ)
└─ StaticCard_2             [DrawCard_1, DrawCard_2, DrawCard_3]
└─ StaticCard_3
                        Khi Card_1 animate:
                        → Toàn bộ PictureLayer dirty
                        → Repaint tất cả cards
                        → GPU: vẽ lại Card_2, Card_3 dù không đổi

Có RepaintBoundary trên AnimatedCard_1:
Widget Tree:            Layer Tree:
ScrollView              TransformLayer (scroll offset)
└─ RepaintBoundary      ├─ PictureLayer (chỉ Card_1)
   └─ AnimatedCard_1    │   [Animated content]
└─ StaticCard_2         └─ PictureLayer (Card_2 + Card_3)
└─ StaticCard_3             [Static content — CACHED in GPU]

Khi Card_1 animate:
→ Chỉ PictureLayer(Card_1) dirty → repaint Card_1 only
→ PictureLayer(Card_2+3) từ GPU cache — KHÔNG repaint
→ Tiết kiệm: 66% paint work

KHI NÀO RepaintBoundary HẠI?
→ Thêm RepaintBoundary = thêm 1 GPU texture (VRAM)
→ Mỗi RepaintBoundary ~1-5MB VRAM tùy kích thước widget
→ Nếu widget không animate/không thay đổi: RepaintBoundary là LÃNG PHÍ
→ Nếu widget thay đổi mỗi frame (trong animation): RepaintBoundary có lợi
```

### 2.4. CustomPainter — `shouldRepaint` tối ưu

```dart
// shouldRepaint quyết định có gọi paint() lại không khi widget rebuild
class ChartPainter extends CustomPainter {
  const ChartPainter({required this.data, required this.color});
  final List<double> data;
  final Color color;

  @override
  void paint(Canvas canvas, Size size) {
    // Vẽ chart... (có thể tốn 2-5ms)
  }

  @override
  bool shouldRepaint(ChartPainter oldDelegate) {
    // ❌ Tệ nhất: luôn repaint
    // return true;

    // ❌ Tệ: so sánh object reference (luôn true vì data là List mới)
    // return data != oldDelegate.data;

    // ✅ Đúng: so sánh giá trị thực sự thay đổi
    return !listEquals(data, oldDelegate.data) || color != oldDelegate.color;
    //              ↑ flutter/foundation: deep comparison của List
  }

  @override
  bool shouldRebuildSemantics(ChartPainter oldDelegate) => false;
  // Tắt semantic rebuild nếu không cần accessibility
}
```

---

## Phần 3 — Production Code Implementation

### 3.1. Fix BackdropFilter trong ListView

```dart
// ❌ VẤN ĐỀ: BackdropFilter trong mỗi list item
class VideoFeedItem extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        VideoThumbnail(url: video.thumbnailUrl),
        // BackdropFilter gọi saveLayer() cho mỗi item
        // 20 items visible → 20 saveLayer() operations mỗi frame
        BackdropFilter(
          filter: ImageFilter.blur(sigmaX: 10, sigmaY: 10),
          child: Container(
            color: Colors.black.withOpacity(0.3),
            child: Text(video.title, style: const TextStyle(color: Colors.white)),
          ),
        ),
      ],
    );
  }
}

// ✅ GIẢI PHÁP 1: Pre-render blur, cache kết quả
class VideoFeedItemOptimized extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        VideoThumbnail(url: video.thumbnailUrl),
        // Thay BackdropFilter bằng semi-transparent gradient
        // Không cần saveLayer — đơn giản là 1 draw call
        DecoratedBox(
          decoration: const BoxDecoration(
            gradient: LinearGradient(
              begin: Alignment.topCenter,
              end: Alignment.bottomCenter,
              colors: [
                Colors.transparent,
                Color(0xBB000000), // 73% opacity black
              ],
            ),
          ),
          child: Padding(
            padding: const EdgeInsets.all(12),
            child: Text(video.title, style: const TextStyle(color: Colors.white)),
          ),
        ),
      ],
    );
  }
}

// ✅ GIẢI PHÁP 2: Nếu PHẢI dùng blur — isolate bằng RepaintBoundary
// Chỉ dùng khi blur là bắt buộc theo thiết kế (VD: iOS style)
class VideoFeedItemWithBlur extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        VideoThumbnail(url: video.thumbnailUrl),
        // RepaintBoundary tạo GPU texture riêng cho blur overlay
        // → BackdropFilter chỉ ảnh hưởng khu vực nhỏ, không cả ListView
        RepaintBoundary(
          child: BackdropFilter(
            filter: ImageFilter.blur(sigmaX: 8, sigmaY: 8),
            child: Container(
              color: Colors.black12,
              child: Text(video.title, style: const TextStyle(color: Colors.white)),
            ),
          ),
        ),
      ],
    );
  }
}
```

### 3.2. Opacity Widget — Cách đúng và sai

```dart
// ❌ NGUYÊN NHÂN JANK: Opacity widget bao container phức tạp
class FadeInProductCard extends StatefulWidget {
  // ...
}

class _FadeInProductCardState extends State<FadeInProductCard>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _opacity;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 500),
    );
    _opacity = CurvedAnimation(parent: _controller, curve: Curves.easeIn);
    _controller.forward();
  }

  @override
  Widget build(BuildContext context) {
    return Opacity(
      opacity: _opacity.value,
      // ❌ Opacity bao toàn bộ ProductCard phức tạp
      // → Mỗi frame animation gọi saveLayer() cho toàn bộ card
      child: ProductCard(product: widget.product),
    );
  }
}

// ✅ GIẢI PHÁP 1: AnimatedOpacity — tự động tối ưu hóa
// Flutter render AnimatedOpacity mà không cần saveLayer trong một số trường hợp
class FadeInProductCardOptimized extends StatefulWidget { /* ... */ }

class _FadeInProductCardOptimizedState extends State<FadeInProductCardOptimized>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  Widget build(BuildContext context) {
    return FadeTransition(
      // FadeTransition dùng opacity layer property thay vì saveLayer
      // → GPU handle opacity compositing ở hardware level, không qua CPU
      opacity: _controller,
      child: ProductCard(product: widget.product),
    );
  }
}

// ✅ GIẢI PHÁP 2: withOpacity() trên màu thuần
// Nếu chỉ cần semi-transparent container (không phải widget tree)
Container(
  color: Colors.black.withOpacity(0.5),
  // withOpacity = 1 paint call với alpha channel
  // Không gọi saveLayer vì không có child cần blend riêng
)

// ✅ GIẢI PHÁP 3: Opacity với child đơn giản — OK
// saveLayer chỉ tốn kém khi child là complex widget tree
Opacity(
  opacity: 0.5,
  child: const Icon(Icons.star), // 1 widget đơn giản — chi phí nhỏ
)
```

### 3.3. ClipRRect tối ưu

```dart
// ❌ ClipRRect với antiAlias = true (mặc định) → saveLayer
ClipRRect(
  borderRadius: BorderRadius.circular(12),
  child: Image.network(url, width: 200, height: 200),
)

// ✅ Thay bằng BoxDecoration → không cần clip, không saveLayer
Container(
  width: 200,
  height: 200,
  decoration: BoxDecoration(
    borderRadius: BorderRadius.circular(12),
    image: DecorationImage(
      image: NetworkImage(url),
      fit: BoxFit.cover,
    ),
  ),
)

// ✅ Hoặc: ClipRRect với clipBehavior = Clip.hardEdge (không antiAlias)
ClipRRect(
  borderRadius: BorderRadius.circular(12),
  clipBehavior: Clip.hardEdge, // Không saveLayer, nhưng edge có thể jagged
  child: Image.network(url, width: 200, height: 200),
)
```

### 3.4. Bật debugRepaintRainbows để visualization

```dart
// Chỉ bật trong debug build — zero cost in release
void main() {
  // Bật rainbow overlay — mỗi lần repaint widget đổi màu ngẫu nhiên
  // Nếu thấy màu thay đổi liên tục mỗi frame → widget đó đang repaint thừa
  debugRepaintRainbowsEnabled = true;
  // debugPaintLayerBordersEnabled = true; // Thấy ranh giới layer
  
  runApp(const MyApp());
}
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Kết quả đo sau khi fix BackdropFilter

```
Device: Pixel 6 (Android 13), Flutter 3.24 (Impeller)
Scenario: VideoFeed ListView với 15 items visible, auto-scroll

                    TRƯỚC FIX        SAU FIX (Solution 1)
────────────────────────────────────────────────────────
FPS (avg)           18 fps           58 fps
Raster thread (ms)  48ms/frame       9ms/frame  (-81%)
VRAM usage          285MB            120MB      (-58%)
Dropped frames      72% (jank)       2% (normal)
────────────────────────────────────────────────────────
```

### RepaintBoundary Trade-off

```
RepaintBoundary trên widget THAY ĐỔI MỖI FRAME:
  CPU overhead: -40% (không repaint static siblings)
  VRAM overhead: +3-8MB per boundary (new GPU texture)
  
  Net result: BENEFIT khi animated widget < 30% viewport area

RepaintBoundary trên widget STATIC (không thay đổi):
  CPU overhead: 0 (static widget không repaint dù có hay không)
  VRAM overhead: +3-8MB per boundary
  
  Net result: WASTE VRAM — không thêm RepaintBoundary vào static widget
```

---

## Phần 5 — Production Checklist

```
[ ] ❌ Dùng Opacity widget bao phủ toàn bộ widget tree phức tạp
    ✅ FadeTransition cho animation, withOpacity() cho màu sắc thuần
    Lý do: Opacity → saveLayer() → off-screen buffer mỗi frame animation

[ ] ❌ BackdropFilter bên trong ListView.builder itemBuilder
    ✅ Pre-render blur into image, hoặc dùng gradient thay thế
    Lý do: N items visible → N saveLayer() → Raster thread overload

[ ] ❌ ClipRRect với antiAlias=true trong list item
    ✅ BoxDecoration với borderRadius, hoặc clipBehavior: Clip.hardEdge
    Lý do: antiAlias = saveLayer, cost tăng linear với số items

[ ] ❌ CustomPainter.shouldRepaint() luôn return true
    ✅ So sánh actual data changes: listEquals(), color!=, ...
    Lý do: Mỗi parent rebuild → paint() gọi lại → tốn CPU không cần thiết

[ ] ❌ Thêm RepaintBoundary cho tất cả widget "để an toàn"
    ✅ Chỉ thêm cho widget có animation hoặc thay đổi thường xuyên
    Lý do: Mỗi RepaintBoundary tốn 3-8MB VRAM → thiết bị RAM ít bị crash

[ ] ❌ Không bật Performance Overlay trong quá trình development
    ✅ flutter run --profile + Performance Overlay luôn bật khi review animation
    Lý do: Debug mode quá chậm để detect jank thực sự

[ ] ❌ Bỏ qua "Raster Cache" trong DevTools Layer panel
    ✅ Kiểm tra widget nào được GPU cache, widget nào không
    Lý do: Widget không được cache → repaint mỗi frame dù không thay đổi

[ ] ❌ saveLayer() trong CustomPainter.paint() thủ công
    ✅ Chỉ dùng saveLayer khi thực sự cần blend complex paths
    Lý do: Mỗi saveLayer thêm 1-10ms Raster time tùy device
```
