# Bài 2.1: BLoC Event Transformers & Concurrency Policies

> **Cấp độ**: Senior Engineer  
> **Thời gian đọc**: ~30 phút  
> **Yêu cầu**: Đã biết BLoC/Cubit cơ bản; hiểu Stream và Dart async

---

## Phần 1 — Architecture & Problem Statement

### Bài toán: Search-as-you-type gây UI freeze

**Hệ thống thương mại điện tử, 500k DAU:**

```
Bug report: User gõ "smartphone" → app bị lag 3-5 giây
Root cause: Mỗi keystroke → 1 API call → 9 calls đồng thời  
            → Response về không theo thứ tự → UI hiện kết quả sai
            
Timeline thực tế:
t=0ms:   User gõ 's'    → Call 1 (search: 's')
t=100ms: User gõ 'sm'   → Call 2 (search: 'sm')
t=200ms: User gõ 'sma'  → Call 3 (search: 'sma')
...
t=800ms: User gõ 'smar' → Call 8 (search: 'smar')

Response về không theo thứ tự:
t=1200ms: Response 8 (smar)  → UI hiển thị "smar" results ✓
t=1350ms: Response 3 (sma)   → UI hiển thị "sma" results ✗ ← BUG!
t=1800ms: Response 1 (s)     → UI hiển thị "s" results ✗✗✗
```

**Root cause kỹ thuật**: BLoC dùng `sequential` transformer mặc định → tất cả event được xử lý đồng thời → race condition trên kết quả trả về.

---

## Phần 2 — Low-Level Mechanics

### 2.1. Kiến trúc EventTransformer

```dart
// EventTransformer là gì về mặt kỹ thuật?
typedef EventTransformer<E> = Stream<E> Function(
  Stream<E> events,           // Stream các event đến
  EventMapper<E> mapper,      // Function xử lý từng event
);

// Mỗi event handler trong BLoC có thể có transformer riêng:
on<SearchQueryChanged>(
  _onSearchQueryChanged,
  transformer: debounce(const Duration(milliseconds: 300)),
  //                ↑ Áp dụng cho chỉ event này, không ảnh hưởng event khác
);
```

### 2.2. 4 Concurrency Policies — Sơ đồ so sánh

```
Events đến: E1──E2──E3──E4──E5
(E1 chưa xử lý xong khi E2 đến)

SEQUENTIAL (default):
E1 ██████████│ E2 ██████████│ E3 ██████████│ E4 ██████████│ E5 ██████████
→ Xử lý tuần tự, E2 chờ E1 xong. Tốt cho: submit form, critical write.
→ VẤN ĐỀ với search: E1 block → tất cả chờ → UI lag.

DROPPABLE:
E1 ██████████│ E2(dropped) │ E3(dropped) │ E4(dropped) │ E5 ██████████
→ Khi E1 đang xử lý, E2,3,4 bị DROP, chỉ E5 được xử lý sau E1 xong.
→ Tốt cho: button tap (tránh double submit), load more (tránh duplicate page).

RESTARTABLE:
E1 ██│cancel│ E2 ██│cancel│ E3 ██│cancel│ E4 ██│cancel│ E5 ██████████
→ Khi E2 đến, HỦY E1 đang chạy, bắt đầu E2. Chỉ event mới nhất được xử lý.
→ ĐÚNG cho search-as-you-type: luôn show kết quả cho từ đang gõ.

CONCURRENT:
E1 ████████████████████│
E2     ████████████│
E3         █████████████████│
E4             ████████│
→ Tất cả xử lý song song. Không có cancel/drop.
→ Tốt cho: analytics logging, fire-and-forget, không cần ordering.
→ NGUY HIỂM nếu events phụ thuộc nhau → race condition.
```

### 2.3. Debounce vs Throttle — Khác biệt quan trọng

```
Input: keystroke mỗi 80ms
       k  k  k  k  k  k  k  [stop 400ms]  k  k
       │  │  │  │  │  │  │               │  │

DEBOUNCE (300ms):
       Chỉ fire sau khi im lặng 300ms:
       ........................................│fire│.........│fire│
       → Tốt cho: search, auto-save draft (tránh quá nhiều API call)

THROTTLE (300ms):
       Fire ngay lần đầu, rồi tối đa 1 lần mỗi 300ms:
       │fire│.......│fire│.......│fire│.............│fire│..........
       → Tốt cho: scroll event, mouse move, sensor data (cần real-time nhưng rate-limited)
```

