# Architecture

> This document is the source of truth for *how the code is organised* in a
> Flutter app built on **Riverpod** + Clean Architecture. Read it before
> adding a feature or touching cross-cutting code. For visual/UI rules see
> [`design.md`](./design.md) *(placeholder — not written yet, added later)*;
> for coding conventions see [`rules.md`](./rules.md).
>
> Treat this as a target-shape template: adapt package versions, feature
> names, and the optional Stack pieces (§1: Firebase, realtime) to the
> actual project.

---

## 1. Stack

| Concern | Choice |
|---|---|
| Framework | Flutter, Dart SDK (see `pubspec.yaml` → `environment.sdk`) |
| State management | `flutter_riverpod` + `riverpod_annotation` (codegen) |
| Routing | `go_router` |
| Networking | `dio`, optionally wrapped by `dio_extended` for shared interceptors/logging |
| Models / serialization | `freezed` + `json_serializable` |
| Entities | plain classes; `equatable` optional |
| Localization | Flutter `gen-l10n` (ARB) |
| Testing | `flutter_test` + `mocktail` (mocking) + `ProviderContainer` overrides (provider/notifier tests) |
| Backend platform (optional) | Firebase (Core, Crashlytics, Messaging) — add only if the project needs it |
| Realtime (optional) | `socket_io_client` or similar — add only if the project has a live-data feature |
| Font | project-specific brand font, bundled with its weights |

Codegen (`freezed`, `json_serializable`, `riverpod_generator`) runs via
`build_runner`. Never edit `*.freezed.dart`, `*.g.dart`, or
`lib/l10n/generated/**` by hand.

---

## 2. Top-level layout

```
lib/
  main_development.dart      # flavor entry → initApp(dev)  → runApp(App)
  main_production.dart       # flavor entry → initApp(prod) → runApp(App)
  app.dart                   # App (root MaterialApp.router) + initApp() bootstrap + Environment enum
  firebase_options.dart      # generated, only if Firebase is used

  core/                      # cross-cutting infrastructure (see §4)
  features/                  # feature modules, one folder each (see §3)
  themes/                    # colors + text styles (see design.md)
  widgets/                   # app-wide shared widgets
  utils/                     # generic helpers (formatting, sessions, l10n, ...)
  l10n/                      # app_<locale>.arb (template), generated/
```

Two flavors, `development` and `production`, expressed as an `Environment`
enum and selected by which `main_*.dart` is run. Each entry point calls
`initApp(env)` then wraps the app in `ProviderScope`. Only add flavors when
the project actually needs environment-specific config.

`initApp()` (in `app.dart`) is the only place for eager, blocking startup
work: load env config, initialize local/encrypted storage, restore any
cached session, initialize backend SDKs if used. Anything not required
before first frame belongs in the splash screen or the relevant feature —
this keeps the platform from showing a blank screen on launch.

---

## 3. Feature modules (`lib/features/`)

Each feature is a self-contained slice organised by **Clean Architecture** in
three layers. Note the folder name is **pluralized** — `presentations/` (not
`presentation`).

```
features/<feature>/
  data/
    datasources/       # talk to the network via a BaseConnection; return ApiResult
    models/            # freezed + json_serializable DTOs (snake_case JSON → camelCase)
    repositories/      # <name>_repository_impl.dart — implements the domain interface
  domain/
    entities/          # plain Dart entities the UI/business layer consumes
    repositories/      # abstract repository interfaces
    usecases/          # thin wrappers over a repository (one action each)
  presentations/
    providers/         # Riverpod providers (DI wiring + notifiers) + generated .g.dart
    views/              # full-page screens
    widgets/            # feature-scoped widgets
    navigation/          # typed route-args classes
    utils/              # params builders, formatters (optional)
```

### The dependency rule

Dependencies point **inward only**:

