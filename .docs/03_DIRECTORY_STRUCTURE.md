# 디렉토리 구조

## 📁 전체 구조 개요

```
grok-cli/
├── .github/                  # GitHub 관련 설정
│   ├── workflows/           # CI/CD 워크플로우
│   │   ├── typecheck.yml   # 타입 체크 자동화
│   │   └── security.yml    # 보안 스캔
│   └── ISSUE_TEMPLATE/     # 이슈 템플릿
│
├── src/                     # 소스 코드 (TypeScript)
│   ├── agent/              # AI 에이전트 로직
│   ├── commands/           # CLI 명령어 구현
│   ├── grok/               # Grok API 통합
│   ├── hooks/              # React hooks
│   ├── mcp/                # MCP 통합
│   ├── tools/              # 도구 구현
│   ├── types/              # TypeScript 타입 정의
│   ├── ui/                 # 터미널 UI (React Ink)
│   ├── utils/              # 유틸리티 함수
│   └── index.ts            # 메인 엔트리 포인트
│
├── dist/                    # 컴파일된 출력 (gitignored)
│
├── .grok/                   # 프로젝트 설정 (gitignored)
│   ├── settings.json       # 프로젝트별 설정
│   └── GROK.md            # 커스텀 AI 지침
│
├── package.json            # 패키지 정보 및 의존성
├── tsconfig.json           # TypeScript 설정
├── .eslintrc.js           # ESLint 설정
├── .env.example           # 환경 변수 템플릿
├── .gitignore             # Git 무시 파일
├── LICENSE                # MIT 라이선스
└── README.md              # 프로젝트 문서
```

## 📂 src/ 디렉토리 상세

### 1. agent/ - AI 에이전트 로직

```
agent/
├── grok-agent.ts          # 메인 에이전트 (781 lines)
│                          # - 도구 실행 루프
│                          # - 스트리밍 지원
│                          # - 도구 레지스트리 관리
│                          # - 웹 검색 통합
│                          # - 취소 처리
│
└── index.ts               # 레거시 에이전트 (간단한 명령 처리)
                           # - 기본 명령 파서
                           # - 간단한 응답 생성
```

**역할**:
- 사용자 입력을 AI에게 전달
- AI 응답을 파싱하고 도구 호출 추출
- 도구 실행 및 결과 처리
- 최대 400라운드까지 반복

**핵심 파일**: `grok-agent.ts`

### 2. commands/ - CLI 명령어 구현

```
commands/
└── mcp.ts                 # MCP 서버 관리 명령어
                           # - add: MCP 서버 추가
                           # - list: 서버 목록
                           # - test: 연결 테스트
                           # - remove: 서버 제거
```

**역할**:
- `grok mcp` 명령어 처리
- MCP 서버 CRUD 작업
- 설정 파일 업데이트

### 3. grok/ - Grok API 통합

```
grok/
├── client.ts              # OpenAI 호환 클라이언트 (154 lines)
│                          # - OpenAI SDK 래퍼
│                          # - 스트리밍/논스트리밍 지원
│                          # - 웹 검색 파라미터
│                          # - 커스텀 엔드포인트 지원
│
└── tools.ts               # 도구 정의 (JSON Schema)
                           # - view_file
                           # - create_file
                           # - str_replace_editor
                           # - edit_file (Morph)
                           # - bash
                           # - search
                           # - create_todo_list
                           # - update_todo_list
```

**역할**:
- Grok API 통신
- 도구 스키마 정의 (Function Calling)
- API 요청/응답 처리

**핵심 파일**: `client.ts`, `tools.ts`

### 4. hooks/ - React Hooks

```
hooks/
├── use-input-handler.ts   # 메인 입력 처리 로직
│                          # - 메시지 전송
│                          # - 스트리밍 응답 처리
│                          # - 도구 실행 조율
│                          # - 에러 처리
│
├── use-enhanced-input.ts  # Raw 입력 처리
│                          # - 키보드 입력 캡처
│                          # - 특수 키 처리 (Shift+Tab, Ctrl+C)
│                          # - 입력 검증
│
└── use-input-history.ts   # 명령 히스토리
                           # - 이전 명령 저장
                           # - 화살표 키 네비게이션
                           # - 히스토리 검색
```

**역할**:
- UI와 로직 연결
- 입력 이벤트 처리
- 상태 관리

