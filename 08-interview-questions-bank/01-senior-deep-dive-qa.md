# Ngân Hàng Câu Hỏi Phỏng Vấn Toàn Diện: Junior $\rightarrow$ Middle $\rightarrow$ Senior

> **Mục tiêu**: Bộ ngân hàng câu hỏi phỏng vấn chuẩn hóa phân loại theo 3 cấp độ năng lực: từ kiến trúc nền tảng (Junior), kỹ thuật thực chiến & kiến trúc dự án (Middle), đến bản chất tầng sâu bên dưới & tối ưu hiệu năng (Senior).

---

## 🔰 CẤP ĐỘ 1: VÒNG PHỎNG VẤN JUNIOR (KIẾN THỨC NỀN TẢNG & BASE)

### Q1.1: Phân biệt sự khác nhau giữa `StatelessWidget` và `StatefulWidget`. Khi nào thì dùng loại nào?
- **StatelessWidget**:
  - Là widget bất biến (Immutable), không duy trì bất kỳ trạng thái nội bộ nào có thể thay đổi sau khi được tạo ra.
  - Giao diện của nó chỉ phụ thuộc duy nhất vào dữ liệu truyền vào qua Constructor.
  - *Khi nào dùng*: Các thành phần giao diện tĩnh như Text, Icon, Button hiển thị cố định, Avatar hình ảnh không đổi.
- **StatefulWidget**:
  - Duy trì một đối tượng `State` có thể biến đổi giá trị theo thời gian (dựa trên tương tác người dùng, nhận dữ liệu mạng, hoặc hẹn giờ Timer).
  - Khi gọi `setState()`, widget sẽ tự động kích hoạt hàm `build()` để vẽ lại giao diện với trạng thái mới.
  - *Khi nào dùng*: Form nhập liệu, nút Checkbox bật/tắt, màn hình danh sách có phân trang, hoạt ảnh chuyển động.

### Q1.2: Trình bày chi tiết các phương thức chính trong vòng đời của một `StatefulWidget` (`State` Lifecycle)?
1. `createState()`: Được Framework gọi để tạo instance của lớp `State`.
2. `initState()`: Chạy **duy nhất 1 lần** khi State được tạo và gắn vào cây Element. Dùng để khởi tạo controllers, lắng nghe Stream, hoặc gọi API lần đầu.
3. `didChangeDependencies()`: Chạy ngay sau `initState()` và mỗi khi một `InheritedWidget` mà widget này phụ thuộc (như `Theme.of(context)` hoặc `MediaQuery.of(context)`) thay đổi giá trị.
4. `build()`: Chạy nhiều lần để trả về cây Widget con mô tả giao diện tương ứng với State hiện tại.
5. `didUpdateWidget()`: Chạy khi Widget cha rebuild và truyền cấu hình Widget mới xuống cho State hiện tại (cùng runtimeType và Key).
6. `dispose()`: Chạy khi Widget bị gỡ bỏ vĩnh viễn khỏi màn hình. Bắt buộc phải hủy (cancel) Stream, Controller, Timer tại đây để tránh rò rỉ bộ nhớ (Memory Leak).

### Q1.3: Tại sao nên đặt từ khóa `const` trước constructor của một Widget bất cứ khi nào có thể?
- **Tối ưu bộ nhớ**: Dart chỉ tạo **duy nhất 1 instance** trên bộ nhớ tại thời điểm biên dịch (Compile-time Canonicalization) và tái sử dụng con trỏ đó ở mọi nơi.
- **Bỏ qua Rebuild**: Khi Widget cha rebuild, nếu Widget con được khai báo là `const`, Flutter nhận biết con trỏ đối tượng không hề thay đổi (`identical == true`), do đó Framework sẽ **bỏ qua hoàn toàn việc gọi lại hàm `build()` của Widget con đó**, giúp tiết kiệm CPU và giữ khung hình 60/120 FPS.

### Q1.4: Phân biệt sự khác nhau giữa `Expanded` và `Flexible` trong `Row` và `Column`?
- Cả hai đều dùng để chia tỷ lệ không gian màn hình theo tham số `flex` (tương đương `android:layout_weight`).
- **`Expanded`**: Ép buộc widget con **bắt buộc phải phình to lấp đầy 100%** không gian còn lại (`fit: FlexFit.tight`).
- **`Flexible`**: Cho phép widget con có kích thước **tối đa bằng** không gian còn lại, nhưng nếu nội dung của con nhỏ hơn, nó được phép co lại vừa vặn (`fit: FlexFit.loose`).