---

## Phần 3 — Production Code Implementation

### 3.1. Custom EventTransformers

```dart
// lib/core/bloc/event_transformers.dart
import 'package:bloc/bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:rxdart/transformers.dart';

/// Debounce: Chờ im lặng [duration] trước khi xử lý
/// Dùng cho: Search-as-you-type, auto-save
EventTransformer<E> debounce<E>(Duration duration) {
  return (events, mapper) => events
      .debounceTime(duration)
      .switchMap(mapper); // switchMap = cancel previous khi event mới đến
}

/// Throttle: Xử lý ngay, rate-limit thành tối đa 1 lần mỗi [duration]
/// Dùng cho: Scroll events, sensor data
EventTransformer<E> throttle<E>(Duration duration) {
  return (events, mapper) => events
      .throttleTime(duration, trailing: false)
      .flatMap(mapper);
}

/// Debounce + Droppable: Chờ [duration] rồi xử lý, bỏ qua các event tiếp theo
/// cho đến khi request hiện tại hoàn thành
/// Dùng cho: Search với expensive API call
EventTransformer<E> debounceDroppable<E>(Duration duration) {
  return (events, mapper) => events
      .debounceTime(duration)
      .flatMap(mapper) // Flat, không cancel — chờ xong rồi nhận event tiếp
      ..take(1); // Chỉ lấy 1 result tại 1 thời điểm
}
```

### 3.2. SearchBloc Production-grade