**핵심 파일**: `use-input-handler.ts`

### 5. mcp/ - Model Context Protocol

```
mcp/
├── client.ts              # MCP 클라이언트 매니저 (170 lines)
│                          # - 다중 서버 관리
│                          # - 도구 레지스트리
│                          # - 연결 상태 관리
│                          # - 도구 실행 라우팅
│
├── config.ts              # MCP 설정 관리
│                          # - 설정 로드/저장
│                          # - 검증
│                          # - 마이그레이션
│
└── transports.ts          # 전송 구현
                           # - StdioTransport (하위 프로세스)
                           # - SSETransport (EventSource)
                           # - HTTPTransport (REST API)
                           # - StreamableHTTPTransport
```

**역할**:
- MCP 서버와의 통신
- 동적 도구 등록
- 다양한 전송 프로토콜 지원

**핵심 파일**: `client.ts`, `transports.ts`

### 6. tools/ - 도구 구현

```
tools/
├── text-editor.ts         # 파일 편집 도구 (669 lines)
│                          # Functions:
│                          # - view_file: 파일/디렉토리 보기
│                          # - create_file: 파일 생성
│                          # - str_replace_editor: 텍스트 교체
│                          # - replace_lines: 라인 교체
│                          # - insert_lines: 라인 삽입
│                          # - undo_edit: 마지막 편집 취소
│                          # - generateDiff: Diff 생성
│
├── morph-editor.ts        # 고속 편집 (393 lines)
│                          # - Morph API 통합
│                          # - Fast Apply 편집
│                          # - 4,500+ tokens/sec
│                          # - 자동 폴백
│
├── bash.ts                # Bash 명령 실행 (86 lines)
│                          # - child_process.exec
│                          # - cd 명령 지원
│                          # - 30초 타임아웃
│                          # - 1MB 버퍼
│
├── search.ts              # 통합 검색 (444 lines)
│                          # - Ripgrep 텍스트 검색
│                          # - 파일명 검색
│                          # - 퍼지 매칭
│                          # - Glob 필터
│
├── todo-tool.ts           # Todo 리스트 (154 lines)
│                          # - create_todo_list
│                          # - update_todo_list
│                          # - 상태: pending/in_progress/completed
│                          # - 우선순위: high/medium/low
│
├── confirmation-tool.ts   # 확인 처리
│                          # - request_confirmation
│                          # - 사용자 응답 대기
│
└── index.ts               # 도구 exports
```

**역할**:
- AI가 호출할 수 있는 기능 구현
- 파일 시스템 조작
- 외부 명령 실행
- 검색 및 분석

**핵심 파일**: `text-editor.ts`, `search.ts`, `bash.ts`

### 7. types/ - TypeScript 타입 정의

```
types/
└── index.ts               # 중앙 타입 정의
                           # Types:
                           # - Message (user/assistant/system/tool)
                           # - ToolCall (도구 호출 정보)
                           # - ToolResult (도구 실행 결과)
                           # - AgentState (에이전트 상태)
                           # - Settings (사용자/프로젝트 설정)
                           # - MCPConfig (MCP 서버 설정)
                           # - ConfirmationData (확인 요청 데이터)
```

**역할**:
- 타입 안전성 보장
- 인터페이스 정의
- 데이터 구조 문서화

### 8. ui/ - 터미널 UI (React Ink)

