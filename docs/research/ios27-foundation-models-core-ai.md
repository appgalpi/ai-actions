# iOS 27 분석: Foundation Models · Core AI — ai-actions에 얼마나 연관되나

> 2026-07 조사 노트. WWDC26(2026-06) 발표 기준이며, 이 스킬(ai-actions)의 관점 — **"어시스턴트가 앱의 기능을 부를 수 있게 노출한다"** — 에서 연관성을 평가한다. 스킬 문서에 반영할 결론은 맨 아래 요약 참고.

## 방향 구분이 먼저다

iOS 27의 AI 발표는 두 방향으로 나뉜다. 이 구분이 이 문서 전체의 판단 기준이다.

| 방향 | 프레임워크 | ai-actions와의 관계 |
|---|---|---|
| **앱 → 모델** (앱이 추론을 호출) | Foundation Models, Core AI | 간접 — "기존 in-app AI 스캐폴드" 감지 신호로만 쓰임 |
| **어시스턴트 → 앱** (Siri가 앱 기능을 호출) | App Intents (**App Schemas**, AppIntentsTesting) | **직접** — 스킬의 핵심 경로 자체가 바뀜 |

사용자가 물어본 세 가지(Third-Party Providers, Image Input, Core AI)는 전부 첫 번째 방향이다. 그러나 조사 과정에서 확인된 두 번째 방향의 변화(App Schemas)가 스킬에는 훨씬 크게 걸린다. 아래에 둘 다 정리한다.

## 1. Foundation Models — Third-Party Providers (`LanguageModel` 프로토콜)

**무엇인가.** iOS 27에서 Foundation Models 프레임워크가 공개 Swift 프로토콜 `LanguageModel`을 노출한다. 온디바이스 모델뿐 아니라 이 프로토콜을 구현한 어떤 모델도 같은 `LanguageModelSession` API 뒤에 꽂을 수 있다.

- Anthropic·Google이 각자 frontier 모델용 Swift 패키지를 배포 (`import AnthropicLanguageModel` → `LanguageModelSession(model: Anthropic.LatestModel())`). SPM으로 모델을 갈아끼워도 downstream 코드(세션·툴 루프·에러 처리)는 그대로.
- 오픈소스 구현체: `CoreAILanguageModel`(Neural Engine 로컬 모델), `MLXLanguageModel`(GPU 로컬 모델).
- PCC(Private Cloud Compute) 모델도 같은 표면: `PrivateCloudComputeLanguageModel()`, 32K 컨텍스트 + reasoning, 키·인증 불필요. 무료 티어는 누적 다운로드 200만 미만 앱.
- 부가 API: **Dynamic Profiles**(세션 하나에서 모델·instructions·tool을 상태에 따라 선언적으로 교체), 토큰 사용량 조회, **Evaluations 프레임워크**(프롬프트 품질 측정), 시스템 tool(OCRTool·BarcodeReaderTool·Spotlight 검색 tool = 로컬 RAG). 프레임워크 자체가 오픈소스화(Linux 포함).

**ai-actions 연관성: 간접이지만 유의미.** 이 스킬의 방향(시스템 어시스턴트에 액션 노출)과 반대 방향의 기능이다. 다만 두 가지 함의가 있다.

