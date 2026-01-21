# OpenCode UI 및 App 패키지 아키텍처

## 1. packages/app - 웹 애플리케이션

**위치:** `packages/app`

**목적:** SolidJS로 구축된 OpenCode IDE용 메인 웹 애플리케이션. 웹 앱 및 Tauri 데스크톱 클라이언트로 배포.

### 진입점 및 빌드 설정

- **진입점:** `src/entry.tsx` - SolidJS 애플리케이션 초기화
- **메인 내보내기:** `src/index.ts` - `PlatformProvider`와 `AppInterface` 내보내기
- **Vite 플러그인:** `vite.js` - SolidJS와 TailwindCSS 설정 제공
- **빌드:** Vite 기반, ESNext 타겟, 개발 모드에서 포트 3000

### 주요 의존성

| 패키지 | 용도 |
|--------|------|
| `@solidjs/router` | 클라이언트 측 라우팅 |
| `@solidjs/meta` | Head/메타 관리 |
| `@kobalte/core` | 헤드리스 UI 컴포넌트 |
| `solid-primitives/*` | 웹 API 프리미티브 |
| `@opencode-ai/sdk` | 백엔드 통신용 TypeScript SDK |
| `@opencode-ai/ui` | 공유 UI 컴포넌트 라이브러리 |
| `shiki` & `marked` | 코드 하이라이팅 및 마크다운 파싱 |
| `virtua` | 대형 목록용 가상 스크롤링 |

### 주요 컴포넌트 및 구조

**페이지:**
- `pages/home.tsx` - 프로젝트 선택 및 표시 (메인 랜딩)
- `pages/session.tsx` - 메인 개발 워크스페이스 (90KB)
- `pages/layout.tsx` - 기본 레이아웃 래퍼 (90KB)
- `pages/directory-layout.tsx` - 디렉토리별 레이아웃
- `pages/error.tsx` - 오류 경계 폴백

**컴포넌트 (29개):**
- `dialog-*.tsx` - 모달 다이얼로그
- `prompt-input.tsx` - 대형 프롬프트 입력 컴포넌트 (65KB)
- `session-*.tsx` - 세션 상태 표시기
- `settings-*.tsx` - 설정 패널
- `terminal.tsx` - 통합 터미널 출력
- `file-tree.tsx` - 파일 브라우저 트리 위젯
- `titlebar.tsx` - 커스텀 데스크톱 타이틀바

**컨텍스트 프로바이더 (13개):**

| 컨텍스트 | 용도 |
|---------|------|
| `platform.tsx` | 플랫폼 추상화 (웹 vs 데스크톱) |
| `global-sdk.tsx` | OpenCode SDK 클라이언트 초기화 |
| `server.tsx` | 서버 URL 관리 및 상태 확인 |
| `global-sync.tsx` | 실시간 상태 동기화 |
| `layout.tsx` | 레이아웃 상태 관리 |
| `settings.tsx` | 사용자 설정 지속성 |
| `language.tsx` | i18n 로케일 관리 |
| `file.tsx` | 파일 작업 컨텍스트 |
| `command.tsx` | 명령 팔레트/실행 |
| `terminal.tsx` | 터미널 상태 |
| `prompt.tsx` | 프롬프트 히스토리 |
| `permission.tsx` | 사용자 권한 |
| `notification.tsx` | 데스크톱 알림 |

---

## 2. packages/ui - 공유 컴포넌트 라이브러리

**위치:** `packages/ui`

**목적:** 웹 앱, 데스크톱 앱, 콘솔에서 사용하는 재사용 가능한 UI 컴포넌트 라이브러리.

### 내보내기 구조

```
./components/*.tsx    - UI 컴포넌트
./i18n/*.ts          - 번역 사전
./pierre/*           - Diff 시각화 컴포넌트
./hooks/             - 커스텀 SolidJS 훅
./context/           - 컨텍스트 프로바이더
./styles/            - CSS 스타일시트
./theme/             - 테마 시스템
./icons/             - 아이콘 타입 정의
./fonts/             - 폰트 에셋
./audio/             - 오디오 에셋
```

### 주요 컴포넌트 (50개 이상)

**핵심 컴포넌트:**
- `logo.tsx` - OpenCode 로고
- `button.tsx` - 기본 버튼 컴포넌트
- `icon.tsx` - 포괄적인 아이콘 시스템 (32KB)
- `card.tsx` - 카드 래퍼
- `dialog.tsx` - 모달 다이얼로그 시스템
- `dropdown-menu.tsx` - 컨텍스트 메뉴 (10KB)
- `list.tsx` - 가상화된 목록 컴포넌트 (10KB)

**코드 및 마크다운:**
- `code.tsx` - 구문 강조된 코드 블록
- `markdown.tsx` - 렌더링된 마크다운 표시
- `diff.tsx` - 비주얼 코드 diff
- `diff-changes.tsx` - 파일 변경 추적기

**입력 컴포넌트:**
- `checkbox.tsx` - 체크박스 컨트롤
- `inline-input.tsx` - 인라인 텍스트 편집
- `keybind.tsx` - 키보드 단축키 표시

