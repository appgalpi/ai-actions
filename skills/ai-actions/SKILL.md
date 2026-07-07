---
name: ai-actions
description: Analyze an iOS or Android project to find which user-facing features are worth exposing to the on-device AI assistant (Siri via App Intents, Gemini via AppFunctions), rank them, and produce an integration plan — with scaffolding templates for when you implement. Use when the user wants their app's functions to be callable by Siri / Apple Intelligence / Gemini, asks "which of my features could the AI trigger", mentions App Intents / AppFunctions / assistant integration, or wants to make an app agent-ready. Works on Swift (iOS) and Kotlin (Android) projects.
---

# AI Actions

Turn an app's existing features into actions the phone's AI assistant can discover and run — Siri/Apple Intelligence via **App Intents** (iOS), Gemini via **AppFunctions** (Android). The goal: the user says something in natural language, the assistant picks the right action from the app's catalog, runs it, and opens the app only for the details.

> **Scope.** This skill's verified core is the **analysis**: detect the platform, inventory candidate features, reconcile with any existing adoption, score them, and hand back a ranked shortlist + integration plan. The scaffolding templates (steps 4–6) are provided as a reference for when you implement — they encode the correct patterns but have not been machine-verified against a compiler, so treat generated code as a starting point to review, not finished output.

## Core principle

