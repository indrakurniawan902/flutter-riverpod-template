# Rules

> Coding conventions and working agreements for a Riverpod + Clean
> Architecture Flutter app. Read this together with
> [`architecture.md`](./architecture.md) (structure) and
> [`design.md`](./design.md) (visuals) *(placeholder — not written yet, added later)*.
>
> These rules describe the target conventions for this codebase. Follow them
> so new code stays consistent as the app grows, rather than each feature
> inventing its own style.

---

## 1. Golden rules

1. **Match the surrounding code.** Once features exist, copy the nearest
   existing feature's structure, naming, and idioms rather than inventing a
   new style.
2. **Respect the layers.** `presentations → domain ← data`. UI never imports
   `data/` types; `domain` imports nothing app-specific.
3. **One provider chain per feature.** `datasource → repository → usecase →
   notifier`, wired with Riverpod codegen.
4. **Never hardcode** colors, text styles, strings, or base URLs — use `AppColor`,
   the text-style functions, ARB keys, and `EnvConfig`/constants.
5. **Never hand-edit generated files** (`*.freezed.dart`, `*.g.dart`,
   `lib/l10n/generated/**`). Change the source, then regenerate.
6. **Verify before claiming done:** `flutter analyze` clean and the app builds.
7. **Don't leave debt behind.** If you notice an inconsistency while touching
   a file, align it to the convention described here rather than copying the
   deviation forward.

---

## 2. Project structure rules

