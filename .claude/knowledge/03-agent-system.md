# OpenCode 에이전트 시스템 아키텍처

## 1. 에이전트 구조

### 핵심 타입 정의 (Agent.Info)

에이전트 시스템은 `/packages/opencode/src/agent/agent.ts`에서 정의된 `Agent.Info` Zod 스키마를 기반으로 구축됩니다:

```typescript
export const Info = z.object({
  name: z.string(),                    // 고유 식별자 (build, plan, general, explore 등)
  description: z.string().optional(),  // 사용자 대면 설명
  mode: z.enum(["subagent", "primary", "all"]),
  native: z.boolean().optional(),      // 내장 vs 커스텀 에이전트
  hidden: z.boolean().optional(),      // UI 자동완성에서 숨김
  topP: z.number().optional(),         // Nucleus 샘플링 파라미터
  temperature: z.number().optional(),  // 모델 창의성 제어
  color: z.string().optional(),        // UI용 Hex 색상
  permission: PermissionNext.Ruleset,  // 도구 접근 제어 규칙
  model: z.object({                    // 선택적 모델 오버라이드
    modelID: z.string(),
    providerID: z.string(),
  }).optional(),
  prompt: z.string().optional(),       // 커스텀 시스템 프롬프트
  options: z.record(z.string(), z.any()),  // 추가 LLM 옵션
  steps: z.number().int().positive().optional(),  // 최대 에이전틱 반복
})
```

### 주요 속성 설명

| 속성 | 설명 |
|------|------|
| **mode** | 가시성 및 호출 제어: `primary` (사용자 대면), `subagent` (특화), `all` (커스텀) |
| **native** | 내장 에이전트 (불변) vs 커스텀 (사용자 구성 가능) |
| **permission** | 도구 사용 권한을 강제하는 `PermissionNext.Ruleset` |
| **steps** | 무한 루프 방지를 위한 에이전틱 반복 제한 |

---

## 2. 에이전트 생명주기

### 초기화 단계

1. **설정 로딩** (`Agent.state()`):
   - 설정 파일에서 읽기 (예: `.opencode/agent/*`)
   - 기본 권한과 사용자 정의 권한 병합
   - `Record<string, Agent.Info>` 반환

2. **내장 에이전트**:
   - **build**: 기본 에이전트, question/plan_enter 권한 허용
   - **plan**: 기본 에이전트, plan_exit 권한으로 계획 처리
   - **general**: 서브에이전트, todo 제한이 있는 범용 실행기
   - **explore**: 서브에이전트, 파일/코드베이스 탐색 특화
   - **compaction**: 숨겨진 기본 에이전트, 컨텍스트 압축용
   - **title**: 숨겨진 기본 에이전트, 세션 제목 생성용
   - **summary**: 숨겨진 기본 에이전트, 세션 요약용

### 런타임 단계

1. **에이전트 선택** (`Agent.get(name)`, `Agent.defaultAgent()`):
   - 특정 에이전트 설정 검색
   - 기본으로 사용될 경우 서브에이전트가 아닌지 검증

2. **도구 해결** (`LLM.stream()` 내):
   - `ToolRegistry.tools()`를 통해 모든 사용 가능한 도구 가져오기
   - 에이전트 권한에 따라 도구 필터링
   - `Tool.init({ agent })`를 통해 도구 구현 해결

3. **모델 호출**:
   - 에이전트 옵션에서 모델 파라미터 구성
   - 에이전트 설정에서 temperature/topP 설정
   - 에이전트 옵션을 기본 프로바이더 옵션과 병합
   - Vercel AI SDK의 `streamText()`를 통해 텍스트 스트리밍

---

## 3. 주요 인터페이스 및 타입

### 도구 통합 (Tool.Info)

```typescript
export interface Info<Parameters extends z.ZodType, M extends Metadata> {
  id: string,
  init: (ctx?: InitContext) => Promise<{
    description: string,
    parameters: Parameters,
    execute(args: z.infer<Parameters>, ctx: Context): Promise<{
      title: string,
      metadata: M,
      output: string,
      attachments?: MessageV2.FilePart[]
    }>,
    formatValidationError?(error: z.ZodError): string
  }>
}
```

### 도구 실행 컨텍스트 (Tool.Context)

```typescript
export type Context<M extends Metadata> = {
  sessionID: string,
  messageID: string,
  agent: string,                    // 도구를 실행하는 에이전트
  abort: AbortSignal,
  callID?: string,
  extra?: Record<string, any>,
  metadata(input: { title?: string; metadata?: M }): void,
  ask(input: PermissionRequest): Promise<void>  // 사용자 권한 요청
}
```

### 권한 시스템 (PermissionNext)

```typescript
export const Rule = z.object({
  permission: z.string(),     // 도구 이름 (예: "read", "bash", "edit")
  pattern: z.string(),        // 패턴 매칭 (예: "*.env", "*")
  action: z.enum(["allow", "deny", "ask"])  // 기본 동작
})

export type Ruleset = Rule[]  // 에이전트는 권한 규칙 배열을 가짐
```

### LLM 스트림 입력 (LLM.StreamInput)

```typescript
export type StreamInput = {
  user: MessageV2.User,          // 사용자 요청 컨텍스트
  sessionID: string,
  model: Provider.Model,         // 사용할 모델
  agent: Agent.Info,             // 에이전트 설정
  system: string[],              // 추가 시스템 프롬프트
  abort: AbortSignal,
  messages: ModelMessage[],      // 채팅 히스토리
  small?: boolean,               // 작은 모델 변형 사용
  tools: Record<string, Tool>,   // 사용 가능한 도구
  retries?: number
}
```