Registering an action is easy; getting the AI to *choose* it is the real work. Favor actions that are **narrow, clearly named, and hard for the AI to do on its own** (they touch the app's own data, domain rules, or account state). Skip anything a general model already does well (plain arithmetic, generic summarization) — the assistant has no reason to call your app for those.

## Workflow

Work through these steps in order. Don't scaffold before the inventory and the user's pick are done.

### 1. Detect the platform

Look for markers, don't assume:
- **iOS**: `*.xcodeproj`, `*.xcworkspace`, `Package.swift`, `*.swift` files, `Info.plist`.
- **Android**: `build.gradle(.kts)`, `settings.gradle`, `AndroidManifest.xml`, `*.kt` files.

If both exist (shared repo / KMP), handle each side separately and say so. Note the language, min OS target, and the DI / architecture in use (MVVM, use-cases, repositories — or none) — the scaffold has to fit it.

**Check the SDK/OS target as a hard prerequisite, not a footnote:**
- iOS → App Intents needs iOS 16+, but **report the routing-capability tier from the deployment target**, because it changes what the user actually gets:
  - **iOS 26/27+**: LLM-powered Siri freely reasons and routes any phrasing to your intent. Full experience.
  - **iOS 16–25**: intents exist but reach the user only via **predefined AppShortcut phrases + Spotlight + Shortcuts** — no free-form routing.
  This is a *runtime capability tier*, not a build gate (unlike Android's SDK requirement). Read `IPHONEOS_DEPLOYMENT_TARGET`. If it's below 26, tell the user their actions work on old OSes only through predefined phrases, and that the branch pattern (`if #available(iOS 26, *)`) lets one build serve both — free routing on new OSes, phrase/Spotlight fallback on old. AppShortcut phrases become **required for backward compat**, not optional.
- Android → AppFunctions needs **compileSdk/targetSdk 36 (Android 16)**. Read `build.gradle(.kts)`. If it's below 36, say so up front: bumping compileSdk affects the whole build and must happen *before* any AppFunctions code. Treat it as step zero, and flag the risk. (minSdk can stay lower; guard the functions with `@RequiresApi(36)`.)

### 2. Build a candidate inventory

Search the codebase for discrete, user-meaningful operations. Good hunting grounds:
- ViewModel / Presenter methods, use-cases / interactors, service or repository methods.
- Anything that takes a small input and produces a result a user would ask for by voice.
- Existing deep links / URL schemes / navigation routes (these become the "open for details" targets).

For each candidate capture: name, file:line, what it does, inputs, output, and whether it **reads** (safe) or **writes/mutates** (needs a confirmation step).

**Also check for an existing in-app AI scaffold** (Firebase AI / Gemini Live / an LLM function-calling tool, e.g. a voice-search ViewModel). If one exists: (a) it's the highest-signal template — its function schemas show exactly which operations are already shaped for AI use, so reuse them; (b) make the distinction explicit to the user — that scaffold runs *inside* the app, whereas App Intents / AppFunctions expose actions to the *system* assistant from outside. They're different layers; you're adding the second.

Read `references/discovery.md` for concrete search patterns and heuristics before this step.

### 2b. Reconcile with what already exists — don't propose duplicates

Apps are often **already partway into adoption.** Before scoring, search for existing declarations and diff them against your candidates:
- iOS: `import AppIntents`, `AppIntent`, `AppShortcutsProvider`, `AppEntity`, `OpensIntent`, widget/Spotlight/`AppShortcut` code (often in an `AppIntent/` folder).
- Android: `@AppFunction`, `AppFunctionSerializable`, existing AppFunctions modules.

For each existing action, judge its **richness**, not just its presence:
- **open-only** (`OpensIntent` / deep-link launch, no result) — the app opens but the assistant says nothing useful. Common and low-value.
- **value-returning** (returns a value + `ProvidesDialog` / a serializable result) — the assistant answers, then optionally opens the app. This is the target.

Then recommend **upgrades over rebuilds**: if `GoToFavoritesIntent` already opens the favorites screen, the move is to make it *return the list/count with a spoken answer*, not to create a second favorites intent. Mark each shortlist row as **new / upgrade (open-only → value-returning) / already-good**.

### 3. Score and shortlist

Rate each candidate on the rubric in `references/discovery.md` (user-meaningful · AI-can't-do-it-alone · clear I/O · single-purpose · read-vs-write). Prefer **value-returning** actions over open-only ones — a spoken/inline answer is the whole point; open-only is a fallback for when there's genuinely nothing to return. Present a ranked shortlist as a short table (with the new/upgrade/already-good marker from 2b) and let the user pick. Do **not** silently implement everything — surface the list first.

### 4. Scaffold the chosen actions

- iOS → follow `references/ios-app-intents.md`.
- Android → follow `references/android-appfunctions.md`.

One action = one intent/function. Reuse the app's existing logic — the intent/function is a thin wrapper that calls into the real code, never a reimplementation. Write clear titles and descriptions (that text is what the AI reads to route). For write actions, add a confirmation/`RequestConfirmation` step.

Two things break the "thin wrapper" assumption in practice — check for both:

- **Logic entangled with UI state.** If the operation you want lives in a ViewModel and depends on observed state (`@Observable` / `StateFlow` / `@Published` that a screen populates), you can't call it from an intent — there's no screen. **Extract the pure logic into a plain callable first** (a function/service method that takes its inputs as parameters and returns a value), have the ViewModel call that too, then wrap the plain callable. This is a small refactor, not a reimplementation — the app keeps one source of truth.
- **The action runs outside the app's lifecycle.** An App Intent / AppFunction executes without the app's normal launch, so it often can't grab a ViewModel, `@Environment` object, or already-configured container. It must **bootstrap its own dependencies**: ensure SDKs are initialized (e.g. `FirebaseApp.configure()`), construct the services directly or open its own data container (`ModelContainer` / Room instance / `AppContainer.get(context)`), and read user selection ("my team") straight from the stored source (UserDefaults / App Group / DataStore) — noting the exact key and encoding, since it may differ from the display value.

### 5. Wire "details → open the app"

Simple answers come back as a spoken/inline result; richer detail opens the app at the right screen via the existing deep link. Use `openAppWhenRun` / `OpensIntent` (iOS) or the return-value + launch pattern (Android). If no deep link exists for a screen, note it as a follow-up.

### 6. Verify

Build the target (XcodeBuildMCP for iOS sim, Gradle for Android). Confirm the intent/function is registered (shows in Shortcuts / the AppFunctions index). Don't claim it's assistant-callable if you only got it to compile — say exactly what you verified.

## Reality checks to tell the user

- **iOS**: App Intents is the *only* supported path now — SiriKit is deprecated as of iOS 27; an app with no App Intents is invisible to the new Siri.
- **Android**: AppFunctions works on Android 16 / One UI 8.5+ (e.g. Galaxy S26), but general-developer Gemini access is still a private preview — implementation is ready now, but live routing may require joining the trusted-tester program.

## Reference files

- `references/discovery.md` — search patterns + the scoring rubric.
- `references/ios-app-intents.md` — App Intents templates (read action, write action with confirmation, open-app, AppShortcut phrases).
- `references/android-appfunctions.md` — AppFunctions templates (annotation, schema, return + launch, manifest/index setup).
