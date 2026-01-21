# OpenCode SDK 패키지 아키텍처

## 1. SDK 구조 및 구성

### 디렉토리 구조

```
packages/sdk/js/
├── src/
│   ├── index.ts                 # 메인 진입점 (v1)
│   ├── client.ts               # V1 클라이언트 팩토리
│   ├── server.ts               # V1 서버 스폰
│   ├── v2/                      # V2 구현
│   │   ├── index.ts            # V2 진입점
│   │   ├── client.ts           # V2 클라이언트 팩토리
│   │   ├── server.ts           # V2 서버 스폰
│   │   └── gen/                # V2 생성된 코드
│   └── gen/                     # V1 생성된 코드
├── example/                     # 사용 예제
├── script/                      # 빌드 스크립트
└── package.json
```

### 패키지 내보내기

```json
{
  ".": "./src/index.ts",
  "./client": "./src/client.ts",
  "./server": "./src/server.ts",
  "./v2": "./src/v2/index.ts",
  "./v2/client": "./src/v2/client.ts",
  "./v2/server": "./src/v2/server.ts"
}
```

---

## 2. 클라이언트 측 API

### 팩토리 함수: createOpencodeClient()

**V1 구현:**
```typescript
export function createOpencodeClient(config?: Config & { directory?: string }) {
  if (!config?.fetch) {
    const customFetch: any = (req: any) => {
      req.timeout = false
      return fetch(req)
    }
    config = {
      ...config,
      fetch: customFetch,
    }
  }

  if (config?.directory) {
    config.headers = {
      ...config.headers,
      "x-opencode-directory": config.directory,
    }
  }

  const client = createClient(config)
  return new OpencodeClient({ client })
}
```

**V2 구현 (향상됨):**
```typescript
export function createOpencodeClient(config?: Config & { directory?: string }) {
  // 동일한 fetch 처리
  // 비ASCII 문자를 위한 향상된 디렉토리 인코딩
  if (config?.directory) {
    const isNonASCII = /[^\x00-\x7F]/.test(config.directory)
    const encodedDirectory = isNonASCII
      ? encodeURIComponent(config.directory)
      : config.directory
    config.headers = {
      ...config.headers,
      "x-opencode-directory": encodedDirectory,
    }
  }

  const client = createClient(config)
  return new OpencodeClient({ client })
}
```

### 설정 옵션

| 옵션 | 설명 |
|------|------|
| `baseUrl` | 서버 기본 URL (기본: "http://localhost:4096") |
| `fetch` | 커스텀 fetch 구현 |
| `directory` | 작업 디렉토리 헤더 |
| `headers` | 커스텀 HTTP 헤더 |
| `parseAs` | 응답 파싱 방법 ("auto", "json", "text", "blob" 등) |
| `responseStyle` | 반환 형식 ("data" 또는 메타데이터가 포함된 "fields") |
| `throwOnError` | HTTP 오류 시 예외 발생 여부 |

---

## 3. OpencodeClient API 그룹

### Global - 전역 이벤트
- `event()` - 이벤트 가져오기 (SSE)

### Project - 프로젝트 관리
- `list()` - 모든 프로젝트 목록
- `current()` - 현재 프로젝트 가져오기

### PTY - 의사 터미널 관리
- `list()` - PTY 세션 목록
- `create()` - 새 PTY 세션 생성
- `remove()` - PTY 세션 제거
- `get()` - PTY 세션 정보
- `update()` - PTY 세션 업데이트
- `connect()` - PTY 세션 연결

### Session - 핵심 세션 관리 (가장 광범위)

| 메서드 | 설명 |
|--------|------|
| `list()` | 세션 목록 |
| `create()` | 세션 생성 |
| `status()` | 세션 상태 |
| `get()` | 세션 상세 |
| `delete()` | 세션 삭제 |
| `update()` | 세션 속성 업데이트 |
| `children()` | 자식 세션 |
| `todo()` | 세션 todo 목록 |
| `init()` | 앱 분석 및 AGENTS.md 생성 |
| `fork()` | 메시지에서 세션 포크 |
| `abort()` | 세션 중단 |
| `share()` / `unshare()` | 공유 관리 |
| `diff()` | 세션 diff |
| `summarize()` | 세션 요약 |
| `messages()` | 메시지 목록 |
| `prompt()` | 메시지 생성 및 전송 |
| `promptAsync()` | 논블로킹 프롬프트 |
| `message()` | 특정 메시지 |
| `command()` | 명령 전송 |
| `shell()` | 셸 명령 실행 |
| `revert()` | 메시지 되돌리기 |
| `unrevert()` | 되돌린 메시지 복원 |

### Config - 설정 관리
- `get()` - 설정 정보
- `update()` - 설정 업데이트
- `providers()` - 프로바이더 목록