### Q1.5: Tại sao việc gọi hàm API trực tiếp trong tham số `future:` của `FutureBuilder` là một sai lầm nghiêm trọng (Anti-pattern)?
- Nếu viết `FutureBuilder(future: api.fetchData(), ...)`, mỗi khi hàm `build()` bị kích hoạt lại (do người dùng mở bàn phím ảo, xoay màn hình, hoặc widget cha rebuild), phương thức `api.fetchData()` sẽ **bị gọi lại từ đầu**.
- Hậu quả: App gửi hàng chục requests thừa lên server và giao diện bị chớp nháy (flickering) liên tục.
- **Cách khắc phục**: Phải khởi tạo và gán `Future` vào một biến trong phương thức `initState()`, sau đó chỉ truyền biến đó vào `FutureBuilder`.

---

## 🔷 CẤP ĐỘ 2: VÒNG PHỎNG VẤN MIDDLE (THỰC CHIẾN & KIẾN TRÚC ỨNG DỤNG)

### Q2.1: So sánh ưu nhược điểm của BLoC và Riverpod. Bạn sẽ chọn giải pháp nào cho dự án của mình?
- **BLoC**:
  - *Ưu điểm*: Cực kỳ chặt chẽ, kiến trúc hướng sự kiện (Event-driven). Dễ dàng kiểm soát concurrency (`droppable`, `restartable`), kiểm toán toàn bộ sự kiện qua `BlocObserver`. Chuẩn hóa rất tốt cho đội ngũ lớn (>15 devs).
  - *Nhược điểm*: Nhiều boilerplate code, độ dốc học tập ban đầu cao.
- **Riverpod**:
  - *Ưu điểm*: Declarative, độc lập hoàn toàn với `BuildContext` (đọc được trong service, isolate), an toàn 100% lúc biên dịch (Compile-time Safe), quản lý caching và tự hủy bộ nhớ cực mượt với `autoDispose` và `AsyncValue`.
  - *Nhược điểm*: Phụ thuộc vào code generation (`riverpod_generator`).
- *Lựa chọn thực tế*: Ưu tiên **BLoC** cho dự án tài chính, ngân hàng, enterprise có luồng sự kiện phức tạp. Ưu tiên **Riverpod** cho ứng dụng vừa và lớn cần tốc độ phát triển nhanh, type-safe và clean code.

### Q2.2: Cảnh báo "Don't use BuildContext across asynchronous gaps" nghĩa là gì và bạn xử lý nó ra sao?
- **Bản chất**: `BuildContext` thực chất là một đối tượng `Element` gắn liền với vị trí của Widget trên cây. Khi bạn gọi một lệnh bất đồng bộ (`await api.login()`), trong thời gian chờ đợi vài giây, người dùng có thể đã bấm nút Back để thoát khỏi màn hình. Lúc này, `Element` đã bị unmounted và tiêu hủy. Nếu bạn tiếp tục dùng `context` sau lệnh `await` (ví dụ: `Navigator.of(context).pop()`), ứng dụng sẽ ném ngoại lệ hoặc truy cập sai vùng nhớ.
- **Cách khắc phục**: Luôn kiểm tra `if (!mounted) return;` (nếu ở trong `State`) hoặc `if (!context.mounted) return;` (từ Flutter 3.7+) ngay sau mỗi khoảng trống bất đồng bộ trước khi sử dụng `context`.

### Q2.3: Trình bày cách bạn xử lý bài toán Refresh Token khi Access Token hết hạn trong thư viện Dio?
- Sử dụng **`QueuedInterceptor`** của Dio.
- Khi nhận mã lỗi `401 Unauthorized`, `QueuedInterceptor` sẽ **tạm khóa (freeze) toàn bộ các request tiếp theo**.
- Sử dụng một instance `Dio` độc lập gọi API `/auth/refresh-token` để lấy cặp token mới nhằm tránh đệ quy vô hạn.
- Lưu token mới vào `flutter_secure_storage`.
- Cập nhật Header của request ban đầu và gọi retry.
- Mở khóa hàng đợi để các request đang chờ tự động lấy token mới đi tiếp mà không làm người dùng bị văng ra màn hình đăng nhập.

### Q2.4: Sự khác nhau giữa `tester.pump()` và `tester.pumpAndSettle()` trong Widget Testing là gì?
- **`tester.pump()`**: Chỉ kích hoạt vẽ lại duy nhất **1 khung hình (1 frame)**. Dùng khi muốn kiểm tra trạng thái chuyển tiếp giữa chừng của một Animation.
- **`tester.pumpAndSettle()`**: Liên tục vẽ các khung hình lặp đi lặp lại cho đến khi **không còn bất kỳ animation hay microtask nào đang chạy** (cây giao diện hoàn toàn tĩnh).
- *Lưu ý*: Nếu màn hình có một `CircularProgressIndicator()` xoay vô tận, gọi `pumpAndSettle()` sẽ bị dính lỗi Timeout vĩnh viễn! Phải dùng `tester.pump(Duration)` để thay thế.