### 테마 시스템

- `src/theme/`에 위치
- 10개 이상의 내장 테마 지원:
  - Tokyo Night, Dracula, Monokai, Solarized
  - Nord, Catppuccin, Ayu, One Dark Pro
  - Shades of Purple, Night Owl, Vesper
- 색상 변환 유틸리티: RGB ↔ Hex ↔ OkLch
- 스케일 생성으로 동적 테마 생성
- CSS 변수 적용

---

## 3. packages/desktop - Tauri 데스크톱 애플리케이션

**위치:** `packages/desktop`

**목적:** Tauri 2.x를 사용하는 OpenCode용 네이티브 데스크톱 애플리케이션 래퍼.

### 빌드 설정

**Vite 설정:**
- 포트: 1420 (strict)
- app의 vite 플러그인 사용
- `src-tauri/**` 변경 감시
- WebSocket 폴백이 있는 HMR 지원

**Tauri 설정:**
- 앱: "OpenCode Dev"
- 창: 오버레이 타이틀바 스타일 (모던 macOS/Windows 룩)
- 번들러 타겟: DMG (macOS), NSIS (Windows), DEB/RPM (Linux)
- 외부 바이너리: `sidecars/opencode-cli`

### 파일 구조

| 파일 | 용도 |
|------|------|
| `src/index.tsx` | 데스크톱별 진입점 |
| `src/menu.ts` | 네이티브 메뉴 설정 |
| `src/updater.ts` | 자동 업데이트 처리 |
| `src/webview-zoom.ts` | WebView 줌 관리 |
| `src/cli.ts` | CLI 통합 |

### 플랫폼 기능

| 기능 | 설명 |
|------|------|
| `openLink()` | 기본 브라우저에서 URL 열기 |
| `restart()` | 애플리케이션 재시작 |
| `notify()` | 시스템 알림 전송 |
| `openDirectoryPickerDialog()` | 네이티브 파일 피커 |
| `openFilePickerDialog()` | 네이티브 파일 피커 |
| `saveFilePickerDialog()` | 파일 저장 다이얼로그 |
| `checkUpdate()` | 앱 업데이트 확인 |
| `update()` | 대기 중인 업데이트 설치 |
| `storage()` | 영구 키-값 저장소 |
| `fetch()` | 인증 헤더 주입이 포함된 HTTP |

### 의존성

- `@tauri-apps/api` - 핵심 Tauri API
- `@tauri-apps/plugin-*` - 모든 주요 플러그인:
  - `dialog`, `shell`, `os`, `notification`
  - `process`, `updater`, `http`, `store`, `window-state`

---

## 4. packages/web - 문서 사이트

**위치:** `packages/web`

**목적:** Astro와 Starlight 문서 테마를 사용하는 공개 문서 및 랜딩 페이지.

### 빌드 설정

**Astro 설정:**
- 프레임워크: Astro 5.7.13 with Starlight
- 어댑터: Cloudflare (서버 사이드 렌더링)
- 기본 경로: `/docs`
- 통합:
  - `@astrojs/solid-js` - SolidJS 컴포넌트 지원
  - `@astrojs/starlight` - 문서 테마
  - `@astrojs/markdown-remark` - 마크다운 처리

### 콘텐츠 구조

**문서 페이지 (Starlight 통해):**
- Getting started
- Configuration
- Providers
- Network
- Enterprise
- Troubleshooting

**사용법 섹션:**
- TUI, CLI, Web, IDE
- Zen, Share, GitHub, GitLab 통합

**설정 섹션:**
- Tools, Rules, Agents
- Models, Themes, Keybinds, Commands
- Formatters, Permissions
- LSP, MCP Servers, ACP, Skills
- Custom tools

**개발 섹션:**
- SDK, Server
- Plugins, Ecosystem

---

## 5. packages/console/app - 관리 콘솔

**위치:** `packages/console/app`

**목적:** 워크스페이스 관리, 빌링, 설정을 위한 관리/관리 콘솔. SolidStart로 구축.

### 빌드 설정

**Vite 설정:**
- 프레임워크: SolidStart (SolidJS용 메타 프레임워크)
- 백엔드: Nitro (Hono와 유사한 노드 프레임워크)
- 호스팅: Cloudflare Workers

### 파일 구조

**진입점:**
- `src/entry-client.tsx` - 클라이언트 측 하이드레이션
- `src/entry-server.tsx` - 서버 렌더링
- `src/app.tsx` - 라우팅이 있는 루트 앱 컴포넌트

**라우팅 시스템** (`src/routes/`):
- 파일 기반 라우팅 (SolidStart FileRoutes)
- API 라우트 (`api/`)
- 인증 라우트 (`auth/`)
- 워크스페이스 관리 (`workspace/`)
- Stripe 빌링 통합
- 법적/엔터프라이즈 페이지
- 404 핸들러

**주요 라우트:**

| 라우트 | 설명 |
|--------|------|
| `/` | 랜딩 페이지 (65KB) |
| `/workspace` | 워크스페이스 관리 |
| `/workspace-picker` | 서버/워크스페이스 선택 |
| `/auth` | 인증 흐름 |
| `/api` | REST API 엔드포인트 |
| `/stripe` | 빌링 페이지 |