```dart
// features/search/presentation/blocs/search_bloc.dart
import 'package:bloc/bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';

part 'search_bloc.freezed.dart';

// Events
@freezed
sealed class SearchEvent with _$SearchEvent {
  /// User gõ từ khóa — debounce + restartable
  const factory SearchEvent.queryChanged(String query) = SearchQueryChanged;
  
  /// User nhấn filter — sequential (filter thay đổi không cancel search)
  const factory SearchEvent.filterApplied(SearchFilter filter) = SearchFilterApplied;
  
  /// User scroll đến cuối — droppable (tránh load page duplicate)
  const factory SearchEvent.loadNextPage() = SearchLoadNextPage;
  
  /// User pull-to-refresh — restartable (reset về trang đầu)
  const factory SearchEvent.refreshed() = SearchRefreshed;
}

// States
@freezed
sealed class SearchState with _$SearchState {
  const factory SearchState.initial() = SearchInitial;
  const factory SearchState.loading() = SearchLoading;
  const factory SearchState.success({
    required List<Product> products,
    required bool hasNextPage,
    required int currentPage,
    required String query,
    required SearchFilter filter,
  }) = SearchSuccess;
  const factory SearchState.failure(Failure failure) = SearchFailure;
  const factory SearchState.loadingMore({
    required List<Product> currentProducts,
    required int currentPage,
  }) = SearchLoadingMore;
}

@injectable
final class SearchBloc extends Bloc<SearchEvent, SearchState> {
  SearchBloc({
    required SearchProductsUseCase searchUseCase,
    required AnalyticsService analytics,
  })  : _searchUseCase = searchUseCase,
        _analytics = analytics,
        super(const SearchState.initial()) {

    // Mỗi event type có transformer riêng biệt — QUAN TRỌNG
    on<SearchQueryChanged>(
      _onQueryChanged,
      transformer: debounce(const Duration(milliseconds: 350)),
      // restartable() cancel request cũ khi query mới đến
      // Kết hợp debounce + restartable qua debounce transformer dùng switchMap
    );

    on<SearchFilterApplied>(
      _onFilterApplied,
      transformer: sequential(), // Đợi search hiện tại xong mới apply filter
    );

    on<SearchLoadNextPage>(
      _onLoadNextPage,
      transformer: droppable(), // Bỏ qua nếu đang load — tránh duplicate page
    );

    on<SearchRefreshed>(
      _onRefreshed,
      transformer: restartable(), // Cancel request cũ, reset từ đầu
    );
  }

  final SearchProductsUseCase _searchUseCase;
  final AnalyticsService _analytics;
  
  // Giữ filter state riêng — không phụ thuộc vào SearchState
  SearchFilter _currentFilter = SearchFilter.empty();
  int _currentPage = 1;

  Future<void> _onQueryChanged(
    SearchQueryChanged event,
    Emitter<SearchState> emit,
  ) async {
    final query = event.query.trim();
    
    // Không search nếu query rỗng hoặc quá ngắn
    if (query.isEmpty) {
      emit(const SearchState.initial());
      return;
    }
    
    if (query.length < 2) return; // Minimum 2 ký tự
    
    emit(const SearchState.loading());
    _currentPage = 1; // Reset pagination khi query thay đổi
    
    await emit.forEach(
      // emit.forEach subscribe Stream — tự động cancel khi bloc đóng
      _searchUseCase.executeStream(SearchParams(
        query: query,
        filter: _currentFilter,
        page: _currentPage,
      )),
      onData: (result) => switch (result) {
        Success(:final value) => SearchState.success(
            products: value.items,
            hasNextPage: value.hasNextPage,
            currentPage: _currentPage,
            query: query,
            filter: _currentFilter,
          ),
        Failure_(:final failure) => SearchState.failure(failure),
      },
      onError: (error, stackTrace) {
        addError(error, stackTrace); // Gửi error đến BlocObserver
        return SearchState.failure(
          ServerFailure(message: error.toString()),
        );
      },
    );
    
    // Log analytics sau khi search xong (fire-and-forget, không await)
    unawaited(_analytics.logSearch(query: query));
  }

  Future<void> _onLoadNextPage(
    SearchLoadNextPage event,
    Emitter<SearchState> emit,
  ) async {
    // Chỉ load more khi đang ở Success state và còn trang tiếp
    final currentState = state;
    if (currentState is! SearchSuccess || !currentState.hasNextPage) return;
    
    emit(SearchState.loadingMore(
      currentProducts: currentState.products,
      currentPage: currentState.currentPage,
    ));
    
    _currentPage++;
    
    final result = await _searchUseCase.execute(SearchParams(
      query: currentState.query,
      filter: _currentFilter,
      page: _currentPage,
    ));
    
    switch (result) {
      case Success(:final value):
        emit(SearchState.success(
          // Append thêm vào list hiện tại — không replace
          products: [...currentState.products, ...value.items],
          hasNextPage: value.hasNextPage,
          currentPage: _currentPage,
          query: currentState.query,
          filter: _currentFilter,
        ));
      case Failure_(:final failure):
        _currentPage--; // Rollback page counter khi fail
        emit(SearchState.failure(failure));
    }
  }
  
  Future<void> _onFilterApplied(
    SearchFilterApplied event,
    Emitter<SearchState> emit,
  ) async {
    _currentFilter = event.filter;
    _currentPage = 1;
    
    final currentQuery = switch (state) {
      SearchSuccess(:final query) => query,
      _ => '',
    };
    
    if (currentQuery.isEmpty) return;
    
    // Trigger lại search với filter mới
    add(SearchEvent.queryChanged(currentQuery));
  }
  
  Future<void> _onRefreshed(
    SearchRefreshed event,
    Emitter<SearchState> emit,
  ) async {
    final currentQuery = switch (state) {
      SearchSuccess(:final query) => query,
      _ => '',
    };
    
    if (currentQuery.isEmpty) return;
    _currentPage = 1;
    add(SearchEvent.queryChanged(currentQuery));
  }
}
```

### 3.3. BlocObserver — Monitor toàn bộ BLoC trong app

