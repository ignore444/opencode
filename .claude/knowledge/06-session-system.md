# OpenCode 세션 관리 시스템 아키텍처

## 1. 세션 생명주기

### 생성 (Session.createNext)

**세션 생성 흐름:**
- 고유한 내림차순 식별자와 slug로 생성
- 각 세션 포함:
  - `id`: 세션 식별자 (내림차순 형식)
  - `slug`: URL 친화적 slug
  - `projectID`: 프로젝트 식별자
  - `directory`: 작업 디렉토리
  - `parentID` (선택): 자식 세션용 (포크)
  - `title`: 제공되지 않으면 자동 생성
  - `permission`: 도구 접근 제어를 위한 선택적 규칙셋
  - `time`: 생성/업데이트/압축/아카이브 타임스탬프

**자동 공유:**
```typescript
if (!result.parentID && (Flag.OPENCODE_AUTO_SHARE || cfg.share === "auto"))
  share(result.id)
```

### 세션 포킹 (Session.fork)

- 기존 세션의 메시지 히스토리에서 새 세션 생성
- 메시지 및 파트 ID를 새 고유 식별자에 매핑
- 선택적으로 특정 메시지 ID까지의 메시지 포함
- 어시스턴트-사용자 메시지 간 부모-자식 관계 유지

### 세션 생명주기 상태

| 상태 | 설명 |
|------|------|
| **Idle** | 활성 처리가 없는 세션 |
| **Busy** | LLM 스트리밍을 통해 활발히 처리 중인 세션 |
| **Retry** | 일시적 오류 발생 및 재시도 중 (시도 횟수 및 다음 재시도 시간 포함) |

---

## 2. 메시지 처리 (V1 vs V2)

현재 표준으로 **MessageV2**를 사용합니다. V1은 레거시입니다.

### MessageV2 구조

**메시지 유형 (구별된 유니온):**

1. **User 메시지** (`MessageV2.User`)
   - `role: "user"`
   - `agent`: 이 메시지를 처리할 에이전트 이름
   - `model`: 프로바이더/모델 선택
   - `system` (선택): 이 메시지에 대한 추가 시스템 프롬프트
   - `tools` (선택): 도구 활성화/비활성화 오버라이드
   - `variant` (선택): 모델 변형 선택
   - `summary` (선택): 변경 사항의 제목 및 diff

2. **Assistant 메시지** (`MessageV2.Assistant`)
   - `role: "assistant"`
   - `parentID`: 부모 사용자 메시지 참조
   - `modelID`, `providerID`: 사용된 모델
   - `agent`: 생성한 에이전트
   - `path`: 작업 디렉토리 컨텍스트 (cwd, root)
   - `cost`: 달러로 누적된 비용
   - `tokens`: 토큰 사용량 추적 (입력, 출력, 추론, 캐시)
   - `error` (선택): 메시지 생성 실패 시
   - `finish` (선택): LLM의 종료 이유
   - `summary` (선택): 컨텍스트 요약으로 표시

### MessageV2.Part 유형 (다중 파트 메시지)

| 파트 유형 | 설명 |
|----------|------|
| **TextPart** | 일반 텍스트 응답 |
| **ReasoningPart** | 확장된 사고 출력 |
| **ToolPart** | 상태 머신을 가진 도구 호출 |
| **FilePart** | 파일 첨부 |
| **StepStartPart** | LLM 단계 시작 경계 |
| **StepFinishPart** | LLM 단계 종료 경계 |
| **SnapshotPart** | 완전한 파일시스템 상태 해시 |
| **PatchPart** | 변경된 파일을 보여주는 파일 diff |
| **CompactionPart** | 압축 지점 표시 (컨텍스트 요약) |
| **SubtaskPart** | 병렬 실행을 위한 서브태스크 정의 |
| **RetryPart** | 재시도 시도 추적 |
| **AgentPart** | 에이전트 호출 참조 |

### 도구 상태 전환

```
Pending → Running → Completed/Error
```

- **Pending**: 도구 입력 처리 중
- **Running**: 도구 실행 시작 (time.start 포함)
- **Completed**: 도구가 출력 반환 (첨부 파일 포함)
- **Error**: 도구가 오류 메시지와 함께 실패

---

## 3. LLM 스트리밍 통합

위치: `packages/opencode/src/session/llm.ts`

### LLM.stream() 함수

**입력 설정:**
```typescript
{
  user: MessageV2.User,
  sessionID: string,
  model: Provider.Model,
  agent: Agent.Info,
  system: string[],
  abort: AbortSignal,
  messages: ModelMessage[],
  small?: boolean,      // 더 작은 모델 사용
  tools: Record<string, Tool>,
  retries?: number
}
```

### 시스템 프롬프트 아키텍처

캐싱 효율성을 위한 2부분 시스템 프롬프트 구조:
1. **Header**: 프로바이더별 초기 지시
2. **Body**: 에이전트 프롬프트 + 프로바이더 프롬프트 + 커스텀 시스템 메시지

