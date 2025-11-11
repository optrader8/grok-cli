# Grok CLI - 커스터마이징 가이드

이 문서는 Grok CLI를 로컬 모델(gpt-oss-20b, qwen3-coder 등)로 커스터마이징하기 위한 기술 문서입니다.

## 목차

1. [프로젝트 구조 분석](#프로젝트-구조-분석)
2. [API 호출 메커니즘](#api-호출-메커니즘)
3. [로컬 모델 통합 방법](#로컬-모델-통합-방법)
4. [설정 관리 시스템](#설정-관리-시스템)
5. [커스터마이징 로드맵](#커스터마이징-로드맵)

## 프로젝트 구조 분석

### 핵심 디렉토리 구조

```
grok-cli/
├── src/
│   ├── agent/              # AI 에이전트 로직
│   │   ├── grok-agent.ts   # 메인 에이전트 (대화 처리, 도구 실행)
│   │   └── index.ts
│   ├── grok/               # Grok API 클라이언트
│   │   ├── client.ts       # OpenAI SDK 기반 API 클라이언트
│   │   └── tools.ts        # 사용 가능한 도구 정의
│   ├── tools/              # 개별 도구 구현
│   │   ├── text-editor.ts  # 파일 편집 도구
│   │   ├── bash.ts         # Bash 명령 실행
│   │   ├── morph-editor.ts # Morph 빠른 편집
│   │   ├── search.ts       # 파일 검색
│   │   └── todo-tool.ts    # TODO 관리
│   ├── ui/                 # 터미널 UI (Ink 기반)
│   ├── utils/              # 유틸리티
│   │   ├── settings-manager.ts  # 설정 관리자
│   │   ├── model-config.ts      # 모델 설정
│   │   └── token-counter.ts     # 토큰 카운팅
│   ├── mcp/                # Model Context Protocol 서버
│   └── index.ts            # CLI 엔트리 포인트
├── .grok/                  # 프로젝트별 설정
│   └── settings.json
└── ~/.grok/                # 사용자 전역 설정
    └── user-settings.json
```

### 주요 컴포넌트

#### 1. `src/grok/client.ts` - API 클라이언트
**역할**: Grok API와의 통신을 담당

**핵심 코드**:
```typescript
export class GrokClient {
  private client: OpenAI;
  private currentModel: string = "grok-code-fast-1";

  constructor(apiKey: string, model?: string, baseURL?: string) {
    this.client = new OpenAI({
      apiKey,
      baseURL: baseURL || process.env.GROK_BASE_URL || "https://api.x.ai/v1",
      timeout: 360000,
    });
  }

  async chat(
    messages: GrokMessage[],
    tools?: GrokTool[],
    model?: string
  ): Promise<GrokResponse> {
    const response = await this.client.chat.completions.create({
      model: model || this.currentModel,
      messages,
      tools: tools || [],
      temperature: 0.7,
      max_tokens: this.defaultMaxTokens,
    });
    return response as GrokResponse;
  }
}
```

**중요 포인트**:
- OpenAI SDK를 사용 (OpenAI 호환 API)
- `baseURL`을 통해 API 엔드포인트 변경 가능
- 스트리밍과 일반 채팅 모두 지원

#### 2. `src/agent/grok-agent.ts` - 에이전트
**역할**: 사용자 메시지 처리 및 도구 실행 관리

**핵심 기능**:
- 메시지 처리 루프 (최대 400라운드)
- 도구 호출 및 실행
- 채팅 히스토리 관리
- 스트리밍 응답 처리

#### 3. `src/utils/settings-manager.ts` - 설정 관리
**역할**: 사용자 및 프로젝트 설정 관리

**설정 우선순위**:
1. 환경 변수 (`GROK_API_KEY`, `GROK_BASE_URL`, `GROK_MODEL`)
2. 명령줄 플래그 (`--api-key`, `--base-url`, `--model`)
3. 프로젝트 설정 (`.grok/settings.json`)
4. 사용자 설정 (`~/.grok/user-settings.json`)
5. 시스템 기본값

#### 4. `src/index.ts` - CLI 엔트리
**역할**: CLI 인터페이스 및 명령 처리

**주요 명령**:
- `grok` - 대화형 모드
- `grok --prompt "..."` - 헤드리스 모드
- `grok git commit-and-push` - Git 자동화
- `grok mcp` - MCP 서버 관리

## API 호출 메커니즘

### OpenAI SDK 기반 구조

Grok CLI는 OpenAI SDK를 사용하여 API를 호출합니다. 이는 다음을 의미합니다:

1. **표준 OpenAI 메시지 형식 사용**
   ```typescript
   interface Message {
     role: "system" | "user" | "assistant" | "tool";
     content: string;
     tool_calls?: ToolCall[];
   }
   ```

2. **도구 호출(Function Calling) 지원**
   ```typescript
   interface GrokTool {
     type: "function";
     function: {
       name: string;
       description: string;
       parameters: {
         type: "object";
         properties: Record<string, any>;
         required: string[];
       };
     };
   }
   ```

3. **스트리밍 응답 지원**
   - `chatStream()` 메서드로 실시간 응답 처리
   - 청크 단위로 토큰 카운팅

### 데이터 흐름

```
사용자 입력
  ↓
ChatInterface (UI)
  ↓
GrokAgent.processUserMessage()
  ↓
GrokClient.chat() / chatStream()
  ↓
OpenAI SDK
  ↓
API 엔드포인트 (baseURL)
  ↓
응답 스트림
  ↓
도구 실행 (필요시)
  ↓
최종 응답 반환
```

## 로컬 모델 통합 방법

### 방법 1: OpenAI 호환 서버 사용

가장 간단한 방법은 로컬 모델을 OpenAI 호환 API로 제공하는 것입니다.

#### 추천 도구:
1. **vLLM** - 고성능 LLM 서빙
2. **LM Studio** - GUI 기반 로컬 모델 서버
3. **Ollama** - 간편한 로컬 모델 실행
4. **text-generation-webui** (oobabooga) - OpenAI 호환 API 제공

#### 예: vLLM으로 gpt-oss-20b 서빙

```bash
# vLLM 설치
pip install vllm

# OpenAI 호환 서버 실행
python -m vllm.entrypoints.openai.api_server \
  --model gpt-oss/gpt-oss-20b \
  --host 0.0.0.0 \
  --port 8000
```

#### Grok CLI 설정

**방법 1: 환경 변수**
```bash
export GROK_API_KEY="not-needed"  # 또는 빈 문자열
export GROK_BASE_URL="http://localhost:8000/v1"
export GROK_MODEL="gpt-oss/gpt-oss-20b"

grok
```

**방법 2: 사용자 설정 파일**
`~/.grok/user-settings.json`:
```json
{
  "apiKey": "not-needed",
  "baseURL": "http://localhost:8000/v1",
  "defaultModel": "gpt-oss/gpt-oss-20b",
  "models": [
    "gpt-oss/gpt-oss-20b",
    "Qwen/Qwen2.5-Coder-32B-Instruct",
    "deepseek-ai/DeepSeek-Coder-V2-Instruct"
  ]
}
```

**방법 3: 명령줄 플래그**
```bash
grok --api-key "not-needed" \
     --base-url "http://localhost:8000/v1" \
     --model "gpt-oss/gpt-oss-20b"
```

### 방법 2: Ollama 사용

Ollama는 로컬 모델 실행을 매우 간단하게 만들어줍니다.

```bash
# Ollama 설치
curl -fsSL https://ollama.com/install.sh | sh

# 모델 다운로드 및 실행
ollama pull qwen2.5-coder:32b
ollama serve
```

Grok CLI 설정:
```bash
export GROK_API_KEY="ollama"
export GROK_BASE_URL="http://localhost:11434/v1"
export GROK_MODEL="qwen2.5-coder:32b"

grok
```

### 방법 3: LM Studio 사용

1. LM Studio 다운로드 및 설치
2. 원하는 모델 다운로드 (예: qwen3-coder)
3. "Local Server" 탭에서 서버 시작 (포트: 1234)

Grok CLI 설정:
```bash
export GROK_BASE_URL="http://localhost:1234/v1"
export GROK_MODEL="qwen3-coder"
grok
```

## 설정 관리 시스템

### 설정 파일 위치

#### 1. 사용자 설정 (`~/.grok/user-settings.json`)
전역 설정으로 모든 프로젝트에 적용됩니다.

```json
{
  "apiKey": "your-api-key",
  "baseURL": "http://localhost:8000/v1",
  "defaultModel": "gpt-oss/gpt-oss-20b",
  "models": [
    "gpt-oss/gpt-oss-20b",
    "Qwen/Qwen2.5-Coder-32B-Instruct"
  ]
}
```

#### 2. 프로젝트 설정 (`.grok/settings.json`)
프로젝트별 설정으로 사용자 설정을 오버라이드합니다.

```json
{
  "model": "Qwen/Qwen2.5-Coder-32B-Instruct",
  "mcpServers": {
    "linear": {
      "name": "linear",
      "transport": "stdio",
      "command": "npx",
      "args": ["@linear/mcp-server"]
    }
  }
}
```

### 설정 우선순위

```
명령줄 플래그 (--model)
  ↓
환경 변수 (GROK_MODEL)
  ↓
프로젝트 설정 (.grok/settings.json)
  ↓
사용자 기본 설정 (~/.grok/user-settings.json)
  ↓
시스템 기본값 (grok-code-fast-1)
```

### SettingsManager API

```typescript
const manager = getSettingsManager();

// API 키 가져오기
const apiKey = manager.getApiKey();

// Base URL 가져오기
const baseURL = manager.getBaseURL();

// 현재 모델 가져오기
const currentModel = manager.getCurrentModel();

// 사용 가능한 모델 목록
const models = manager.getAvailableModels();

// 설정 업데이트
manager.updateUserSetting('defaultModel', 'new-model');
manager.updateProjectSetting('model', 'project-model');
```

## 커스터마이징 로드맵

### Phase 1: 기본 로컬 모델 통합 ✅ (현재)

**목표**: Grok API 대신 로컬 모델 사용

**작업 항목**:
- [x] 코드베이스 분석
- [x] API 호출 메커니즘 이해
- [ ] vLLM/Ollama 서버 설정
- [ ] 환경 변수 및 설정 파일 구성
- [ ] 기본 동작 테스트

**예상 소요 시간**: 1-2시간

### Phase 2: 다중 모델 지원

**목표**: 여러 로컬 모델 간 쉽게 전환

**작업 항목**:
- [ ] 모델 프리셋 생성 (gpt-oss-20b, qwen3-coder, deepseek-coder 등)
- [ ] UI에 모델 선택 기능 추가
- [ ] 모델별 파라미터 튜닝 (temperature, max_tokens 등)
- [ ] 모델 성능 벤치마킹

**예상 소요 시간**: 3-4시간

### Phase 3: 도구 호출 최적화

**목표**: 로컬 모델의 도구 호출 성능 향상

**작업 항목**:
- [ ] 로컬 모델의 function calling 능력 평가
- [ ] 필요시 프롬프트 엔지니어링으로 도구 호출 개선
- [ ] ReAct 패턴 구현 (모델이 도구 호출을 잘 못하는 경우)
- [ ] 도구 사용 로그 및 디버깅

**예상 소요 시간**: 4-6시간

### Phase 4: 성능 최적화

**목표**: 로컬 모델의 응답 속도 및 품질 향상

**작업 항목**:
- [ ] 토큰 카운팅 최적화 (로컬 모델에 맞게)
- [ ] 컨텍스트 윈도우 관리
- [ ] 배치 처리 및 캐싱
- [ ] GPU 메모리 최적화

**예상 소요 시간**: 5-8시간

### Phase 5: 고급 기능

**목표**: 추가 기능 및 사용자 경험 개선

**작업 항목**:
- [ ] 모델 앙상블 (여러 모델 동시 사용)
- [ ] RAG (Retrieval-Augmented Generation) 통합
- [ ] 커스텀 도구 플러그인 시스템
- [ ] 웹 UI 개발 (선택사항)

**예상 소요 시간**: 10-15시간

## 기술적 고려사항

### 1. 도구 호출 (Function Calling)

로컬 모델은 Grok/GPT-4와 달리 도구 호출 능력이 제한적일 수 있습니다.

**해결 방법**:
- **Qwen2.5-Coder**: 네이티브 function calling 지원
- **기타 모델**: 프롬프트 엔지니어링으로 JSON 출력 유도

```typescript
// 도구 호출을 위한 프롬프트 예시
const systemPrompt = `
You are an AI assistant that can use tools.
When you need to use a tool, output JSON in this format:
{
  "tool": "view_file",
  "arguments": {
    "path": "/path/to/file"
  }
}
`;
```

### 2. 컨텍스트 윈도우

로컬 모델의 컨텍스트 윈도우는 다양합니다:
- GPT-OSS-20B: ~8K 토큰
- Qwen2.5-Coder-32B: ~32K 토큰
- DeepSeek-Coder-V2: ~128K 토큰

**관리 방법**:
```typescript
// src/grok/client.ts에서 모델별 max_tokens 설정
const MODEL_CONFIGS = {
  'gpt-oss/gpt-oss-20b': { maxTokens: 2048, contextWindow: 8192 },
  'Qwen/Qwen2.5-Coder-32B-Instruct': { maxTokens: 4096, contextWindow: 32768 },
  'deepseek-ai/DeepSeek-Coder-V2-Instruct': { maxTokens: 8192, contextWindow: 131072 }
};
```

### 3. 토큰 카운팅

현재 `tiktoken` 라이브러리는 OpenAI 모델용입니다. 로컬 모델용으로 교체가 필요할 수 있습니다.

**대안**:
```typescript
// 간단한 토큰 추정 (정확도는 떨어지지만 빠름)
function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}

// 모델별 토크나이저 사용
import { AutoTokenizer } from '@xenova/transformers';
const tokenizer = await AutoTokenizer.from_pretrained('Qwen/Qwen2.5-Coder-32B-Instruct');
```

### 4. 스트리밍

OpenAI SDK의 스트리밍은 대부분의 OpenAI 호환 서버에서 작동합니다.

**확인 사항**:
- vLLM: ✅ 스트리밍 지원
- Ollama: ✅ 스트리밍 지원
- LM Studio: ✅ 스트리밍 지원

## 코드 수정 포인트

### 최소한의 수정으로 로컬 모델 사용하기

실제로는 **코드 수정 없이** 환경 변수만으로 로컬 모델을 사용할 수 있습니다:

```bash
# 1. 로컬 서버 실행 (예: vLLM)
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-Coder-32B-Instruct \
  --port 8000

# 2. Grok CLI 실행
export GROK_API_KEY="local"
export GROK_BASE_URL="http://localhost:8000/v1"
export GROK_MODEL="Qwen/Qwen2.5-Coder-32B-Instruct"
grok
```

### 선택적 개선 사항

더 나은 사용자 경험을 위해 다음 파일들을 수정할 수 있습니다:

#### 1. `src/utils/settings-manager.ts`
로컬 모델 프리셋 추가:

```typescript
const DEFAULT_USER_SETTINGS: Partial<UserSettings> = {
  baseURL: "http://localhost:8000/v1",  // 로컬 서버로 변경
  defaultModel: "Qwen/Qwen2.5-Coder-32B-Instruct",
  models: [
    "Qwen/Qwen2.5-Coder-32B-Instruct",
    "gpt-oss/gpt-oss-20b",
    "deepseek-ai/DeepSeek-Coder-V2-Instruct",
  ],
};
```

#### 2. `src/grok/client.ts`
모델별 설정 추가:

```typescript
const MODEL_CONFIGS: Record<string, { maxTokens: number; temperature: number }> = {
  'Qwen/Qwen2.5-Coder-32B-Instruct': {
    maxTokens: 4096,
    temperature: 0.3  // 코드 생성에 최적화
  },
  'gpt-oss/gpt-oss-20b': {
    maxTokens: 2048,
    temperature: 0.7
  },
};

constructor(apiKey: string, model?: string, baseURL?: string) {
  // ...
  const config = MODEL_CONFIGS[model || this.currentModel];
  if (config) {
    this.defaultMaxTokens = config.maxTokens;
  }
}
```

#### 3. `src/index.ts`
로컬 모델 사용시 API 키 검증 완화:

```typescript
// API 키 검증 수정
const apiKey = options.apiKey || loadApiKey() || "local-model";

// 로컬 서버 사용시 경고 제거
if (baseURL?.includes('localhost') || baseURL?.includes('127.0.0.1')) {
  console.log('🏠 Using local model server at', baseURL);
}
```

## 디버깅 및 문제 해결

### 로깅 활성화

```bash
# 환경 변수로 디버그 모드
export DEBUG=grok:*
export GROK_LOG_LEVEL=debug

grok
```

### 일반적인 문제

#### 1. 연결 오류
```
Error: Grok API error: connect ECONNREFUSED 127.0.0.1:8000
```

**해결책**: 로컬 서버가 실행 중인지 확인
```bash
curl http://localhost:8000/v1/models
```

#### 2. 도구 호출 실패
```
Error: Tool execution error: ...
```

**해결책**: 모델이 function calling을 지원하는지 확인. 지원하지 않으면 ReAct 패턴 구현 필요.

#### 3. 토큰 제한 초과
```
Error: Token limit exceeded
```

**해결책**: `GROK_MAX_TOKENS` 조정 또는 대화 히스토리 정리
```bash
export GROK_MAX_TOKENS=2048
```

## 참고 자료

### 로컬 모델 서빙
- [vLLM 문서](https://docs.vllm.ai/)
- [Ollama 문서](https://ollama.com/docs)
- [LM Studio](https://lmstudio.ai/)

### 추천 모델
- **Qwen2.5-Coder-32B**: 코드 생성 및 도구 호출에 우수
- **DeepSeek-Coder-V2**: 긴 컨텍스트 지원
- **GPT-OSS-20B**: 오픈소스 대안

### OpenAI 호환 API
- [OpenAI API 레퍼런스](https://platform.openai.com/docs/api-reference)
- [vLLM OpenAI 호환 서버](https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html)

## 다음 단계

1. **로컬 서버 설정**: vLLM 또는 Ollama로 원하는 모델 실행
2. **환경 변수 구성**: `GROK_BASE_URL`과 `GROK_MODEL` 설정
3. **테스트**: 기본 명령으로 동작 확인
4. **최적화**: 모델별 파라미터 튜닝
5. **고급 기능**: 커스텀 도구 및 프롬프트 추가

궁금한 점이나 문제가 발생하면 이슈를 열어주세요!