```
ui/
├── components/            # UI 컴포넌트
│   ├── chat-interface.tsx      # 메인 채팅 UI (416 lines)
│   │                           # - 입력 처리
│   │                           # - 메시지 히스토리
│   │                           # - 확인 대화상자
│   │                           # - 로딩 상태
│   │                           # - 모델 선택
│   │
│   ├── chat-history.tsx        # 메시지 히스토리 표시
│   │                           # - 사용자/AI 메시지 렌더링
│   │                           # - 마크다운 지원
│   │                           # - 코드 구문 강조
│   │                           # - Diff 표시
│   │
│   ├── chat-input.tsx          # 입력 필드 표시
│   │                           # - 프롬프트 표시
│   │                           # - 입력 텍스트 렌더링
│   │
│   ├── confirmation-dialog.tsx # 확인 대화상자
│   │                           # - 변경사항 표시
│   │                           # - 승인/거부 버튼
│   │                           # - "다시 묻지 않기" 옵션
│   │
│   ├── diff-renderer.tsx       # Diff 시각화
│   │                           # - 추가/삭제 라인 색상
│   │                           # - 통합 Diff 형식
│   │
│   ├── model-selection.tsx     # 모델 선택기
│   │                           # - 사용 가능 모델 목록
│   │                           # - 현재 모델 표시
│   │
│   ├── mcp-status.tsx          # MCP 연결 상태
│   │                           # - 연결된 서버 목록
│   │                           # - 연결 상태 아이콘
│   │
│   ├── loading-spinner.tsx     # 로딩 인디케이터
│   │                           # - 애니메이션 스피너
│   │                           # - 토큰 카운트 표시
│   │
│   ├── command-suggestions.tsx # 명령 자동완성
│   │                           # - 명령어 추천
│   │                           # - 단축키 힌트
│   │
│   └── api-key-input.tsx       # API 키 입력
│                               # - 초기 설정 UI
│                               # - 키 검증
│
├── shared/                # 공유 컴포넌트
│   └── max-sized-box.tsx  # 크기 제한 Box
│
├── utils/                 # UI 유틸리티
│   ├── colors.ts          # 색상 정의
│   │                      # - 브랜드 색상
│   │                      # - 상태 색상
│   │                      # - 구문 강조 색상
│   │
│   ├── markdown-renderer.tsx  # 마크다운 렌더링
│   │                          # - marked + marked-terminal
│   │                          # - 코드 블록 처리
│   │
│   └── code-colorizer.tsx     # 구문 강조
│                              # - 언어별 색상
│                              # - 토큰 파싱
│
└── app.tsx                # 루트 UI 컴포넌트
                           # - 앱 초기화
                           # - 전역 상태
                           # - 에러 바운더리
```

**역할**:
- 사용자 인터페이스 렌더링
- 사용자 입력 캡처
- 실시간 UI 업데이트

**핵심 파일**: `chat-interface.tsx`, `chat-history.tsx`

### 9. utils/ - 유틸리티 모듈

```
utils/
├── settings-manager.ts         # 설정 관리 (326 lines)
│                              # - 사용자 설정 (~/.grok/)
│                              # - 프로젝트 설정 (.grok/)
│                              # - 우선순위 병합
│                              # - 파일 권한 관리
│
├── confirmation-service.ts     # 확인 서비스
│                              # - EventEmitter 기반
│                              # - 확인 요청 큐
│                              # - 세션별 플래그
│
├── custom-instructions.ts      # 커스텀 지침 로드
│                              # - .grok/GROK.md 읽기
│                              # - 시스템 프롬프트에 추가
│
├── token-counter.ts            # 토큰 카운팅
│                              # - tiktoken 사용
│                              # - 인코딩 캐싱
│                              # - 실시간 카운팅
│
├── model-config.ts             # 모델 설정
│                              # - 모델별 파라미터
│                              # - 토큰 제한
│
├── settings.ts                 # 설정 유틸리티
│                              # - 설정 검증
│                              # - 기본값
│
└── text-utils.ts               # 텍스트 처리
                               # - 문자열 조작
                               # - 퍼지 매칭
                               # - 라인 처리
```

**역할**:
- 공통 기능 제공
- 설정 관리
- 헬퍼 함수

**핵심 파일**: `settings-manager.ts`, `confirmation-service.ts`

### 10. index.ts - 메인 엔트리 포인트

```
index.ts                   # CLI 진입점 (463 lines)
                          # - Commander.js 설정
                          # - 명령어 등록
                          # - API 키 로드
                          # - 모드 선택 (interactive/headless/git)
                          # - 시그널 핸들링
```

**역할**:
- CLI 파싱
- 앱 초기화
- 모드별 실행

## 📄 루트 레벨 설정 파일

### package.json

```json
{
  "name": "@vibe-kit/grok-cli",
  "version": "0.0.33",
  "type": "module",
  "bin": {
    "grok": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsx src/index.ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": { },
  "devDependencies": { }
}
```

**역할**: 패키지 정보, 의존성, 스크립트 정의

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": false,
    "esModuleInterop": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "jsx": "react"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**역할**: TypeScript 컴파일 설정

### .eslintrc.js

```javascript
export default [
  {
    languageOptions: {
      parser: tsParser,
      parserOptions: {
        ecmaVersion: 'latest',
        sourceType: 'module'
      }
    },
    plugins: {
      '@typescript-eslint': tsPlugin
    },
    rules: {
      // ESLint 규칙
    }
  }
];
```

