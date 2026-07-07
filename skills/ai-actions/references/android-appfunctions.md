# Android — AppFunctions templates

Target Android 16 / API 36 (`compileSdk = 36`, `targetSdk = 36`). Uses the `androidx.appfunctions` Jetpack library plus the platform index. Gemini discovers and calls annotated functions locally — Google calls it "on-device MCP".

General-developer Gemini routing is still a **private preview** (trusted testers). The code below is ready to ship; live assistant routing may need program enrollment. Tell the user this.

The function is a **thin wrapper** — call the existing use-case/repository (or the global API object if there's no DI), don't reimplement.

## Return real data, don't just open the app

A function *can* return only a deep link that launches the app — but that wastes the assistant. Prefer returning the **actual answer** as a serializable result the assistant can speak/show, and include a deep link *in addition* for "see full detail." Open-only is the fallback for cases with genuinely nothing to answer.

If the app already exposes `@AppFunction`s, **upgrade the thin/open-only ones to return real data** rather than adding parallel functions for the same operation.

## 1. Dependencies

```kotlin
// build.gradle.kts (module)
dependencies {
    implementation("androidx.appfunctions:appfunctions:<latest>")
    ksp("androidx.appfunctions:appfunctions-compiler:<latest>")
}
```

Check for the current version before writing — the artifact is moving fast. `compileSdk`/`targetSdk` must be 36+.

## 2. A read function

```kotlin
import androidx.appfunctions.AppFunction
import androidx.appfunctions.AppFunctionContext

class FinanceFunctions(
    private val accountRepo: AccountRepository,   // existing app code
) {
    @AppFunction(
        // Description is what Gemini reads to route — be specific.
        description = "지정한 계좌의 현재 잔액을 조회한다."
    )
    suspend fun checkBalance(
        appFunctionContext: AppFunctionContext,
        accountName: String,
    ): BalanceResult {
        val balance = accountRepo.balanceByName(accountName)
        return BalanceResult(accountName = accountName, amount = balance)
    }
}

// Return types are serializable data holders.
@AppFunctionSerializable
data class BalanceResult(
    val accountName: String,
    val amount: Long,
)
```

## 3. A write function (confirm in-app)

AppFunctions has no built-in confirmation UI like iOS. For mutations, either return a "pending" result that the assistant surfaces for confirmation, or perform the write and open the app so the user sees and can undo it. Keep destructive actions behind an explicit user-visible step.

```kotlin
@AppFunction(description = "새 지출 내역을 추가한다.")
suspend fun addExpense(
    appFunctionContext: AppFunctionContext,
    amount: Long,
    memo: String,
): AddExpenseResult {
    val id = ledgerRepo.add(amount, memo)
    return AddExpenseResult(id = id, confirmationDeepLink = "myapp://ledger/$id")
}
```

## 4. Return + open the app for details

Put a deep link in the return value; the assistant/app can launch it for the full screen.

```kotlin
@AppFunctionSerializable
data class ReportResult(
    val summary: String,
    val deepLink: String,   // e.g. "myapp://report/2026-07"
)
```

Make sure the target deep link is declared in `AndroidManifest.xml`:

```xml
<activity android:name=".MainActivity" android:exported="true">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" />
    </intent-filter>
</activity>
```

## 5. Discovery / registration

The KSP compiler generates the function metadata that the platform indexes at install time — no manual manifest entry for the functions themselves beyond the library setup. Verify with the AppFunctions inventory/index (via `AppFunctionManager` in a debug build) that each function is registered and its schema looks right.

## Checklist

- [ ] `compileSdk` / `targetSdk` = 36+, library + KSP compiler added.
- [ ] Function calls existing logic, no reimplementation.
- [ ] `@AppFunction(description=...)` is specific (Gemini reads it to route).
- [ ] Return types are `@AppFunctionSerializable` data classes.
- [ ] Mutations are confirmed or open the app; no silent destructive calls.
- [ ] Deep-link scheme declared in the manifest for "open for details".
- [ ] Builds; function shows in the AppFunctions index.
- [ ] User told about the trusted-tester preview status if they want live Gemini routing.