- Features live in `lib/features/<feature>/` following Clean Architecture and
  the **plural** `presentations/` folder (see
  [`architecture.md §3`](./architecture.md#3-feature-modules-libfeatures)).
- Promote a widget to `lib/widgets/` only when a second feature needs it;
  otherwise keep it in the feature's `presentations/widgets/`.
- Cross-cutting infra goes in `lib/core/`; generic helpers in `lib/utils/`.
- Route-args classes belong in the feature's `presentations/navigation/`.

---

## 3. Naming

| Kind | Convention | Example |
|---|---|---|
| Files | `snake_case.dart` | `<feature>_view.dart` |
| Classes / enums | `UpperCamelCase` | `<Feature>RepositoryImpl` |
| Members / vars | `lowerCamelCase` | `fetch<Thing>` |
| Constants | `lowerCamelCase` (`static const`) | `AppColor.primary50` |
| Datasource | `<Feature>RemoteDataSource` | `<Feature>RemoteDataSource` |
| Repo interface / impl | `<Feature>Repository` / `<Feature>RepositoryImpl` | `<Feature>Repository` / `<Feature>RepositoryImpl` |
| Usecase | `<Feature>Usecase` | `<Feature>Usecase` |
| Notifier | `<Thing>Notifier` or `<Thing>` extends `_$<Thing>` | `<Feature>Notifier` |
| Model (DTO) | `<Thing>Model` | `<Feature>Model` |
| Entity | `<Thing>Entities` / `<Thing>Entity` | `<Feature>Entity` |
| View (screen) | `<Thing>View` | `<Feature>View` |
| Route-args | `<Thing>RouteArgs` / `<Thing>Args` | `<Feature>DetailRouteArgs` |

Pick one entity suffix (`Entity` or `Entities`) and use it consistently
across every feature, not just the first one.

JSON keys are commonly snake_case in an API; map them with
`@JsonKey(name: '...')` to camelCase Dart fields.

---

## 4. State management (Riverpod)

Follow the provider and async notifier patterns in [`architecture.md §6`](./architecture.md#6-state-management-riverpod):

- **Provider styles:** `@riverpod` for functional/DI providers; `@Riverpod(keepAlive: true)` for persistent state (session, locale, etc).
- **Async guard & mounted checks:** Notifiers expose `AsyncValue<T>`. Always guard post-await state writes with `if (!ref.mounted) return;`.
- **Reactivity:** Use `ref.read` in action callbacks; use `ref.watch` in reactive builds and provider wiring.
- **No new `StateProvider`s:** Do not introduce legacy `StateProvider`s.
- **Codegen:** Run `build_runner` after adding or changing annotated providers.

---

## 5. Networking & data

Follow the client and data mapping architecture in [`architecture.md §4`](./architecture.md#4-core-infrastructure-libcore):

- **No raw Dio at call sites:** All network calls must go through the appropriate `BaseConnection*` singleton via `callApiRequest(request:, parseData:)`.
- **Centralized 401/refresh handling:** Do not write manual token refresh or retry logic at call sites.
- **DTO to Entity boundary:** Datasources return `ApiResult<T>` wrapping Freezed models. Repository implementations unwrap models, map them to Domain Entities, and throw a typed exception on failure. The UI layer must never consume DTO models directly.
- **Model rules:** `freezed` + `json_serializable` with `@JsonKey` snake_case mapping where the backend uses it. Domain entities are plain classes exposing only needed fields.

---

## 6. UI & widgets

- Widgets that need text styles or theme-reactive values are `ConsumerWidget`s.
- Colors via `AppColor.*`; text via the shared text-style functions. No bare `TextStyle(...)`, no literal hex in widgets.
- Reuse `lib/widgets/` components before building new UI.
- Handle all three `AsyncValue` branches: Loading (shimmer/skeleton) → Error (human-readable message) → Success (content).
- Follow the spacing/radius scale in [`design.md`](./design.md) once it exists.

---

## 7. Localization

Follow the localization setup in [`architecture.md §9`](./architecture.md#9-localization):

- **Sync every locale:** Every user-facing string must be an ARB key added to every supported locale's ARB file, keeping keys in sync.
- **String access:** Use generated `AppLocalizations` (or a non-context accessor where a `BuildContext` is unavailable).
- **No hardcoding or concatenation:** Never concatenate translated fragments — use ICU placeholders for dynamic values.

---

## 8. Config & secrets

- Read configuration via `EnvConfig` / a constants class, never by reading env keys inline.
- `.env.development` / `.env.production` and key files hold secrets — **do not**
  commit new secrets, print them, or paste values into logs, code, or PRs.
- Environment selection is via `main_development.dart` / `main_production.dart`
  (the `Environment` enum) — don't branch on env ad-hoc in feature code.

---

## 9. Error handling & logging

Follow the typed-exception shape in [`architecture.md §5`](./architecture.md#5-error-model):

- Convert exceptions to user-facing copy through a shared error-mapping
  helper; never surface raw exception text.
- If crash reporting is wired up, uncaught errors flow through the handlers
  installed in `initApp()` — keep any deliberate fatal/non-fatal
  classification intact.
- **No `print()` in committed code.** Use proper logging / crash reporting
  where needed.

---

## 10. Code generation & tooling

Run codegen after touching freezed models, json models, annotated providers, or
ARB files:

```bash
dart run build_runner build --delete-conflicting-outputs
flutter gen-l10n            # if only ARB changed
```

Lint/analyze before finishing:

```bash
flutter analyze
```

Lints are configured in `analysis_options.yaml` (`flutter_lints`). Keep analyze
output clean — fix warnings introduced by your change.

---

## 11. Testing

Follow the tooling and layout in
[`architecture.md §13`](./architecture.md#13-testing):

- **Mock one layer down, never further.** A notifier test mocks its usecase;
  a usecase test mocks its repository interface; a repository test mocks its
  datasource.
- **Notifier tests assert the `AsyncValue` sequence**, not just the final
  state — build a `ProviderContainer` with the usecase provider overridden,
  and check it emits `AsyncLoading()` then `AsyncData(...)`/`AsyncError(...)`
  in order.
- **New usecases and notifiers ship with tests** as part of the same change
  that adds them — don't defer coverage to a follow-up.
- **Widget tests** are for a screen's branching (loading/error/success), not
  for verifying static layout — skip them for widgets with no conditional
  rendering.
- Test files mirror `lib/` under `test/`: `test/<same path>/<file>_test.dart`.

---

## 12. Definition of done

Before marking a task complete:

1. Code follows the layer + naming rules above and matches nearby code.
2. `build_runner` has been run if generated code is affected.
3. `flutter analyze` is clean (no new warnings/errors).
4. New/changed strings exist in every supported locale's ARB file.
5. UI uses `AppColor` + text-style functions + shared components; all three
   `AsyncValue` branches handled.
6. No secrets, no `print()`, no hand-edited generated files.
7. The change is scoped to the request — no unrelated refactors.
8. New usecases/notifiers have tests, and `flutter test` passes (see §11).

---

## 13. UI slicing from Figma

When the user shares a Figma link (a `figma.com/...` URL) for a screen or
component to implement:

- **Use the Figma MCP, not the screenshot alone.** Load the
  `figma-design-to-code` skill, then call `get_design_context` to pull real
  layout, spacing, and token data before writing any widget code.
- **Map tokens, don't hardcode them.** Translate the colors/text styles the
  MCP returns onto `AppColor` and the shared text-style functions — never
  paste raw hex values or `TextStyle(...)` literals pulled from Figma
  directly into a widget (see [§6](#6-ui--widgets)).
- **Reuse before creating.** Check `lib/widgets/` and the feature's
  `presentations/widgets/` for an existing component that matches a sliced
  element before building a new one; keep new screen-local widgets inline
  until a second feature needs them.
- **Slice UI only.** A Figma slice produces the widget tree — wire it into
  the feature's existing provider/notifier chain (§2–§4) rather than
  hardcoding sample data from the design.

---

## 14. Git & PRs

- Keep changes scoped to the task; avoid drive-by refactors.
- Don't commit unless asked; stage specific files rather than `git add .`.
- Never commit `.env*` or key/secret files.
- PR description: what changed, why, what was verified (analyze/build/manual).