### Tool - 도구 관리
- `ids()` - 도구 ID 목록
- `list()` - JSON 스키마와 함께 도구 목록

### File - 파일 작업
- `list()` - 파일/디렉토리 목록
- `read()` - 파일 읽기
- `status()` - 파일 상태

### Find - 검색 작업
- `text()` - 파일에서 텍스트 찾기
- `files()` - 파일 찾기
- `symbols()` - 워크스페이스 심볼 찾기

### Provider - 프로바이더 관리
- `list()` - 프로바이더 목록
- `auth()` - 인증 방법
- `oauth.authorize()` - OAuth 인증
- `oauth.callback()` - OAuth 콜백 처리

### MCP - Model Context Protocol
- `status()` - MCP 서버 상태
- `add()` - MCP 서버 추가
- `connect()` - 서버 연결
- `disconnect()` - 서버 연결 해제
- `auth.remove()` - 자격 증명 제거
- `auth.start()` - OAuth 흐름 시작
- `auth.callback()` - OAuth 콜백
- `auth.authenticate()` - 브라우저로 OAuth 흐름

### TUI - 터미널 UI
- `appendPrompt()` - 프롬프트 추가
- `openHelp()` - 도움말 다이얼로그 열기
- `openSessions()` - 세션 다이얼로그 열기
- `openThemes()` - 테마 다이얼로그 열기
- `openModels()` - 모델 다이얼로그 열기
- `submitPrompt()` - 프롬프트 제출
- `clearPrompt()` - 프롬프트 지우기
- `executeCommand()` - TUI 명령 실행
- `showToast()` - 토스트 알림 표시
- `publish()` - TUI 이벤트 게시
- `control.next()` - 다음 TUI 요청
- `control.response()` - TUI 응답 제출

### 기타 API 그룹
- **Instance** - 인스턴스 관리 (`dispose()`)
- **Path** - 경로 유틸리티 (`get()`)
- **VCS** - 버전 관리 (`get()`)
- **LSP** - Language Server Protocol (`status()`)
- **Formatter** - 코드 포맷팅 (`status()`)
- **Command** - 명령 관리 (`list()`)
- **Event** - 이벤트 구독 (`subscribe()`)
- **Auth** - 인증 (`set()`)

---

## 4. 서버 측 API

### 팩토리 함수: createOpencodeServer()

```typescript
export async function createOpencodeServer(options?: ServerOptions) {
  options = Object.assign({
    hostname: "127.0.0.1",
    port: 4096,
    timeout: 5000,
  }, options ?? {})

  const args = [`serve`, `--hostname=${options.hostname}`, `--port=${options.port}`]
  if (options.config?.logLevel) args.push(`--log-level=${options.config.logLevel}`)

  const proc = spawn(`opencode`, args, {
    signal: options.signal,
    env: {
      ...process.env,
      OPENCODE_CONFIG_CONTENT: JSON.stringify(options.config ?? {}),
    },
  })

  const url = await new Promise<string>((resolve, reject) => {
    // "opencode server listening on" 메시지 대기
    // 출력에서 URL 파싱 후 서버 URL로 resolve
  })

  return {
    url,
    close() {
      proc.kill()
    },
  }
}
```

### ServerOptions 타입

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `hostname` | "127.0.0.1" | 호스트 이름 |
| `port` | 4096 | 포트 |
| `signal` | - | 취소용 AbortSignal |
| `timeout` | 5000 | 타임아웃 (ms) |
| `config` | - | SDK 설정 |

### TUI 함수: createOpencodeTui()

```typescript
export function createOpencodeTui(options?: TuiOptions) {
  const args = []

  if (options?.project) args.push(`--project=${options.project}`)
  if (options?.model) args.push(`--model=${options.model}`)
  if (options?.session) args.push(`--session=${options.session}`)
  if (options?.agent) args.push(`--agent=${options.agent}`)

  const proc = spawn(`opencode`, args, {
    signal: options?.signal,
    stdio: "inherit",
    env: {
      ...process.env,
      OPENCODE_CONFIG_CONTENT: JSON.stringify(options?.config ?? {}),
    },
  })

  return {
    close() {
      proc.kill()
    },
  }
}
```

---

## 5. 통합 팩토리: createOpencode()

```typescript
export async function createOpencode(options?: ServerOptions) {
  const server = await createOpencodeServer({
    ...options,
  })

  const client = createOpencodeClient({
    baseUrl: server.url,
  })

  return {
    client,
    server,
  }
}
```

이 편의 함수는:
1. OpenCode 서버 스폰
2. 해당 서버에 연결하도록 구성된 클라이언트 생성
3. 조정된 사용을 위해 둘 다 반환

---

## 6. 타입 정의

자동 생성된 타입 포함:

