# Bài 2.1: BLoC Event Transformers & Concurrency Policies

> **Cấp độ**: Senior / Staff Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Nắm vững BLoC/Cubit cơ bản, Streams, Reactive Programming (Rx) và Dart Event Loop.

---

## Dẫn Chiếu Tài Liệu Chính Thức
- **bloc_concurrency Package Documentation**: [pub.dev/packages/bloc_concurrency](https://pub.dev/packages/bloc_concurrency)
- **BLoC Library Architecture Guide — Event Transformers**: [bloclibrary.dev/bloc-concepts/#event-transformers](https://bloclibrary.dev/bloc-concepts/#event-transformers)
- **ReactiveX Operators (ConcatMap, SwitchMap, ExhaustMap)**: [reactivex.io/documentation/operators.html](https://reactivex.io/documentation/operators.html)
- **Dart Streams & Asynchronous Programming**: [dart.dev/tutorials/language/streams](https://dart.dev/tutorials/language/streams)

---

## Phần 1 — Khái Niệm & Bài Toán Kiến Trúc (Nó Là Gì & Giải Quyết Bài Toán Gì?)

### 1.1 — Nó Là Gì? EventTransformer & Chính Sách Concurrency Trong BLoC
Trong kiến trúc BLoC (Business Logic Component), một `EventTransformer` là một hàm bậc cao (Higher-order Function) can thiệp vào luồng `Stream<Event>` đầu vào trước khi sự kiện được chuyển giao cho bộ xử lý `EventHandler`.

Mặc định trong package `bloc`, các sự kiện gửi vào được xử lý theo cơ chế **`concurrent()`** (xử lý đồng thời không khóa) hoặc **`sequential()`** (xử lý tuần tự theo hàng đợi FIFO). Tuy nhiên, các bài toán thực tế trên ứng dụng di động đòi hỏi các chiến lược kiểm soát luồng bất đồng bộ phức tạp hơn nhằm giải quyết tải dồn dập (Backpressure).

Package `bloc_concurrency` cung cấp 4 chính sách đồng thời cốt lõi:
1. **`sequential()`**: Xử lý tuần tự từng sự kiện một theo thứ tự xuất hiện trong hàng đợi FIFO. Sự kiện kế tiếp chỉ bắt đầu khi sự kiện trước hoàn tất.
2. **`droppable()`**: Bỏ qua (drop) hoàn toàn các sự kiện mới đến nếu sự kiện hiện tại đang trong quá trình thực thi.
3. **`restartable()`**: Hủy bỏ (cancel) tác vụ đang chạy để ưu tiên xử lý ngay sự kiện mới nhất vừa xuất hiện.
4. **`concurrent()`**: Xử lý song song tất cả các sự kiện cùng lúc mà không có bất kỳ ràng buộc thứ tự hay hủy bỏ nào.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      KIẾN TRÚC EVENT TRANSFORMER                        │
│                                                                         │
│   UI Events Stream:  E1 ───► E2 ───► E3 ───► E4                         │
│                               │                                         │
│                               ▼                                         │
│   EventTransformer:  [ Debounce / Throttle / Concurrency Policy ]       │
│                               │                                         │
│                               ▼                                         │
│   Transformed Stream:        E1' ──────────► E4'                        │
│                               │                                         │
│                               ▼                                         │
│   Event Handler:     on<Event>(_handler) ──► emit(NewState)             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 1.2 — Giải Quyết Bài Toán Gì? Thách Thức Khi Xử Lý Dữ Liệu Bất Đồng Bộ Dồn Dập
Khi ứng dụng phục vụ số lượng người dùng lớn (High DAU), việc thiếu kiểm soát Concurrency Policy sẽ gây ra 3 lỗi nghiêm trọng:

1. **Sự cố Search-as-you-type và Race Condition kết quả mạng**:
   - Khi người dùng gõ từ khóa `"smartphone"`, 10 sự kiện gõ phím được phát xạ liên tục trong 800ms.
   - Nếu xử lý đồng thời, 10 request mạng được gửi đi cùng lúc. Do độ trễ mạng biến thiên (Jitter), request thứ 3 (tìm `"sma"`) có thể phản hồi sau request thứ 10 (tìm `"smartphone"`).
   - Giao diện người dùng sẽ hiển thị kết quả của từ khóa `"sma"` sau cùng, gây lỗi hiển thị sai lệch dữ liệu trầm trọng (Stale Response Bug).
2. **Sự cố nhấn nút kép (Double-Tap / Spam Click)**:
   - Tại màn hình thanh toán, người dùng nhấn liên tục 2 lần vào nút "Xác Nhận Đặt Hàng" trong khoảng 200ms do mạng lag. Nếu dùng chính sách mặc định, hệ thống gửi 2 request trừ tiền song song, gây trùng lặp giao dịch (Duplicate Charge).
3. **Hiện tượng nghẽn hàng đợi (Queue Head-of-Line Blocking)**:
   - Nếu áp dụng `sequential()` cho sự kiện tìm kiếm, người dùng gõ 10 ký tự sẽ buộc hệ thống phải chờ lần lượt 10 API calls hoàn tất. Mỗi call mất 500ms khiến ứng dụng bị đóng băng (Lag) tới 5 giây sau khi người dùng đã ngừng gõ.

---

### 1.3 — Bảng So Sánh 4 Chính Sách Concurrency

| Chính Sách | Cơ Chế Xử Lý | Toán Tử Rx Tương Ứng | Kịch Bản Sử Dụng Chuẩn |
| :--- | :--- | :--- | :--- |
| **`sequential()`** | Hàng đợi FIFO; chờ sự kiện trước xong mới chạy sự kiện sau. | `concatMap` | Đồng bộ dữ liệu offline, các tác vụ ghi CSDL tuần tự. |
| **`droppable()`** | Bỏ qua sự kiện mới nếu đang bận xử lý sự kiện cũ. | `exhaustMap` | Nhấn nút Submit form, nút Mua hàng, tải thêm trang (Pagination). |
| **`restartable()`** | Hủy tác vụ cũ đang chạy, lập tức xử lý sự kiện mới. | `switchMap` | Tìm kiếm (Search-as-you-type), lọc danh mục, tab chuyển đổi. |
| **`concurrent()`** | Chạy song song độc lập, không chờ đợi, không hủy. | `flatMap` | Ghi log phân tích (Analytics), gửi telemetry, tác vụ độc lập. |

---

### 1.4 — Mục Tiêu Kỹ Thuật Cần Đạt Được
- Nắm vững kiến trúc nội tại của `EventTransformer<E>` và các toán tử Stream cơ sở trong Dart.
- Phân biệt bản chất cơ chế giữa **Debounce** (chờ im lặng) và **Throttle** (giới hạn tần suất).
- Xây dựng lớp `SearchBloc` chuẩn Enterprise tích hợp đa transformers trên từng loại sự kiện riêng biệt.
- Sử dụng `emit.forEach` để quản lý vòng đời Stream an toàn, triệt tiêu rò rỉ bộ nhớ.
- Thiết lập hệ sinh thái giám sát tập trung với `BlocObserver`.

---

## Phần 2 — Bản Chất Là Gì? (Under the Hood & Cơ Chế Hoạt Động)

### 2.1 — Cấu Trúc Typedef Của `EventTransformer<E>`
Trong thư viện `bloc`, `EventTransformer` được định nghĩa như sau:

```dart
typedef EventMapper<Event> = Stream<Transition<Event, State>> Function(Event event);

typedef EventTransformer<Event> = Stream<Event> Function(
  Stream<Event> events,
  EventMapper<Event> mapper,
);
```

- `events`: Luồng Stream chứa toàn bộ các sự kiện thô phát ra từ giao diện.
- `mapper`: Hàm closure ánh xạ từng sự kiện thành một Stream các chuyển đổi trạng thái (Transitions).
- *Bản chất*: Transformer chính là một toán tử biến đổi Stream. Thay vì trực tiếp đưa từng event vào handler, BLoC chuyển quyền điều phối luồng cho transformer để quyết định khi nào và làm thế nào để thực thi `mapper`.

---

### 2.2 — Giải Phẫu Cơ Chế Concurrency Policies Dưới Góc Nhìn Stream

```text
Chuỗi sự kiện đầu vào:  ──E1────E2──E3────────E4─────────►

1. SEQUENTIAL (concatMap):
   E1: [===== Xử lý =====]
   E2:                     [===== Xử lý =====]
   E3:                                         [===== Xử lý =====]
   E4:                                                             [===== Xử lý =====]

2. DROPPABLE (exhaustMap):
   E1: [===== Xử lý =====]
   E2: (BỊ DROP BỎ QUA)
   E3: (BỊ DROP BỎ QUA)
   E4:                     [===== Xử lý =====]

3. RESTARTABLE (switchMap):
   E1: [=Hủy=]
   E2:         [=Hủy=]
   E3:                 [===== Xử lý =====]
   E4:                                         [===== Xử lý =====]

4. CONCURRENT (flatMap):
   E1: [===== Xử lý =====]
   E2:     [===== Xử lý =====]
   E3:       [===== Xử lý =====]
   E4:                 [===== Xử lý =====]
```

---

### 2.3 — Debounce So Với Throttle: Phân Biệt Cơ Chế Kiểm Soát Tần Suất

```text
Luồng phím bấm:  ──k──k──k──k──────[Nghỉ 400ms]──────k──k──────►

DEBOUNCE (Thời gian chờ 300ms):
Chỉ kích hoạt khi luồng sự kiện ngừng phát xạ (im lặng) đủ 300ms:
                 ...............................│FIRE│..........

THROTTLE (Thời gian giới hạn 300ms):
Kích hoạt ngay tại sự kiện đầu tiên, sau đó khóa luồng trong 300ms tiếp theo:
                 │FIRE│.........................│FIRE│..........
```

- **Debounce**: Tối ưu tuyệt đối cho việc tiết kiệm băng thông và tài nguyên CPU. Thích hợp cho ô nhập tìm kiếm (chờ người dùng gõ xong từ mới gọi API) hoặc tính năng tự động lưu bản nháp (Auto-save draft).
- **Throttle**: Đảm bảo phản hồi tức thì lần đầu và duy trì tốc độ cập nhật ổn định theo chu kỳ. Thích hợp cho sự kiện cuộn trang (Scroll Listener), cử chỉ kéo thả (Drag Gestures), hoặc cập nhật tọa độ GPS.

---

## Phần 3 — Triển Khai Kỹ Thuật (Triển Khai Như Nào? Step-by-Step Implementation)

Xây dựng module Tìm Kiếm Sản Phẩm Chuẩn Enterprise (`SearchBloc`) tích hợp đa chính sách concurrency.

### 3.1 — Bước 1: Xây Dựng Thư Viện Custom EventTransformers

```dart
// lib/core/bloc/event_transformers.dart

import 'package:bloc/bloc.dart';
import 'package:rxdart/rxdart.dart';

/// Debounce kết hợp Restartable (switchMap):
/// Chờ im lặng [duration], sau đó hủy bỏ request trước đó nếu có query mới đến
EventTransformer<E> debounceRestartable<E>(Duration duration) {
  return (events, mapper) => events
      .debounceTime(duration)
      .switchMap(mapper);
}

/// Throttle: Kích hoạt tức thì, giới hạn tần suất tối đa 1 lần mỗi [duration]
EventTransformer<E> throttleDroppable<E>(Duration duration) {
  return (events, mapper) => events
      .throttleTime(duration, leading: true, trailing: false)
      .exhaustMap(mapper);
}
```

---

### 3.2 — Bước 2: Khai Báo Events & States Đa Dạng

```dart
// lib/features/search/presentation/bloc/search_event.dart

import 'package:flutter/foundation.dart';

@immutable
sealed class SearchEvent {
  const SearchEvent();
}

final class SearchQueryChanged extends SearchEvent {
  final String query;
  const SearchQueryChanged(this.query);
}

final class SearchNextPageRequested extends SearchEvent {
  const SearchNextPageRequested();
}

final class SearchFilterApplied extends SearchEvent {
  final String category;
  const SearchFilterApplied(this.category);
}
```

```dart
// lib/features/search/presentation/bloc/search_state.dart

import 'package:flutter/foundation.dart';

@immutable
sealed class SearchState {
  const SearchState();
}

final class SearchInitialState extends SearchState {
  const SearchInitialState();
}

final class SearchLoadingState extends SearchState {
  const SearchLoadingState();
}

final class SearchSuccessState extends SearchState {
  final List<String> items;
  final bool hasNextPage;
  final int pageIndex;
  final String query;

  const SearchSuccessState({
    required this.items,
    required this.hasNextPage,
    required this.pageIndex,
    required this.query,
  });
}

final class SearchFailureState extends SearchState {
  final String errorMessage;
  const SearchFailureState(this.errorMessage);
}
```

---

### 3.3 — Bước 3: Triển Khai `SearchBloc` Với Concurrency Policies Chuyên Biệt

```dart
// lib/features/search/presentation/bloc/search_bloc.dart

import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';
import '../../../../core/bloc/event_transformers.dart';
import 'search_event.dart';
import 'search_state.dart';

abstract interface class SearchRepository {
  Future<List<String>> searchProducts(String query, int page);
}

class SearchBloc extends Bloc<SearchEvent, SearchState> {
  final SearchRepository _repository;
  int _currentPage = 1;

  SearchBloc(this._repository) : super(const SearchInitialState()) {
    // 1. Tìm kiếm: Debounce 300ms + Restartable (Hủy request cũ khi có từ khóa mới)
    on<SearchQueryChanged>(
      _onQueryChanged,
      transformer: debounceRestartable(const Duration(milliseconds: 300)),
    );

    // 2. Tải thêm trang: Droppable (Bỏ qua thao tác cuộn nếu trang trước đang tải)
    on<SearchNextPageRequested>(
      _onNextPageRequested,
      transformer: droppable(),
    );

    // 3. Áp dụng bộ lọc: Sequential (Chờ request hiện tại xong mới áp dụng bộ lọc mới)
    on<SearchFilterApplied>(
      _onFilterApplied,
      transformer: sequential(),
    );
  }

  Future<void> _onQueryChanged(
    SearchQueryChanged event,
    Emitter<SearchState> emit,
  ) async {
    final query = event.query.trim();
    if (query.isEmpty) {
      emit(const SearchInitialState());
      return;
    }

    emit(const SearchLoadingState());
    _currentPage = 1;

    try {
      final results = await _repository.searchProducts(query, _currentPage);
      emit(SearchSuccessState(
        items: results,
        hasNextPage: results.length >= 20,
        pageIndex: _currentPage,
        query: query,
      ));
    } catch (e) {
      emit(SearchFailureState(e.toString()));
    }
  }

  Future<void> _onNextPageRequested(
    SearchNextPageRequested event,
    Emitter<SearchState> emit,
  ) async {
    final currentState = state;
    if (currentState is! SearchSuccessState || !currentState.hasNextPage) return;

    try {
      _currentPage++;
      final nextItems = await _repository.searchProducts(
        currentState.query,
        _currentPage,
      );

      emit(SearchSuccessState(
        items: [...currentState.items, ...nextItems],
        hasNextPage: nextItems.length >= 20,
        pageIndex: _currentPage,
        query: currentState.query,
      ));
    } catch (e) {
      _currentPage--; // Hoàn tác số trang nếu request thất bại
      emit(SearchFailureState(e.toString()));
    }
  }

  Future<void> _onFilterApplied(
    SearchFilterApplied event,
    Emitter<SearchState> emit,
  ) async {
    final currentState = state;
    if (currentState is! SearchSuccessState) return;

    emit(const SearchLoadingState());
    _currentPage = 1;

    try {
      final filteredResults = await _repository.searchProducts(
        '${currentState.query}&category=${event.category}',
        _currentPage,
      );
      emit(SearchSuccessState(
        items: filteredResults,
        hasNextPage: filteredResults.length >= 20,
        pageIndex: _currentPage,
        query: currentState.query,
      ));
    } catch (e) {
      emit(SearchFailureState(e.toString()));
    }
  }
}
```

---

### 3.4 — Bước 4: Triển Khai `CheckoutBloc` Với Chính Sách `droppable()`

Để loại trừ triệt để nguy cơ người dùng nhấn đúp (Double-tap) hoặc spam nút "Xác nhận thanh toán" làm gửi nhiều request trừ tiền song song:

```dart
// lib/features/checkout/presentation/bloc/checkout_bloc.dart

import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';

sealed class CheckoutEvent {
  const CheckoutEvent();
}

final class CheckoutSubmitRequested extends CheckoutEvent {
  final String orderId;
  final int amountCents;
  const CheckoutSubmitRequested({required this.orderId, required this.amountCents});
}

sealed class CheckoutState {
  const CheckoutState();
}

final class CheckoutIdleState extends CheckoutState {
  const CheckoutIdleState();
}

final class CheckoutProcessingState extends CheckoutState {
  const CheckoutProcessingState();
}

final class CheckoutSuccessState extends CheckoutState {
  final String transactionId;
  const CheckoutSuccessState(this.transactionId);
}

final class CheckoutFailureState extends CheckoutState {
  final String error;
  const CheckoutFailureState(this.error);
}

abstract interface class PaymentRepository {
  Future<String> charge({required String orderId, required int amountCents});
}

final class CheckoutBloc extends Bloc<CheckoutEvent, CheckoutState> {
  final PaymentRepository _paymentRepository;

  CheckoutBloc(this._paymentRepository) : super(const CheckoutIdleState()) {
    // droppable(): Nếu đang có giao dịch xử lý dở dang, BỎ QUA mọi sự kiện click tiếp theo
    on<CheckoutSubmitRequested>(
      _onSubmitRequested,
      transformer: droppable(),
    );
  }

  Future<void> _onSubmitRequested(
    CheckoutSubmitRequested event,
    Emitter<CheckoutState> emit,
  ) async {
    emit(const CheckoutProcessingState());
    try {
      final transactionId = await _paymentRepository.charge(
        orderId: event.orderId,
        amountCents: event.amountCents,
      );
      emit(CheckoutSuccessState(transactionId));
    } catch (e) {
      emit(CheckoutFailureState(e.toString()));
    }
  }
}
```

---

### 3.5 — Bước 5: Triển Khai `OfflineSyncBloc` Với Chính Sách `sequential()`

Trong kịch bản ứng dụng hoạt động ngoại tuyến, người dùng thực hiện liên tiếp các thao tác: Thêm sản phẩm $\to$ Cập nhật số lượng $\to$ Xóa sản phẩm. Nếu các sự kiện này được gửi lên server song song (`concurrent`), thao tác "Xóa" có thể đến server trước thao tác "Thêm", gây xung đột dữ liệu nghiêm trọng. Chính sách `sequential()` đảm bảo xử lý nghiêm ngặt theo hàng đợi FIFO:

```dart
// lib/features/sync/presentation/bloc/sync_bloc.dart

import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';

sealed class SyncEvent {
  const SyncEvent();
}

final class SyncMutationQueued extends SyncEvent {
  final String entityId;
  final String mutationType; // 'CREATE', 'UPDATE', 'DELETE'
  final Map<String, dynamic> payload;

  const SyncMutationQueued({
    required this.entityId,
    required this.mutationType,
    required this.payload,
  });
}

sealed class SyncState {
  const SyncState();
}

final class SyncIdleState extends SyncState {
  const SyncIdleState();
}

final class SyncInProgressState extends SyncState {
  final int remainingTasks;
  const SyncInProgressState(this.remainingTasks);
}

final class SyncCompletedState extends SyncState {
  const SyncCompletedState();
}

abstract interface class SyncRemoteGateway {
  Future<void> pushMutation(String type, Map<String, dynamic> payload);
}

final class OfflineSyncBloc extends Bloc<SyncEvent, SyncState> {
  final SyncRemoteGateway _gateway;
  int _pendingCount = 0;

  OfflineSyncBloc(this._gateway) : super(const SyncIdleState()) {
    // sequential(): Cưỡng chế thực thi tuần tự từng mutation theo thứ tự FIFO
    on<SyncMutationQueued>(
      _onMutationQueued,
      transformer: sequential(),
    );
  }

  Future<void> _onMutationQueued(
    SyncMutationQueued event,
    Emitter<SyncState> emit,
  ) async {
    _pendingCount++;
    emit(SyncInProgressState(_pendingCount));

    try {
      await _gateway.pushMutation(event.mutationType, event.payload);
    } finally {
      _pendingCount--;
      if (_pendingCount == 0) {
        emit(const SyncCompletedState());
      } else {
        emit(SyncInProgressState(_pendingCount));
      }
    }
  }
}
```

---

### 3.6 — Bước 6: Kiểm Thử Hành Vi Concurrency Với `bloc_test`

Để kiểm chứng tính đúng đắn của Concurrency Policy tại compile-time và CI/CD, ta sử dụng package `bloc_test` với các chuỗi phát xạ sự kiện tốc độ cao:

```dart
// test/features/search/presentation/bloc/search_bloc_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:bloc_test/bloc_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:app/features/search/presentation/bloc/search_bloc.dart';
import 'package:app/features/search/presentation/bloc/search_event.dart';
import 'package:app/features/search/presentation/bloc/search_state.dart';

class MockSearchRepository extends Mock implements SearchRepository {}

void main() {
  group('SearchBloc Concurrency Policy Test', () {
    late MockSearchRepository repository;

    setUp(() {
      repository = MockSearchRepository();
    });

    blocTest<SearchBloc, SearchState>(
      'Chính sách debounceRestartable phải hủy bỏ request trung gian và chỉ thực thi từ khóa cuối cùng',
      build: () {
        when(() => repository.searchProducts('flutter', 1))
            .thenAnswer((_) async => ['Flutter Architecture', 'Flutter Testing']);
        return SearchBloc(repository);
      },
      act: (bloc) async {
        // Phát 3 sự kiện dồn dập trong khoảng thời gian < 300ms debounce
        bloc.add(const SearchQueryChanged('f'));
        await Future.delayed(const Duration(milliseconds: 50));
        bloc.add(const SearchQueryChanged('flut'));
        await Future.delayed(const Duration(milliseconds: 50));
        bloc.add(const SearchQueryChanged('flutter'));
        // Chờ vượt quá ngưỡng debounce (300ms) để request cuối cùng được kích hoạt
        await Future.delayed(const Duration(milliseconds: 350));
      },
      expect: () => [
        const SearchLoadingState(),
        const SearchSuccessState(
          items: ['Flutter Architecture', 'Flutter Testing'],
          hasNextPage: false,
          pageIndex: 1,
          query: 'flutter',
        ),
      ],
      verify: (_) {
        // Xác minh chỉ có DUY NHẤT 1 cuộc gọi mạng được thực hiện cho từ khóa cuối
        verify(() => repository.searchProducts('flutter', 1)).called(1);
        verifyNever(() => repository.searchProducts('f', 1));
        verifyNever(() => repository.searchProducts('flut', 1));
      },
    );
  });
}
```

---

### 3.7 — Bước 7: Thiết Lập Hệ Sinh Thái Giám Sát Tập Trung Với `AppBlocObserver`

```dart
// lib/core/bloc/app_bloc_observer.dart

import 'package:flutter/foundation.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

final class AppBlocObserver extends BlocObserver {
  const AppBlocObserver();

  @override
  void onEvent(Bloc<dynamic, dynamic> bloc, Object? event) {
    super.onEvent(bloc, event);
    debugPrint('[BLoC Event] ${bloc.runtimeType} -> ${event.runtimeType}');
  }

  @override
  void onTransition(
    Bloc<dynamic, dynamic> bloc,
    Transition<dynamic, dynamic> transition,
  ) {
    super.onTransition(bloc, transition);
    assert(() {
      debugPrint(
        '[BLoC Transition] ${bloc.runtimeType}: '
        '${transition.currentState.runtimeType} -> ${transition.nextState.runtimeType}',
      );
      return true;
    }());
  }

  @override
  void onError(BlocBase<dynamic> bloc, Object error, StackTrace stackTrace) {
    super.onError(bloc, error, stackTrace);
    debugPrint('[BLoC ERROR] ${bloc.runtimeType}: $error');
    // Gửi lỗi lên Crashlytics hoặc Sentry tại môi trường Production
  }
}
```

---

## Phần 4 — Best Practices & Phòng Chống Cạm Bẫy (Defensive Engineering)

### 4.1 — ❌ Anti-pattern 1: Sử Dụng `restartable()` Cho Sự Kiện Thanh Toán Hoặc Giao Dịch

#### Mô tả lỗi:
Gắn `restartable()` vào sự kiện bấm nút thanh toán tiền:

```dart
// ❌ LỖI NGUY HIỂM: Gán restartable cho thao tác trừ tiền
on<SubmitPaymentEvent>(_onPayment, transformer: restartable());
```

#### Phân tích cơ chế gây lỗi:
Khi người dùng bấm nút 2 lần liên tiếp, sự kiện thứ nhất vừa gửi gói tin HTTP đi thì bị `restartable()` hủy bỏ luồng Stream phía client. Client ngắt kết nối và gửi request thứ hai. Tuy nhiên, backend đã kịp nhận request thứ nhất và tiến hành trừ tiền. Kết quả là tài khoản bị trừ tiền 2 lần và client chỉ ghi nhận 1 giao dịch.

#### Biện pháp phòng chống:
Các tác vụ thanh toán hoặc ghi dữ liệu quan trọng **bắt buộc phải sử dụng `droppable()`** hoặc **`sequential()`** kết hợp với Idempotency Key.

---

### 4.2 — ❌ Anti-pattern 2: Tự Đăng Ký `stream.listen()` Thủ Công Trong BLoC

#### Mô tả lỗi:
Lắng nghe một Stream bên ngoài bằng `_repository.watchData().listen(...)` mà không quản lý biến hủy `StreamSubscription`:

```dart
// ❌ LỖI: Rò rỉ StreamSubscription
SearchBloc(...) {
  _repository.dataStream.listen((data) {
    add(DataReceivedEvent(data)); // Rò rỉ bộ nhớ khi BLoC bị close()!
  });
}
```

#### Biện pháp phòng chống:
Luôn luôn sử dụng phương thức tích hợp sẵn **`emit.forEach`** hoặc **`emit.onEach`** của BLoC. Cơ chế này tự động quản lý vòng đời hủy StreamSubscription ngay khi BLoC được giải phóng (`close()`).

---

## Phần 5 — Khảo Sát Bản Chất Kỹ Thuật & Thử Thách Thẩm Định

### 5.1 — Khảo Sát Bản Chất Kỹ Thuật

#### Câu hỏi 1: Về mặt kiến trúc Dart Event Loop, sự khác biệt giữa `asyncExpand` (dùng trong sequential) và `switchMap` (dùng trong restartable) là gì?
*Phân tích kỹ thuật:*
- **`asyncExpand` (Sequential)**: Khi một phần tử mới đến trên Stream, `asyncExpand` chuyển nó vào hàng đợi FIFO và chờ đợi `Stream` con được trả về bởi mapper hoàn tất phát xạ (Emit) xong toàn bộ dữ liệu mới tiếp tục xử lý phần tử kế tiếp trong Event Loop.
- **`switchMap` (Restartable)**: Khi một phần tử mới xuất hiện, `switchMap` ngay lập tức gọi phương thức hủy `cancel()` trên `StreamSubscription` của Stream con đang chạy dở dang, giải phóng tài nguyên CPU/Network của tác vụ trước đó và đăng ký lắng nghe ngay Stream con mới.

---

#### Câu hỏi 2: Tại sao `droppable()` lại là giải pháp tối ưu nhất cho tính năng Infinite Scrolling Pagination?
*Phân tích kỹ thuật:*
Khi người dùng cuộn nhanh về đáy danh sách, sự kiện `ScrollNotification` có thể kích hoạt nhiều lần trong vài chục miligiây. Nếu dùng `sequential()`, hệ thống sẽ tải dồn dập trang 2, trang 3, trang 4 cùng lúc. Nếu dùng `concurrent()`, các trang phản hồi lộn xộn làm sai thứ tự danh sách. `droppable()` bỏ qua hoàn toàn các tín hiệu cuộn thừa thãi trong lúc trang hiện tại đang tải, chỉ tiếp nhận yêu cầu tải tiếp theo sau khi trang cũ đã được nối vào danh sách thành công.

---

### 5.2 — Bài Tập Thực Hành: Kiểm Thử Concurrency Policy Bằng Virtual Time

**Yêu cầu**:
1. Sử dụng package `bloc_test` để viết Unit Test cho `SearchBloc`.
2. Kiểm thử kịch bản: Gửi 3 sự kiện `SearchQueryChanged` với độ trễ giữa các lần là 100ms (nhỏ hơn khoảng thời gian debounce 300ms).
3. Xác minh rằng: Repository chỉ nhận duy nhất 1 cuộc gọi mạng với từ khóa của sự kiện cuối cùng, và BLoC phát xạ chính xác chuỗi trạng thái: `[SearchInitialState, SearchLoadingState, SearchSuccessState]`.
