# OpenCode 도구 시스템 아키텍처

## 1. 도구 정의 및 등록 아키텍처

### 도구 정의 프레임워크 (tool.ts)

도구 시스템은 Zod를 사용한 스키마 검증 기반의 기본 추상화 위에 구축됩니다:

```typescript
Tool.Info<Parameters extends z.ZodType, M extends Metadata>
├── id: string (고유 도구 식별자)
├── init(ctx?: InitContext): Promise<{
│   ├── description: string
│   ├── parameters: Parameters (Zod 스키마)
│   ├── execute(args, ctx): Promise<{
│   │   ├── title: string
│   │   ├── metadata: M
│   │   ├── output: string
│   │   └── attachments?: FilePart[]
│   │ }
│   └── formatValidationError?(error): string
├── Tool.Context: {
│   ├── sessionID: string
│   ├── messageID: string
│   ├── agent: string
│   ├── abort: AbortSignal
│   ├── callID?: string
│   ├── extra?: { [key: string]: any }
│   ├── metadata(input): void (실시간 업데이트용)
│   └── ask(권한 요청): Promise<void>
│ }
```

**주요 기능:**
- `Tool.define()` 래퍼를 통한 지연 초기화
- 자동 출력 잘림 (metadata.truncated 설정 시 건너뜀)
- 커스텀 오류 포맷팅을 통한 파라미터 검증
- 실행 중 실시간 메타데이터 업데이트
- `ctx.ask()`를 통한 권한 기반 작업

### 도구 레지스트리 (registry.ts)

레지스트리는 도구 가용성과 모델별 필터링을 관리합니다:

```
ToolRegistry.all()
├── 핵심 내장 도구 (항상 사용 가능):
│   ├── InvalidTool (오류 처리)
│   ├── BashTool
│   ├── ReadTool
│   ├── GlobTool
│   ├── GrepTool
│   ├── EditTool
│   ├── WriteTool
│   ├── TaskTool
│   ├── WebFetchTool
│   ├── WebSearchTool
│   ├── CodeSearchTool
│   ├── SkillTool
│   └── ApplyPatchTool
├── 조건부 도구 (기능 플래그):
│   ├── QuestionTool (app/cli/desktop 클라이언트만)
│   ├── LspTool (OPENCODE_EXPERIMENTAL_LSP_TOOL)
│   ├── BatchTool (config.experimental.batch_tool)
│   ├── PlanExitTool & PlanEnterTool (OPENCODE_EXPERIMENTAL_PLAN_MODE)
├── 커스텀 도구 (설정 디렉토리에서):
│   └── 설정의 {tool,tools}/*.{js,ts}에서 로드
└── 플러그인 도구:
    └── 등록된 플러그인에서
```

---

## 2. 내장 도구 참조

### A. 파일 작업

#### ReadTool (read.ts)

스마트 바이너리 감지로 파일 내용 읽기

| 파라미터 | 설명 |
|---------|------|
| `filePath` | 절대 경로 (필수) |
| `offset` | 0 기반 줄 번호 (선택) |
| `limit` | 기본: 2000줄 (선택) |

**기능:**
- 바이너리 파일 감지 (30% 비인쇄 임계값)
- 특수 처리: 이미지 (base64로 임베드), PDF
- 최대 줄 길이: 2000자, 최대 바이트: 50KB
- 권한: "read" with "always": ["*"]
- 코드 완성을 위한 LSP 워밍

#### EditTool (edit.ts)

다중 폴백 전략을 가진 스마트 텍스트 교체

| 파라미터 | 설명 |
|---------|------|
| `filePath` | 절대 경로 (필수) |
| `oldString` | 찾을 텍스트 (필수) |
| `newString` | 교체 텍스트 (필수) |
| `replaceAll` | 기본: false (선택) |

**교체 전략 (우선순위 순서):**
1. SimpleReplacer: 정확한 매칭
2. LineTrimmedReplacer: 줄 레벨 트림 매칭
3. BlockAnchorReplacer: 첫/마지막 줄 앵커
4. WhitespaceNormalizedReplacer: 공백 정규화
5. IndentationFlexibleReplacer: 들여쓰기 무시
6. EscapeNormalizedReplacer: 문자 이스케이프 해제
7. TrimmedBoundaryReplacer: 경계 트림
8. ContextAwareReplacer: 컨텍스트 앵커
9. MultiOccurrenceReplacer: 모든 매칭

#### WriteTool (write.ts)

파일 생성 또는 덮어쓰기

| 파라미터 | 설명 |
|---------|------|
| `content` | 파일 내용 (필수) |
| `filePath` | 절대 경로 (필수) |

**기능:**
- 부모 디렉토리 자동 생성
- LSP 오류 보고 (프로젝트에서 최대 5개 파일)
- 권한: "edit"

