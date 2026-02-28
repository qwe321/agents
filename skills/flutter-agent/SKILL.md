# Flutter Agent Rules

You already know Flutter best practices for humans. This document overrides those where they cause agent bugs. Follow these rules strictly.

## Rule 1: No Code Generation

No `freezed`, `json_serializable`, `injectable`, `auto_route`, `build_runner`. Ever.
Write `fromJson`/`toJson` by hand. Write routes by hand. The 2 minutes saved is not worth the `.g.dart` staleness bugs you will create.

```dart
// CORRECT — hand-written model
class User {
  final String id;
  final String name;
  final DateTime createdAt;

  User({required this.id, required this.name, required this.createdAt});

  factory User.fromJson(Map<String, dynamic> json) => User(
    id: json['id'] as String,
    name: json['name'] as String,
    createdAt: DateTime.parse(json['created_at'] as String),
  );

  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'created_at': createdAt.toIso8601String(),
  };
}
```

## Rule 2: Strict Explicit Typing

Never use `dynamic`. Never rely on type inference for anything except literals.

Annotate every: variable declaration, function return type, function parameter, and generic type parameter.
Local variable type inference is allowed only when the type is explicitly visible on the right-hand side.

```dart
// FORBIDDEN — implicit dynamic
var data = json['key'];
final items = response.data;
final mapped = list.map((e) => e.name);

// CORRECT
final String name = json['key'] as String;
final List<Post> items = response.data;
final Iterable<String> mapped = list.map((Post e) => e.name);

// ALLOWED:
final foo = SomeType();
```

`Map<String, dynamic>` from JSON must be converted to a typed model immediately. It never flows past a service boundary.

```dart
// FORBIDDEN — dynamic leaking into feature code
Future<Map<String, dynamic>> getUser() async { ... }

// CORRECT — typed at the boundary
Future<User> getUser() async {
  final Map<String, dynamic> json = await _get('/user');
  return User.fromJson(json);
}
```

Non-null assertion operator (!) is allowed only after explicit null check in same scope.

Enable and enforce in `analysis_options.yaml`:
```yaml
analyzer:
  strict-casts: true
  strict-raw-types: true
  errors:
    inference_failure_on_untyped_parameter: error
    inference_failure_on_function_return_type: error
    inference_failure_on_collection_literal: error
```

## Rule 3: Imports — Relative Only

Use relative imports everywhere. Never use `package:` imports for your own code.

```dart
// FORBIDDEN
import 'package:my_app/core/models/user.dart';

// CORRECT
import '../../core/models/user.dart';
```

Mixing styles causes `type 'X' is not a subtype of type 'X'` errors that are impossible to diagnose. This rule has zero exceptions.

`package:` imports are only for external packages:
```dart
import 'package:flutter/material.dart';       // external — package: OK
import 'package:get_it/get_it.dart';          // external — package: OK
import '../../core/models/user.dart';          // own code — relative ONLY
```

## Rule 4: State Management — ChangeNotifier Only

Do NOT use Riverpod, BLoC, Cubit, GetX, Redux, or MobX.
Use `ChangeNotifier` + `ListenableBuilder`. Use `get_it` for service location (without `injectable`).

```dart
class HomeState extends ChangeNotifier {
  final ApiClient _api;
  HomeState(this._api);

  List<Post> posts = [];
  bool isLoading = false;
  String? error;

  Future<void> loadPosts() async {
    isLoading = true;
    error = null;
    notifyListeners();
    try {
      posts = await _api.getPosts();
    } catch (e) {
      error = e.toString();
    } finally {
      isLoading = false;
      notifyListeners();
    }
  }
}
```

```dart
class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});
  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  late final HomeState _state;

  @override
  void initState() {
    super.initState();
    _state = HomeState(getIt<ApiClient>());
    _state.loadPosts();
  }

  @override
  void dispose() {
    _state.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: _state,
      builder: (context, _) {
        if (_state.isLoading) return const Center(child: CircularProgressIndicator());
        if (_state.error != null) return Center(child: Text(_state.error!));
        return ListView.builder(
          itemCount: _state.posts.length,
          itemBuilder: (context, i) => PostTile(post: _state.posts[i]),
        );
      },
    );
  }
}
```