---

## 4. 에이전트-도구 상호작용 흐름

### 도구 접근 제어

```typescript
async function resolveTools(input: Pick<StreamInput, "tools" | "agent" | "user">) {
  // 이 에이전트에 대해 비활성화된 도구 가져오기
  const disabled = PermissionNext.disabled(
    Object.keys(input.tools),
    input.agent.permission  // 에이전트의 권한 규칙셋
  )

  // 거부되거나 비활성화된 도구를 사용 가능한 세트에서 제거
  for (const tool of Object.keys(input.tools)) {
    if (input.user.tools?.[tool] === false || disabled.has(tool)) {
      delete input.tools[tool]
    }
  }
  return input.tools
}
```

### 권한 평가 (PermissionNext.evaluate())

- 규칙셋에 대해 권한 + 패턴 매칭
- 첫 번째 일치하는 규칙의 동작 반환 ("allow", "deny", "ask")
- 일치하지 않으면 기본 거부
- 와일드카드 패턴 지원

---

## 5. 네이티브 에이전트 개요

| 에이전트 | 모드 | 네이티브 | 목적 | 주요 권한 |
|---------|------|---------|------|----------|
| **build** | primary | yes | 일반 작업 실행 | question, plan_enter |
| **plan** | primary | yes | 다단계 계획 | question, plan_exit |
| **general** | subagent | yes | 다중 작업 병렬화 | todoread/todowrite 거부 |
| **explore** | subagent | yes | 코드베이스 탐색 | glob, grep, read, bash (만) |
| **compaction** | primary | yes | 컨텍스트 압축 | (모두 거부) |
| **title** | primary | yes | 세션 제목 지정 | (모두 거부) |
| **summary** | primary | yes | 세션 요약 | (모두 거부) |

---

## 6. 에이전트 설정 및 생성

### 설정 소스

1. **기본값** (agent.ts에 하드코딩)
2. **사용자 설정** (`.opencode/agent/` 또는 설정 파일에서)
3. **런타임 오버라이드** (CLI 인수에서)

### 병합 전략

```typescript
// 사용자 설정이 기본값을 오버라이드
item.temperature = value.temperature ?? item.temperature
item.topP = value.top_p ?? item.topP
item.prompt = value.prompt ?? item.prompt
item.permission = PermissionNext.merge(item.permission,
  PermissionNext.fromConfig(value.permission ?? {}))
```

### 에이전트 생성 (Agent.generate())

- Vercel AI SDK의 `generateObject()`를 사용하여 커스텀 에이전트 생성
- 새 에이전트 이름이 기존 에이전트와 충돌하지 않는지 검증
- 출력: `identifier`, `whenToUse`, `systemPrompt`을 포함한 JSON

---

## 7. 시스템 아키텍처 패턴

### 프롬프트 시스템

- **헤더 프롬프트** (`SystemPrompt.header()`):
  - 프로바이더별 헤더 (예: 호환성을 위한 Anthropic 스푸프)

- **에이전트 프롬프트** (`agent.prompt`):
  - 파일에 저장된 커스텀 시스템 프롬프트 (예: `/agent/prompt/explore.txt`)
  - 사용자 설정에서 오버라이드 가능

- **시스템 구성**:
  ```
  1. 프로바이더 헤더
  2. 에이전트 커스텀 프롬프트 (또는 프로바이더 기본값)
  3. 사용자 시스템 프롬프트
  4. 마지막 메시지 시스템 컨텍스트
  ```

### 모델 설정 병합 순서

1. 기본 프로바이더 옵션
2. 모델별 옵션
3. **에이전트 옵션** (최고 우선순위)
4. 변형 옵션 (해당되는 경우)

---

## 8. 주요 파일 참조

| 파일 | 목적 |
|------|------|
| `/agent/agent.ts` | 핵심 에이전트 정의, 상태 관리 |
| `/agent/generate.txt` | 새 에이전트 생성을 위한 프롬프트 |
| `/agent/prompt/explore.txt` | explore 에이전트용 시스템 프롬프트 |
| `/agent/prompt/compaction.txt` | 컨텍스트 압축 프롬프트 |
| `/agent/prompt/summary.txt` | 세션 요약 프롬프트 |
| `/agent/prompt/title.txt` | 세션 제목 생성 프롬프트 |
| `/session/llm.ts` | 도구 해결이 포함된 LLM 스트리밍 |
| `/tool/tool.ts` | 도구 인터페이스 및 정의 헬퍼 |
| `/tool/registry.ts` | 도구 등록 및 필터링 |
| `/tool/task.ts` | 서브에이전트 호출을 위한 Task 도구 |
| `/permission/next.ts` | 권한 규칙 평가 |

---

## 요약

에이전트 시스템은 **조합 가능하고, 권한 인식적인 작업 실행 프레임워크**입니다:

1. **에이전트**는 동작, 권한, LLM 파라미터를 지정하는 설정 프로필
2. **도구**는 권한 요구사항과 컨텍스트 인식 실행을 가진 기능
3. **권한**은 도구를 에이전트에 매칭하는 규칙 기반 접근 제어 시스템
4. **모드**는 계층적 에이전트 구성 가능 (기본 에이전트가 서브에이전트 조율)
5. **유연성**은 시스템 에이전트를 보호하면서 설정을 통한 커스텀 에이전트 허용
