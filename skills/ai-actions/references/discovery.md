# Discovery: finding and scoring candidate actions

## Where candidate actions live

Search these first — they're where discrete, user-meaningful operations usually sit.

### iOS (Swift)
- `func` in `*ViewModel.swift`, `*Presenter.swift`, `*Service.swift`, `*Manager.swift`, `*UseCase.swift`, `*Repository.swift`.
- `@MainActor` methods that a button calls.
- URL scheme / universal link handlers (`onOpenURL`, `application(_:open:)`), `NavigationPath` routes — these are your "open for details" targets.

Useful greps:
- `grep -rn "func .*(" --include=*.swift` then filter to public/internal, non-UI helpers.
- `grep -rn "URL(string\|onOpenURL\|widgetURL\|\.deeplink\|scheme" --include=*.swift`

### Android (Kotlin)
- `fun` in `*ViewModel.kt`, `*UseCase.kt`, `*Interactor.kt`, `*Repository.kt`, `*Service.kt`.
- `sealed`/`data class` action or intent models.
- `AndroidManifest.xml` `<intent-filter>`, deep-link `<data android:scheme>` — "open for details" targets.

Useful greps:
- `grep -rn "fun .*(" --include=*.kt` then filter to domain/use-case layer.
- `grep -rn "android:scheme\|deepLink\|NavDeepLink" --include=*.xml --include=*.kt`

### Don't assume clean architecture (both platforms)

Some apps follow clean architecture (UseCase/Repository/Interactor); others have **no such layer at all** and put operations in a thin API/service layer plus ViewModels. Handle both — start with the layer greps above, and if they turn up nothing, fall back to:

- **Android**: a top-level **API manager** `object`, a Retrofit **service interface**, or plain **`Service` classes**; ViewModel methods calling them; a Room **`@Dao`** for local reads. Grep `object .*Api`, `interface .*Service`/`.*Api`, `class .*Service`, `@Dao`.
- **iOS**: API clients / **`Service` classes** in a networking or data module; Core Data / SwiftData / an App-Group store for local reads. Grep `class .*Service`, `struct .*APIType`, `class .*APIClient`, `NSManagedObject`, `@Model`.

**Don't assume the backend is REST.** Many apps have no HTTP client at all — data comes through **Firebase (RTDB/Firestore), GraphQL, gRPC, or another SDK**. So don't anchor discovery on `@GET`/`@POST` or Moya; anchor on the **Service layer** (`*Service`, `*Repository`), which is the reliable seam regardless of transport. The action wraps a Service method whether it hits HTTP, Firebase, or local storage underneath.

**Detect the DI framework and match it — don't assume either way.** Android: `@Inject`/`@Module`/Hilt/Koin, or a **manual container** (`AppContainer.get(context)`). iOS: `@Dependency`/a container, `.shared` singletons, `@Environment`-shared objects, or services **instantiated inline inside ViewModels** (no shared instance at all). If DI exists, take the dependency the app's way; if there's genuinely none, construct/fetch the service the way the app does. Either way, **don't introduce a DI pattern the app doesn't already use, and don't strip one it does.** (See the bootstrap note below — an action runs outside the app's normal lifecycle, so "just grab the ViewModel" often isn't available.)

Local reads (recently-viewed, saved queries) are frequently the **easiest action to ship** on either platform — no auth, no network.

### Capture natural-language → internal-ID resolvers (both platforms)

Spoken parameters are rarely the app's internal keys. A user says a place, product, or category by *name*; the API wants an *id* or *code*. During inventory, hunt for the **resolver** that bridges them and note it alongside the action it feeds — an action that takes a coded id is only voice-usable if a resolver turns the spoken value into that id.

Example shape: a place-name → region-code resolver (address string → administrative code). Look for name→id / geocode / category-lookup helpers (`grep -rin "geocode\|lookup\|resolve\|byName\|findCode\|toId"` and whatever the app's domain calls it). If a param is a place, category, or named entity and **no resolver exists**, flag it: the action needs one before it can be voice-driven (or the parameter must be modeled as a resolvable entity — see the platform templates).

## What makes a good candidate

Capture per candidate: **name · file:line · what it does · inputs · output · read-or-write**.

A strong action is something a person would *say out loud* — "check my …", "add a …", "how much …". A weak one is an internal helper, a UI-only toggle, or plumbing.

## Scoring rubric

Score each candidate 0–2 on five axes. Shortlist the highest totals.

| Axis | 0 | 1 | 2 |
|---|---|---|---|
| **User-meaningful** | internal/plumbing | vaguely useful | a user would ask for it by name |
| **AI can't do it alone** | plain math/generic text | partly generic | needs app data / domain rules / account state |
| **Clear I/O** | fuzzy, stateful | some ambiguity | well-defined inputs → one clear output |
| **Single-purpose** | does many things | two-ish | one job, one verb |
| **Safety (read/write)** | destructive, no undo | mutating, reversible | read-only |

Notes:
- **AI-can't-do-it-alone is the tie-breaker.** An action the assistant could answer without your app will rarely get routed to you. Domain-specific, data-backed actions win.
- Low **Safety** score doesn't disqualify — it means the intent needs a confirmation step (see platform templates), not that you skip it.
- Prefer several **narrow** actions over one god-action. The list itself is the catalog the AI chooses from, so more well-named entries = better routing.

## Presenting the shortlist

Show a ranked table: action name · one-line description · inputs → output · read/write · score. Then ask which to implement. Never scaffold the whole list unprompted.