## Rule 5: File Structure

```
lib/
├── app.dart
├── main.dart
├── core/
│   ├── keys/
│   │   └── app_keys.dart        # ALL widget keys live here
│   ├── services/
│   │   ├── api_client.dart      # Abstract interface — agent reads this
│   │   ├── auth_service.dart    # Abstract interface
│   │   ├── storage_service.dart # Abstract interface
│   │   ├── event_bus.dart       # Concrete (simple enough)
│   │   └── impl/               # AGENT DOES NOT MODIFY
│   │       ├── api_client_impl.dart
│   │       ├── auth_service_impl.dart
│   │       └── storage_service_impl.dart
│   ├── models/
│   │   ├── app_exception.dart   # Domain exceptions
│   │   └── user.dart
│   ├── utils/
│   └── widgets/                 # Shared widgets only
├── features/
│   ├── auth/
│   │   ├── auth_state.dart
│   │   ├── login_screen.dart
│   │   └── widgets/
│   ├── home/
│   │   ├── home_state.dart
│   │   ├── home_screen.dart
│   │   └── widgets/
│   └── settings/
└── routing/
    └── app_router.dart          # GoRouter, single file
```

Rules:
- One `*_state.dart` per feature, colocated in feature folder.
- One screen per file. Do NOT split into controller/view/binding.
- Feature-local widgets go in feature's `widgets/` subfolder.
- No cross-feature state imports. Cross-feature communication goes through core services or EventBus.

## Rule 6: Widget Keys

Every interactive widget MUST have a key from the centralized registry. Never hardcode key strings in widgets or tests.

```dart
// core/keys/app_keys.dart
abstract class K {
  // Auth
  static const emailField = Key('auth_email_field');
  static const passwordField = Key('auth_password_field');
  static const loginButton = Key('auth_login_button');

  // Home
  static const postsList = Key('home_posts_list');
  static const refreshButton = Key('home_refresh_button');
}
```

**When adding a new interactive widget:**
1. Add key to `app_keys.dart` FIRST.
2. Reference as `key: K.loginButton` in widget.
3. Reference as `$(K.loginButton)` in tests.

Naming: `{feature}_{element}_{type}`. List items use `ValueKey(item.id)` instead.

## Rule 7: Global Events and Notifications

Use a broadcast `StreamController`-based EventBus for background→UI communication. Do NOT model this as shared state.

Events are a sealed class hierarchy. Each event type carries only its own typed fields — no `Map<String, dynamic>`.

```dart
// core/services/event_bus.dart
sealed class AppEvent {}

class SyncErrorEvent extends AppEvent {
  final String message;
  SyncErrorEvent(this.message);
}

class NewMessageEvent extends AppEvent {
  final String chatId;
  final String senderName;
  final String preview;
  NewMessageEvent({required this.chatId, required this.senderName, required this.preview});
}

class AchievementUnlockedEvent extends AppEvent {
  final String achievementId;
  final String title;
  AchievementUnlockedEvent({required this.achievementId, required this.title});
}
```

`SessionExpiredEvent` does not exist. Auth state changes are handled by the router via `refreshListenable` (Rule 7). Do NOT route from the event bus.

```dart
class EventBus {
  final StreamController<AppEvent> _controller = StreamController<AppEvent>.broadcast();
  Stream<AppEvent> get stream => _controller.stream;
  void fire(AppEvent event) => _controller.add(event);
  void dispose() => _controller.close();
}
```

Adding a new event type: add a new subclass to this file. The compiler enforces exhaustive handling via `switch`. Do not include a default case in the event switch. Let the compiler force exhaustiveness.

Screens opt in via `EventListenerWrapper`. Screens suppress event types they handle internally:

