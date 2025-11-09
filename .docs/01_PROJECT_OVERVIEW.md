# 프로젝트 개요

## 🎯 프로젝트 소개

**Grok CLI**는 X.AI의 Grok을 기반으로 한 대화형 AI 명령줄 인터페이스 도구입니다. 터미널 환경에서 AI 에이전트와 상호작용하며 파일 편집, 코드 검색, 명령 실행 등의 작업을 자연어로 수행할 수 있습니다.

### 기본 정보

- **패키지명**: `@vibe-kit/grok-cli`
- **현재 버전**: 0.0.33
- **라이선스**: MIT
- **저장소**: GitHub
- **최소 요구사항**: Node.js ≥ 18.0.0
- **권장 런타임**: Bun 1.0+ (더 빠른 성능)

## 💡 핵심 가치

Grok CLI는 다음과 같은 가치를 제공합니다:

1. **자연어 코딩**: 복잡한 파일 편집 작업을 자연어로 수행
2. **AI 기반 자동화**: 반복적인 개발 작업을 AI가 자동으로 처리
3. **확장성**: Model Context Protocol(MCP)을 통한 무한한 확장 가능성
4. **안전성**: 모든 파일 변경 및 명령 실행 전 사용자 확인
5. **빠른 성능**: 스트리밍과 병렬 처리로 최적화된 성능

## 🚀 주요 기능

### 1. 지능형 파일 편집

```typescript
// AI가 자연어 명령을 이해하고 파일을 편집합니다
"Add error handling to the API endpoint in src/api.ts"
"Refactor the UserService class to use dependency injection"
"Fix all TypeScript errors in the project"
```

**지원 편집 작업**:
- 파일 보기 (View)
- 파일 생성 (Create)
- 텍스트 교체 (String Replace)
- 라인 교체 (Replace Lines)
- 라인 삽입 (Insert)
- 실행 취소 (Undo)
- Fast Apply (Morph API 사용 시 4,500+ tokens/sec)

### 2. 통합 코드 검색

```bash
# 파일 내용과 파일명을 동시에 검색 (Cursor 스타일)
"Find all authentication related code"
"Search for TODO comments"
```

**검색 기능**:
- Ripgrep 기반 고속 텍스트 검색
- 퍼지 파일명 검색
- 정규식 지원
- Glob 패턴 필터링
- 대소문자 구분 옵션
- 최대 50개 결과 (기본값)

### 3. Bash 명령 실행

```bash
"Run the test suite"
"Install the missing dependencies"
"Check the git status"
```

**특징**:
- 안전한 명령 실행 (사용자 확인 필수)
- `cd` 명령 지원 (프로세스 디렉토리 변경)
- 30초 타임아웃 (기본값)
- 1MB 출력 버퍼

### 4. Model Context Protocol (MCP) 통합

외부 도구와 서비스를 AI에게 제공:

```bash
# Linear 이슈 관리
grok mcp add linear -t sse -u https://mcp.linear.app/sse

# GitHub 통합
grok mcp add github -t stdio -c "npx" -a "-y @modelcontextprotocol/server-github"

# 커스텀 MCP 서버
grok mcp add myserver -t stdio -c "bun" -a "server.js"
```

**지원 전송 방식**:
- **stdio**: 하위 프로세스로 실행
- **SSE**: Server-Sent Events (HTTP 스트리밍)
- **HTTP**: REST API
- **streamable-http**: 스트리밍 HTTP

### 5. 대화형 Todo 리스트

AI가 작업을 관리하고 진행 상황을 시각화:

```
○ 대기 중 (Pending)
◐ 진행 중 (In Progress)
● 완료 (Completed)
```

**우선순위 레벨**: High, Medium, Low

### 6. 다중 AI 모델 지원

OpenAI 호환 API를 지원하는 모든 서비스와 호환:

**지원 모델 예시**:
- X.AI: `grok-code-fast-1`, `grok-4-latest`, `grok-3-latest`
- OpenAI: `gpt-4-turbo`, `gpt-3.5-turbo`
- Anthropic: `claude-3-opus`, `claude-3-sonnet` (via OpenRouter)
- Groq: `llama3-70b-8192`, `mixtral-8x7b-32768`
- 기타 OpenAI 호환 서비스

## 📋 사용 모드

### 1. 대화형 모드 (Interactive Mode)

