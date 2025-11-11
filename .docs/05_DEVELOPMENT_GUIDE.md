# 개발 가이드

이 문서는 Grok CLI 개발을 위한 환경 설정, 빌드, 테스트, 배포 방법을 안내합니다.

## 📋 목차

1. [개발 환경 설정](#1-개발-환경-설정)
2. [프로젝트 구조 이해](#2-프로젝트-구조-이해)
3. [빌드 및 실행](#3-빌드-및-실행)
4. [개발 워크플로우](#4-개발-워크플로우)
5. [디버깅](#5-디버깅)
6. [테스트](#6-테스트)
7. [배포](#7-배포)
8. [코딩 가이드라인](#8-코딩-가이드라인)
9. [문제 해결](#9-문제-해결)

---

## 1. 개발 환경 설정

### 필수 요구사항

```bash
# Node.js 18 이상
node --version  # v18.0.0+

# 또는 Bun (권장)
bun --version   # 1.0.0+

# Git
git --version

# 시스템 의존성
ripgrep --version  # rg 명령어
```

### 개발 도구 설치

#### Option 1: Bun (권장)

```bash
# Bun 설치
curl -fsSL https://bun.sh/install | bash

# 의존성 설치
bun install

# 장점: 더 빠른 설치 및 실행
```

#### Option 2: npm/yarn

```bash
# npm
npm install

# 또는 yarn
yarn install
```

### Ripgrep 설치

검색 기능에 필수입니다.

```bash
# macOS
brew install ripgrep

# Ubuntu/Debian
sudo apt install ripgrep

# Fedora
sudo dnf install ripgrep

# Windows (Scoop)
scoop install ripgrep

# 설치 확인
rg --version
```

### VS Code 설정 (선택사항)

```json
// .vscode/settings.json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "files.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.grok": true
  }
}
```

### 환경 변수 설정

```bash
# .env 파일 생성
cp .env.example .env

# .env 편집
GROK_API_KEY=xai-your-api-key-here
GROK_BASE_URL=https://api.x.ai/v1
GROK_MODEL=grok-code-fast-1
MORPH_API_KEY=your-morph-api-key  # 선택사항
```

---

## 2. 프로젝트 구조 이해

### 핵심 디렉토리

```
src/
├── agent/          # AI 에이전트 로직 수정
├── tools/          # 새 도구 추가
├── ui/components/  # UI 수정
├── mcp/            # MCP 관련 작업
└── utils/          # 유틸리티 추가
```

### 개발 시작 체크리스트

- [ ] `src/index.ts` - CLI 엔트리 포인트 이해
- [ ] `src/agent/grok-agent.ts` - 에이전트 로직 이해
- [ ] `src/tools/` - 도구 구현 방법 이해
- [ ] `src/ui/components/` - UI 컴포넌트 구조 파악
- [ ] `src/types/index.ts` - 타입 정의 확인

---

## 3. 빌드 및 실행

### 개발 모드

```bash
# TypeScript를 직접 실행 (tsx 사용)
bun run dev

# 또는 npm
npm run dev

# 특정 명령어로 시작
bun run dev "show me package.json"
bun run dev -d /path/to/project
```

### 빌드

```bash
# TypeScript 컴파일
bun run build

# 출력: dist/ 디렉토리
# - dist/index.js (진입점)
# - dist/**/*.js (모든 모듈)
# - dist/**/*.d.ts (타입 정의)
# - dist/**/*.js.map (소스맵)
```

### 빌드된 버전 실행

```bash
# 직접 실행
node dist/index.js

# 또는 npm start
npm start
```

### 로컬 설치 (글로벌)

```bash
# 현재 디렉토리를 글로벌 링크
npm link

# 또는 bun
bun link

# 이제 어디서든 사용 가능
grok "hello world"

# 언링크
npm unlink -g @vibe-kit/grok-cli
```

---

## 4. 개발 워크플로우

### 일반적인 개발 사이클

```bash
# 1. 브랜치 생성
git checkout -b feature/my-new-feature

# 2. 코드 수정
# ...

# 3. 타입 체크
bun run typecheck

# 4. 린트
bun run lint

# 5. 빌드
bun run build

# 6. 테스트 (수동)
node dist/index.js --prompt "test my changes"

# 7. 커밋
git add .
git commit -m "feat: add new feature"

# 8. 푸시
git push origin feature/my-new-feature
```

### 핫 리로드 개발

```bash
# tsx watch 모드
npx tsx watch src/index.ts

# 파일 변경 시 자동 재시작
```

---

## 5. 디버깅

### VS Code 디버그 설정

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Grok CLI",
      "runtimeExecutable": "tsx",
      "runtimeArgs": ["--inspect-brk", "src/index.ts"],
      "args": ["--prompt", "hello world"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Debug with Directory",
      "runtimeExecutable": "tsx",
      "runtimeArgs": ["--inspect-brk", "src/index.ts"],
      "args": ["-d", "/path/to/test/project"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal"
    }
  ]
}
```

### 로깅

```typescript
// 개발 중 디버그 로그 추가
if (process.env.DEBUG) {
  console.log('[DEBUG] Agent state:', this.conversationHistory);
}

// 실행 시
DEBUG=1 bun run dev
```

### MCP 서버 디버깅

```bash
# MCP 서버 테스트
grok mcp test <server-name>

# 연결 상태 확인
grok mcp list

# 수동으로 도구 호출 테스트
# (AI에게 특정 MCP 도구 사용 요청)
grok "use the linear server to list my issues"
```

### 네트워크 디버깅

```bash
# API 요청 로그
export NODE_DEBUG=http,https
bun run dev

# 또는 프록시 사용
export HTTPS_PROXY=http://localhost:8080
bun run dev
```

---

## 6. 테스트

### 타입 체크

```bash
# TypeScript 타입 체크 (빌드 없이)
bun run typecheck

# 또는
npx tsc --noEmit
```

### 린팅

```bash
# ESLint 실행
bun run lint

# 자동 수정
npx eslint --fix src/
```

### 수동 테스트

```bash
# 헤드리스 모드로 빠른 테스트
grok --prompt "view package.json"
grok --prompt "search for TODO"
grok --prompt "run ls -la"

# 대화형 모드 테스트
grok "hello"  # 간단한 응답 테스트
grok "create a test file"  # 파일 생성 테스트
grok "search for main function"  # 검색 테스트
```

### 통합 테스트 시나리오

```bash
# 1. 파일 편집 테스트
grok --prompt "view src/index.ts lines 1-20"
grok --prompt "create test.txt with content 'hello world'"
grok --prompt "replace 'hello' with 'hi' in test.txt"

# 2. 검색 테스트
grok --prompt "search for 'export function' in src/"
grok --prompt "find files matching '*.ts' in src/tools/"

# 3. MCP 테스트
grok mcp add test-server -t stdio -c "bun" -a "server.js"
grok "list all tools available"

# 4. Git 테스트
grok git commit-and-push
```

### CI/CD 테스트

GitHub Actions에서 자동으로 실행됩니다:

```yaml
# .github/workflows/typecheck.yml
- run: bun run typecheck
- run: bun run lint
```

---

## 7. 배포

### NPM 배포 준비

```bash
# 1. 버전 업데이트
npm version patch  # 0.0.33 -> 0.0.34
npm version minor  # 0.0.33 -> 0.1.0
npm version major  # 0.0.33 -> 1.0.0

# 2. 빌드
bun run build

# 3. 빌드 검증
node dist/index.js --version

# 4. 패키지 검증
npm pack
tar -xzf vibe-kit-grok-cli-*.tgz
ls package/
```

### NPM 배포

```bash
# 로그인 (최초 1회)
npm login

# 배포
npm publish --access public

# 특정 태그로 배포
npm publish --tag beta
```

### 배포 후 확인

```bash
# 설치 테스트
npm install -g @vibe-kit/grok-cli@latest

# 실행 확인
grok --version
grok --help
```

### GitHub Release

```bash
# 태그 생성
git tag -a v0.0.34 -m "Release v0.0.34"
git push origin v0.0.34

# GitHub에서 Release 생성
# - Changelog 작성
# - 빌드 아티팩트 첨부 (선택)
```

---

## 8. 코딩 가이드라인

### TypeScript 스타일

```typescript
// ✅ 좋은 예
export interface UserConfig {
  apiKey: string;
  model?: string;
  maxTokens?: number;
}

export async function processMessage(
  message: string,
  config: UserConfig
): Promise<string> {
  // 구현
}

// ❌ 나쁜 예
export async function process(msg: any, cfg: any): Promise<any> {
  // any 타입 사용 지양
}
```

### 파일 명명 규칙

```
kebab-case.ts         # 파일명
PascalCase            # 클래스, 인터페이스, 타입
camelCase             # 함수, 변수
UPPER_SNAKE_CASE      # 상수
```

### Import 순서

```typescript
// 1. Node.js 내장 모듈
import fs from 'fs';
import path from 'path';

// 2. 외부 패키지
import { Command } from 'commander';
import OpenAI from 'openai';

// 3. 내부 모듈
import { GrokAgent } from './agent/grok-agent.js';
import { createTools } from './tools/index.js';

// 4. 타입
import type { Message, ToolResult } from './types/index.js';
```

### 에러 처리

```typescript
// ✅ 구체적인 에러 메시지
try {
  const result = await executeTool(toolCall);
} catch (error) {
  console.error(`Failed to execute tool ${toolCall.name}:`, error);
  return {
    output: `Error: ${error.message}\nTool: ${toolCall.name}`
  };
}

// ❌ 불분명한 에러 처리
try {
  const result = await executeTool(toolCall);
} catch (error) {
  console.error(error);  // 불충분한 정보
}
```

### 비동기 코드

```typescript
// ✅ async/await 사용
async function loadConfig(): Promise<Config> {
  const data = await fs.promises.readFile('config.json', 'utf-8');
  return JSON.parse(data);
}

// ✅ 병렬 실행
const [user, project] = await Promise.all([
  loadUserSettings(),
  loadProjectSettings()
]);

// ❌ Promise 체인 (레거시)
function loadConfig(): Promise<Config> {
  return fs.promises.readFile('config.json', 'utf-8')
    .then(data => JSON.parse(data));
}
```

### 주석

```typescript
// ✅ 복잡한 로직에만 주석
// Fuzzy matching to handle whitespace differences in multi-line strings
const normalizedTarget = normalizeWhitespace(target);

// ❌ 불필요한 주석
// Get the file path
const filePath = args.path;
```

### 커밋 메시지

```bash
# Conventional Commits 사용
feat: add new search feature
fix: resolve ripgrep timeout issue
docs: update API documentation
refactor: simplify tool execution logic
test: add unit tests for text-editor
chore: update dependencies

# 상세 설명 추가
feat: add fuzzy file name search

Implements fuzzy matching algorithm for file names
- Scores based on character sequence matching
- Integrates with existing search tool
- Max 50 results to prevent performance issues

Closes #123
```

---

## 9. 문제 해결

### 일반적인 문제

#### 1. "ripgrep not found" 에러

```bash
# 해결: ripgrep 설치
brew install ripgrep  # macOS
sudo apt install ripgrep  # Ubuntu

# 확인
which rg
```

#### 2. "API key not found" 에러

```bash
# 해결: API 키 설정
export GROK_API_KEY=xai-your-key

# 또는 .env 파일에 추가
echo "GROK_API_KEY=xai-your-key" >> .env

# 확인
echo $GROK_API_KEY
```

#### 3. MCP 서버 연결 실패

```bash
# 디버그 모드로 실행
DEBUG=mcp grok

# 서버 설정 확인
cat .grok/settings.json

# 서버 재시작
grok mcp remove <server-name>
grok mcp add <server-name> ...
```

#### 4. TypeScript 컴파일 에러

```bash
# 의존성 재설치
rm -rf node_modules package-lock.json
bun install

# 타입 정의 업데이트
bun add -D @types/node@latest
```

#### 5. "Module not found" 에러

```bash
# .js 확장자 확인 (ES modules 필수)
# ❌ import { foo } from './bar'
# ✅ import { foo } from './bar.js'

# tsconfig.json 확인
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler"
  }
}
```

### 디버깅 체크리스트

- [ ] Node.js/Bun 버전 확인
- [ ] 의존성 설치 완료
- [ ] 환경 변수 설정
- [ ] 빌드 성공
- [ ] 타입 체크 통과
- [ ] Ripgrep 설치 확인
- [ ] API 키 유효성
- [ ] 네트워크 연결

### 퍼포먼스 최적화

```typescript
// 1. 캐싱 활용
const cache = new Map();
function expensiveOperation(key: string) {
  if (cache.has(key)) {
    return cache.get(key);
  }
  const result = /* ... */;
  cache.set(key, result);
  return result;
}

// 2. 병렬 처리
const results = await Promise.all(
  items.map(item => processItem(item))
);

// 3. 스트리밍 사용
for await (const chunk of stream) {
  processChunk(chunk);  // 메모리 효율적
}
```

### 메모리 관리

```typescript
// 큰 파일 처리 시 스트림 사용
import { createReadStream } from 'fs';

const stream = createReadStream('large-file.txt');
stream.on('data', (chunk) => {
  // 청크별 처리
});

// 캐시 크기 제한
const MAX_CACHE_SIZE = 100;
if (cache.size > MAX_CACHE_SIZE) {
  const firstKey = cache.keys().next().value;
  cache.delete(firstKey);
}
```

---

## 🎓 추가 리소스

### 내부 문서

- [아키텍처](02_ARCHITECTURE.md) - 시스템 설계 이해
- [핵심 컴포넌트](04_CORE_COMPONENTS.md) - 주요 파일 분석
- [API 통합](06_API_INTEGRATIONS.md) - 외부 서비스 연동

### 외부 리소스

- [TypeScript 공식 문서](https://www.typescriptlang.org/docs/)
- [React Ink 문서](https://github.com/vadimdemedes/ink)
- [MCP SDK](https://github.com/modelcontextprotocol/sdk)
- [OpenAI API](https://platform.openai.com/docs/api-reference)

### 커뮤니티

- GitHub Issues: 버그 리포트 및 기능 요청
- Pull Requests: 코드 기여

---

## 📝 개발 팁

1. **작게 시작하기**: 먼저 작은 기능이나 버그 수정부터 시작
2. **테스트 자주 하기**: 코드 변경 후 즉시 테스트
3. **타입 활용하기**: TypeScript의 타입 시스템을 최대한 활용
4. **문서 업데이트**: 코드 변경 시 관련 문서도 함께 업데이트
5. **코드 리뷰**: Pull Request 전에 스스로 코드 리뷰

## 🚀 다음 단계

1. 개발 환경 설정 완료
2. 간단한 기능 추가해보기
3. MCP 서버 만들어보기
4. 커스텀 도구 구현하기
5. 커뮤니티에 기여하기
