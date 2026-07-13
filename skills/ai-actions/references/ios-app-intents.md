# iOS — App Intents templates

Target iOS 16+ (App Intents). iOS 26/27 gets the LLM-routed Siri, so free-form phrasing works — but keep titles/descriptions clean regardless. SiriKit is deprecated as of iOS 27; App Intents is the only path.

The intent is a **thin wrapper** over existing app code. Never reimplement logic inside `perform()` — call the real service/use-case.

## 0. App Schemas first (iOS 27+): check for a system schema before writing a custom intent

iOS 27 adds **App Schemas** — system-defined intent and entity shapes grouped into domains (`.messages`, `.mail`, `.photos`, task management, …). An intent that conforms to a schema gets routed from **any natural phrasing with zero predefined phrases**, and inherits future Siri language improvements without code changes. So for every shortlisted action, check the schema catalog first:

- **A schema matches** → declare conformance and shape parameters to the schema's slots. The wrapper body is the same thin call into existing logic:

```swift
struct SendMessageIntent: AppIntent {
    static let schema: Intent.Schema = .messages.sendMessage

    @Parameter var recipient: ContactEntity
    @Parameter var messageContent: String

    func perform() async throws -> some IntentResult {
        let sent = try await chatService.send(to: recipient.id, text: messageContent)
        return .result(value: sent.entity)
    }
}
```

- **No schema matches** (the usual case for domain-specific actions) → write a custom intent per the templates below. Nothing is lost; schemas are a routing shortcut, not a gate.

Two cautions: Xcode validates schema adoption at build time and will demand **related schemas** (adopting `sendMessage` pulls in `draftMessage` — expect Fix-Its, and budget for the siblings), and schema conformance requires iOS 27, so on lower deployment targets keep the custom intent + AppShortcut phrases as the shipping path.

## Entity schemas + semantic index (iOS 27+): let the system resolve spoken names

The old problem — "the user *says* a name, the API wants an *id*" — gets a platform answer. Conform the entity to a schema and index it, and Siri resolves the spoken value for you:

```swift
@AppEntity(schema: .messages.contact)
struct ContactEntity: IndexedEntity {
    @Property(indexingKey: \.textContent)
    var name: String
    // indexed into the system semantic index → meaning-based matching
}
```

For data too large, server-backed, or fast-changing to index ahead of time, implement **`EntityStringQuery`** instead — a runtime string → entities lookup that wraps whatever search the app already has. Prefer one of these two over a hand-rolled resolver on iOS 27+; keep the custom resolver only for older targets.

## Prefer value-returning intents; open-only is the fallback

Two shapes exist, and the difference is the whole point:
- **value-returning** — `perform()` returns `ReturnsValue` + `ProvidesDialog`, so the assistant *speaks/shows the answer* and can then open the app. This is what you want.
- **open-only** — `OpensIntent` / `openAppWhenRun` with no returned value: the app just launches. Use only when there's genuinely nothing to answer (e.g. "open the settings screen").

If the app already has open-only intents (e.g. one that opens the favorites screen), **upgrade them to return a value** rather than adding parallel intents. Template 1 below is the value-returning shape; template 3 is open-only.

## Backward compatibility: free routing on new OS, phrases on old

Free-form assistant routing is iOS 26+. On iOS 16–25 the intent is reachable only through predefined **AppShortcut phrases**, Spotlight, and the Shortcuts app — so those phrases (§5) are **required for backward compat**, not optional. When behavior must differ by OS, branch explicitly:

```swift
if #available(iOS 26, *) {
    // richer conversational result — the LLM Siri handles free phrasing
} else {
    // rely on the AppShortcut phrases + a simple dialog/deep-link
}
```

Keep one build serving both tiers; don't drop the phrases just because you target the new OS too.

## 1. Read action (returns a value + spoken answer)

```swift
import AppIntents

struct CheckBalanceIntent: AppIntent {
    static let title: LocalizedStringResource = "잔액 확인"
    // Description text is read by the assistant for routing — be specific.
    static let description = IntentDescription("현재 계좌 잔액을 조회합니다.")

    @Parameter(title: "계좌")
    var account: AccountEntity

    @Dependency
    var accountService: AccountService   // existing app service

    func perform() async throws -> some IntentResult & ReturnsValue<Double> & ProvidesDialog {
        let balance = try await accountService.balance(for: account.id)
        return .result(
            value: balance,
            dialog: "\(account.name) 잔액은 \(balance.formatted(.currency(code: "KRW")))입니다."
        )
    }
}
```