```bash
grok                                    # 기본 대화형 모드 시작
grok -d /path/to/project               # 특정 디렉토리에서 시작
grok "initial message"                 # 초기 메시지와 함께 시작
grok --model grok-code-fast-1          # 특정 모델 지정
```

**인터랙티브 기능**:
- 실시간 스트리밍 응답
- 명령 히스토리 (화살표 키)
- 자동 편집 모드 (Shift+Tab)
- 모델 전환 (`/model`)
- 작업 취소 (Ctrl+C)
- 도움말 (`/help`)

### 2. 헤드리스 모드 (Headless Mode)

CI/CD 파이프라인이나 스크립트에서 사용:

```bash
grok --prompt "show me package.json"
grok -p "create a test file" -d /project
grok --prompt "run tests" --max-tool-rounds 50
```

**특징**:
- 모든 작업 자동 승인
- 단일 프롬프트 실행
- 스크립트 친화적
- 반환 코드 지원

### 3. Git 명령 모드

```bash
grok git commit-and-push    # AI가 변경사항을 분석하고 커밋 메시지 생성
```

## 🎨 사용자 인터페이스

### 터미널 UI (React Ink 기반)

```
┌─────────────────────────────────────┐
│  Grok CLI v0.0.33                  │
│  Model: grok-code-fast-1           │
│  MCP: ● linear  ● github           │
├─────────────────────────────────────┤
│                                     │
│  User: Help me fix the bug in...   │
│                                     │
│  Grok: I'll help you fix that.     │
│  Let me first examine the file...  │
│                                     │
│  [Tool: view_file]                 │
│  ✓ Completed                       │
│                                     │
│  📝 Todo List:                     │
│  ● Examine error logs              │
│  ◐ Fix the null pointer issue      │
│  ○ Run tests                       │
│                                     │
├─────────────────────────────────────┤
│  > _                               │
└─────────────────────────────────────┘
```

**UI 요소**:
- 실시간 메시지 스트리밍
- 구문 강조된 코드 블록
- 마크다운 렌더링
- Diff 시각화
- 확인 대화상자
- 로딩 스피너 (토큰 카운트 포함)
- MCP 연결 상태
- Todo 리스트 표시

## 🔧 CLI 명령어

### 기본 옵션

```bash
grok [options] [initial-message]
```

**옵션**:
- `-k, --api-key <key>` - Grok API 키
- `-u, --base-url <url>` - 커스텀 API 엔드포인트
- `-m, --model <model>` - AI 모델 선택
- `-d, --directory <path>` - 작업 디렉토리
- `-p, --prompt <text>` - 헤드리스 모드 프롬프트
- `--max-tool-rounds <n>` - 도구 실행 제한 (기본: 400)
- `-h, --help` - 도움말 표시
- `-V, --version` - 버전 표시

### MCP 관리 명령어

```bash
# MCP 서버 추가 (stdio)
grok mcp add <name> -t stdio -c <command> -a <args>

# MCP 서버 추가 (SSE)
grok mcp add <name> -t sse -u <url>

# MCP 서버 추가 (HTTP)
grok mcp add <name> -t http -u <url>

# JSON 설정으로 추가
grok mcp add-json <name> '{"command": "node", "args": ["server.js"]}'

# MCP 서버 목록
grok mcp list

# MCP 서버 테스트
grok mcp test <name>

# MCP 서버 제거
grok mcp remove <name>
```

## 📁 설정 파일

### 1. 사용자 설정 (`~/.grok/user-settings.json`)

전역 사용자 설정:

```json
{
  "apiKey": "xai-...",
  "baseURL": "https://api.x.ai/v1",
  "defaultModel": "grok-code-fast-1",
  "models": [
    "grok-code-fast-1",
    "grok-4-latest",
    "grok-3-latest"
  ]
}
```

### 2. 프로젝트 설정 (`.grok/settings.json`)

프로젝트별 설정:

```json
{
  "model": "grok-3-fast",
  "mcpServers": {
    "linear": {
      "name": "linear",
      "transport": {
        "type": "sse",
        "url": "https://mcp.linear.app/sse"
      }
    }
  }
}
```

### 3. 커스텀 지침 (`.grok/GROK.md`)

프로젝트별 AI 동작 지침:

```markdown
# Project Guidelines

- Use TypeScript strict mode
- Follow Airbnb style guide
- Write unit tests for all new features
- Use conventional commits
```

### 4. 환경 변수 (`.env`)

