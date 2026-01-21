# OpenCode 프로바이더 시스템 아키텍처

## 1. 프로바이더 구조 및 등록

### 핵심 아키텍처

프로바이더 시스템은 `packages/opencode/src/provider/`에 위치하며 모듈식, 레지스트리 기반 패턴을 따릅니다.

**파일 구조:**
- `provider.ts` - 핵심 프로바이더 레지스트리 및 SDK 초기화
- `auth.ts` - 인증 흐름 관리
- `models.ts` - models.dev API에서 모델 데이터 로딩
- `transform.ts` - 프로바이더별 변환
- `sdk/openai-compatible/` - 커스텀 OpenAI 호환 SDK 구현

### 프로바이더 등록 프로세스

**1단계: 모델 발견**
- 모델은 `https://models.dev/api.json`에서 가져와 로컬에 캐시
- 주기적 새로고침 (60분마다)
- 로컬 캐시 관리
- 임베디드 매크로 데이터로 폴백

**2단계: 프로바이더 로딩 (우선순위 순서)**
```
1. models.dev 데이터베이스에서 로드
2. 설정 파일 오버라이드와 병합
3. 환경 변수 적용
4. 인증 자격 증명 적용
5. 커스텀 로더 설정 적용
6. 활성화/비활성화 프로바이더 목록으로 필터링
7. deprecated/alpha 모델 제거 (플래그가 없으면)
8. 모델 레벨 블랙리스트/화이트리스트 적용
```

**3단계: SDK 초기화**
1. 모델의 npm 패키지 조회
2. 번들 프로바이더인지 확인
3. 필요시 패키지 설치 (BunProc를 통해)
4. 옵션 해시별로 SDK 인스턴스 캐시

---

## 2. 사용 가능한 내장 프로바이더 (30+)

### 번들 프로바이더 (직접 임포트)

| 프로바이더 ID | NPM 패키지 | 서비스 |
|--------------|-----------|--------|
| `amazon-bedrock` | `@ai-sdk/amazon-bedrock` | AWS Bedrock |
| `anthropic` | `@ai-sdk/anthropic` | Anthropic Claude |
| `azure` | `@ai-sdk/azure` | Azure OpenAI |
| `google` | `@ai-sdk/google` | Google Generative AI (Gemini) |
| `google-vertex` | `@ai-sdk/google-vertex` | Google Vertex AI |
| `google-vertex-anthropic` | `@ai-sdk/google-vertex/anthropic` | Vertex AI + Anthropic |
| `openai` | `@ai-sdk/openai` | OpenAI GPT |
| `openai-compatible` | `@ai-sdk/openai-compatible` | OpenAI API 호환 |
| `openrouter` | `@openrouter/ai-sdk-provider` | OpenRouter |
| `xai` | `@ai-sdk/xai` | xAI (Grok) |
| `mistral` | `@ai-sdk/mistral` | Mistral AI |
| `groq` | `@ai-sdk/groq` | Groq |
| `deepinfra` | `@ai-sdk/deepinfra` | DeepInfra |
| `cerebras` | `@ai-sdk/cerebras` | Cerebras |
| `cohere` | `@ai-sdk/cohere` | Cohere |
| `gateway` | `@ai-sdk/gateway` | Vercel AI Gateway |
| `togetherai` | `@ai-sdk/togetherai` | Together AI |
| `perplexity` | `@ai-sdk/perplexity` | Perplexity |
| `vercel` | `@ai-sdk/vercel` | Vercel AI |
| `gitlab` | `@gitlab/gitlab-ai-provider` | GitLab Duo |

### 특수 처리가 있는 커스텀 로더

| 프로바이더 ID | 설정 | 기능 |
|--------------|------|------|
| `anthropic` | 베타 헤더 | 확장된 사고, 인터리브 스트리밍 |
| `opencode` | 조건부 공개키 지원 | 내부 OpenCode API |
| `openai` | Responses API 지원 | 추론 모델용 `sdk.responses()` |
| `github-copilot` | 동적 모델 선택 | GPT-5는 Responses API 사용 |
| `azure` | 완료 URL 처리 | URL 기반 모델 요청 |
| `amazon-bedrock` | 복잡한 리전/자격증명 처리 | AWS 자격증명 체인 |
| `google-vertex` | 프로젝트/위치 설정 | GCP 환경 변수 |
| `gitlab` | API 키/OAuth 인증 | 기능 플래그, 에이전틱 채팅 |

---

## 3. 프로바이더 인증 흐름

### 인증 유형

**1. API 키 인증**
```typescript
await Auth.set(providerID, {
  type: "api",
  key: "api-key-here"
})
```

**2. OAuth 인증**
```typescript
const auth: Auth.Info = {
  type: "oauth",
  access: "access_token",
  refresh: "refresh_token",
  expires: timestamp,
  accountId?: "optional-account-id"
}
await Auth.set(providerID, info)
```

### 인증 소스 (우선순위 순서)

