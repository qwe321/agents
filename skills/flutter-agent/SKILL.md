---
name: flutter-agent
description: >
  Use this skill for all Flutter/Dart development tasks. Enforces project-specific
  architecture: ChangeNotifier state management, hand-written models (no codegen),
  relative imports only, abstract service interfaces with impl/ boundary, centralized
  widget keys, sealed-class EventBus, and GoRouter with auth via refreshListenable.
  Operates in two modes: Feature Mode (default — screens, state, models, tests) and
  Service Mode (explicit — implement service internals). Load REFERENCE.md alongside
  this file when implementing a pattern for the first time.
references:
  - REFERENCE.md
---

# Flutter Agent Rules

Every task begins with a mode declaration. Follow only the rules permitted by your current mode. For full code examples of any rule, see `REFERENCE.md`.

## Task Modes

### Feature Mode (default)
Build screens, state classes, models, tests. Read abstract service interfaces but never open `impl/`.

**You may edit:** `features/**`, `core/models/**`, `core/keys/app_keys.dart`, `core/widgets/**`, `core/services/*.dart` (abstract classes only — add method signatures with `throw UnimplementedError`), `core/services/event_bus.dart` (add event subclasses), `routing/app_router.dart`, `test/**`, `integration_test/**`

**You may NOT edit:** `core/services/impl/**`

### Service Mode
Implement or modify service internals. You are given one specific service to work on.

**You may edit:** `core/services/impl/<specified_service>_impl.dart` only, `core/services/<specified_service>.dart` (abstract class — adjust signatures if needed)

**You may NOT edit:** `features/**`, `routing/**`, any other `impl/` file

**Service Mode rules:**
1. Read the full abstract interface first. Implement every `throw UnimplementedError` method.
2. Follow patterns already in the impl file. Match error handling, naming, structure.
3. Catch all library exceptions (`DioException`, `PlatformException`, etc.) → rethrow as `AppException` subtypes. No library types escape `impl/`.
4. Return only domain model types. Convert JSON to models inside impl.
5. Run existing unit tests for the service. Do not modify feature tests.

### Mode triggers
- No declaration → Feature Mode.
- `SERVICE MODE: <ServiceName>` → Service Mode for that service.

---

## Rule 1: No Code Generation

No `freezed`, `json_serializable`, `injectable`, `auto_route`, `build_runner`. Ever. Write `fromJson`/`toJson` by hand. Write routes by hand.

## Rule 2: Strict Explicit Typing

No `dynamic`. Annotate function return types, parameter types, class fields, and any variable where the type isn't obvious from the RHS. Inference is OK only when type is unambiguous from context (e.g. `final posts = <Post>[];`). The test: if you'd have to open another file to know the type, annotate explicitly.

`Map<String, dynamic>` from JSON must be converted to a typed model immediately — it never flows past a service boundary.

Enable in `analysis_options.yaml`: `strict-casts: true`, `strict-raw-types: true`, `non_exhaustive_switch_statement: error`.

### Null safety
- No `!` operator — restructure with null check + promotion instead. If `user` could be null, do `if (user == null) return;` then use the promoted variable.
- `required` in constructors by default. Nullable fields only when the domain genuinely requires it.

## Rule 3: Imports — Relative Only

Relative imports for all own code. `package:` only for external packages. Mixing causes `type 'X' is not a subtype of type 'X'` bugs. Zero exceptions.

```dart
import 'package:flutter/material.dart';       // external — OK
import '../../core/models/user.dart';          // own code — relative ONLY
```

## Rule 4: State Management — ChangeNotifier Only

No Riverpod, BLoC, Cubit, GetX, Redux, MobX. Use `ChangeNotifier` + `ListenableBuilder`. Use `get_it` for service location (without `injectable`).

## Rule 5: Mutable State, No copyWith

State classes use mutable fields + `notifyListeners()`. No immutable state objects, no `copyWith`. Models (`core/models/`) are also mutable plain classes — no `const` constructors, no `@immutable`.