```dart
// core/widgets/event_listener_wrapper.dart
class EventListenerWrapper extends StatefulWidget {
  final Widget child;
  final Set<Type> suppressedEvents;
  const EventListenerWrapper({
    required this.child,
    this.suppressedEvents = const {},
    super.key,
  });
  @override
  State<EventListenerWrapper> createState() => _EventListenerWrapperState();
}

class _EventListenerWrapperState extends State<EventListenerWrapper> {
  late final StreamSubscription<AppEvent> _sub;

  @override
  void initState() {
    super.initState();
    _sub = getIt<EventBus>().stream.listen(_onEvent);
  }

  void _onEvent(AppEvent event) {
    if (suppressedEvents.contains(event.runtimeType)) return;
    if (!mounted) return;

    switch (event) {
      case SyncErrorEvent(:final message):
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(message)),
        );
      case NewMessageEvent(:final senderName, :final preview):
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('$senderName: $preview')),
        );
      case AchievementUnlockedEvent(:final title):
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Unlocked: $title')),
        );
    }
  }

  @override
  void dispose() {
    _sub.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => widget.child;
}
```

Usage in a screen:

```dart
@override
Widget build(BuildContext context) {
  return EventListenerWrapper(
    suppressedEvents: {NewMessageEvent}, // chat handles these inline
    child: Scaffold(...),
  );
}
```

Place a root-level `EventListenerWrapper` with no suppressions above `MaterialApp` for catch-all snackbars/dialogs.

## Rule 8: Routing

Use `GoRouter`. Define all routes in a single file. No code generation. No deep nesting of route guards.

Auth redirects are handled by the router via `refreshListenable`, NOT by the `EventListenerWrapper`. This ensures redirects work even when no screen is mounted.

```dart
// core/services/auth_service.dart
abstract class AuthService extends ChangeNotifier {
  bool get isAuthenticated;
  User? get currentUser;
  Future<void> login({required String email, required String password});
  Future<void> logout();
  // Implementation calls notifyListeners() on auth state changes.
  // This includes: successful login, logout, and session expiry
  // detected by token refresh failure.
}
```

```dart
// routing/app_router.dart
final GoRouter appRouter = GoRouter(
  initialLocation: '/home',
  refreshListenable: getIt<AuthService>(), // re-evaluates redirect on auth changes
  redirect: (BuildContext context, GoRouterState state) {
    final bool authed = getIt<AuthService>().isAuthenticated;
    final bool onAuthRoute = state.matchedLocation.startsWith('/auth');

    if (!authed && !onAuthRoute) return '/auth/login';
    if (authed && onAuthRoute) return '/home';
    return null;
  },
  routes: <RouteBase>[
    GoRoute(path: '/auth/login', builder: (_, __) => const LoginScreen()),
    GoRoute(path: '/auth/signup', builder: (_, __) => const SignupScreen()),
    GoRoute(path: '/home', builder: (_, __) => const HomeScreen()),
    GoRoute(path: '/settings', builder: (_, __) => const SettingsScreen()),
  ],
);
```

When a background token refresh fails, the `AuthService` implementation sets `isAuthenticated = false` and calls `notifyListeners()`. The router picks this up automatically and redirects to login. No event bus involvement, no UI-layer navigation calls.

**Auth redirects NEVER go through `EventListenerWrapper`.** The `EventBus` is for UI notifications (snackbars, popups). Routing is the router's job.

## Rule 9: Testing

Use **Patrol** for integration tests. Use standard `flutter_test` for unit tests.

### Unit tests
Test every state class as plain Dart. Mock services via constructor injection.

**Every model with `fromJson`/`toJson` MUST have a round-trip test.** This catches key name typos and serialization bugs that hand-written JSON code is prone to.

```dart
test('User round-trip serialization', () {
  final User original = User(
    id: 'abc-123',
    name: 'Test User',
    createdAt: DateTime.utc(2025, 1, 15),
  );

  final Map<String, dynamic> json = original.toJson();
  final User restored = User.fromJson(json);

  expect(restored.id, original.id);
  expect(restored.name, original.name);
  expect(restored.createdAt, original.createdAt);
});
```

Rules for model tests:
- Test EVERY field, not just a subset.
- Use non-default values for every field so defaults don't mask bugs.
- For nullable fields, write two tests: one with a value, one with null.
- Place in `test/unit/models/` mirroring `core/models/`.

```dart
test('loadPosts sets posts on success', () async {
  final mockApi = MockApiClient();
  when(() => mockApi.getPosts()).thenAnswer((_) async => [testPost]);

  final state = HomeState(mockApi);
  await state.loadPosts();

  expect(state.posts, [testPost]);
  expect(state.isLoading, false);
  expect(state.error, null);
});
```