```bash
GROK_API_KEY=xai-...               # 필수
GROK_BASE_URL=https://api.x.ai/v1 # 선택
GROK_MODEL=grok-code-fast-1       # 선택
MORPH_API_KEY=morph-...            # 선택 (Fast Apply용)
GROK_MAX_TOKENS=1536               # 선택
```

## 🎯 일반적인 사용 사례

### 1. 코드 리팩토링

```
User: "Refactor the authentication module to use async/await instead of promises"

Grok: I'll help you refactor the authentication code. Let me first examine the current implementation...

[AI examines files, makes changes, runs tests]
```

### 2. 버그 수정

```
User: "There's a memory leak in the WebSocket handler, can you fix it?"

Grok: I'll investigate the WebSocket handler for memory leaks...

[AI analyzes code, identifies issues, proposes fixes]
```

### 3. 새 기능 추가

```
User: "Add rate limiting to the API endpoints using express-rate-limit"

Grok: I'll add rate limiting to your API. This involves:
○ Installing express-rate-limit
○ Creating rate limiter middleware
○ Applying to routes
○ Adding tests

[AI implements feature step by step]
```

### 4. 문서 작성

```
User: "Generate API documentation for all endpoints"

Grok: I'll create comprehensive API documentation...

[AI extracts route definitions, generates markdown docs]
```

### 5. 테스트 작성

```
User: "Write unit tests for the UserService class"

Grok: I'll create comprehensive tests for UserService...

[AI analyzes class, writes tests, runs them]
```

## 🔒 보안 및 안전성

### 사용자 확인 시스템

모든 파일 작업과 Bash 명령은 실행 전 확인 필요:

```
┌─────────────────────────────────────┐
│  Confirmation Required              │
├─────────────────────────────────────┤
│  Tool: str_replace_editor           │
│  File: src/api/auth.ts              │
│                                     │
│  Changes:                           │
│  + Add error handling               │
│  + Improve validation               │
│                                     │
│  ✓ Approve  ✗ Reject               │
│  □ Don't ask again this session    │
└─────────────────────────────────────┘
```

### 보안 기능

1. **API 키 보호**: 파일 권한 `0o600`으로 저장
2. **명령 검증**: Bash 명령 실행 전 검증
3. **자동 편집 모드**: 세션별 선택적 활성화
4. **헤드리스 모드**: CI/CD에서만 자동 승인
5. **환경 분리**: MCP 서버용 독립적인 환경 변수

## 📊 성능 특징

### 스트리밍 아키텍처

- 첫 토큰까지의 시간 단축
- 실시간 토큰 카운팅 (250ms마다 업데이트)
- 도구 호출 조기 생성 (함수명 확인 즉시)
- AbortController를 통한 우아한 취소

### Fast Apply (Morph API)

- **속도**: 4,500+ tokens/sec
- **형식**: `<instruction>`, `<code>`, `<update>` 태그
- **자동 폴백**: API 키 없으면 일반 편집으로 전환

### 병렬 처리

- MCP 서버 백그라운드 초기화
- 가능한 경우 도구 병렬 실행
- 토큰 카운팅 캐싱

## 🌍 호환성

### 운영체제

- ✅ Linux
- ✅ macOS
- ⚠️ Windows (일부 기능 제한 가능)

### 런타임

- ✅ Node.js 18+
- ✅ Bun 1.0+ (권장)

### AI 제공자

- ✅ X.AI (네이티브)
- ✅ OpenAI
- ✅ OpenRouter
- ✅ Groq
- ✅ 기타 OpenAI 호환 API

## 📈 프로젝트 통계

- **코드 라인 수**: ~6,231 lines
- **TypeScript**: 100%
- **React 컴포넌트**: 15+
- **도구 구현**: 8개 (+ MCP 동적 도구)
- **최대 도구 실행**: 400 라운드 (기본값)
- **최대 토큰**: 1,536 (기본값)

## 🎓 학습 리소스

- **아키텍처**: [02_ARCHITECTURE.md](02_ARCHITECTURE.md)
- **개발 가이드**: [05_DEVELOPMENT_GUIDE.md](05_DEVELOPMENT_GUIDE.md)
- **API 통합**: [06_API_INTEGRATIONS.md](06_API_INTEGRATIONS.md)
- **핵심 컴포넌트**: [04_CORE_COMPONENTS.md](04_CORE_COMPONENTS.md)

## 🤝 커뮤니티

- **이슈 리포팅**: GitHub Issues
- **기여**: Pull Requests 환영
- **라이선스**: MIT (자유롭게 사용 가능)
