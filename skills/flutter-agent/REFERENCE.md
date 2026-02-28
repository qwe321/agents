# Flutter Agent Rules — Reference Examples

Full code examples for each rule in `SKILL.md`. Consult when implementing a pattern for the first time or when unsure of the correct form.

---

## Rule 1: Hand-Written Model Example

```dart
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

---

## Rule 2: Typing & Null Safety Examples

### Inference OK vs Forbidden

```dart
// Inference OK — type is obvious from RHS
final homeState = HomeState(api);
final posts = <Post>[];
const padding = EdgeInsets.all(16.0);
for (final post in posts) { ... }
final name = 'hello';

// Inference FORBIDDEN — type is ambiguous or hidden
final data = json['key'];                      // dynamic!
final items = response.data;                   // dynamic from Dio!
final mapped = list.map((e) => e.name);        // Iterable<dynamic> if list is untyped
final result = await _api.fetchSomething();    // unclear without checking return type

// When in doubt, annotate
final List<Post> result = await _api.getPosts();
final String name = user.displayName;
```

### Service boundary typing

```dart
// FORBIDDEN — dynamic leaking into feature code
Future<Map<String, dynamic>> getUser() async { ... }

// CORRECT — typed at the boundary
Future<User> getUser() async {
  final Map<String, dynamic> json = await _get('/user');
  return User.fromJson(json);
}
```

### analysis_options.yaml

```yaml
analyzer:
  strict-casts: true
  strict-raw-types: true
  errors:
    non_exhaustive_switch_statement: error
```

### Null safety — `!` operator

```dart
// FORBIDDEN — no null check
final String name = user!.name;

// FORBIDDEN — null check in different scope
if (user != null) { ... }
doSomething(user!.name);

// CORRECT — null check + return promotes the variable
if (user == null) return;
final String name = user.name;

// CORRECT — safe local binding
final User? currentUser = _state.currentUser;
if (currentUser == null) {
  error = 'Not logged in';
  notifyListeners();
  return;
}
final String name = currentUser.name;
```

### Nullable fields

```dart
// FORBIDDEN — lazy nullability
class Post {
  String? title;
  String? body;
}

// CORRECT — required unless genuinely optional
class Post {
  final String title;
  final String body;
  final String? subtitle;  // genuinely optional in the domain

  Post({required this.title, required this.body, this.subtitle});
}
```

---

## Rule 4: ChangeNotifier + ListenableBuilder Pattern

### State class

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

### Screen

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
        final String? error = _state.error;
        if (error != null) return Center(child: Text(error));
        return ListView.builder(
          itemCount: _state.posts.length,
          itemBuilder: (context, i) => PostTile(post: _state.posts[i]),
        );
      },
    );
  }
}
```

---

## Rule 5: Mutable State Example

```dart
// FORBIDDEN — immutable + copyWith
class HomeState {
  final List<Post> posts;
  final bool isLoading;
  final String? error;
  const HomeState({this.posts = const [], this.isLoading = false, this.error});

  HomeState copyWith({List<Post>? posts, bool? isLoading, String? error}) =>
      HomeState(
        posts: posts ?? this.posts,
        isLoading: isLoading ?? this.isLoading,
        error: error ?? this.error,   // BUG: can never set error back to null
      );
}

// CORRECT — mutable ChangeNotifier
class HomeState extends ChangeNotifier {
  List<Post> posts = [];
  bool isLoading = false;
  String? error;

  void clearError() {
    error = null;
    notifyListeners();
  }
}
```

---

## Rule 7: Widget Keys Registry

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

---

## Rule 8: EventBus — Full Implementation

### Event hierarchy

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

### EventBus class

```dart
class EventBus {
  final StreamController<AppEvent> _controller = StreamController<AppEvent>.broadcast();
  Stream<AppEvent> get stream => _controller.stream;
  void fire(AppEvent event) => _controller.add(event);
  void dispose() => _controller.close();
}
```

### EventListenerWrapper

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
    if (widget.suppressedEvents.contains(event.runtimeType)) return;
    if (!mounted) return;

    // Switch EXPRESSION on sealed type — compile error if case missing
    final String? snackText = switch (event) {
      SyncErrorEvent(:final message) => message,
      NewMessageEvent(:final senderName, :final preview) => '$senderName: $preview',
      AchievementUnlockedEvent(:final title) => 'Unlocked: $title',
    };

    if (snackText != null) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(snackText)),
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