### Integration tests
Use Patrol `$` syntax. Test user flows, not individual screens.

```dart
patrolTest('login and view posts', ($) async {
  app.main();
  await $.pumpAndSettle();

  await $(K.emailField).enterText('test@example.com');
  await $(K.passwordField).enterText('password123');
  await $(K.loginButton).tap();

  await $(K.postsList).waitUntilVisible();
});
```

### Test rules
- NEVER use `await Future.delayed()` in tests. Use Patrol's `waitUntilVisible`/`waitUntilExists`.
- NEVER hardcode key strings in tests. Always use `K.*`.
- Each test file covers one user flow.
- All tests start from a clean state via `test_setup.dart` with mock service registrations in `get_it`.
- Prefer keys over text. Use text only for verifying user-visible content.

## Rule 10: Dependency Injection

Use `get_it` as a flat service locator. No `injectable`. Register everything in one setup function.

```dart
// main.dart
void setupDependencies() {
  getIt.registerLazySingleton<EventBus>(() => EventBus());
  getIt.registerLazySingleton<ApiClient>(() => ApiClient());
  getIt.registerLazySingleton<AuthService>(() => AuthService(getIt<ApiClient>()));
}

void main() {
  setupDependencies();
  runApp(const MyApp());
}
```

## Rule 11: Things You Must NOT Do

| Banned | Reason |
|---|---|
| `dynamic` type | Turns compile errors into runtime errors |
| `var` without type annotation | Implicit typing leads to silent `dynamic` |
| `package:my_app/` imports for own code | Mixing with relative imports causes 'X is not a subtype of X' |
| `part` / `part of` directives | You will desync files |
| Mixins for shared logic | Use helper classes/functions instead |
| `extends` for state reuse | Use composition |
| `BuildContext` in state classes | Pass dependencies via constructor |
| Passing `BuildContext` to any method outside a `Widget` or `State` | Context belongs to the UI layer exclusively |
| `StreamController` in state classes (use ChangeNotifier) | Exception: EventBus only |
| Barrel files (`export`) | You will create circular imports |
| Implicit animations without explicit controllers | You forget to dispose them |
| `late` without `final` | You will reassign it by mistake |

### The `mounted` / `context` boundary

`mounted` checks and `BuildContext` usage belong EXCLUSIVELY in `State<T>` classes (the UI layer). State classes (`ChangeNotifier`) never see `context`, never check `mounted`, and never perform navigation or show UI.

The pattern is always: state class does async work and updates fields → calls `notifyListeners()` → `ListenableBuilder` rebuilds → the widget tree reads fields and decides what to render.

```dart
// FORBIDDEN — passing context into state
class BadState extends ChangeNotifier {
  Future<void> submit(BuildContext context) async {
    await _api.submit(data);
    if (context.mounted) GoRouter.of(context).go('/success'); // WRONG
  }
}

// FORBIDDEN — async logic in widget to keep context access
class _BadScreenState extends State<BadScreen> {
  Future<void> _onTap() async {
    final result = await getIt<ApiClient>().submit(data); // WRONG: business logic in UI
    if (mounted) GoRouter.of(context).go('/success');
  }
}

// CORRECT — state class owns async work, widget reacts to state changes
class SubmitState extends ChangeNotifier {
  final ApiClient _api;
  SubmitState(this._api);

  bool isSubmitted = false;
  String? error;

  Future<void> submit(FormData data) async {
    try {
      await _api.submit(data);
      isSubmitted = true;
    } on AppException catch (e) {
      error = e.message;
    }
    notifyListeners(); // UI layer handles what happens next
  }
}

class _SubmitScreenState extends State<SubmitScreen> {
  late final SubmitState _state;

  @override
  void initState() {
    super.initState();
    _state = SubmitState(getIt<ApiClient>());
    _state.addListener(_onStateChanged);
  }

  void _onStateChanged() {
    if (!mounted) return; // mounted check HERE, in the UI layer
    if (_state.isSubmitted) {
      GoRouter.of(context).go('/success'); // navigation HERE, in the UI layer
    }
  }

  @override
  void dispose() {
    _state.removeListener(_onStateChanged);
    _state.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: _state,
      builder: (BuildContext context, _) {
        if (_state.error != null) return Center(child: Text(_state.error!));
        return ElevatedButton(
          key: K.submitButton,
          onPressed: () => _state.submit(formData),
          child: const Text('Submit'),
        );
      },
    );
  }
}
```