### B. 검색 작업

#### GlobTool (glob.ts)

패턴 기반 파일 발견

| 파라미터 | 설명 |
|---------|------|
| `pattern` | glob 패턴 (예: "**/*.ts") (필수) |
| `path` | 검색 디렉토리 (선택) |

**기능:**
- 효율성을 위해 ripgrep 사용
- mtime 기준 정렬된 상위 100개 매칭 반환
- 100개 초과 시 잘림 경고
- 권한: "glob"

#### GrepTool (grep.ts)

정규식으로 내용 검색

| 파라미터 | 설명 |
|---------|------|
| `pattern` | 정규식 패턴 (필수) |
| `path` | 검색 디렉토리 (선택) |
| `include` | 파일 glob 필터 (선택) |

**기능:**
- ripgrep 백엔드 (-nH --hidden --follow)
- mtime 기준 정렬된 상위 100개 매칭 반환
- 권한: "grep"

### C. 명령 실행

#### BashTool (bash.ts)

안전 검사가 포함된 셸 명령 실행

| 파라미터 | 설명 |
|---------|------|
| `command` | 셸 명령 (필수) |
| `workdir` | 작업 디렉토리 (선택) |
| `timeout` | 밀리초 (선택) |
| `description` | 5-10 단어 요약 (필수) |

**안전 기능:**
- 권한 감지를 위한 Tree-sitter bash 파싱
- 추적: cd, rm, cp, mv, mkdir, touch, chmod, chown
- 외부 디렉토리 권한 검사
- BashArity 기반 명령 패턴 추출
- 타임아웃/중단 시 프로세스 트리 종료
- 메타데이터 스트리밍 (메타데이터에서 최대 30KB)

기본 타임아웃: 2분
권한: "bash" + "external_directory"

### D. 웹 작업

#### WebFetchTool (webfetch.ts)

포맷 변환이 포함된 HTTP 내용 가져오기

| 파라미터 | 설명 |
|---------|------|
| `url` | http/https URL (필수) |
| `format` | "text" / "markdown" / "html" (기본: markdown) |
| `timeout` | 초 (최대 120) (선택) |

**기능:**
- HTML to Markdown 변환 (Turndown 서비스)
- 콘텐츠 타입 인식 포맷팅
- 최대 응답: 5MB
- User-Agent 스푸핑 (Chrome 143)
- Script/style/iframe 제거
- 권한: "webfetch"

#### WebSearchTool (websearch.ts)

Exa API를 통한 웹 검색 (MCP)

| 파라미터 | 설명 |
|---------|------|
| `query` | 검색 쿼리 (필수) |
| `numResults` | 기본: 8 (선택) |
| `livecrawl` | "fallback" / "preferred" (선택) |
| `type` | "auto" / "fast" / "deep" (선택) |
| `contextMaxCharacters` | 기본: 10000 (선택) |

**기능:**
- Server-Sent Events (SSE) 응답 처리
- 모델: Exa의 web_search_exa
- 필요: OpenCode 프로바이더 또는 OPENCODE_ENABLE_EXA 플래그
- 타임아웃: 25초
- 권한: "websearch"

#### CodeSearchTool (codesearch.ts)

API/SDK 코드 컨텍스트 검색

| 파라미터 | 설명 |
|---------|------|
| `query` | 예: "React useState hook examples" (필수) |
| `tokensNum` | 1000-50000, 기본: 5000 (필수) |

**기능:**
- API, 라이브러리, SDK 타겟팅
- LLM에 최적화된 컨텍스트 반환
- Server-Sent Events (SSE) 응답 처리
- 모델: Exa의 get_code_context_exa
- 타임아웃: 30초
- 권한: "codesearch"

### E. 작업 오케스트레이션

#### TaskTool (task.ts)

특화된 서브에이전트 생성

| 파라미터 | 설명 |
|---------|------|
| `description` | 3-5 단어 작업 요약 (필수) |
| `prompt` | 에이전트를 위한 작업 (필수) |
| `subagent_type` | 에이전트 이름 (필수) |
| `session_id` | 기존 작업 계속 (선택) |
| `command` | 트리거링 명령 (선택) |

**워크플로:**
1. 새 자식 세션 생성 (또는 기존 재사용)
2. todo 권한 상속: todowrite/todoread 모두 거부
3. 도구 제한 구성 (재귀적 "task" 거부 가능)
4. 부모 모델로 서브에이전트 초기화 (또는 커스텀)
5. 도구 실행을 메타데이터 업데이트로 스트리밍
6. 반환: 작업 요약 + session_id

#### SkillTool (skill.ts)

특화된 작업 지침 로드

| 파라미터 | 설명 |
|---------|------|
| `name` | 스킬 식별자 (필수) |