**프로바이더별 프롬프트:**
- Claude/Anthropic: `PROMPT_ANTHROPIC`
- GPT-4/5/O-series: `PROMPT_BEAST`
- Gemini: `PROMPT_GEMINI`
- 기타 (Qwen 등): `PROMPT_ANTHROPIC_WITHOUT_TODO`

### 모델 옵션 해결

계층적 옵션 병합:
```
기본 옵션
  ↓ (병합)
모델 옵션
  ↓ (병합)
에이전트 옵션
  ↓ (병합)
변형 옵션 (해당되는 경우)
```

---

## 4. 세션 프로세서 워크플로

위치: `packages/opencode/src/session/processor.ts`

`SessionProcessor`는 LLM 스트리밍 출력 처리를 위한 핵심 오케스트레이터입니다.

### 처리 루프

**스트림 이벤트 처리:**

1. **스트림 시작** (`start`) - 상태를 "busy"로 설정

2. **추론 블록** (`reasoning-start/delta/end`)
   - ReasoningPart 생성
   - 델타를 통해 텍스트 누적
   - 타이밍 및 메타데이터 기록

3. **도구 호출** (`tool-input-start/delta/end`, `tool-call`)
   - pending 상태로 ToolPart 생성
   - 실행 시작 시 running으로 전환
   - 도구 호출 ID 매핑 추적

4. **Doom 루프 감지** (DOOM_LOOP_THRESHOLD = 3)
   - 동일한 입력으로 같은 도구가 3번 호출될 때 감지
   - PermissionNext.ask()를 통해 사용자 권한 요청
   - 무한 도구 재귀 방지

5. **도구 실행** (`tool-result`, `tool-error`)
   - 출력/오류로 ToolPart 업데이트
   - 실행 시간 기록
   - 첨부 파일 저장 (도구가 반환한 파일)

6. **단계 마커** (`start-step`, `finish-step`)
   - StepStart: 초기 파일시스템 스냅샷 캡처
   - StepFinish:
     - 최종 스냅샷 캡처
     - 토큰 사용량 및 비용 계산
     - SessionSummary 트리거
     - 압축 필요 여부 확인

7. **텍스트 생성** (`text-start/delta/end`)
   - 델타를 통해 텍스트 콘텐츠 스트리밍
   - 공백 트림
   - 플러그인 텍스트 변환 적용
   - 타이밍 기록

### 오류 처리 및 재시도 로직

```
오류 발생
  ↓
fromError()가 MessageV2 오류 유형으로 변환
  ↓
retryable()이 재시도 가능 여부 확인
  ↓
YES: SessionRetry.delay() → sleep → 재시도 루프
NO: 어시스턴트 메시지에 오류 저장, 루프 종료
```

**재시도 가능한 오류:**
- `isRetryable: true`인 APIError
- Rate limit 오류
- 프로바이더 과부하 오류
- 서버 오류

**재시도 불가:**
- 인증 오류
- 출력 길이 초과
- 중단된 오류

### 처리 결과 상태

| 결과 | 설명 |
|------|------|
| `"continue"` | 정상 완료, 다음 사용자 메시지를 위해 루프 준비 |
| `"stop"` | 오류 발생 또는 권한 거부 |
| `"compact"` | 토큰 제한 초과, 압축 필요 |

---

## 5. 압축 및 요약

### 압축 (SessionCompaction)

위치: `packages/opencode/src/session/compaction.ts`

**트리거 조건:**
```typescript
isOverflow() 확인:
  count (입력 + cache_read + 출력 토큰) > 사용 가능한 컨텍스트 창
```

**압축 프로세스:**

1. **자동 압축:**
   - 토큰이 컨텍스트를 초과하면 step-finish 후
   - 새 어시스턴트 메시지 생성 (`summary: true`)
   - 프롬프트: "대화를 계속하기 위한 상세한 프롬프트 제공..."

2. **수동 압축:**
   - 사용자가 명시적으로 압축 요청 가능
   - CompactionPart가 있는 사용자 메시지 생성

### 프루닝 (SessionCompaction.prune())

**전략:**
1. 메시지를 역순으로 탐색
2. 마지막 2개의 대화 턴 건너뜀 (최근 컨텍스트 보호)
3. `PRUNE_PROTECT` (40k 토큰)부터 도구 호출 수집
4. 총 도구 출력 > `PRUNE_MINIMUM` (20k)인 경우:
   - 해당 도구 출력을 `compacted: Date.now()`로 표시
   - 도구 출력을 "[Old tool result content cleared]"로 교체

**보호되는 도구:**
- `skill` 도구 출력은 절대 프루닝되지 않음 (참조용으로 유지)

### 요약 (SessionSummary)

위치: `packages/opencode/src/session/summary.ts`

**세션 요약:**
- 세션 전체의 모든 패치 집계
- 추가/삭제/수정된 파일 수 계산
- 저장소에 전체 diff 저장

**메시지 요약 (비동기):**
1. **Diff 계산:**
   - 첫 번째 step-start 스냅샷 찾기 (초기 상태)
   - 마지막 step-finish 스냅샷 찾기 (최종 상태)
   - 둘 사이의 diff 계산

