# AI Actions

**English** · [한국어](#한국어)

A Claude Code skill that analyzes an iOS or Android project and tells you **which of its features are worth exposing to the phone's AI assistant** — Siri / Apple Intelligence via **App Intents**, Gemini via **AppFunctions** — then hands back a ranked shortlist and an integration plan.

The idea: on modern phones the OS assistant can call an app's actions directly. The user says something in plain language, the assistant picks the right action from the app's catalog, runs it, and opens the app only for the details. But an action the assistant could already answer on its own (plain math, generic text) never gets routed to your app. The hard part isn't registering actions — it's choosing the ones the assistant will actually pick. This skill is built around that judgment.

## What it does

It works **analysis-first**. Point it at a project and it:

1. **Detects the platform** — Swift/iOS or Kotlin/Android — plus the language, architecture, DI style, and the OS/SDK target (which decides what the assistant can actually do).
2. **Inventories candidate actions** — the discrete, user-meaningful operations buried in ViewModels, use-cases, services, or a flat API layer — recording inputs, outputs, and whether each reads or writes.
3. **Reconciles with existing adoption** — if the app already declares some App Intents / AppFunctions, it recommends upgrading the weak ones rather than proposing duplicates.
4. **Scores and shortlists** on a five-axis rubric (user-meaningful · AI-can't-do-it-alone · clear I/O · single-purpose · read-vs-write) and surfaces a ranked table for you to choose from.

It also flags the things that quietly block an integration: missing natural-language → internal-id resolvers (a spoken "Jamsil" has to become a stadium code before you can query), logic entangled with UI state that must be extracted first, no deep-link route for "open for details," and OS/SDK gates.

Reference templates for the actual App Intents / AppFunctions code are included for when you implement — see **Scope** below.

## Scope

The **analysis** is the verified core, exercised across multiple real iOS and Android codebases. The **scaffolding templates** (the code-generation half) encode the correct patterns but have not been machine-verified against a compiler — treat any generated code as a reviewed starting point, not finished output. Contributions that harden the templates are welcome.

## Platform notes

- **iOS** — App Intents is the only supported path (SiriKit is deprecated as of iOS 27). Free-form assistant routing arrives with the LLM-powered Siri on iOS 26/27; iOS 27 adds **App Schemas** (system-defined intent/entity shapes that route natural language with no predefined phrases), and the skill flags candidates that match a schema domain. On iOS 16–25 actions reach the user only through predefined phrases, Spotlight, and Shortcuts. The skill reports which tier your deployment target lands in.
- **Android** — AppFunctions needs Android 16 / API 36 (`compileSdk 36`). General-developer Gemini routing is still rolling out via a trusted-tester preview, so implementation is ready before live routing is universally available.

## Installation

Install into Claude Code with [`skills`](https://www.npmjs.com/package/skills):

```sh
npx skills install appgalpi/ai-actions -g
```

The `-g` flag installs globally (`~/.claude/skills/`); drop it to install into the current project (`.claude/skills/`).

<details>
<summary>Manual install</summary>

Copy the skill folder into your skills directory:

```sh
git clone https://github.com/appgalpi/ai-actions
cp -R ai-actions/skills/ai-actions ~/.claude/skills/
```

Place it under `~/.claude/skills/` (global) or `.claude/skills/` (per project).
</details>

Once installed, the skill runs when you ask Claude Code to find which of your app's features the AI assistant could trigger, or to make your app agent-ready. You can also invoke it directly with `/ai-actions`.

## License

MIT © 2026 앱갈피

---

# 한국어

**[English](#ai-actions)** · 한국어

iOS·Android 프로젝트를 분석해서, **어떤 기능을 스마트폰 AI 어시스턴트가 부를 수 있게 노출할 만한지** 짚어주는 Claude Code 스킬입니다 — iOS는 **App Intents**(Siri / Apple Intelligence), Android는 **AppFunctions**(Gemini). 결과로 우선순위 순위표와 연동 계획을 돌려줍니다.

발상은 이렇습니다. 요즘 폰은 OS 어시스턴트가 앱의 동작을 직접 부를 수 있습니다. 사용자가 평범하게 말하면, 어시스턴트가 앱의 도구 목록에서 알맞은 걸 골라 실행하고, 상세한 건 그때 앱을 엽니다. 그런데 어시스턴트가 혼자서도 답할 수 있는 동작(단순 계산·일반적인 글)은 절대 당신 앱으로 넘어오지 않아요. 어려운 건 동작을 등록하는 게 아니라, **어시스턴트가 실제로 골라줄 동작을 고르는 판단**입니다. 이 스킬은 그 판단을 중심으로 짜였습니다.

## 무엇을 하나

**분석 우선**으로 동작합니다. 프로젝트를 가리키면:

1. **플랫폼 감지** — Swift/iOS인지 Kotlin/Android인지, 언어·아키텍처·DI 방식, 그리고 OS/SDK 타깃(어시스턴트가 실제로 뭘 할 수 있는지 결정)을 파악합니다.
2. **후보 동작 인벤토리** — ViewModel·유스케이스·서비스나 평평한 API 층에 묻혀 있는, 사용자가 말로 시킬 만한 개별 동작들을 입력·출력·읽기/쓰기까지 정리합니다.
3. **기존 도입분과 대조** — 앱이 이미 App Intents / AppFunctions를 일부 선언했다면, 중복 생성 대신 약한 것을 업그레이드하도록 제안합니다.
4. **점수화·순위표** — 5축 루브릭(사용자 의미성 · AI가 혼자 못함 · 입출력 명확성 · 단일 목적 · 읽기/쓰기)으로 매겨 고를 수 있는 표로 보여줍니다.

또한 연동을 조용히 막는 것들도 짚어줍니다: 자연어 → 내부 id 변환기 부재(말로 한 "잠실"을 구장 코드로 바꿔야 조회 가능), UI 상태에 얽혀 먼저 떼어내야 하는 로직, "상세 열기"용 딥링크 부재, OS/SDK 요건 등.

실제 App Intents / AppFunctions 코드용 레퍼런스 템플릿도 포함돼 있습니다(아래 **범위** 참고).

## 범위

**분석**이 검증된 핵심으로, 여러 실제 iOS·Android 코드베이스에서 돌려봤습니다. **스캐폴딩 템플릿**(코드 생성 부분)은 올바른 패턴을 담았지만 컴파일러로 기계 검증한 것은 아닙니다 — 생성된 코드는 완성물이 아니라 검토용 출발점으로 다뤄주세요. 템플릿을 단단하게 만드는 기여를 환영합니다.

## 플랫폼 참고

- **iOS** — App Intents가 유일한 경로입니다(SiriKit은 iOS 27부터 폐기). 자유 문장 라우팅은 iOS 26/27의 LLM 기반 Siri부터 되고, iOS 27에는 **App Schemas**(문구 등록 없이 자연어를 라우팅하는 시스템 정의 인텐트/엔티티 양식)가 추가돼 스킬이 스키마 도메인에 맞는 후보를 표시해줍니다. iOS 16–25에서는 미리 등록한 문구·Spotlight·단축어로만 도달합니다. 스킬이 당신 배포 타깃이 어느 등급인지 알려줍니다.
- **Android** — AppFunctions는 Android 16 / API 36(`compileSdk 36`)이 필요합니다. 일반 개발자용 Gemini 라우팅은 아직 신뢰 테스터 프리뷰로 확대 중이라, 구현은 미리 해둘 수 있어도 라이브 라우팅은 점진적으로 열립니다.

## 설치

[`skills`](https://www.npmjs.com/package/skills)로 Claude Code에 설치합니다:

```sh
npx skills install appgalpi/ai-actions -g
```

`-g` 플래그는 전역(`~/.claude/skills/`)에 설치합니다. 빼면 현재 프로젝트(`.claude/skills/`)에 설치됩니다.

<details>
<summary>수동 설치</summary>

스킬 폴더를 스킬 디렉터리로 복사하세요:

```sh
git clone https://github.com/appgalpi/ai-actions
cp -R ai-actions/skills/ai-actions ~/.claude/skills/
```

`~/.claude/skills/`(전역) 또는 `.claude/skills/`(프로젝트별)에 두면 됩니다.
</details>

설치하면, 앱의 어떤 기능을 AI 어시스턴트가 부를 수 있을지 찾아달라거나 앱을 에이전트 대응으로 만들어달라고 할 때 자동으로 실행됩니다. `/ai-actions`로 직접 호출할 수도 있습니다.

## 라이선스

MIT © 2026 앱갈피