**역할**: 코드 품질 검사 규칙

### .env.example

```bash
GROK_API_KEY=your-api-key-here
GROK_BASE_URL=https://api.x.ai/v1
GROK_MODEL=grok-code-fast-1
MORPH_API_KEY=your-morph-api-key-here
```

**역할**: 환경 변수 템플릿

### .gitignore

```
node_modules/
dist/
.env
.grok/
*.log
.DS_Store
```

**역할**: Git에서 제외할 파일 지정

## 🗂️ 사용자 디렉토리 구조

### ~/.grok/ (사용자 전역 설정)

```
~/.grok/
└── user-settings.json    # 사용자 설정
                          # {
                          #   "apiKey": "xai-...",
                          #   "baseURL": "https://api.x.ai/v1",
                          #   "defaultModel": "grok-code-fast-1",
                          #   "models": [...]
                          # }
```

**권한**: `0o600` (소유자만 읽기/쓰기)

### .grok/ (프로젝트 로컬 설정)

```
project/.grok/
├── settings.json         # 프로젝트 설정
│                         # {
│                         #   "model": "grok-3-fast",
│                         #   "mcpServers": {...}
│                         # }
│
└── GROK.md              # 커스텀 AI 지침
                         # - 프로젝트별 코딩 스타일
                         # - 특수 요구사항
                         # - AI 동작 지침
```

**Git**: 일반적으로 `.gitignore`에 추가

## 📊 파일 통계

### 코드 라인 수 (추정)

```
src/
├── agent/          ~800 lines
├── commands/       ~100 lines
├── grok/           ~250 lines
├── hooks/          ~200 lines
├── mcp/            ~400 lines
├── tools/          ~1,900 lines
├── types/          ~200 lines
├── ui/             ~2,000 lines
├── utils/          ~700 lines
└── index.ts        ~463 lines
─────────────────────────────
Total:              ~6,231 lines
```

### 파일 수

- **TypeScript 파일**: ~40개
- **설정 파일**: ~6개
- **문서**: README.md, LICENSE

## 🔍 주요 파일 빠른 참조

| 기능 | 파일 경로 | 라인 수 |
|------|----------|--------|
| 메인 에이전트 | `src/agent/grok-agent.ts` | 781 |
| 파일 편집 | `src/tools/text-editor.ts` | 669 |
| 통합 검색 | `src/tools/search.ts` | 444 |
| 메인 UI | `src/ui/components/chat-interface.tsx` | 416 |
| 고속 편집 | `src/tools/morph-editor.ts` | 393 |
| 설정 관리 | `src/utils/settings-manager.ts` | 326 |
| MCP 클라이언트 | `src/mcp/client.ts` | 170 |
| Grok API | `src/grok/client.ts` | 154 |
| Todo 도구 | `src/tools/todo-tool.ts` | 154 |
| Bash 실행 | `src/tools/bash.ts` | 86 |

## 🎯 파일 찾기 팁

### 기능별 위치

- **AI 로직 수정**: `src/agent/grok-agent.ts`
- **새 도구 추가**: `src/tools/` + `src/grok/tools.ts`
- **UI 변경**: `src/ui/components/`
- **설정 로직**: `src/utils/settings-manager.ts`
- **MCP 관련**: `src/mcp/`
- **타입 정의**: `src/types/index.ts`
- **CLI 명령**: `src/index.ts`, `src/commands/`

### 일반적인 수정 시나리오

| 하고 싶은 것 | 수정할 파일 |
|-------------|-----------|
| 새 CLI 옵션 추가 | `src/index.ts` |
| 도구 실행 로직 변경 | `src/agent/grok-agent.ts` |
| 파일 편집 기능 개선 | `src/tools/text-editor.ts` |
| 검색 알고리즘 개선 | `src/tools/search.ts` |
| UI 레이아웃 변경 | `src/ui/components/chat-interface.tsx` |
| 확인 대화상자 수정 | `src/ui/components/confirmation-dialog.tsx` |
| MCP 서버 추가 명령 | `src/commands/mcp.ts` |
| 새 전송 방식 구현 | `src/mcp/transports.ts` |
| 색상 테마 변경 | `src/ui/utils/colors.ts` |
| 타입 추가/수정 | `src/types/index.ts` |