```dart
// lib/core/bloc/app_bloc_observer.dart
final class AppBlocObserver extends BlocObserver {
  const AppBlocObserver({required AnalyticsService analytics})
      : _analytics = analytics;
  final AnalyticsService _analytics;

  @override
  void onEvent(Bloc<dynamic, dynamic> bloc, Object? event) {
    super.onEvent(bloc, event);
    debugPrint('[${bloc.runtimeType}] Event: ${event.runtimeType}');
  }

  @override
  void onTransition(
    Bloc<dynamic, dynamic> bloc,
    Transition<dynamic, dynamic> transition,
  ) {
    super.onTransition(bloc, transition);
    // Log transition chỉ trong debug mode
    assert(() {
      debugPrint(
        '[${bloc.runtimeType}] '
        '${transition.currentState.runtimeType} → '
        '${transition.nextState.runtimeType}',
      );
      return true;
    }());
  }

  @override
  void onError(BlocBase<dynamic> bloc, Object error, StackTrace stackTrace) {
    super.onError(bloc, error, stackTrace);
    // Gửi lên Crashlytics trong production
    unawaited(_analytics.logError(
      error: error,
      stackTrace: stackTrace,
      context: bloc.runtimeType.toString(),
    ));
  }
}

// main.dart
Bloc.observer = AppBlocObserver(analytics: getIt<AnalyticsService>());
```

---

## Phần 4 — Profiling & Performance Trade-offs

### Benchmark: Sequential vs Restartable vs Droppable với search 500ms latency

```
Test: User gõ 5 ký tự (mỗi 100ms), API latency 500ms

SEQUENTIAL:
- Tổng thời gian: 5 * 500ms = 2500ms (tuần tự)
- API calls: 5 (tất cả chạy)
- Kết quả hiển thị: Sai (hiện kết quả cũ sau cùng)
- Memory (peak): 5 concurrent responses in buffer

RESTARTABLE (debounce 300ms):
- Tổng thời gian: 300ms wait + 500ms API = 800ms
- API calls: 1 (chỉ query cuối cùng)
- Kết quả hiển thị: Đúng (luôn là query mới nhất)
- Memory: 1 response in buffer
- Cost reduction: 80% fewer API calls → giảm server cost

DROPPABLE (cho load-next-page):
- Tổng thời gian: 500ms (1 call)
- API calls: 1 (event thứ 2,3,4,5 bị drop)
- Kết quả: Không duplicate page load
- Memory: 1 response in buffer
```

| Scenario | Transformer | API calls | Kết quả đúng | UX |
|:---|:---:|:---:|:---:|:---|
| Search typing | `restartable + debounce` | 1 | ✅ | Nhanh, đúng |
| Submit form | `sequential` hoặc `droppable` | 1 | ✅ | Không double-submit |
| Load more | `droppable` | 1 | ✅ | Không duplicate |
| Analytics log | `concurrent` | N | ✅ | Fire-and-forget |
| Filter change | `sequential` | 1 | ✅ | Đợi search cũ xong |

---

## Phần 5 — Production Checklist

```
[ ] ❌ Dùng sequential (default) cho search/filter events
    ✅ restartable hoặc debounce(restartable) cho search
    Lý do: Sequential block UI khi latency cao, kết quả stale

[ ] ❌ Không debounce keystroke event
    ✅ Debounce 250-350ms cho text input
    Lý do: 1 user gõ "iphone 14" = 9 API calls thay vì 1

[ ] ❌ Dùng concurrent cho event có shared mutable state
    ✅ concurrent chỉ cho fire-and-forget, analytics, non-critical
    Lý do: Race condition khi 2 event cùng write cùng state

[ ] ❌ Event handler có try-catch nuốt lỗi im lặng
    ✅ Dùng addError() để gửi đến BlocObserver
    Lý do: Lỗi im lặng không xuất hiện trong Crashlytics

[ ] ❌ BLoC subscribe stream mà không dùng emit.forEach
    ✅ Luôn dùng emit.forEach hoặc emit.onEach cho stream
    Lý do: emit.forEach tự động cancel subscription khi BLoC closed

[ ] ❌ BLoC giữ reference đến BuildContext
    ✅ BLoC không biết về Flutter Widget/Context — chỉ emit states
    Lý do: Memory leak và crash khi context bị dispose

[ ] ❌ 1 BLoC xử lý quá nhiều concern (search + filter + cart + analytics)
    ✅ 1 BLoC = 1 domain concern; delegate sang UseCase để orchestrate
    Lý do: Khó test, khó maintain, event transformer conflict

[ ] ❌ Không có BlocObserver trong production
    ✅ Implement AppBlocObserver với Crashlytics integration
    Lý do: Không có visibility khi BLoC throw error ở production
```