```
presentations  ──▶  domain  ◀──  data
   (UI)          (entities,      (datasources,
                  usecases,       models,
                  repo iface)     repo impl)
```

- `domain` depends on nothing outside itself. Entities are plain Dart.
- `data` implements the interfaces declared in `domain/repositories/` and maps
  its `models/` (DTOs) down to `domain/entities/`.
- `presentations` depends on `domain` (usecases + entities), never on `data`
  types directly.

### The standard call chain

Every feature follows the same vertical wiring, one provider file per feature:

```
View (ConsumerWidget)
  → ref.read(<feature>Provider.notifier).someAction()   # a Notifier
      → ref.read(<feature>UsecaseProvider)               # Usecase
          → <Feature>RepositoryImpl                       # implements domain iface
              → <Feature>RemoteDataSource                 # BaseConnection.callApiRequest
                  → REST API
```

The `datasource → repository → usecase → notifier` objects are assembled with
Riverpod codegen providers (see §6). This is the pattern to copy when adding a
feature — not every feature needs every layer; skip what a purely local or
static feature doesn't use.

---

## 4. Core infrastructure (`lib/core/`)

| Folder | Responsibility |
|---|---|
| `configs/` | `EnvConfig` (typed env getters), `.env` loading, secret keys, crypto helpers |
| `constants/` | base URLs and other app-wide constants |
| `extensions/` | date/time helpers, human-readable error mapping |
| `interceptors/` | shared Dio interceptors (headers, content-type fixes, logging) |
| `models/` | `GlobalResponse`, `ApiException`, `Result`/`ResultEntities`, push-notification model (if used) |
| `router/` | `appRouter` (GoRouter), `Routes` (route-name constants), route observer |
| `services/` | `BaseConnection*` HTTP clients, messaging/notification service, global events |

### Environments & config

`EnvConfig.load(env.name)` reads `.env.development` / `.env.production`.
Access config through `EnvConfig` / a constants class, never by reading env
keys directly at call sites.

### Networking

One or more singleton HTTP clients extend a shared base client, one per
backend surface — e.g. `BaseConnection` for the main API, plus an additional
one per additional backend the app talks to.

Key behaviours to keep consistent across every `BaseConnection*`:

- **Auth is header-based**: attach any session token/key to every request
  from a single place, not per call site.
- **401 handling**: centralize token-refresh/re-auth in the base client —
  call sites should not implement their own retry logic.
- Reasonable timeouts (shorter in debug, longer in release, if the project
  wants that split).
- A shared interceptor layer for logging/debugging in non-production builds.

Datasources call `_connection.callApiRequest(request: ..., parseData: ...)`,
which returns an `ApiResult<T>`; `parseData` maps raw JSON into a model
(often wrapped in a shared envelope type). Repository impls unwrap the
`ApiResult`/envelope, map to domain entities, and throw a typed exception on
failure (see §5).

### API envelope

If the backend wraps every response in a shared shape (e.g.
`GlobalResponse { int? statusCode; String? message; String? token; dynamic
data; }`), model it once and have every datasource's `parseData` extract
from it — don't re-parse the envelope per feature.

---

## 5. Error model

- Datasources throw typed exceptions (e.g. `ApiException`, `CacheException`)
  on failure — never let a raw `DioException` or platform exception escape a
  datasource.
- Repository impls catch the datasource's exception, map it to a
  human-readable message via a shared error-mapping helper (see
  `core/extensions/`), and rethrow a typed app exception. Pick one error
  style — plain typed exceptions, or a functional `Result`/`Either` wrapper
  — and use it consistently across every repository; don't mix styles per
  feature.
- Usecases and notifiers never construct user-facing error copy themselves —
  a notifier's `AsyncValue.guard(...)` call captures whatever the usecase
  throws and lands it in `AsyncError`, so views only ever branch on
  `AsyncValue`'s data/error/loading (see §6).
- `core/models/` is where any shared response envelope and typed exception
  classes live — see §4's "API envelope".