Only exception: event classes (`sealed class AppEvent` subclasses) are immutable — fire-and-forget.

## Rule 6: File Structure

```
lib/
├── app.dart
├── main.dart
├── core/
│   ├── keys/
│   │   └── app_keys.dart        # ALL widget keys live here
│   ├── services/
│   │   ├── api_client.dart      # Abstract interface
│   │   ├── auth_service.dart    # Abstract interface
│   │   ├── storage_service.dart # Abstract interface
│   │   ├── event_bus.dart       # Concrete (simple enough)
│   │   └── impl/               # AGENT DOES NOT MODIFY (Feature Mode)
│   ├── models/
│   ├── utils/
│   └── widgets/                 # Shared widgets only
├── features/
│   ├── auth/
│   │   ├── auth_state.dart
│   │   ├── login_screen.dart
│   │   └── widgets/
│   ├── home/
│   └── settings/
└── routing/
    └── app_router.dart          # GoRouter, single file
```

One `*_state.dart` per feature, colocated. One screen per file. Feature-local widgets in feature's `widgets/`. No cross-feature state imports — use core services or EventBus.

## Rule 7: Widget Keys

Every interactive widget MUST have a key from the centralized `K` registry. Never hardcode key strings.

```dart
// core/keys/app_keys.dart
abstract class K {
  static const emailField = Key('auth_email_field');
  static const loginButton = Key('auth_login_button');
}
```

Workflow: add key to `app_keys.dart` FIRST → use `key: K.loginButton` in widget → use `$(K.loginButton)` in tests.
Naming: `{feature}_{element}_{type}`. List items use `ValueKey(item.id)`.

## Rule 8: Global Events — EventBus with Sealed Classes

Broadcast `StreamController`-based EventBus for background→UI communication. Events are a `sealed class AppEvent` hierarchy with typed fields only — no `Map<String, dynamic>`.

Key constraints:
- `SessionExpiredEvent` does NOT exist. Auth state changes go through the router's `refreshListenable` (Rule 9).
- No `default` / `_` case in switch on sealed types — handle every subclass explicitly for exhaustiveness.
- Use switch **expressions** (not statements) on sealed types as primary exhaustiveness safeguard.
- Screens opt in via `EventListenerWrapper` with `suppressedEvents` set. Place a root-level `EventListenerWrapper` above `MaterialApp` for catch-all handling.

## Rule 9: Routing — GoRouter with Auth via refreshListenable

`GoRouter` in a single file. No code generation. Auth redirects via `refreshListenable` bound to `AuthService` (which extends `ChangeNotifier`), NOT via EventBus.

When background token refresh fails, `AuthService` impl sets `isAuthenticated = false` + `notifyListeners()`. Router picks it up automatically. No event bus, no UI-layer navigation for auth redirects.

## Rule 10: Testing

**Unit tests:** Test every state class as plain Dart with mock services via constructor injection. Every model with `fromJson`/`toJson` MUST have a round-trip test covering ALL fields with non-default values. Nullable fields get two tests (with value, with null). Place in `test/unit/models/`.

**Integration tests:** Patrol with `$` syntax. Test user flows, not individual screens.

**Hard rules:**
- NEVER `await Future.delayed()` in tests — use Patrol's `waitUntilVisible`/`waitUntilExists`.
- NEVER hardcode key strings — always `K.*`.
- All tests start from clean state via `test_setup.dart` with mock service registrations.

## Rule 11: Dependency Injection

`get_it` as flat service locator. No `injectable`. Register everything in one `setupDependencies()` function in `main.dart` — the only file that imports `impl/`.

## Rule 12: Banned Patterns