**기능:**
- 설명에 사용 가능한 스킬 나열
- 에이전트당 권한 필터링된 접근
- ConfigMarkdown에서 로드
- 권한: "skill"

### F. 파일 패치

#### ApplyPatchTool (apply_patch.ts)

Unified diff 적용

| 파라미터 | 설명 |
|---------|------|
| `patchText` | 전체 unified diff 패치 (필수) |

**지원되는 작업:**
- Add: 새 파일
- Update: 청크 기반 재구성으로 기존 파일
- Delete: 파일 제거
- Move: 내용 업데이트와 함께 파일 재배치

**기능:**
- Patch.parsePatch()로 패치 검증
- Levenshtein 기반 hunk 매칭 폴백
- 파일당 원자적 작업
- 적용 후 LSP 진단
- 파일 변경 이벤트 게시
- external_directory 권한 준수

### G. LSP 통합

#### LspTool (lsp.ts) - 실험적

Language Server Protocol 작업

| 파라미터 | 설명 |
|---------|------|
| `operation` | 열거형 (아래 참조) (필수) |
| `filePath` | 절대/상대 경로 (필수) |
| `line` | 1 기반 (필수) |
| `character` | 1 기반 (필수) |

**작업:**
- goToDefinition: 정의로 이동
- findReferences: 모든 사용처 찾기
- hover: 호버 정보 가져오기
- documentSymbol: 현재 파일 개요
- workspaceSymbol: 프로젝트 전체 심볼 검색
- goToImplementation: 구현 찾기
- prepareCallHierarchy: 호출 체인 준비
- incomingCalls: 들어오는 호출 표시
- outgoingCalls: 나가는 호출 표시

권한: "lsp"
플래그: OPENCODE_EXPERIMENTAL_LSP_TOOL

### H. 유틸리티 도구

#### QuestionTool (question.ts)

대화형 사용자 프롬프트 (app/cli/desktop만)

| 파라미터 | 설명 |
|---------|------|
| `questions` | Question.Info 배열 (필수) |

#### BatchTool (batch.ts) - 실험적

병렬 도구 실행

| 파라미터 | 설명 |
|---------|------|
| `tool_calls` | {tool: string, parameters: object} 배열 (필수) |

**기능:**
- 최대 25개 도구를 병렬로 실행
- 허용되지 않음: batch, invalid 도구
- 도구별 파라미터 검증
- 개별 오류 처리
- 각 호출에 대해 Session.updatePart()
- 집계된 결과 + 첨부 파일
- 권한: 각 도구에서 상속

플래그: config.experimental.batch_tool

#### TodoWriteTool / TodoReadTool (todo.ts)

세션 범위 todo 관리

- TodoWrite 파라미터: todos[] (status: pending|in_progress|completed)
- TodoRead 파라미터: {} (없음)
- OpenAI 모델에서는 비활성화
- 권한: "todowrite" / "todoread"

### I. 실험적: 계획 모드

#### PlanEnterTool (plan.ts)

계획 에이전트로 전환

#### PlanExitTool (plan.ts)

빌드로 계획 모드 종료

---

## 3. 권한 처리 아키텍처

### 권한 모델 (permission/next.ts)

요청/응답 패턴을 가진 3단계 권한 시스템:

```typescript
Action = "allow" | "deny" | "ask"

Rule = {
  permission: string (도구 이름)
  pattern: string (glob 패턴 또는 명령)
  action: Action
}

Ruleset = Rule[]
```

### 권한 요청 흐름

```
1. 도구가 ctx.ask(permission_request) 호출
   Request = {
     id: string (권한 ID)
     sessionID: string
     permission: string (도구 이름)
     patterns: string[] (확인할 특정 패턴)
     always: string[] (즉시 부여)
     metadata: object (컨텍스트 데이터)
     tool?: { messageID, callID } (UI 링크)
   }

2. PermissionNext.ask()가 다음에 대해 평가:
   - 현재 규칙셋 (전달됨)
   - 이전에 승인된 패턴 (Storage에서)
   - 와일드카드 매칭 규칙

3. 동작 결정:
   if rule.action === "allow" → 진행
   if rule.action === "deny" → DeniedError 던지기
   if rule.action === "ask" → Event.Asked 발행, Reply 대기

4. 사용자가 PermissionNext.reply()를 통해 응답:
   Reply = "once" | "always" | "reject"

5. "always"인 경우 승인 저장:
   Storage.write(["permission", projectID], approved)
```

### 외부 디렉토리 보호

```typescript
assertExternalDirectory(ctx, target, options?)
├── 확인: Instance.containsPath(target)
├── 외부인 경우: "external_directory" 권한 요청
└── 패턴: directory/* (부모 glob)
```

---

## 4. 실행 흐름

