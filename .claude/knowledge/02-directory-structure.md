# OpenCode 디렉토리 구조

## packages/opencode/src 구조

### 진입점

- `index.ts` - CLI 진입점 (yargs 기반)

---

## 핵심 기능 디렉토리

### 1. acp/ - Agent Client Protocol

**목적**: 외부 통합을 위한 Agent Client Protocol 구현 (예: Zed 에디터)

| 파일 | 역할 |
|------|------|
| `agent.ts` | @agentclientprotocol/sdk의 Agent 인터페이스 구현 |
| `session.ts` | ACP 세션 상태 관리 및 내부 세션 매핑 |
| `types.ts` | 내부 사용 타입 정의 |
| `README.md` | ACP 구현 문서 |

### 2. agent/ - 핵심 에이전트 로직

**목적**: 작업 처리 및 실행을 위한 메인 에이전트 구현

| 파일 | 역할 |
|------|------|
| `agent.ts` | 핵심 에이전트 구현 |
| `generate.txt` | 에이전트 생성 프롬프트 템플릿 |
| `prompt/compaction.txt` | 메시지 압축 전략 |
| `prompt/explore.txt` | 코드 탐색 프롬프트 |
| `prompt/summary.txt` | 세션 요약 생성 |
| `prompt/title.txt` | 제목 생성 프롬프트 |

### 3. cli/ - 명령줄 인터페이스

**목적**: 메인 CLI 진입점 및 명령 프레임워크

#### 최상위 파일

| 파일 | 역할 |
|------|------|
| `bootstrap.ts` | CLI 초기화 및 설정 |
| `error.ts` | 오류 포맷팅 및 처리 |
| `network.ts` | 네트워크 관련 유틸리티 |
| `ui.ts` | 사용자 인터페이스 출력 포맷팅 |
| `upgrade.ts` | CLI 업그레이드/버전 관리 |

#### cli/cmd/ 서브명령

| 파일 | 역할 |
|------|------|
| `acp.ts` | Agent Client Protocol 서버 명령 |
| `agent.ts` | 에이전트 관리 명령 |
| `auth.ts` | 인증 명령 |
| `generate.ts` | 생성/코드 생성 명령 |
| `run.ts` | 저장소에서 OpenCode 실행 |
| `serve.ts` | 서버 모드 명령 |
| `export.ts` | 세션 데이터 내보내기 |
| `import.ts` | 세션 데이터 가져오기 |
| `github.ts` | GitHub 통합 |
| `mcp.ts` | Model Context Protocol 관리 |
| `models.ts` | LLM 모델 관리 |
| `pr.ts` | Pull request 작업 |
| `session.ts` | 세션 관리 |
| `stats.ts` | 통계 표시 |
| `upgrade.ts` | CLI 업그레이드 |
| `uninstall.ts` | 제거 명령 |
| `web.ts` | 웹 서버 명령 |

#### cli/cmd/debug/ 디버그 유틸리티

- `agent.ts`, `config.ts`, `file.ts`, `lsp.ts`
- `ripgrep.ts`, `scrap.ts`, `skill.ts`, `snapshot.ts`

#### cli/cmd/tui/ 터미널 UI

| 디렉토리/파일 | 역할 |
|---------------|------|
| `app.tsx` | 메인 TUI 애플리케이션 (SolidJS) |
| `attach.ts` | 기존 세션 연결 |
| `thread.ts` | 스레드 명령 |
| `worker.ts` | TUI 워커 스레드 |
| `event.ts` | 이벤트 처리 |
| `component/` | UI 컴포넌트들 |
| `context/` | 상태 관리 컨텍스트 |
| `routes/` | TUI 라우트 |
| `ui/` | 공통 UI 컴포넌트 |
| `util/` | TUI 유틸리티 |

### 4. session/ - 세션 관리

**목적**: 대화 세션, 메시지 히스토리, 세션 상태 관리

| 파일 | 역할 |
|------|------|
| `index.ts` | 세션 코어 구현 |
| `message.ts` | 메시지 처리 (v1) |
| `message-v2.ts` | 메시지 처리 (v2) |
| `llm.ts` | LLM 상호작용 로직 |
| `processor.ts` | 메시지 처리 파이프라인 |
| `status.ts` | 세션 상태 추적 |
| `summary.ts` | 세션 요약 |
| `compaction.ts` | 메시지 히스토리 압축 |
| `retry.ts` | 실패한 작업 재시도 |
| `revert.ts` | 세션 변경 되돌리기 |
| `todo.ts` | TODO 항목 관리 |
| `system.ts` | 시스템 메시지 관리 |
| `prompt.ts` | 세션 프롬프트 처리 |

### 5. server/ - HTTP 서버

**목적**: API 엔드포인트를 위한 Hono 기반 HTTP 서버

| 파일 | 역할 |
|------|------|
| `server.ts` | 메인 서버 설정 |
| `error.ts` | 서버 오류 처리 |
| `event.ts` | 서버 이벤트 |
| `mdns.ts` | mDNS 서비스 디스커버리 |

#### server/routes/ 라우트

- `config.ts`, `session.ts`, `file.ts`, `pty.ts`
- `project.ts`, `mcp.ts`, `provider.ts`, `permission.ts`
- `question.ts`, `tui.ts`, `global.ts`, `experimental.ts`