| Banned | Why |
|---|---|
| `dynamic` type | Compile errors → runtime errors |
| `var` (use `final`) | Mutable locals invite reassignment bugs |
| `copyWith` methods | Nullable field ambiguity, boilerplate drift |
| `@immutable` on state/models | Mutable state is the standard here |
| `!` without null check + promotion in same scope | Runtime crash — restructure instead |
| `default`/`_` in switch on sealed types | Defeats exhaustiveness checking |
| `package:my_app/` for own code | Causes subtype errors when mixed with relative |
| `part` / `part of` | File desync |
| Mixins for shared logic | Use helper classes/functions |
| `extends` for state reuse | Use composition |
| `BuildContext` in state classes or any non-Widget/State method | Context belongs to UI layer exclusively |
| `StreamController` in state classes | Exception: EventBus only |
| Barrel files (`export`) | Circular imports |
| `AnimationController` without `dispose()` | Use implicit animations instead |
| `late` without `final` | Reassignment bugs |

### The `mounted` / `context` boundary

`mounted` and `BuildContext` belong EXCLUSIVELY in `State<T>`. ChangeNotifier state classes never see `context`, never check `mounted`, never navigate or show UI. Pattern: state does async work → updates fields → `notifyListeners()` → `ListenableBuilder` rebuilds → widget reads fields and renders. See `REFERENCE.md` for the full correct/forbidden examples.

## Rule 13: Deep Modules — Abstract Interface + `impl/` Boundary

Services expose simple abstract interfaces returning domain types. Implementations hide all complexity in `impl/`.

**Adding a new method (Feature Mode):**
1. Add to abstract class with `=> throw UnimplementedError('ClassName.methodName not yet implemented')`.
2. Add `// TODO: implement in impl/` comment.
3. Write feature code calling it. Write tests with mock. Everything compiles.
4. Do NOT open `impl/` files.

**Rules:** Feature code imports only abstract classes. One impl per interface. Domain return types only. Errors as `AppException` subtypes — state classes catch `NetworkException`, `UnauthorizedException`, `AppException`, never `DioException`/`SocketException`/`PlatformException`.

## Rule 14: Workflow — Cross-Cutting Files First

Before writing any feature code, review and update if needed:
1. `core/keys/app_keys.dart` — if adding interactive widgets
2. `routing/app_router.dart` — if adding/changing screens
3. `core/services/event_bus.dart` — if adding global notifications
4. `core/services/*.dart` (abstract classes) — if new data methods needed

Read each file before editing. Do NOT start screen/state code until cross-cutting updates are done.

## Rule 15: Self-Evaluation (Mandatory)

After completing code, output a `SELF-CHECK` block. Every item MUST include a one-line evidence citation. A `[PASS]` without evidence is a `[FAIL]`. If any `[FAIL]`, fix immediately and re-output.

```
SELF-CHECK:
- [PASS] No dynamic types → home_state.dart: all fields typed (List<Post>, bool, String?)
- [FAIL] Missing dispose → home_screen.dart:42 _sub never cancelled → fixing now
```

**Service Mode self-check format:**
```
SELF-CHECK (Service Mode):
- [PASS] All UnimplementedError methods implemented → api_client_impl.dart: getComments(), getComment()
- [PASS] No library types in return signatures → all return domain models
- [PASS] All DioExceptions caught → _makeRequest wraps all calls
- [PASS] Follows existing patterns → getComments() mirrors getPosts()
```

**Checklist items:**
1. No `.g.dart` or `.freezed.dart` imports
2. No `dynamic` — ambiguous types annotated, `Map<String, dynamic>` stays inside `impl/`
3. No `!` without null check + promotion in same scope
4. All own-code imports are relative
5. `analysis_options.yaml` strict mode passes
6. Every new interactive widget has key in `app_keys.dart`
7. State class extends `ChangeNotifier`, nothing else
8. Screen calls `_state.dispose()` in its `dispose()`
9. No `context`/`mounted` in ChangeNotifier — only in `State<T>` callbacks
10. StreamSubscriptions cancelled in `dispose()`
11. New routes added to `app_router.dart`
12. New services registered in `setupDependencies()`
13. Feature code imports abstract interfaces, never `impl/`
14. `Map<String, dynamic>` never crosses service boundary
15. Unit test exists for new state class
16. Round-trip serialization test for every new/modified model
17. Integration test updated if user flow changed