Summary:
- `ChangeNotifier`: owns async logic, updates fields, calls `notifyListeners()`. Never touches `context` or `mounted`.
- `State<T>`: listens to state changes, checks `mounted`, performs navigation and shows UI. Never calls APIs directly.

## Rule 12: Deep Modules — Hide Complexity Behind Narrow Interfaces

Every core service must be a **deep module**: simple interface, complex internals. You interact with services ONLY through their abstract interface. You never look at or modify the implementation.

### Pattern: Abstract class with throwing defaults, implementation in `impl/`

```dart
// core/services/api_client.dart — THIS IS ALL YOU SEE
abstract class ApiClient {
  Future<List<Post>> getPosts();
  Future<Post> getPost(String id);
  Future<Post> createPost({required String title, required String body});
  Future<void> deletePost(String id);

  // When you need a new method, add it here with a default:
  Future<List<Comment>> getComments(String postId) =>
      throw UnimplementedError('ApiClient.getComments not yet implemented');
}
```

When adding a new method to an interface:
1. Add it to the abstract class with `=> throw UnimplementedError('ClassName.methodName not yet implemented')`.
2. Write your feature code that calls it.
3. Write unit tests with a mock that returns test data — tests will pass.
4. The app compiles. The method throws at runtime until `impl/` is updated.
5. Leave a `// TODO: implement in impl/` comment on the method.

This keeps the agent unblocked while making the gap impossible to miss.

```dart
// core/services/impl/api_client_impl.dart — YOU DO NOT TOUCH THIS
class ApiClientImpl implements ApiClient {
  final Dio _dio;
  final TokenStore _tokenStore;
  final ConnectivityChecker _connectivity;

  // Retry logic, token refresh, caching, error mapping —
  // all hidden here. Agent never sees or modifies this.

  @override
  Future<List<Post>> getPosts() async {
    final response = await _makeRequest(() => _dio.get('/posts'));
    return (response.data as List).map((j) => Post.fromJson(j)).toList();
  }

  Future<Response> _makeRequest(Future<Response> Function() request) async {
    // retry, token refresh, connectivity check, error mapping
    // ... complex but irrelevant to feature development
  }
}
```

```dart
// Registration — agent copies this pattern, never changes impl
getIt.registerLazySingleton<ApiClient>(() => ApiClientImpl(...));
```

### Which services MUST be deep modules

| Service | Interface exposes | Implementation hides |
|---|---|---|
| `ApiClient` | Domain methods (`getPosts`, `createUser`) | HTTP, retry, auth tokens, caching, error mapping |
| `StorageService` | `get(key)`, `set(key, value)`, `delete(key)` | SharedPreferences vs Hive vs secure storage, encryption, migration |
| `AuthService` | `login()`, `logout()`, `isAuthenticated`, `currentUser` | Token refresh, biometrics, keychain, session expiry |
| `NotificationService` | `requestPermission()`, `onNotification` stream | FCM/APNs setup, channel creation, payload parsing |
| `LocationService` | `getCurrentLocation()`, `locationStream` | Permission handling, platform differences, accuracy tuning |
| `AnalyticsService` | `track(event, params)` | Provider SDK (Firebase, Mixpanel), batching, user properties |

### Rules

1. **Feature code imports only the abstract class.** Never import from `impl/`.
2. **One implementation per interface.** Don't create elaborate inheritance hierarchies. If you need a mock for tests, the abstract class is already mockable.
3. **Domain return types only.** The interface returns your models (`Post`, `User`), never library types (`Response`, `DocumentSnapshot`, `SharedPreferences`).
4. **Errors as domain exceptions.** The implementation catches library exceptions and throws your own:

```dart
// core/models/app_exception.dart
class AppException implements Exception {
  final String message;
  final String? code;
  AppException(this.message, {this.code});
}

class NetworkException extends AppException {
  NetworkException([String message = 'Network error']) : super(message, code: 'NETWORK');
}

class UnauthorizedException extends AppException {
  UnauthorizedException() : super('Session expired', code: 'UNAUTHORIZED');
}
```

