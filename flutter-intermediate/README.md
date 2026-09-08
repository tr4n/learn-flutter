# Tập 1.5: Flutter Intermediate — Bridge to Enterprise

> **Vị trí trong lộ trình học:**
> `flutter-fundamentals-v2/` → **`flutter-intermediate/`** (bạn đang ở đây) → `flutter-advanced/`

---

## Mục đích của Volume này

Sau khi hoàn thành Fundamentals V2, bạn đã nắm vững:
- Dart 3, Null Safety, Event Loop, Streams
- Widget–Element–RenderObject tree, BuildContext, Keys
- Layout engine, Navigation cơ bản, Provider cơ bản
- Networking với Dio, Animations, Forms

Tuy nhiên, trước khi bước vào **Advanced** (Enterprise Architecture, BLoC Event Transformers, Riverpod AsyncNotifier, Platform Interop, Testing Pyramid...), bạn cần thêm nền tảng về:

| Gap | Module bổ sung |
|:----|:---------------|
| Material 3 Theming — tất cả code Advanced dùng M3 | Module 01 |
| BLoC/Cubit — Advanced nhảy thẳng vào Event Transformers | Module 02 |
| Riverpod — Advanced nhảy vào AsyncNotifier & Codegen | Module 03 |
| Clean Architecture — nền tảng cho toàn bộ Volume 2 | Module 04 |
| Testing cơ bản — Advanced giả định bạn biết viết test | Module 05 |
| GoRouter thực hành — Advanced đi vào StatefulShellRoute, deep links | Module 06 |

---

## Bốn Trụ Cột của Volume 1.5

1. **Design System** — Hiểu Material 3 để build app đúng chuẩn Google
2. **Reactive State** — BLoC và Riverpod: hai hướng state management phổ biến nhất production
3. **Architecture** — Clean Architecture: tách biệt concern, testable, scalable
4. **Quality** — Testing pyramid: unit, widget, integration test

---

## Bảng Mục Lục

| # | Module | Chapters | Trọng tâm |
|:---:|:---|:---:|:---|
| **01** | Material 3 Theming & Design System | 3 | ThemeData, ColorScheme, Typography, dark mode |
| **02** | BLoC Pattern & Cubit | 4 | Cubit, BLoC, Events/States, BlocBuilder, BlocListener |
| **03** | Riverpod Modern | 3 | ProviderScope, ConsumerWidget, AsyncNotifier, autoDispose |
| **04** | Clean Architecture | 4 | Layers, UseCase, Entity vs DTO, DI với get_it |
| **05** | Testing Fundamentals | 4 | Unit test, mocking, widget test, integration test |
| **06** | GoRouter Practical | 3 | GoRoute, path params, redirect, ShellRoute |

**Tổng: 6 module / 21 chapter**

---

## 5-Part Article Blueprint

Mỗi chapter tuân theo cấu trúc:

```markdown
## Phần 1 — Khái Niệm & Mục Tiêu Bài Học
> "Why it matters" — tại sao cần học concept này.

## Phần 2 — Cơ Chế Hoạt Động (Under the Hood)
> Diagram. Phân tích luồng. Trả lời "Tại sao?".

## Phần 3 — Code Mẫu Chuẩn Google
> Dart 3+, Sound Null Safety, flutter_lints. Material 3.

## Phần 4 — Lỗi Sai Phổ Biến & Best Practices
> ❌ Anti-pattern → ✅ Đúng + giải thích kỹ thuật.

## Phần 5 — Bài Tập Củng Cố Tư Duy
> 1 challenge cụ thể + câu hỏi thẩm định chuyên sâu liên quan.
```

---

## Module 01 — Material 3 Theming & Design System (3 chapters)

| Chapter | File | Nội dung |
|---------|------|----------|
| 1.1 | `01-themedata-colorscheme-m3.md` | ThemeData, ColorScheme.fromSeed, M3 color roles |
| 1.2 | `02-typography-texttheme-custom-fonts.md` | TextTheme, M3 type scale, Google Fonts |
| 1.3 | `03-dark-mode-dynamic-theming.md` | Dark/light mode, system theme, runtime switching |

## Module 02 — BLoC Pattern & Cubit (4 chapters)

| Chapter | File | Nội dung |
|---------|------|----------|
| 2.1 | `01-cubit-state-management.md` | Cubit, emit(), CubitBuilder, states pattern |
| 2.2 | `02-bloc-events-states.md` | BLoC vs Cubit, Events, sealed state classes |
| 2.3 | `03-bloc-builder-listener-consumer.md` | BlocBuilder, BlocListener, BlocConsumer, context.read/watch |
| 2.4 | `04-bloc-repository-pattern.md` | BLoC + Repository, error handling, loading states |

## Module 03 — Riverpod Modern (3 chapters)

| Chapter | File | Nội dung |
|---------|------|----------|
| 3.1 | `01-provider-scope-consumer-widget.md` | ProviderScope, ConsumerWidget, ref.watch/read/listen |
| 3.2 | `02-notifier-async-notifier.md` | Notifier, AsyncNotifier, AsyncValue (data/loading/error) |
| 3.3 | `03-provider-modifiers-autodispose-family.md` | autoDispose, family, keepAlive, provider composition |

## Module 04 — Clean Architecture (4 chapters)

| Chapter | File | Nội dung |
|---------|------|----------|
| 4.1 | `01-layers-domain-data-presentation.md` | 3 layers, dependency rule, folder structure |
| 4.2 | `02-entity-dto-repository-interface.md` | Entity vs DTO vs Model, Repository interface |
| 4.3 | `03-usecase-pattern.md` | UseCase class, orchestration, single-responsibility |
| 4.4 | `04-dependency-injection-get-it.md` | get_it, Singleton/Factory/LazySingleton, service locator |

## Module 05 — Testing Fundamentals (4 chapters)

| Chapter | File | Nội dung |
|---------|------|----------|
| 5.1 | `01-unit-testing-dart.md` | test(), expect(), group(), setUp/tearDown |
| 5.2 | `02-mocking-mocktail.md` | Mock, Stub, Fake, mocktail, verify() |
| 5.3 | `03-widget-testing.md` | testWidgets(), pumpWidget(), find, tester actions |
| 5.4 | `04-bloc-riverpod-testing.md` | BlocTest, ProviderContainer, async testing |

## Module 06 — GoRouter Practical (3 chapters)

| Chapter | File | Nội dung |
|---------|------|----------|
| 6.1 | `01-gorouter-routes-navigation.md` | GoRouter setup, GoRoute, go/push/replace, path params |
| 6.2 | `02-shell-route-bottom-nav.md` | ShellRoute, bottom nav persistence, nested navigation |
| 6.3 | `03-redirect-auth-guard.md` | redirect, auth guard, listenable redirect, deep link basics |

---

## Tiêu Chuẩn Chất Lượng

- **Độ dài**: 250–350 dòng/chapter
- **Code blocks**: tối thiểu 3 Dart block/chapter, production-ready
- **Diagrams**: ít nhất 1 Mermaid diagram/chapter
- **Anti-patterns**: ít nhất 2 ❌/✅ pairs/chapter
- **Ngôn ngữ**: Tiếng Việt kỹ thuật, thuật ngữ Anh giữ nguyên
