# OpenCode 프로젝트 개요

## 기본 정보

| 항목 | 내용 |
|------|------|
| **프로젝트명** | OpenCode |
| **버전** | 1.1.28 |
| **런타임** | Bun 1.3.5, Node.js 22+ |
| **라이선스** | MIT |

## 기술 스택

| 영역 | 기술 |
|------|------|
| 런타임 | Bun, Node.js 22+ |
| 언어 | TypeScript 5.8.2 |
| 프론트엔드 | Solid.js, Vite, Tailwind CSS |
| 백엔드 | Hono, Cloudflare Workers |
| 데이터베이스 | Drizzle ORM, PlanetScale |
| AI SDK | Vercel AI SDK (@ai-sdk/*) |
| 빌드 | Turbo (모노레포) |
| 데스크톱 | Tauri 2 |

## 패키지 구조 (17개)

### 핵심 패키지

| 패키지 | 경로 | 역할 |
|--------|------|------|
| **opencode** | `packages/opencode/` | CLI/TUI 메인 애플리케이션 |
| **sdk** | `packages/sdk/js/` | JavaScript SDK (클라이언트/서버) |
| **plugin** | `packages/plugin/` | 플러그인 시스템 |
| **util** | `packages/util/` | 공유 유틸리티 |

### UI/앱 패키지

| 패키지 | 경로 | 역할 |
|--------|------|------|
| **app** | `packages/app/` | 웹 애플리케이션 |
| **ui** | `packages/ui/` | 공유 UI 컴포넌트 |
| **desktop** | `packages/desktop/` | Tauri 데스크톱 앱 |
| **web** | `packages/web/` | 문서 사이트 (Astro) |

### 콘솔/엔터프라이즈 패키지

| 패키지 | 경로 | 역할 |
|--------|------|------|
| **console-app** | `packages/console/app/` | 관리 콘솔 UI |
| **console-core** | `packages/console/core/` | 백엔드 로직/DB |
| **console-function** | `packages/console/function/` | Serverless API |
| **enterprise** | `packages/enterprise/` | SaaS 대시보드 |

### 통합 패키지

| 패키지 | 경로 | 역할 |
|--------|------|------|
| **slack** | `packages/slack/` | Slack 봇 통합 |
| **function** | `packages/function/` | 서버리스 함수 |

## 아키텍처 패턴

- **Namespace 패턴**: 모듈별 상수, 스키마, 함수 캡슐화
- **Instance Context**: 프로젝트별 격리된 컨텍스트
- **Zod 스키마**: 타입 안전 검증
- **이벤트 버스**: 비동기 통신 (Bus.publish/subscribe)

## 코드 흐름

```
CLI 명령어 → Instance 컨텍스트 → Session 생성
    ↓
SessionProcessor (메시지 루프)
    ↓
LLM.stream() (Vercel AI SDK)
    ↓
Tool Call 감지 → Permission 확인 → Tool 실행
    ↓
결과 수집 → Message 저장 → 루프 계속/종료
```

## AI 프로바이더 지원

30+ 개의 AI 프로바이더 지원:

- **Anthropic** - Claude 시리즈
- **OpenAI** - GPT 시리즈
- **Google** - Gemini, Vertex AI
- **Azure** - Azure OpenAI
- **AWS** - Bedrock
- **Mistral** - Mistral AI
- **Groq** - Groq Cloud
- **DeepInfra** - DeepInfra
- **GitHub Copilot** - GitHub AI
- **GitLab AI** - GitLab AI
- 기타 다수

## 주요 CLI 명령어

```bash
opencode run [message]   # 에이전트 실행
opencode serve           # HTTP 서버 시작
opencode auth            # 인증 설정
opencode models          # 모델 목록
opencode session         # 세션 관리
opencode mcp             # MCP 서버 관리
```

## 개발 명령어

```bash
bun install              # 의존성 설치
bun run dev              # 개발 모드
bun run typecheck        # 타입 체크
bun turbo build          # 전체 빌드
```

## 설정 우선순위

1. 인라인 설정 (OPENCODE_CONFIG_CONTENT) - 최고
2. 프로젝트 설정 (opencode.jsonc)
3. 커스텀 설정 (OPENCODE_CONFIG)
4. 전역 설정 (~/.opencode/config.json)
5. 원격 Well-Known 설정 - 최저