1. **in-app AI 스캐폴드 감지 신호가 크게 늘어난다.** SKILL.md step 2는 "기존 in-app AI 스캐폴드(Firebase AI / Gemini Live 등)"를 최고 신호 템플릿으로 취급하는데, iOS 쪽 등가물이 이제 명확한 마커 집합을 가진다: `import FoundationModels`, `LanguageModelSession`, `: Tool`, `@Generable`, `DynamicProfile`, 그리고 provider 패키지들(`AnthropicLanguageModel`, `GoogleLanguageModel`, `MLXLanguageModel`, `CoreAILanguageModel`). 온디바이스 한계 때문에 도입을 망설이던 앱들이 provider 교체 옵션 덕에 Foundation Models를 채택할 유인이 커졌으므로, 이 마커를 가진 앱을 만날 확률 자체가 올라간다. Foundation Models `Tool`로 감싸인 연산은 이미 "AI가 부를 수 있는 형태"로 다듬어진 것 — App Intents 후보 인벤토리의 1급 소스다.
2. **레이어 구분 설명이 더 중요해진다.** 같은 `Tool`이라도 Foundation Models의 tool은 *앱 안* 세션에서만 보이고, App Intents는 *시스템* 어시스턴트에 보인다. 순수 로직을 한 번 추출하면 양쪽 레이어에 동시에 재사용된다(아래 결론 참고).

## 2. Foundation Models — Image Input (멀티모달)

**무엇인가.** 온디바이스 시스템 모델이 vision을 지원한다. 프롬프트에 이미지를 첨부하면 된다 — 별도 파이프라인·세션 타입 없음:

```swift
let response = try await session.respond {
    "What animal is this?"
    Attachment(uiImage)
}
```

UIImage/NSImage/CGImage/CIImage/픽셀버퍼/파일 URL 지원, 임의 크기·비율(큰 이미지는 토큰·지연 증가). guided generation(`@Generable`)은 멀티모달 프롬프트에서도 그대로 동작. Vision 프레임워크 기반 OCR·바코드 tool을 모델이 직접 호출 가능.

**ai-actions 연관성: 낮음.** 전적으로 in-app 기능이다. 어시스턴트 → 앱 라우팅에서 이미지 입력을 받는 경로는 App Intents 쪽 별개 메커니즘(View Annotations / Visual Intelligence)이지 Foundation Models image input이 아니다. 스킬에 반영할 것은 인벤토리 단계의 각주 수준: "입력이 이미지인 연산은 in-app AI 후보이지 어시스턴트 액션 후보로는 여전히 약하다."

## 3. Core AI 프레임워크

**무엇인가.** 온디바이스 Apple Intelligence를 구동하던 추론 스택을 개발자에게 공개한 것. Core ML의 차세대 격으로, 생성형·대형 모델 워크로드에 초점.

- 전 Apple Silicon 플랫폼(iOS·iPadOS·macOS·tvOS·visionOS·watchOS), CPU/GPU/Neural Engine 활용.
- PyTorch 변환(`coreai-torch`) → `.aimodel`, AOT 컴파일, 디바이스별 specialization + 캐시(`AIModelCache`), 소형 모델부터 70B급 LLM까지.
- Swift API: `AIModel` → `InferenceFunction` → `NDArray` 입출력, KV 캐시 상태 관리.
- Xcode instruments·디버거·디버그 게이지 제공.

**ai-actions 연관성: 사실상 없음(마커 1개 제외).** 커스텀 모델을 *앱 안에서* 돌리는 인프라라 어시스턴트 노출과 무관하다. 유일한 접점은 `CoreAILanguageModel`로 Foundation Models 프레임워크에 꽂을 수 있다는 것 — 즉 위 1번의 감지 마커 목록에 한 항목을 보탤 뿐이다. 스킬 문서에 Core AI 자체를 다룰 이유는 없다.

## 4. (조사 중 확인) App Intents — App Schemas · AppIntentsTesting

사용자 질문 범위 밖이지만, iOS 27에서 **이 스킬에 가장 직접적인 변화**라 기록한다.

**Intent Schemas.** 시스템 정의 스키마(`AppSchema` 도메인: `.messages`, `.mail`, `.photos`, task management, communication 등)에 인텐트를 맞추면(`static let schema: Intent.Schema = .messages.sendMessage`) **사전 등록 문구 없이** Siri의 언어 모델이 자연어를 해당 인텐트로 라우팅한다. Siri 언어 이해가 개선되거나 새 언어·방언으로 확장돼도 코드 변경이 불필요. Xcode가 연관 스키마 누락을 빌드 타임에 검증(Fix-It 제공).