The agent's state classes catch `AppException` subtypes. They never handle `DioException`, `SocketException`, `PlatformException`, etc.

```dart
// In a state class — clean, predictable error handling
Future<void> loadPosts() async {
  isLoading = true;
  error = null;
  notifyListeners();
  try {
    posts = await _api.getPosts();
  } on NetworkException {
    error = 'Check your connection';
  } on UnauthorizedException {
    // Auth redirect handled automatically by router's refreshListenable.
    // AuthService.isAuthenticated is already false at this point.
    error = 'Session expired';
  } on AppException catch (e) {
    error = e.message;
  } finally {
    isLoading = false;
    notifyListeners();
  }
}
```

5. **When creating a new feature that needs a new service method:** Add the method signature with a `throw UnimplementedError(...)` default to the abstract class. Add a `// TODO: implement in impl/` comment. Do NOT open `impl/` files.

### The `impl/` folder

```
core/
├── services/
│   ├── api_client.dart          # Abstract — you read this
│   ├── auth_service.dart        # Abstract — you read this
│   ├── storage_service.dart     # Abstract — you read this
│   ├── event_bus.dart           # Concrete (simple enough to be its own impl)
│   └── impl/                    # YOU DO NOT MODIFY FILES HERE
│       ├── api_client_impl.dart
│       ├── auth_service_impl.dart
│       └── storage_service_impl.dart
```

**The `impl/` directory is a boundary.** Treat it as a third-party package you cannot edit. This single constraint eliminates an entire class of bugs you would otherwise produce: misconfigured HTTP clients, broken token refresh, incorrect storage serialization, wrong permission request flows.

## Rule 13: Workflow — Update Cross-Cutting Files First

**Before writing feature code, review and update cross-cutting files if applicable.**
When creating or modifying a feature, edit files in this order:

1. `core/keys/app_keys.dart` — add keys for every new interactive widget
2. `routing/app_router.dart` — add or update route if the feature has a screen
3. `core/services/event_bus.dart` — add new event subclass if feature needs global notifications
4. `core/services/*.dart` — add method signatures to abstract interfaces if feature needs new data
5. Feature files — now build the feature

This order is mandatory. Do NOT start writing screen or state code before steps 1–4 are done. Open and read each cross-cutting file before editing it to verify current contents.

## Rule 14: Self-Evaluation (Mandatory)

After completing code for any task, you MUST output a `SELF-CHECK` block before finishing. Evaluate your code against every applicable item. Skip items that don't apply to the current change.

Every item MUST include a one-line evidence citation: the specific file and detail that proves compliance. A `[PASS]` without evidence is a `[FAIL]`.

Format:

```
SELF-CHECK:
- [PASS] No dynamic types → home_state.dart: all fields typed (List<Post>, bool, String?), loadPosts return Future<void>
- [PASS] Keys registered → app_keys.dart: added K.commentInput, K.commentSubmitButton
- [FAIL] Missing dispose for StreamSubscription → home_screen.dart:42 _sub never cancelled → fixing now
```

If any item is `[FAIL]`, fix it immediately, then re-output the full `SELF-CHECK` with the fix confirmed.

Checklist items:

1. No `.g.dart` or `.freezed.dart` imports anywhere
2. No `dynamic` types — every variable, return type, parameter explicitly typed
3. All own-code imports are relative — no `package:my_app/` anywhere
4. `analysis_options.yaml` strict mode passes with zero errors
4. Every new interactive widget has a key in `app_keys.dart`
5. State class extends `ChangeNotifier`, nothing else
6. Screen calls `_state.dispose()` in its `dispose()`
7. No `context` or `mounted` in ChangeNotifier classes — only in `State<T>` listener callbacks
8. StreamSubscriptions cancelled in `dispose()`
9. New routes added to `app_router.dart`
10. New services registered in `setupDependencies()`
11. Feature code imports abstract service interfaces, never `impl/`
12. `Map<String, dynamic>` never crosses a service boundary
13. Unit test exists for new state class
14. Round-trip serialization test exists for every new or modified model
15. Integration test updated if user flow changed
