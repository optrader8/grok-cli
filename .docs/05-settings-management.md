# Grok CLI - 설정 관리 상세 분석

## 목차

1. [개요](#개요)
2. [SettingsManager 클래스](#settingsmanager-클래스)
3. [설정 계층 구조](#설정-계층-구조)
4. [사용자 설정 (User Settings)](#사용자-설정-user-settings)
5. [프로젝트 설정 (Project Settings)](#프로젝트-설정-project-settings)
6. [우선순위 처리](#우선순위-처리)
7. [설정 로드/저장](#설정-로드저장)
8. [모델 관리](#모델-관리)
9. [실제 사용 예](#실제-사용-예)

## 개요

설정 시스템은 사용자 전역 설정과 프로젝트별 설정을 관리합니다. 싱글톤 패턴을 사용하여 애플리케이션 전역에서 접근할 수 있습니다.

**파일 위치**: `src/utils/settings-manager.ts`

**핵심 책임**:
- 사용자 설정 관리
- 프로젝트 설정 관리
- 설정 우선순위 처리
- 모델 및 API 키 관리
- MCP 서버 설정

## SettingsManager 클래스

### 클래스 구조

```typescript
export class SettingsManager {
  private static instance: SettingsManager;

  private userSettingsPath: string;      // ~/.grok/user-settings.json
  private projectSettingsPath: string;   // .grok/settings.json

  private constructor()
  static getInstance(): SettingsManager

  // 사용자 설정
  loadUserSettings(): UserSettings
  saveUserSettings(settings: UserSettings): void
  updateUserSetting(key: string, value: any): void

  // 프로젝트 설정
  loadProjectSettings(): ProjectSettings
  saveProjectSettings(settings: ProjectSettings): void
  updateProjectSetting(key: string, value: any): void

  // 통합 설정
  getCurrentModel(): string
  setCurrentModel(model: string): void
  getAvailableModels(): string[]
}
```

### 싱글톤 패턴

```typescript
export class SettingsManager {
  private static instance: SettingsManager;

  public static getInstance(): SettingsManager {
    if (!SettingsManager.instance) {
      SettingsManager.instance = new SettingsManager();
    }
    return SettingsManager.instance;
  }
}

// 사용법
const manager = SettingsManager.getInstance();
const settings = manager.loadUserSettings();
```

**장점**:
- 메모리 효율성: 인스턴스 하나만 생성
- 일관성: 모든 코드가 동일한 설정 사용
- 스레드 안전성: 중앙화된 접근

## 설정 계층 구조

### 우선순위 (높음 → 낮음)

```
1. 환경 변수
   GROK_API_KEY
   GROK_BASE_URL
   GROK_MODEL
   GROK_MAX_TOKENS

2. 명령줄 플래그
   --api-key
   --base-url
   --model
   --max-tool-rounds

3. 프로젝트 설정
   .grok/settings.json

4. 사용자 설정
   ~/.grok/user-settings.json

5. 기본값 (하드코딩)
   "https://api.x.ai/v1"
   "grok-code-fast-1"
```

### 결정 흐름

```
사용자가 API 키 필요
  │
  ├─ 환경 변수 GROK_API_KEY 있는가?
  │  YES → 사용
  │  NO  ↓
  │
  ├─ 명령줄 --api-key 있는가?
  │  YES → 사용
  │  NO  ↓
  │
  ├─ 프로젝트 설정 apiKey 있는가?
  │  YES → 사용
  │  NO  ↓
  │
  ├─ 사용자 설정 apiKey 있는가?
  │  YES → 사용
  │  NO  ↓
  │
  └─ 사용자 입력 요청
```

## 사용자 설정 (User Settings)

### 저장 위치

```
~/.grok/user-settings.json
```

예: `/home/user/.grok/user-settings.json`

### UserSettings 인터페이스

```typescript
export interface UserSettings {
  apiKey?: string;        // Grok API 키
  baseURL?: string;       // API 엔드포인트 URL
  defaultModel?: string;  // 기본 모델
  models?: string[];      // 사용 가능한 모델 목록
}
```

### 예제

```json
{
  "apiKey": "xai-abc123def456...",
  "baseURL": "https://api.x.ai/v1",
  "defaultModel": "grok-code-fast-1",
  "models": [
    "grok-code-fast-1",
    "grok-4-latest",
    "grok-3-latest",
    "grok-3-fast",
    "grok-3-mini-fast"
  ]
}
```

### 기본값

```typescript
const DEFAULT_USER_SETTINGS: Partial<UserSettings> = {
  baseURL: "https://api.x.ai/v1",
  defaultModel: "grok-code-fast-1",
  models: [
    "grok-code-fast-1",
    "grok-4-latest",
    "grok-3-latest",
    "grok-3-fast",
    "grok-3-mini-fast",
  ],
};
```

### 로드 및 저장

```typescript
// 로드
loadUserSettings(): UserSettings {
  try {
    if (fs.existsSync(this.userSettingsPath)) {
      const content = fs.readFileSync(this.userSettingsPath, 'utf-8');
      return JSON.parse(content);
    }
    return { ...DEFAULT_USER_SETTINGS };
  } catch (error) {
    return { ...DEFAULT_USER_SETTINGS };
  }
}

// 저장
saveUserSettings(settings: UserSettings): void {
  const dir = path.dirname(this.userSettingsPath);
  if (!fs.existsSync(dir)) {
    fs.mkdirSync(dir, { recursive: true });
  }

  fs.writeFileSync(
    this.userSettingsPath,
    JSON.stringify(settings, null, 2),
    { mode: 0o600 }  // 보안: 사용자만 읽을 수 있음
  );
}

// 개별 설정 업데이트
updateUserSetting(key: string, value: any): void {
  const settings = this.loadUserSettings();
  settings[key as keyof UserSettings] = value;
  this.saveUserSettings(settings);
}
```

## 프로젝트 설정 (Project Settings)

### 저장 위치

```
.grok/settings.json  (프로젝트 루트)
```

예: `/path/to/project/.grok/settings.json`

### ProjectSettings 인터페이스

```typescript
export interface ProjectSettings {
  model?: string;                     // 현재 프로젝트 모델
  mcpServers?: Record<string, any>;   // MCP 서버 설정
}
```

### 예제

```json
{
  "model": "grok-4-latest",
  "mcpServers": {
    "linear": {
      "name": "linear",
      "transport": "sse",
      "url": "https://mcp.linear.app/sse"
    },
    "github": {
      "name": "github",
      "transport": "stdio",
      "command": "npx",
      "args": ["@github/mcp-server"]
    }
  }
}
```

### 기본값

```typescript
const DEFAULT_PROJECT_SETTINGS: Partial<ProjectSettings> = {
  model: "grok-code-fast-1",
};
```

### 로드 및 저장

```typescript
// 로드
loadProjectSettings(): ProjectSettings {
  try {
    if (fs.existsSync(this.projectSettingsPath)) {
      const content = fs.readFileSync(this.projectSettingsPath, 'utf-8');
      return JSON.parse(content);
    }
    return { ...DEFAULT_PROJECT_SETTINGS };
  } catch (error) {
    return { ...DEFAULT_PROJECT_SETTINGS };
  }
}

// 저장
saveProjectSettings(settings: ProjectSettings): void {
  const dir = path.dirname(this.projectSettingsPath);
  if (!fs.existsSync(dir)) {
    fs.mkdirSync(dir, { recursive: true });
  }

  fs.writeFileSync(
    this.projectSettingsPath,
    JSON.stringify(settings, null, 2)
  );
}

// 개별 설정 업데이트
updateProjectSetting(key: string, value: any): void {
  const settings = this.loadProjectSettings();
  settings[key as keyof ProjectSettings] = value;
  this.saveProjectSettings(settings);
}
```

## 우선순위 처리

### API 키 결정 로직

```typescript
function getAPIKey(): string {
  // 1. 환경 변수
  if (process.env.GROK_API_KEY) {
    return process.env.GROK_API_KEY;
  }

  // 2. 명령줄 플래그 (args 객체에 저장됨)
  if (this.commandLineArgs?.apiKey) {
    return this.commandLineArgs.apiKey;
  }

  // 3. 프로젝트 설정
  const projectSettings = this.loadProjectSettings();
  if (projectSettings.apiKey) {
    return projectSettings.apiKey;
  }

  // 4. 사용자 설정
  const userSettings = this.loadUserSettings();
  if (userSettings.apiKey) {
    return userSettings.apiKey;
  }

  // 5. 기본값 없음
  throw new Error("API key not found. Set GROK_API_KEY or configure settings.");
}
```

### 모델 결정 로직

```typescript
function getCurrentModel(): string {
  // 1. 환경 변수
  if (process.env.GROK_MODEL) {
    return process.env.GROK_MODEL;
  }

  // 2. 명령줄 플래그
  if (this.commandLineArgs?.model) {
    return this.commandLineArgs.model;
  }

  // 3. 프로젝트 설정
  const projectSettings = this.loadProjectSettings();
  if (projectSettings.model) {
    return projectSettings.model;
  }

  // 4. 사용자 설정
  const userSettings = this.loadUserSettings();
  if (userSettings.defaultModel) {
    return userSettings.defaultModel;
  }

  // 5. 기본값
  return "grok-code-fast-1";
}
```

### Base URL 결정 로직

```typescript
function getBaseURL(): string {
  // 1. 환경 변수
  if (process.env.GROK_BASE_URL) {
    return process.env.GROK_BASE_URL;
  }

  // 2. 명령줄 플래그
  if (this.commandLineArgs?.baseUrl) {
    return this.commandLineArgs.baseUrl;
  }

  // 3. 사용자 설정
  const userSettings = this.loadUserSettings();
  if (userSettings.baseURL) {
    return userSettings.baseURL;
  }

  // 4. 기본값
  return "https://api.x.ai/v1";
}
```

## 설정 로드/저장

### 초기화 프로세스

```
프로그램 시작
  │
  ├─ 환경 변수 로드
  │  (process.env)
  │
  ├─ 명령줄 인자 파싱
  │  (commander로 파싱)
  │
  ├─ SettingsManager 초기화
  │  ├─ 사용자 설정 로드: ~/.grok/user-settings.json
  │  └─ 프로젝트 설정 로드: .grok/settings.json
  │
  └─ GrokClient 생성
     (API 키, Base URL, 모델 결정됨)
```

### 파일 시스템 구조

```
홈 디렉토리 (~)
└── .grok/
    └── user-settings.json          # 사용자 전역 설정

프로젝트 디렉토리
├── .grok/
│   └── settings.json               # 프로젝트 설정
├── src/
├── package.json
└── ...
```

## 모델 관리

### 모델 목록 조회

```typescript
getAvailableModels(): string[] {
  // 1. 환경 변수에서 모델 목록
  const envModels = process.env.GROK_MODELS?.split(',') || [];

  // 2. 사용자 설정에서 모델 목록
  const userSettings = this.loadUserSettings();
  const userModels = userSettings.models || [];

  // 3. 프로젝트 설정에서 현재 모델
  const projectSettings = this.loadProjectSettings();
  const currentModel = projectSettings.model ? [projectSettings.model] : [];

  // 모두 합치기 (중복 제거)
  const allModels = new Set([
    ...envModels,
    ...userModels,
    ...currentModel
  ]);

  return Array.from(allModels);
}
```

### 모델 추가

```typescript
addModel(modelName: string): void {
  const userSettings = this.loadUserSettings();
  const models = userSettings.models || [];

  if (!models.includes(modelName)) {
    models.push(modelName);
    userSettings.models = models;
    this.saveUserSettings(userSettings);
  }
}
```

### 모델 변경

```typescript
setCurrentModel(model: string): void {
  // 프로젝트 설정에 저장
  this.updateProjectSetting('model', model);

  // 모델이 목록에 없으면 추가
  this.addModel(model);
}
```

## 실제 사용 예

### 예 1: 초기 설정

```bash
# API 키 설정
export GROK_API_KEY="xai-..."
export GROK_BASE_URL="https://api.x.ai/v1"
export GROK_MODEL="grok-code-fast-1"

# 또는 설정 파일로
grok --api-key "xai-..." --base-url "..." --model "..."
```

### 예 2: 영구 설정 (사용자 전역)

```bash
# 프로그래밍 방식
const manager = SettingsManager.getInstance();
manager.updateUserSetting('apiKey', 'xai-...');
manager.updateUserSetting('defaultModel', 'grok-4-latest');

# 또는 직접 파일 편집
cat ~/.grok/user-settings.json
{
  "apiKey": "xai-...",
  "baseURL": "https://api.x.ai/v1",
  "defaultModel": "grok-4-latest"
}
```

### 예 3: 프로젝트별 설정

```bash
# 프로젝트 디렉토리에서
mkdir -p .grok
cat > .grok/settings.json <<EOF
{
  "model": "grok-4-latest",
  "mcpServers": {
    "linear": {
      "name": "linear",
      "transport": "sse",
      "url": "https://mcp.linear.app/sse"
    }
  }
}
EOF
```

### 예 4: 로컬 모델로 변경

```typescript
// 로컬 모델 서버 실행 (vLLM)
// python -m vllm.entrypoints.openai.api_server \
//   --model Qwen/Qwen2.5-Coder-32B-Instruct \
//   --port 8000

const manager = SettingsManager.getInstance();

// 옵션 1: 환경 변수
process.env.GROK_BASE_URL = "http://localhost:8000/v1";
process.env.GROK_MODEL = "Qwen/Qwen2.5-Coder-32B-Instruct";

// 옵션 2: 명령줄
// grok --base-url "http://localhost:8000/v1" --model "Qwen/Qwen2.5-Coder-32B-Instruct"

// 옵션 3: 사용자 설정 저장
manager.updateUserSetting('baseURL', 'http://localhost:8000/v1');
manager.updateUserSetting('defaultModel', 'Qwen/Qwen2.5-Coder-32B-Instruct');
manager.addModel('Qwen/Qwen2.5-Coder-32B-Instruct');
```

### 예 5: 모델 선택

```typescript
const manager = SettingsManager.getInstance();
const availableModels = manager.getAvailableModels();

console.log("Available models:", availableModels);
// 출력: ['grok-code-fast-1', 'grok-4-latest', 'grok-3-latest', ...]

// 모델 변경
manager.setCurrentModel('grok-4-latest');

const currentModel = manager.getCurrentModel();
console.log("Current model:", currentModel);
// 출력: 'grok-4-latest'
```

## 보안

### 파일 권한

```typescript
// 사용자 설정 저장 시 0o600 (사용자만 읽기/쓰기)
fs.writeFileSync(
  this.userSettingsPath,
  JSON.stringify(settings, null, 2),
  { mode: 0o600 }  // ← 보안 설정
);
```

### API 키 보호

```
권장사항:
1. 절대 Git에 커밋하지 않기
2. .gitignore에 ~/.grok/ 추가
3. 환경 변수로 관리
4. CI/CD에서 Secrets 사용
```

### .gitignore 예

```
# .gitignore
.grok/settings.json
.env
.env.local
*.key
```

## 설정 검증

### 필수 항목 확인

```typescript
function validateSettings(): boolean {
  const apiKey = this.getAPIKey();
  if (!apiKey) {
    console.error('Error: API key not configured');
    return false;
  }

  const model = this.getCurrentModel();
  if (!model) {
    console.error('Error: Model not configured');
    return false;
  }

  return true;
}
```

## 결론

설정 관리 시스템의 주요 특징:

1. **계층적 우선순위**: 환경 변수 > CLI 플래그 > 프로젝트 > 사용자 > 기본값
2. **유연성**: 다양한 설정 방식 지원
3. **보안**: 파일 권한으로 API 키 보호
4. **확장성**: 새로운 설정 추가 용이
5. **일관성**: 싱글톤 패턴으로 중앙화된 관리