```
┌─ 에이전트가 도구 호출 받음 ─┐
│                             │
├─ ToolRegistry.tools()       │ ← 모델에 사용 가능한 도구 가져오기
│  └─ 모델/플래그별 필터링   │
│                             │
├─ Tool.init(context)         │ ← 도구 초기화
│  └─ description,            │
│     parameters 스키마 반환  │
│                             │
├─ 파라미터 검증              │ ← Zod 검증
│  └─ 커스텀 오류 포맷        │
│                             │
├─ Tool.execute(args, ctx)    │ ← 도구 실행
│  ├─ ctx.ask() 권한          │ ← 권한 확인
│  ├─ ctx.metadata() 업데이트 │ ← 실시간 업데이트
│  └─ 출력 반환               │
│                             │
├─ 출력 잘림 (자동)           │ ← metadata.truncated 설정 시 건너뜀
│                             │
└─ 결과 포맷                  │ ← title, metadata, output
```

---

## 5. 전체 도구 목록

| 도구 | 목적 | 상태 | 모델별 | 비동기 Init |
|------|------|------|--------|------------|
| bash | 셸 명령 실행 | 핵심 | 아니오 | 아니오 |
| read | 파일 내용 읽기 | 핵심 | 아니오 | 아니오 |
| glob | 패턴 파일 매칭 | 핵심 | 아니오 | 아니오 |
| grep | 내용 검색 | 핵심 | 아니오 | 아니오 |
| edit | 스마트 텍스트 교체 | 핵심 | 조건부* | 아니오 |
| write | 파일 생성/덮어쓰기 | 핵심 | 조건부* | 아니오 |
| apply_patch | Unified diff 적용 | 핵심 | 조건부** | 아니오 |
| webfetch | HTTP 내용 가져오기 | 핵심 | 아니오 | 아니오 |
| websearch | 웹 검색 (Exa) | 핵심 | 조건부*** | 비동기 |
| codesearch | 코드 컨텍스트 검색 (Exa) | 핵심 | 조건부*** | 비동기 |
| task | 서브에이전트 생성 | 핵심 | 아니오 | 비동기 |
| skill | 지침 로드 | 핵심 | 아니오 | 비동기 |
| question | 대화형 프롬프트 | 조건부 | 데스크톱만 | 아니오 |
| batch | 병렬 실행 | 실험적 | 아니오 | 비동기 |
| lsp | Language Server 작업 | 실험적 | 아니오 | 아니오 |
| todowrite | todo 목록 업데이트 | 핵심 | OpenAI 제외 | 아니오 |
| todoread | todo 목록 읽기 | 핵심 | OpenAI 제외 | 아니오 |
| plan_enter | 계획 모드 진입 | 실험적 | CLI만 | 아니오 |
| plan_exit | 계획 모드 종료 | 실험적 | CLI만 | 아니오 |
| invalid | 오류 처리 | 핵심 | N/A | 아니오 |

**범례:**
- `*` edit/write는 apply_patch 활성화 시 비활성화
- `**` apply_patch는 GPT 모델에서만
- `***` OpenCode 프로바이더 또는 OPENCODE_ENABLE_EXA 플래그에서만

---

## 6. 주요 아키텍처 패턴

### 출력 잘림 시스템 (truncation.ts)
- 최대 출력: 32KB (에이전트별 구성 가능)
- 초과 시 마지막 줄로 잘림
- 오버플로우를 디스크에 저장 (메타데이터의 outputPath)
- 도구가 `metadata.truncated` 설정 시 건너뜀

### 컨텍스트 메타데이터 스트리밍
```typescript
ctx.metadata({
  title?: string
  metadata?: object
})
```
장시간 실행 작업(bash, batch)에 대해 UI로 실시간 업데이트 전송

### 파일 잠금 패턴
```typescript
FileTime.withLock(filePath, async () => {
  // 원자적 편집 작업
  // 경쟁 조건 방지
})
```

### 이벤트 게시
- File.Event.Edited → 파일 감시자
- FileWatcher.Event.Updated → LSP
- Bus.publish() 비동기 구독자용

---

## 7. 설정 및 플래그

### 기능 플래그

```
OPENCODE_CLIENT = "app" | "cli" | "desktop" (QuestionTool 활성화)
OPENCODE_EXPERIMENTAL_LSP_TOOL (LspTool 활성화)
OPENCODE_EXPERIMENTAL_BATCH_TOOL (BatchTool 활성화)
OPENCODE_EXPERIMENTAL_PLAN_MODE (계획 도구 활성화)
OPENCODE_ENABLE_EXA (websearch/codesearch 활성화)
OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS (기본: 2분)
```

### 설정 옵션

```
config.experimental.batch_tool: boolean
config.experimental.primary_tools: string[] (제한된 도구)
```