2. **제목 생성:**
   - "title" 에이전트 사용 (작은 모델)
   - 첫 번째 사용자 메시지 텍스트를 요약하도록 프롬프트
   - 사용자 메시지 요약에 제목 저장

---

## 6. 세션 내 TODO 관리

위치: `packages/opencode/src/session/todo.ts`

### Todo 스키마

```typescript
{
  id: string,           // 고유 식별자
  content: string,      // 간략한 설명
  status: string,       // pending|in_progress|completed|cancelled
  priority: string      // high|medium|low
}
```

### Todo 작업

**저장소:**
- Todo는 키 `["todo", sessionID]`에 세션별로 저장
- 찾지 못하면 빈 배열 반환

**업데이트:**
```typescript
Todo.update({
  sessionID: string,
  todos: Info[]
})
```
- todos 목록과 함께 `Todo.Event.Updated` 게시

---

## 7. 추가 아키텍처 컴포넌트

### 상태 추적 (SessionStatus)

- 인메모리 상태 (지속되지 않음)
- 세 가지 상태: idle, busy, retry
- `SessionStatus.Event.Status`를 통해 게시

### 되돌리기 메커니즘 (SessionRevert)

- 변경 전 원본 스냅샷 저장
- 파일시스템을 이전 상태로 되돌릴 수 있음
- 되돌린 메시지/파트를 깔끔하게 제거
- diff 일관성 유지

### 재시도 로직 (SessionRetry)

```
초기 지연: 2초
백오프 팩터: 2배
최대 지연: 30초 (retry-after 헤더 없이)
최대 지연: 2^31-1ms (retry-after 헤더 있음)
```

HTTP `retry-after` 및 `retry-after-ms` 헤더 준수.

---

## 8. 시스템 프롬프트 및 설정

위치: `packages/opencode/src/session/system.ts`

**3단계 프롬프트 시스템:**

1. **커스텀 지시** (최고 우선순위)
   - 로컬 파일: CLAUDE.md, AGENTS.md, CONTEXT.md
   - 전역 파일: ~/.claude/CLAUDE.md, config/AGENTS.md
   - 설정의 커스텀 URL

2. **프로바이더 프롬프트** (중간 우선순위)
   - 모델별 동작 가이드
   - LLM에 따라 다름 (Claude, GPT, Gemini 등)

3. **에이전트 프롬프트** (최저 우선순위)
   - 에이전트별 지시
   - 프로바이더 프롬프트 오버라이드 가능

**환경 컨텍스트:**
```
<env>
  Working directory: ...
  Is directory a git repo: yes|no
  Platform: linux|darwin|win32
  Today's date: ...
</env>
```

---

## 9. 이벤트 버스 통합

모든 주요 작업이 이벤트를 게시합니다:

**세션 이벤트:**
- `session.created`: 새 세션 생성
- `session.updated`: 세션 메타데이터 변경
- `session.deleted`: 세션 제거
- `session.diff`: 파일 변경 게시
- `session.error`: 처리 오류 발생
- `session.status`: 상태 변경 (idle/busy/retry)

**메시지 이벤트:**
- `message.updated`: 메시지 생성/수정
- `message.removed`: 메시지 삭제
- `message.part.updated`: 파트 수정 (델타 포함)
- `message.part.removed`: 파트 삭제

---

## 10. 요약 다이어그램

```
세션 생명주기:
┌─────────────────────────────────────────────────────────┐
│ 1. 세션 생성                                             │
│    └─ createNext() → Session.Info 저장                 │
│    └─ 부모 세션이면 자동 공유                           │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 2. 사용자 메시지 전송                                    │
│    └─ SessionPrompt.prompt()                           │
│    └─ User MessageV2 생성                              │
│    └─ 파일 파트, 에이전트 해결                          │
│    └─ message.updated 이벤트 게시                       │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 3. 처리 루프 (SessionPrompt.loop)                       │
│    └─ Assistant MessageV2 생성                         │
│    └─ 도구 해결 (ToolRegistry + MCP)                   │
│    └─ LLM.stream() 호출                                │
│    └─ SessionProcessor가 스트림 이벤트 처리             │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 4. 스트림 이벤트 처리                                    │
│    ├─ Reasoning → ReasoningPart                        │
│    ├─ Tool calls → ToolPart (상태 머신)                 │
│    ├─ Text → TextPart                                  │
│    ├─ Step-finish → 요약 + 압축 트리거                  │
│    └─ Error → fromError() + 재시도 로직                 │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 5. 후처리                                               │
│    ├─ SessionSummary.summarize()                       │
│    ├─ SessionCompaction.isOverflow()                   │
│    └─ SessionCompaction.prune()                        │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 6. 루프 결정                                            │
│    ├─ "continue" → 3단계로 돌아감 (다음 턴)             │
│    ├─ "stop" → 루프 종료                               │
│    └─ "compact" → 압축 트리거, 그 후 계속              │
└─────────────────────────────────────────────────────────┘
```