---

## 🔶 CẤP ĐỘ 3: VÒNG PHỎNG VẤN SENIOR (UNDER-THE-HOOD & SYSTEM DESIGN)

### Q3.1: Trình bày chi tiết cơ chế hoạt động của 3 cây: Widget Tree, Element Tree và RenderObject Tree. Thuật toán `canUpdate` vận hành như thế nào?
- **Widget Tree**: Bản thiết kế bất biến (Immutable Blueprint), cực nhẹ, sinh ra và chết đi liên tục sau mỗi khung hình.
- **Element Tree**: Bộ não điều phối sống lâu dài (Persistent), quản lý vòng đời và trạng thái (`State`).
- **RenderObject Tree**: Cấu trúc hình học nặng nề, trực tiếp tính toán Layout, Paint pixel và Hit Testing.
- **Thuật toán `canUpdate`**:
  ```dart
  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType && oldWidget.key == newWidget.key;
  }
  ```
  Nếu cùng `runtimeType` và `key`, Flutter sẽ **tái sử dụng nguyên vẹn Element và RenderObject cũ**, chỉ cập nhật thuộc tính bị thay đổi. Điều này giúp tránh việc cấp phát lại các đối tượng đồ họa đắt đỏ, đạt hiệu năng 60/120 FPS.

### Q3.2: Trình bày cơ chế hoạt động của Dart Event Loop và Generational Garbage Collection?
- **Event Loop**: Quản lý 2 hàng đợi: Microtask Queue (ưu tiên tuyệt đối) và Event Queue (I/O, Timer, Gestures). Event Loop luôn dọn sạch Microtask Queue trước khi lấy bất kỳ sự kiện nào từ Event Queue.
- **Generational GC**:
  - **New Generation (Scavenger)**: Sử dụng thuật toán Semi-Space Copying (Cheney's Algorithm). Cấp phát cực nhanh bằng Bump Pointer ($O(1)$). Chi phí dọn rác chỉ tỉ lệ thuận với số lượng object còn sống. Vì 99% widget chết ngay sau khi `build()`, Scavenger dọn rác trong $<1$ms mà hoàn toàn không bị phân mảnh bộ nhớ.
  - **Old Generation (Mark-Sweep-Compact)**: Dành cho các object sống lâu (Singletons, State dài hạn, Cache). Chạy Mark song song (Concurrent Mark) và Sweep/Compact khi bộ nhớ bị phân mảnh.

### Q3.3: Shader Compilation Jank trong Skia là gì và Impeller đã giải quyết triệt để vấn đề này như thế nào?
- **Vấn đề của Skia**: Dựa trên JIT (Just-In-Time) shader compilation. Lần đầu tiên một hiệu ứng phức tạp (blur, shadow, clip) xuất hiện lúc runtime, GPU driver phải dừng toàn bộ tiến trình render trong 100-200ms để biên dịch mã GLSL sang mã máy GPU $\rightarrow$ Gây đơ giật khung hình (Jank).
- **Giải pháp của Impeller**: Sử dụng **AOT (Ahead-Of-Time) Precompiled Shaders**. Toàn bộ các shader được công cụ `impellerc` biên dịch sẵn sang Metal (iOS) hoặc SPIR-V/Vulkan (Android) ngay tại thời điểm build ứng dụng. Lúc runtime không có bất kỳ shader nào cần biên dịch $\rightarrow$ Triệt tiêu 100% Shader Jank.

### Q3.4: Khi phân tích Heap Snapshot trên Flutter DevTools để bắt Memory Leak, sự khác nhau giữa `Shallow Size` và `Retained Size` là gì?
- **Shallow Size**: Dung lượng bộ nhớ được cấp phát chỉ để chứa bản thân đối tượng đó (các con trỏ, biến nguyên thủy). Thường rất nhỏ (vài chục bytes).
- **Retained Size**: **Tổng dung lượng bộ nhớ sẽ được giải phóng ngay lập tức** nếu đối tượng này bị Garbage Collector thu gom. Nó bao gồm Shallow Size của chính nó cộng với toàn bộ các đối tượng con mà chỉ một mình nó đang nắm giữ độc quyền.
- *Kinh nghiệm Senior*: Luôn sắp xếp theo cột **Retained Size giảm dần** để tìm ra đối tượng đang giam giữ hàng chục Megabytes RAM của hệ thống.