---

## 6. State management (Riverpod)

The app is wrapped in a single `ProviderScope`. Two annotation styles are
used together:

- `@riverpod` — auto-dispose functional providers, used for **DI wiring**
  (datasource → repository → usecase).
- `@Riverpod(keepAlive: true)` — persistent **Notifiers** for session / auth /
  locale state that must survive navigation.

Prefer codegen providers (`@riverpod`) for everything new; avoid introducing
legacy `StateProvider`s (from `flutter_riverpod/legacy.dart`) except to match
an existing one already in the codebase.

### DI wiring pattern (copy this per feature)

```dart
@riverpod
<Feature>RemoteDataSource <feature>Datasource(Ref ref) => <Feature>RemoteDataSource();

@riverpod
<Feature>Repository <feature>Repository(Ref ref) =>
    <Feature>RepositoryImpl(ref.watch(<feature>DatasourceProvider));

@riverpod
<Feature>Usecase <feature>Usecase(Ref ref) =>
    <Feature>Usecase(ref.watch(<feature>RepositoryProvider));
```

### Notifier pattern (async state)

Notifiers expose `AsyncValue<T>` state, set `AsyncLoading()`, run the call
through `AsyncValue.guard`, and guard the write with `if (!ref.mounted) return;`
before assigning `state`:

```dart
@riverpod
class <Feature> extends _$<Feature> {
  @override
  AsyncValue<List<Thing>> build() => const AsyncData([]);

  Future<void> load() async {
    state = const AsyncLoading();
    final usecase = ref.read(<feature>UsecaseProvider);
    final result = await AsyncValue.guard(() => usecase.call());
    if (!ref.mounted) return;
    state = result;
  }
}
```

Views render `AsyncValue` with `.when(data/error/loading)`.

---

## 7. Routing

`go_router` with a single `appRouter` instance (`lib/core/router/app_router.dart`)
wired into `MaterialApp.router`. Route-name constants live in
`lib/core/router/routes.dart` (`abstract class Routes`).

- Shell: `StatefulShellRoute.indexedStack` when the app has a bottom-nav
  shell, one branch per primary tab.
- Non-shell screens are top-level `GoRoute`s.
- **Arguments** are passed via `state.extra`, cast to a typed *route-args*
  class kept in the feature's `presentations/navigation/` folder, with a
  fallback cast:

```dart
GoRoute(
  path: Routes.<feature>Detail,
  builder: (context, state) {
    final extra = state.extra;
    if (extra is <Feature>DetailRouteArgs) {
      return <Feature>DetailView(id: extra.id);
    }
    return <Feature>DetailView(id: extra as String? ?? '');
  },
),
```

Add new routes by: (1) a constant in `Routes`, (2) a `GoRoute` in `appRouter`,
(3) a typed args class in the feature. Navigate with `context.push`/`.go` using
the `Routes.*` constant and the typed args as `extra`.

---

## 8. Models & serialization

Network DTOs are `freezed` + `json_serializable`. Convention:

- snake_case JSON keys mapped with `@JsonKey(name: '...')` to camelCase fields,
- nullable fields where the backend is permissive about omitting them,
- three generated files per model: `x.dart`, `x.freezed.dart`, `x.g.dart`.

```dart
@freezed
abstract class <Feature>Model with _$<Feature>Model {
  const factory <Feature>Model({
    int? id,
    String? name,
    @JsonKey(name: 'created_at') String? createdAt,
  }) = _<Feature>Model;

  factory <Feature>Model.fromJson(Map<String, dynamic> json) =>
      _$<Feature>ModelFromJson(json);
}
```

Domain entities are plain (non-freezed) Dart classes with only the fields the
app actually needs. Repository impls translate DTO → entity so the UI never sees
raw JSON shapes.

---

## 9. Localization