1. **설정 파일** (`opencode.json`)
2. **환경 변수** (예: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`)
3. **인증 저장소** (`~/.opencode/auth.json`)
4. **플러그인 인증 로더**

### 인증 흐름 함수

```typescript
// OAuth 흐름 시작
const auth = await ProviderAuth.authorize({
  providerID: "github-copilot",
  method: 0
})
// 반환: { url, method, instructions }

// OAuth 콜백 완료
await ProviderAuth.callback({
  providerID: "github-copilot",
  method: 0,
  code: "auth-code"
})

// API 키 직접 설정
await ProviderAuth.api({
  providerID: "openai",
  key: "sk-..."
})
```

---

## 4. 모델 해결 및 설정

### 모델 데이터 구조

```typescript
interface Model {
  id: string                           // 모델 ID (예: "gpt-5")
  providerID: string                   // 프로바이더 (예: "openai")
  api: {
    id: string                         // API 모델 ID
    url: string                        // 기본 URL
    npm: string                        // npm 패키지
  }
  name: string                         // 표시 이름
  family?: string                      // 모델 패밀리

  // 기능
  capabilities: {
    temperature: boolean
    reasoning: boolean                 // 확장된 사고 지원
    attachment: boolean
    toolcall: boolean
    input: { text, audio, image, video, pdf: boolean }
    output: { text, audio, image, video, pdf: boolean }
    interleaved: boolean | { field: string }
  }

  // 가격 (1M 토큰당)
  cost: {
    input: number
    output: number
    cache: { read: number, write: number }
  }

  // 컨텍스트/출력 제한 (토큰)
  limit: {
    context: number
    input?: number
    output: number
  }

  status: "alpha" | "beta" | "deprecated" | "active"
  options: Record<string, any>
  headers: Record<string, string>
  release_date: string
  variants?: Record<string, any>       // 추론 노력 변형
}
```

### 모델 해결 함수

```typescript
// 모든 프로바이더 가져오기
const providers = await Provider.list()

// 특정 모델 가져오기
const model = await Provider.getModel(providerID, modelID)

// 언어 모델로 해결
const languageModel = await Provider.getLanguage(model)

// 가장 가까운 매칭 가져오기
const closest = await Provider.closest(providerID, query)

// 빠른 작업을 위한 작은 모델 가져오기
const smallModel = await Provider.getSmallModel(providerID)

// 기본 모델 가져오기
const defaultModel = await Provider.defaultModel()
```

### 모델 필터링

**설정을 통해:**
```json
{
  "provider": {
    "openai": {
      "whitelist": ["gpt-5", "gpt-5-mini"],
      "blacklist": ["gpt-5-deprecated"]
    }
  }
}
```

**플래그를 통해:**
- `OPENCODE_ENABLE_EXPERIMENTAL_MODELS` - alpha 상태 모델 활성화
- `OPENCODE_DISABLE_MODELS_FETCH` - models.dev 새로고침 건너뜀

---

## 5. 커스텀 OpenAI 호환 SDK 구현

### 개요

OpenCode는 **Responses API**를 지원하는 GitHub Copilot용 커스텀 OpenAI 호환 프로바이더를 구현합니다.

**위치:** `packages/opencode/src/provider/sdk/openai-compatible/`

### 핵심 기능

**1. 이중 API 지원**
```typescript
interface OpenaiCompatibleProvider {
  (modelId: string): LanguageModelV2        // 기본: chat
  chat(modelId: string): LanguageModelV2    // 표준 chat completions
  responses(modelId: string): LanguageModelV2  // Responses API
  languageModel(modelId: string): LanguageModelV2
}
```

**2. Responses API 특화**
- 고급 추론 모델 (GPT-5, o-series, codex)
- 확장된 도구 지원 (file search, code interpreter, web search, image generation)
- 암호화된 추론 출력
- 스트리밍 지원

### 내장 도구 지원

```typescript
type OpenAIResponsesTool =
  | { type: "function"; name: string; parameters: JSONSchema7 }
  | { type: "web_search"; ... }
  | { type: "code_interpreter"; container?: "auto" | {...} }
  | { type: "file_search"; vector_store_ids: string[] }
  | { type: "image_generation"; size?: string; quality?: string }
  | { type: "local_shell" }
```

### 응답 스트리밍 아키텍처

**스트림 이벤트 유형:**
- `response.created` - 응답 초기화
- `response.output_item.added` - 새 출력 항목 시작
- `response.output_text.delta` - 텍스트 청크
- `response.function_call_arguments.delta` - 함수 인수
- `response.reasoning_summary_text.delta` - 추론 텍스트
- `response.completed` / `response.incomplete` - 응답 완료

---

## 6. 프로바이더 변환 파이프라인

### 메시지 변환

```typescript
ProviderTransform.message(
  msgs: ModelMessage[],
  model: Model,
  options: Record<string, unknown>
)
```

- 지원되지 않는 모달리티 제거
- 도구 호출 ID 정규화
- 프롬프트 캐싱 적용
- 인터리브 추론 처리

### 모델 변형 (추론 노력)

```typescript
ProviderTransform.variants(model): Record<string, any>
// 프로바이더별 추론 노력 설정 반환:
// - OpenAI: "none"|"minimal"|"low"|"medium"|"high"|"xhigh"
// - Anthropic: 사고 예산 (16k, 31.999k)
// - Google: thinkingLevel "low"|"high"
// - Bedrock: 노력이 포함된 reasoningConfig
```

### 온도 기본값

```typescript
{
  "qwen": 0.55,
  "claude": undefined,
  "gemini": 1.0,
  "glm-4.6": 1.0,
  "kimi-k2-thinking": 1.0,
  "kimi-k2": 0.6
}
```

---

## 요약

OpenCode 프로바이더 시스템은 다음을 제공하는 정교하고 확장 가능한 아키텍처입니다:

1. **30개 이상의 LLM 프로바이더를 동적으로 등록** - 통합 인터페이스
2. **인증 관리** - API 키, OAuth, 설정, 환경 변수를 통해
3. **모델 발견 처리** - models.dev에서 로컬 캐싱으로
4. **정교한 변환** - 크로스 프로바이더 호환성을 위해
5. **커스텀 OpenAI 호환 SDK** - Responses API 지원
6. **고급 기능 지원** - 확장된 사고, 추론, 도구 실행, 스트리밍