### 6. tool/ - 도구 레지스트리 및 구현

**목적**: 에이전트가 사용할 수 있는 도구/기능

| 도구 파일 | 역할 |
|----------|------|
| `bash.ts` | 셸 명령 실행 |
| `edit.ts` | 파일 편집 |
| `read.ts` | 파일 읽기 |
| `write.ts` | 파일 쓰기 |
| `glob.ts` | 파일 글로빙/패턴 매칭 |
| `grep.ts` | 코드 검색 |
| `ls.ts` | 디렉토리 목록 |
| `apply_patch.ts` | 패치 적용 |
| `codesearch.ts` | 고급 코드 검색 |
| `lsp.ts` | Language Server Protocol 상호작용 |
| `batch.ts` | 배치 작업 |
| `multiedit.ts` | 다중 파일 편집 |
| `skill.ts` | 스킬 호출 |
| `task.ts` | 작업 관리 |
| `todo.ts` | TODO 목록 처리 |
| `question.ts` | 사용자 질문 프롬프트 |
| `plan.ts` | 계획 도구 |
| `webfetch.ts` | 웹 콘텐츠 가져오기 |
| `websearch.ts` | 웹 검색 |

### 7. provider/ - LLM 프로바이더

**목적**: 다양한 LLM 프로바이더 및 모델 통합

| 파일 | 역할 |
|------|------|
| `provider.ts` | 기본 프로바이더 구현 |
| `models.ts` | 모델 목록 및 관리 |
| `models-macro.ts` | 모델 매크로 |
| `auth.ts` | 프로바이더 인증 |
| `transform.ts` | 응답 변환 |

---

## 인프라 및 유틸리티 디렉토리

### 8. lsp/ - Language Server Protocol

| 파일 | 역할 |
|------|------|
| `client.ts` | LSP 클라이언트 구현 |
| `server.ts` | LSP 서버 구현 |
| `language.ts` | 언어별 설정 |

### 9. mcp/ - Model Context Protocol

| 파일 | 역할 |
|------|------|
| `index.ts` | 메인 MCP 구현 |
| `auth.ts` | MCP 인증 |
| `oauth-provider.ts` | OAuth 프로바이더 지원 |
| `oauth-callback.ts` | OAuth 콜백 처리 |

### 10. project/ - 프로젝트 관리

| 파일 | 역할 |
|------|------|
| `project.ts` | 메인 프로젝트 클래스 |
| `bootstrap.ts` | 프로젝트 초기화 |
| `instance.ts` | 프로젝트 인스턴스 관리 |
| `state.ts` | 프로젝트 상태 추적 |
| `vcs.ts` | 버전 관리 시스템 통합 (Git) |

### 11-35. 기타 디렉토리

| 디렉토리 | 목적 |
|---------|------|
| `util/` | 재사용 가능한 유틸리티 함수 (20+ 파일) |
| `file/` | 파일 시스템 추상화 |
| `patch/` | Git 패치 관리 |
| `worktree/` | Git worktree 작업 |
| `snapshot/` | 세션 스냅샷 저장/복원 |
| `skill/` | 커스텀 스킬 구현 |
| `plugin/` | 서드파티 플러그인 지원 |
| `bus/` | 모듈 간 이벤트 통신 |
| `storage/` | 영구 데이터 저장소 |
| `format/` | 코드 및 텍스트 포맷팅 |
| `permission/` | 사용자 권한 및 기능 관리 |
| `question/` | 사용자 상호작용 및 질문 |
| `shell/` | 셸 명령 실행 및 통합 |
| `pty/` | 터미널 에뮬레이션 및 프로세스 관리 |
| `scheduler/` | 작업 스케줄링 |
| `share/` | 세션 공유 및 내보내기 |
| `id/` | 고유 식별자 생성 |
| `flag/` | 기능 플래그 관리 |
| `global/` | 전역 애플리케이션 상태 |
| `env/` | 환경 변수 관리 |
| `bun/` | Bun 런타임 통합 |
| `auth/` | 사용자 인증 처리 |
| `installation/` | CLI 설치, 버전 관리, 업그레이드 |
| `ide/` | IDE별 통합 |
| `command/` | 사전 정의된 명령 템플릿 |
| `config/` | 설정 파싱 및 관리 |

---

## 모듈 구성 요약

코드베이스는 명확한 아키텍처 패턴을 따릅니다:

1. **진입점**: `index.ts` - yargs로 CLI 설정
2. **명령 레이어**: `cli/cmd/` - 모든 CLI 명령
3. **핵심 로직**: `session/`, `agent/`, `project/` - 메인 비즈니스 로직
4. **인프라**: `server/`, `lsp/`, `mcp/`, `acp/` - 외부 통합
5. **실행**: `tool/`, `shell/`, `pty/` - 코드 실행 기능
6. **데이터**: `storage/`, `snapshot/`, `worktree/`, `patch/` - 데이터 지속성
7. **유틸리티**: `util/`, `file/`, `format/` - 헬퍼 함수
8. **UI**: `cli/cmd/tui/` - 터미널 사용자 인터페이스 (SolidJS 기반)
9. **확장성**: `skill/`, `plugin/`, `permission/` - 플러그인 시스템
10. **설정**: `config/`, `env/`, `auth/`, `provider/` - 설정 관리