- **Event 타입**: `EventServerInstanceDisposed`, `EventInstallationUpdated`, `EventLspClientDiagnostics` 등
- **Message 타입**: `UserMessage`, `AssistantMessage`, `ToolPart`, `TextPart`, `FilePart`, `AgentPart` 등
- **Error 타입**: `ApiError`, `ProviderAuthError`, `MessageAbortedError`, `MessageOutputLengthError`, `UnknownError`
- **Session 타입**: 세션 메타데이터, 상태, 권한
- **File 타입**: 추가/삭제 추적이 있는 `FileDiff`
- **Provider 타입**: 인증, OAuth 설정
- **Permission 타입**: 권한 요청 및 승인
- **Tool 타입**: JSON 스키마 파라미터가 있는 도구 정의

---

## 7. 코드 생성 및 빌드

### 빌드 프로세스

SDK는 **OpenAPI 스펙에서 생성**됩니다:

1. **생성 도구**: `@hey-api/openapi-ts` (v0.90.4)
2. **입력**: 메인 opencode 패키지의 OpenAPI 스펙
3. **출력 디렉토리**:
   - `src/v2/gen/` - V2 클라이언트 코드
   - `src/gen/` - V1 클라이언트 코드 (히스토리)
4. **사용된 플러그인**:
   - `@hey-api/typescript` - TypeScript 생성
   - `@hey-api/sdk` - SDK 클래스 생성
   - `@hey-api/client-fetch` - Fetch 클라이언트 구현

**빌드 단계:**
```bash
1. 메인 패키지에서 OpenAPI JSON 생성
2. @hey-api/openapi-ts 코드 생성기 실행
3. Prettier로 포맷팅
4. TypeScript를 dist/로 컴파일
5. 임시 파일 정리
```

---

## 8. 메인 OpenCode 패키지와 통합

### SDK 사용 위치

- `packages/opencode/src/acp/` - 에이전트 생명주기 관리
- `packages/opencode/src/cli/cmd/` - SDK 클라이언트를 사용하는 CLI 명령
- `packages/opencode/src/cli/cmd/tui/` - SDK 타입을 임포트하는 터미널 UI 컴포넌트

### 일반적인 임포트 패턴

```typescript
// V2 (권장)
import { createOpencodeClient, type OpencodeClient } from "@opencode-ai/sdk/v2"
import type { Event, SessionMessageResponse } from "@opencode-ai/sdk/v2"

// V1 (deprecated)
import type { Path } from "@opencode-ai/sdk"
```

---

## 9. HTTP 클라이언트 구현

### 기능

- **Fetch 기반**: 네이티브 Fetch API 사용
- **요청 생명주기**:
  1. 설정 병합
  2. 보안/인증 설정
  3. 요청 검증
  4. 바디 직렬화
  5. 헤더 병합
  6. 요청 인터셉터
  7. 실제 fetch
  8. 응답 인터셉터
  9. 응답 파싱 (JSON, text, blob 등)
  10. 응답 검증 및 변환

- **오류 처리**:
  - HTTP 오류 응답은 JSON으로 파싱
  - 텍스트 파싱으로 폴백
  - 선택적 `throwOnError` 모드
  - 구별된 유니온으로 오류 타이핑

- **응답 스타일**:
  - `"data"` - 데이터만 반환
  - `"fields"` - `{ data, request, response }` 반환

- **특수 처리**:
  - 204 No Content는 빈 객체 반환
  - Content-Type 자동 감지
  - SSE (Server-Sent Events) 지원

---

## 10. 사용 예제

```typescript
import { createOpencodeClient, createOpencodeServer } from "@opencode-ai/sdk"

const server = await createOpencodeServer()
const client = createOpencodeClient({ baseUrl: server.url })

const input = await Array.fromAsync(new Bun.Glob("packages/core/*.ts").scan())

for await (const file of input) {
  const session = await client.session.create()
  await client.session.prompt({
    path: { id: session.data.id },
    body: {
      parts: [
        {
          type: "file",
          mime: "text/plain",
          url: `file://${file}`,
        },
        {
          type: "text",
          text: `Write tests for every public function in this file.`,
        },
      ],
    },
  })
}
```

---

## 11. 기술 세부사항

| 항목 | 값 |
|------|-----|
| **TypeScript 타겟** | ES2022 |
| **모듈** | Node ESNext |
| **버전** | 1.1.28 |
| **라이선스** | MIT |
| **배포** | npm에 `@opencode-ai/sdk`로 `dist/`에서 게시 |

이 SDK는 클라이언트 및 서버 측 기능을 모두 갖춘 OpenCode 서버에 대한 포괄적이고 타입 안전한 인터페이스를 제공하며, 실시간 이벤트를 위한 SSE 및 OAuth 인증과 같은 고급 기능을 지원합니다.