**주요 기능:**
- 서버 사이드 렌더링
- API 라우트 (파일 기반)
- 빌링/Stripe 통합
- 워크스페이스 관리
- 사용자 인증
- 관리 설정

---

## 6. 백엔드 통합 - SDK 및 서버 통신

### SDK 아키텍처

**내보내기:**
- `./` - 메인 SDK
- `./client` - 브라우저/Node 클라이언트
- `./v2/client` - 최신 클라이언트 API
- `./v2/server` - 서버 유틸리티

### 글로벌 SDK 프로바이더 패턴

**App 패키지에서:**
```typescript
// global-sdk.tsx 초기화:
- createOpencodeClient() - 메인 SDK 인스턴스
- /global/event 엔드포인트에서 이벤트 스트리밍
- UI 업데이트를 위한 통합된 이벤트 배치 (16ms)
- 컴포넌트 구독을 위한 글로벌 이벤트 이미터
```

### 사용되는 API 엔드포인트

- `/global/event` - 실시간 업데이트를 위한 Server-Sent Events 스트림
- `/global/health` - 서버 상태 확인
- 다양한 명령/세션/에이전트 엔드포인트 (SDK에서)

### 상태 동기화

**글로벌 싱크 컨텍스트** (`global-sync.tsx`) 유지:
- 에이전트 목록
- 세션
- 파트가 있는 메시지
- 프로젝트
- 명령
- 설정
- 권한
- MCP/LSP 상태
- VCS 정보
- 파일 diff
- Todo 항목

---

## 7. 통합 흐름 다이어그램

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenCode 애플리케이션                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─ WEB (packages/app) ──────────────────────────────────────────┐  │
│  │  entry.tsx → app.tsx                                          │  │
│  │  Router (SolidJS) → Pages (home, session, layout)            │  │
│  │  Providers: Platform, Server, GlobalSDK, GlobalSync           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│  ┌─ DESKTOP (packages/desktop) ──────────────────────────────────┐ │
│  │  packages/app을 감싸는 Tauri 래퍼                              │ │
│  │  index.tsx: OS API가 있는 플랫폼 구현                          │ │
│  │  Menu, Updater, Storage (Tauri Store)                         │ │
│  │  백엔드용 사이드카 CLI                                         │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              ↓                                       │
│  ┌─ UI 컴포넌트 (packages/ui) ───────────────────────────────────┐│
│  │  50개 이상의 재사용 가능한 SolidJS 컴포넌트                    ││
│  │  테마 시스템, 아이콘, 코드 하이라이팅                          ││
│  │  Contexts: Marked, Diff, Code, Dialog, Theme                  ││
│  └──────────────────────────────────────────────────────────────┘│
│                              ↓                                       │
│  ┌─ SDK (packages/sdk/js) ────────────────────────────────────────┐│
│  │  createOpencodeClient() → OpenCode 서버 API                   ││
│  │  SSE 스트리밍이 있는 v2 API                                   ││
│  │  타입 안전 작업                                               ││
│  └──────────────────────────────────────────────────────────────┘│
│                              ↓                                       │
│             ┌────────────────────────────────────┐                  │
│             │  OpenCode 백엔드 (packages/opencode) │                │
│             │  - Hono HTTP 서버                   │                │
│             │  - 에이전트, 세션, 명령              │                │
│             │  - SSE 이벤트 스트리밍              │                │
│             └────────────────────────────────────┘                  │
│                                                                       │
│  ┌─ CONSOLE (packages/console/app) ──────────────────────────────┐ │
│  │  SolidStart 풀스택 앱                                         │ │
│  │  파일 기반 라우팅, Nitro 백엔드                               │ │
│  │  관리 UI, 워크스페이스 관리, 빌링                             │ │
│  │  Cloudflare Workers 호스팅                                    │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌─ WEB DOCS (packages/web) ─────────────────────────────────────┐ │
│  │  Astro + Starlight 문서                                       │ │
│  │  Cloudflare에서 호스팅되는 정적 사이트                         │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. 주요 기술 요약

| 패키지 | 프레임워크 | 빌드 | 상태 관리 | 스타일링 | 호스팅 |
|--------|-----------|------|----------|---------|--------|
| **app** | SolidJS | Vite | Solid Store + Context | TailwindCSS | 클라우드 불가지론 |
| **ui** | SolidJS | Vite | N/A (lib) | TailwindCSS + CSS | NPM 패키지 |
| **desktop** | SolidJS + Tauri | Vite | Solid Store | TailwindCSS | 네이티브 앱 |
| **console** | SolidStart | Vite | Solid Store | CSS modules | Cloudflare Workers |
| **web** | Astro | Astro CLI | N/A (정적) | Tailwind | Cloudflare |

---

## 9. 개발 명령어

```bash
# 모든 패키지는 Bun 런타임 사용
bun run dev           # 개발 서버 시작
bun run typecheck     # 타입 체크
bun run build         # 프로덕션 빌드
bun run test          # 테스트 실행 (app만 Playwright로 e2e)
```