### Usage in a screen

```dart
@override
Widget build(BuildContext context) {
  return EventListenerWrapper(
    suppressedEvents: {NewMessageEvent}, // chat handles these inline
    child: Scaffold(...),
  );
}
```

---

## Rule 9: GoRouter with Auth Redirect

### AuthService abstract class

```dart
// core/services/auth_service.dart
abstract class AuthService extends ChangeNotifier {
  bool get isAuthenticated;
  User? get currentUser;
  Future<void> login({required String email, required String password});
  Future<void> logout();
  // Implementation calls notifyListeners() on auth state changes.
}
```

### Router

```dart
// routing/app_router.dart
final GoRouter appRouter = GoRouter(
  initialLocation: '/home',
  refreshListenable: getIt<AuthService>(),
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

---

## Rule 10: Testing Examples

### Unit test — state class

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

### Unit test — model round-trip

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

### Integration test — Patrol

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

---

## Rule 11: Dependency Injection Setup

```dart
// main.dart — the only place that imports impl/
import 'core/services/impl/api_client_impl.dart';
import 'core/services/impl/auth_service_impl.dart';

void setupDependencies() {
  getIt.registerLazySingleton<EventBus>(() => EventBus());
  getIt.registerLazySingleton<ApiClient>(() => ApiClientImpl());
  getIt.registerLazySingleton<AuthService>(() => AuthServiceImpl(getIt<ApiClient>()));
}

void main() {
  setupDependencies();
  runApp(const MyApp());
}
```

---

## Rule 12: The `mounted` / `context` Boundary — Full Examples

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
    notifyListeners();
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
    if (!mounted) return;
    if (_state.isSubmitted) {
      GoRouter.of(context).go('/success');
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
        final String? error = _state.error;
        if (error != null) return Center(child: Text(error));
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

---

## Rule 13: Deep Modules — Full Examples

### Abstract interface with UnimplementedError default

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

### Implementation (for Service Mode reference)

```dart
// core/services/impl/api_client_impl.dart
class ApiClientImpl implements ApiClient {
  final Dio _dio;
  final TokenStore _tokenStore;
  final ConnectivityChecker _connectivity;

  @override
  Future<List<Post>> getPosts() async {
    final response = await _makeRequest(() => _dio.get('/posts'));
    return (response.data as List).map((j) => Post.fromJson(j)).toList();
  }

  Future<Response> _makeRequest(Future<Response> Function() request) async {
    // retry, token refresh, connectivity check, error mapping
  }
}
```

### Domain exception hierarchy

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

### Error handling in state classes

```dart
Future<void> loadPosts() async {
  isLoading = true;
  error = null;
  notifyListeners();
  try {
    posts = await _api.getPosts();
  } on NetworkException {
    error = 'Check your connection';
  } on UnauthorizedException {
    error = 'Session expired';
  } on AppException catch (e) {
    error = e.message;
  } finally {
    isLoading = false;
    notifyListeners();
  }
}
```

### Deep module service table

| Service | Interface exposes | Implementation hides |
|---|---|---|
| `ApiClient` | Domain methods (`getPosts`, `createUser`) | HTTP, retry, auth tokens, caching, error mapping |
| `StorageService` | `get(key)`, `set(key, value)`, `delete(key)` | SharedPreferences vs Hive vs secure storage, encryption, migration |
| `AuthService` | `login()`, `logout()`, `isAuthenticated`, `currentUser` | Token refresh, biometrics, keychain, session expiry |
| `NotificationService` | `requestPermission()`, `onNotification` stream | FCM/APNs setup, channel creation, payload parsing |
| `LocationService` | `getCurrentLocation()`, `locationStream` | Permission handling, platform differences, accuracy tuning |
| `AnalyticsService` | `track(event, params)` | Provider SDK (Firebase, Mixpanel), batching, user properties |

### The `impl/` folder structure

```
core/
├── services/
│   ├── api_client.dart          # Abstract — you read this
│   ├── auth_service.dart        # Abstract — you read this
│   ├── storage_service.dart     # Abstract — you read this
│   ├── event_bus.dart           # Concrete (simple enough to be its own impl)
│   └── impl/                    # YOU DO NOT MODIFY FILES HERE (Feature Mode)
│       ├── api_client_impl.dart
│       ├── auth_service_impl.dart
│       └── storage_service_impl.dart
```