**Entity Schemas.** `@AppEntity(schema: .messages.contact)` + `IndexedEntity`(+`indexingKey`)로 앱 콘텐츠를 Spotlight **시맨틱 인덱스**에 기여 — 의미 기반 매칭으로 Siri가 앱 콘텐츠를 출처 표기와 함께 서페이스. 대규모·서버 기반·자주 바뀌는 데이터는 `EntityStringQuery`로 런타임 해석. **이것이 스킬이 말하는 "자연어 → 내부 ID 리졸버" 문제의 플랫폼 네이티브 해법이다** — 말한 "잠실"을 엔티티로 푸는 일을 커스텀 리졸버 대신 시스템이 맡는다.

**AppIntentsTesting.** Siri·Shortcuts·Spotlight 경로를 UI 자동화 없이 실제 시스템 경로로 검증하는 테스트 프레임워크. 스킬의 step 6(Verify)이 "Shortcuts에 뜨는지 확인"에서 "라우팅을 테스트로 검증"으로 올라설 수 있다.

**View Annotations API.** 화면 위 뷰를 엔티티에 매핑해 "이 사람한테 전화해" 같은 화면 참조 발화를 가능하게 함. (스킬에는 후순위 — 딥링크 섹션의 각주 감.)

## 결론 요약 — 스킬에 반영할 것

연관성 판정: **사용자가 지목한 세 항목 자체는 간접(1번) ~ 무관(2·3번)**이다. 그러나 같은 릴리스의 App Intents 변화(4번)가 스킬의 핵심 문서 세 곳을 직접 갱신할 근거가 된다. 이를 합쳐 적용안은:

1. **App Schemas를 라우팅 티어·스캐폴딩·루브릭에 반영** — SKILL.md step 1의 iOS 티어 표에 "iOS 27+: 스키마 채택 시 문구 없는 자연어 라우팅" 추가; step 4 / `references/ios-app-intents.md`에 "후보가 시스템 스키마 도메인에 맞으면 커스텀 인텐트보다 스키마 준수를 우선" 지침 + 템플릿; discovery 루브릭에 스키마 도메인 일치를 가점 요인으로.
2. **리졸버 섹션을 entity schema 중심으로 업그레이드** — `references/discovery.md`의 자연어→ID 리졸버 항목에: iOS 27+는 파라미터를 `IndexedEntity`(사전 인덱싱) 또는 `EntityStringQuery`(대규모·동적)로 모델링하는 것이 1차 해법, 커스텀 리졸버는 iOS 16–26 폴백.
3. **in-app AI 스캐폴드 마커 확장 + Verify에 AppIntentsTesting** — SKILL.md step 2의 감지 예시에 Foundation Models 마커 일체(`LanguageModelSession`·`Tool`·`@Generable`·`DynamicProfile`·provider 패키지·`CoreAILanguageModel`) 추가, "같은 순수 함수가 App Intent와 FM `Tool` 양쪽에 재사용된다" 명시; step 6에 `AppIntentsTesting` 기반 검증 추가.

## 출처

- [WWDC26 Apple Intelligence guide](https://developer.apple.com/wwdc26/guides/apple-intelligence/)
- [What's new in the Foundation Models framework (WWDC26 세션 241)](https://developer.apple.com/videos/play/wwdc2026/241/)
- [Build intelligent Siri experiences with App Schemas (WWDC26 세션 240)](https://developer.apple.com/videos/play/wwdc2026/240/)
- [Meet Core AI (WWDC26 세션 324)](https://developer.apple.com/videos/play/wwdc2026/324/)
- [Apple Newsroom — next generation of Apple Intelligence](https://www.apple.com/newsroom/2026/06/apple-unveils-next-generation-of-apple-intelligence-siri-ai-and-more/)
