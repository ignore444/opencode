# OpenCode 프로젝트 분석 문서

이 폴더에는 OpenCode 프로젝트의 상세 분석 문서가 포함되어 있습니다.

## 문서 목록

| 파일 | 내용 |
|------|------|
| [01-project-overview.md](./01-project-overview.md) | 프로젝트 개요, 기술 스택, 패키지 구조 |
| [02-directory-structure.md](./02-directory-structure.md) | 상세 디렉토리 구조 및 모듈 설명 |
| [03-agent-system.md](./03-agent-system.md) | 에이전트 시스템 아키텍처 |
| [04-tool-system.md](./04-tool-system.md) | 도구 시스템 및 내장 도구 참조 |
| [05-provider-system.md](./05-provider-system.md) | LLM 프로바이더 시스템 |
| [06-session-system.md](./06-session-system.md) | 세션 관리 및 메시지 처리 |
| [07-sdk-package.md](./07-sdk-package.md) | JavaScript SDK 패키지 |
| [08-ui-app-packages.md](./08-ui-app-packages.md) | UI/App 패키지 (웹, 데스크톱, 콘솔) |

## 빠른 참조

### 핵심 개념

- **에이전트**: 작업 처리를 위한 설정 프로필 (build, plan, explore 등)
- **도구**: 에이전트가 사용하는 기능 (bash, read, edit, grep 등)
- **프로바이더**: LLM 서비스 연결 (OpenAI, Anthropic, Google 등 30+)
- **세션**: 대화 상태 및 메시지 히스토리 관리

### 주요 파일 경로

```
packages/opencode/
├── src/agent/agent.ts      # 에이전트 정의
├── src/tool/tool.ts        # 도구 인터페이스
├── src/provider/provider.ts # 프로바이더 레지스트리
├── src/session/index.ts    # 세션 관리
├── src/server/server.ts    # HTTP 서버
└── src/cli/               # CLI 명령어

packages/sdk/js/            # JavaScript SDK
packages/app/               # 웹 애플리케이션
packages/ui/                # 공유 UI 컴포넌트
packages/desktop/           # Tauri 데스크톱 앱
```

### 기술 스택 요약

| 영역 | 기술 |
|------|------|
| 런타임 | Bun 1.3.5 |
| 언어 | TypeScript 5.8.2 |
| 프론트엔드 | SolidJS |
| 백엔드 | Hono |
| AI SDK | Vercel AI SDK |
| 데스크톱 | Tauri 2 |
| 빌드 | Turbo (모노레포) |

### 개발 명령어

```bash
bun install              # 의존성 설치
bun run dev              # 개발 모드
bun run typecheck        # 타입 체크
bun turbo build          # 전체 빌드
```

---

*마지막 업데이트: 2026-01-21*