- `gen-l10n` config in `l10n.yaml`: `arb-dir: lib/l10n`, template = the app's
  primary/source-of-truth locale's ARB file, output `AppLocalizations` into
  `lib/l10n/generated/`.
- A `LocaleNotifier` (`@Riverpod(keepAlive: true)`) holds the active locale,
  persists changes, and exposes a non-context accessor for places without a
  `BuildContext`.
- Add strings to **every** supported locale's ARB file with matching keys,
  then run `build_runner` / `flutter gen-l10n`. Never hardcode user-facing
  copy.

---

## 10. Auth (optional)

If the app needs authentication, follow the same layered shape as any other
feature:

```
AuthRemoteDataSource (wraps whatever auth SDK/backend is used)
  → AuthRepositoryImpl                                 # implements AuthRepository
      → LoginUsecase / LogoutUsecase / CheckLoginUsecase
          → AuthNotifier (@Riverpod(keepAlive: true))  # drives LoginView / SplashView, survives navigation
```

Register the SDK client via a provider the same way any other core service is
registered (see §4). Auth error handling doesn't have to go through the
shared error-mapping helper (see §5) if the SDK throws exceptions that need
bespoke messages — a dedicated try/catch mapping SDK exceptions to the app's
typed exception is fine here.

---

## 11. Bootstrapping & error reporting

`initApp()` installs global error handlers, if crash reporting is used:

- `FlutterError.onError` and `PlatformDispatcher.instance.onError` forward to
  the crash-reporting backend.
- Consider classifying transient/expected failures (e.g. a network-image load
  failing because of a flaky connection) as non-fatal, so they don't pollute
  crash-free-rate metrics — keep this classification consistent as new error
  handling is added.

---

## 12. Feature assembly flow

When assembling a new feature module, construct the components in dependency order:

1. **Domain**: Define plain entity classes, repository interfaces, and usecases (`domain/`).
2. **Data**: Build freezed DTO models (`data/models/`), remote datasources (`data/datasources/`), and repository implementations (`data/repositories/`).
3. **State & DI**: Wire up Riverpod providers and async notifiers (`presentations/providers/`).
4. **UI & Navigation**: Implement views, widgets, and typed route-args (`presentations/views/`, `presentations/widgets/`, `presentations/navigation/`).
5. **Routing & L10n**: Register routes in `Routes` + `appRouter`, and add strings to every locale's ARB file.

---

## 13. Testing

| Layer | Tool | What it covers |
|---|---|---|
| Usecase / repository | `flutter_test` + `mocktail` | mock the layer below (a repository mocks its datasource; a usecase mocks its repository interface), assert the returned entity or thrown exception |
| Notifier / provider | `flutter_test` + `ProviderContainer` + `mocktail` | build a `ProviderContainer` with `overrides: [<feature>UsecaseProvider.overrideWithValue(mockUsecase)]`, call the notifier's method, assert the emitted `AsyncValue` sequence (`AsyncLoading` → `AsyncData`/`AsyncError`) |
| Widget/screen | `flutter_test` (`testWidgets`) | wrap the widget in a `ProviderScope(overrides: [...])`, assert each of the three `AsyncValue` render branches |

Test files mirror `lib/`'s structure under `test/`, one `<file>_test.dart` per
source file.

Prefer `mocktail` — no code generation. Build a fresh `ProviderContainer` per
test, `addTearDown(container.dispose)`, then read/listen to the provider
under test to assert its emitted states — this is the Riverpod equivalent of
asserting a Cubit's state sequence.

**Mock one layer down, never further** — a notifier test mocks its usecase,
not the repository or datasource beneath it; a usecase test mocks its
repository interface. Not every class needs a test: prioritize usecases
(business logic) and notifiers (state transitions) first — widget tests earn
their cost on a screen's branching logic, not on pure layout.

> For coding standards, naming conventions, and the complete verification checklist before marking work complete, refer to [`rules.md`](./rules.md) (especially §12 Definition of done).