Register the dependency once at launch: `AppDependencyManager.shared.add { AccountService() }`.

**`@Dependency` is optional — match the app's wiring.** If the app has no DI and uses `.shared` singletons (common), skip `@Dependency` entirely and call the singleton directly inside `perform()`:

```swift
func perform() async throws -> some IntentResult & ReturnsValue<Double> & ProvidesDialog {
    let balance = try await AccountService.shared.balance(for: account.id)   // no DI
    return .result(value: balance, dialog: "…")
}
```

Don't introduce `AppDependencyManager` into an app that doesn't otherwise use it.

## 2. Write / mutating action (with confirmation)

Anything that changes state should confirm first.

```swift
struct AddExpenseIntent: AppIntent {
    static let title: LocalizedStringResource = "지출 기록"
    static let description = IntentDescription("새 지출 내역을 추가합니다.")

    @Parameter(title: "금액") var amount: Double
    @Parameter(title: "항목") var memo: String

    @Dependency var ledger: LedgerService

    func perform() async throws -> some IntentResult & ProvidesDialog {
        try await requestConfirmation(
            result: .result(dialog: "\(memo)에 \(amount.formatted())원을 기록할까요?")
        )
        try await ledger.add(amount: amount, memo: memo)
        return .result(dialog: "기록했어요.")
    }
}
```

## 3. Open the app for details

Return-and-open, so a rich screen shows after the quick answer.

```swift
struct OpenReportIntent: AppIntent {
    static let title: LocalizedStringResource = "리포트 열기"
    static let openAppWhenRun = true

    @Parameter(title: "월") var month: Int

    func perform() async throws -> some IntentResult & OpensIntent {
        // Route via the app's existing deep link / navigation.
        return .result(opensIntent: ShowReportDeepLink(month: month))
    }
}
```

If the app already handles a URL scheme, an alternative is to open that URL from `perform()` — reuse whatever navigation exists rather than inventing new routing.

## 4. Entities (for typed parameters)

When a parameter is a domain object (an account, a document), model it as an `AppEntity` with an `EntityQuery` so Siri/Shortcuts can resolve it by name. Only add this when a parameter genuinely refers to app data. (On iOS 27+, also consider schema conformance + `IndexedEntity` / `EntityStringQuery` — see §0 above — which upgrades resolution from exact-name to semantic matching.)

```swift
struct AccountEntity: AppEntity {
    var id: String
    var name: String
    static var typeDisplayRepresentation: TypeDisplayRepresentation = "계좌"
    var displayRepresentation: DisplayRepresentation { .init(title: "\(name)") }
    static var defaultQuery = AccountQuery()
}
```

## 5. AppShortcuts — give the assistant example phrases

Predefined phrases help routing (and are required for zero-setup Siri on pre-27). Include the app name token.

```swift
struct AppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: CheckBalanceIntent(),
            phrases: ["\(.applicationName)에서 잔액 확인", "내 잔액 얼마야 \(.applicationName)"],
            shortTitle: "잔액 확인",
            systemImageName: "wonsign.circle"
        )
    }
}
```

## Checklist

- [ ] (iOS 27+) Checked the App Schema catalog; schema-matching actions declare `static let schema:` conformance.
- [ ] Intent calls existing logic, no reimplementation.
- [ ] `title` + `description` are specific (assistant reads them to route).
- [ ] Write actions have `requestConfirmation`.
- [ ] Dependencies registered in `AppDependencyManager`.
- [ ] AppShortcut phrases include `\(.applicationName)`.
- [ ] Deep link exists for any "open for details" target.
- [ ] Entity parameters resolvable by spoken name (`EntityQuery`; iOS 27+: `IndexedEntity` / `EntityStringQuery`).
- [ ] Builds; intent appears in the Shortcuts app. (iOS 27+: routing verified with an `AppIntentsTesting` test, not just presence in Shortcuts.)
