# Grok CLI - API 클라이언트 상세 분석

## 목차

1. [개요](#개요)
2. [GrokClient 클래스](#grokclient-클래스)
3. [OpenAI SDK 통합](#openai-sdk-통합)
4. [메시지 형식](#메시지-형식)
5. [도구 호출 (Function Calling)](#도구-호출-function-calling)
6. [스트리밍 구현](#스트리밍-구현)
7. [웹 검색 통합](#웹-검색-통합)
8. [에러 처리](#에러-처리)
9. [로컬 모델 호환성](#로컬-모델-호환성)

## 개요

`GrokClient` 클래스는 Grok CLI와 AI 모델 간의 모든 통신을 담당합니다. OpenAI SDK를 사용하여 OpenAI 호환 API와 통신하므로, Grok 뿐만 아니라 로컬 모델 서버와도 호환됩니다.

**파일 위치**: `src/grok/client.ts`

**핵심 기능**:
- OpenAI 호환 API 통신
- 스트리밍 및 블로킹 응답 지원
- 도구 호출 (Function Calling) 지원
- 웹 검색 통합
- 모델 전환

## GrokClient 클래스

### 클래스 구조

```typescript
export class GrokClient {
  private client: OpenAI;                    // OpenAI SDK 클라이언트
  private currentModel: string;              // 현재 사용 중인 모델
  private defaultMaxTokens: number;          // 최대 토큰 수

  constructor(apiKey: string, model?: string, baseURL?: string)
  setModel(model: string): void
  getCurrentModel(): string
  async chat(...): Promise<GrokResponse>
  async *chatStream(...): AsyncGenerator<any, void, unknown>
  async search(...): Promise<GrokResponse>
}
```

### 생성자 (Constructor)

```typescript
constructor(apiKey: string, model?: string, baseURL?: string) {
  this.client = new OpenAI({
    apiKey,
    baseURL: baseURL || process.env.GROK_BASE_URL || "https://api.x.ai/v1",
    timeout: 360000,  // 6분 타임아웃
  });

  // 환경 변수에서 최대 토큰 수 로드
  const envMax = Number(process.env.GROK_MAX_TOKENS);
  this.defaultMaxTokens = Number.isFinite(envMax) && envMax > 0
    ? envMax
    : 1536;  // 기본값: 1536

  if (model) {
    this.currentModel = model;
  }
}
```

**매개변수**:
- `apiKey`: API 인증 키
- `model`: 사용할 모델 (선택사항, 기본값: `grok-code-fast-1`)
- `baseURL`: API 엔드포인트 (선택사항, 기본값: `https://api.x.ai/v1`)

**환경 변수**:
- `GROK_BASE_URL`: API 엔드포인트 URL
- `GROK_MAX_TOKENS`: 최대 생성 토큰 수

**타임아웃**:
- `360000ms` (6분) - 복잡한 작업에 충분한 시간 제공

### 모델 관리 메서드

#### setModel()

```typescript
setModel(model: string): void {
  this.currentModel = model;
}
```

런타임에 모델을 변경할 수 있습니다.

**사용 예**:
```typescript
grokClient.setModel("grok-4-latest");
grokClient.setModel("Qwen/Qwen2.5-Coder-32B-Instruct");  // 로컬 모델
```

#### getCurrentModel()

```typescript
getCurrentModel(): string {
  return this.currentModel;
}
```

현재 사용 중인 모델 이름을 반환합니다.

## OpenAI SDK 통합

### OpenAI 클라이언트 초기화

```typescript
this.client = new OpenAI({
  apiKey,           // 인증 키
  baseURL,          // API 엔드포인트
  timeout: 360000,  // 타임아웃 (ms)
});
```

### API 호환성

GrokClient는 **OpenAI Chat Completions API**를 사용합니다. 따라서 다음 API와 호환됩니다:

| 제공자 | Base URL | 비고 |
|--------|----------|------|
| Grok (X.AI) | `https://api.x.ai/v1` | 기본값 |
| OpenAI | `https://api.openai.com/v1` | GPT-4 등 |
| OpenRouter | `https://openrouter.ai/api/v1` | 여러 모델 통합 |
| Groq | `https://api.groq.com/openai/v1` | Llama 등 |
| vLLM | `http://localhost:8000/v1` | 로컬 모델 |
| Ollama | `http://localhost:11434/v1` | 로컬 모델 |
| LM Studio | `http://localhost:1234/v1` | 로컬 모델 |

### 요청 페이로드 구조

```typescript
const requestPayload = {
  model: "grok-code-fast-1",
  messages: [
    { role: "system", content: "You are a helpful assistant..." },
    { role: "user", content: "Hello!" },
    { role: "assistant", content: "Hi there!" },
    { role: "user", content: "Create a file called test.txt" }
  ],
  tools: [
    {
      type: "function",
      function: {
        name: "create_file",
        description: "Create a new file",
        parameters: {
          type: "object",
          properties: {
            path: { type: "string" },
            content: { type: "string" }
          },
          required: ["path", "content"]
        }
      }
    }
  ],
  tool_choice: "auto",    // 도구 사용 여부를 AI가 결정
  temperature: 0.7,       // 생성 무작위성 (0-2)
  max_tokens: 1536,       // 최대 생성 토큰
  stream: false           // 스트리밍 여부
};
```

## 메시지 형식

### GrokMessage 타입

```typescript
export type GrokMessage = ChatCompletionMessageParam;

// OpenAI SDK 타입
type ChatCompletionMessageParam =
  | ChatCompletionSystemMessageParam      // role: "system"
  | ChatCompletionUserMessageParam        // role: "user"
  | ChatCompletionAssistantMessageParam   // role: "assistant"
  | ChatCompletionToolMessageParam;       // role: "tool"
```

### 메시지 역할 (Roles)

#### 1. System Message

시스템 지침을 정의합니다.

```typescript
{
  role: "system",
  content: `You are Grok CLI, an AI assistant that helps with coding tasks.
You have access to these tools:
- view_file: View file contents
- create_file: Create new files
- str_replace_editor: Edit existing files
...`
}
```

#### 2. User Message

사용자 입력입니다.

```typescript
{
  role: "user",
  content: "Create a file called example.js with a hello world function"
}
```

#### 3. Assistant Message

AI의 응답입니다. 도구 호출을 포함할 수 있습니다.

```typescript
// 일반 응답
{
  role: "assistant",
  content: "I'll create that file for you."
}

// 도구 호출 포함
{
  role: "assistant",
  content: null,
  tool_calls: [
    {
      id: "call_abc123",
      type: "function",
      function: {
        name: "create_file",
        arguments: '{"path":"example.js","content":"function hello() { ... }"}'
      }
    }
  ]
}
```

#### 4. Tool Message

도구 실행 결과입니다.

```typescript
{
  role: "tool",
  content: "File created successfully: example.js",
  tool_call_id: "call_abc123"  // 해당 도구 호출 ID
}
```

### 대화 흐름 예시

```typescript
const messages: GrokMessage[] = [
  // 1. 시스템 프롬프트
  {
    role: "system",
    content: "You are a coding assistant with file editing tools."
  },

  // 2. 사용자 요청
  {
    role: "user",
    content: "Create a file called test.txt with 'Hello World'"
  },

  // 3. AI 응답 (도구 호출)
  {
    role: "assistant",
    content: null,
    tool_calls: [{
      id: "call_1",
      type: "function",
      function: {
        name: "create_file",
        arguments: '{"path":"test.txt","content":"Hello World"}'
      }
    }]
  },

  // 4. 도구 실행 결과
  {
    role: "tool",
    content: "File created successfully",
    tool_call_id: "call_1"
  },

  // 5. AI 최종 응답
  {
    role: "assistant",
    content: "I've created test.txt with the content 'Hello World'."
  }
];
```

## 도구 호출 (Function Calling)

### GrokTool 타입

```typescript
export interface GrokTool {
  type: "function";
  function: {
    name: string;                    // 도구 이름
    description: string;             // 도구 설명
    parameters: {
      type: "object";
      properties: Record<string, any>;  // 매개변수 스키마
      required: string[];              // 필수 매개변수
    };
  };
}
```

### 도구 정의 예시

```typescript
const CREATE_FILE_TOOL: GrokTool = {
  type: "function",
  function: {
    name: "create_file",
    description: "Create a new file with the given content",
    parameters: {
      type: "object",
      properties: {
        path: {
          type: "string",
          description: "The file path"
        },
        content: {
          type: "string",
          description: "The file content"
        }
      },
      required: ["path", "content"]
    }
  }
};
```

### 도구 호출 흐름

```
1. API 요청 시 도구 목록 전송
   ├─ tools: [CREATE_FILE_TOOL, EDIT_FILE_TOOL, ...]
   └─ tool_choice: "auto"

2. AI가 도구 사용 필요성 판단
   ├─ 필요 없음 → 일반 텍스트 응답
   └─ 필요함 → tool_calls 배열 생성

3. tool_calls 반환
   {
     id: "call_123",
     type: "function",
     function: {
       name: "create_file",
       arguments: '{"path":"test.txt","content":"..."}'
     }
   }

4. 클라이언트가 도구 실행
   ├─ arguments를 JSON 파싱
   ├─ 해당 도구 함수 호출
   └─ 결과 생성

5. 결과를 tool 메시지로 다시 전송
   {
     role: "tool",
     content: "Success",
     tool_call_id: "call_123"
   }

6. AI가 최종 응답 생성
```

### tool_choice 옵션

```typescript
const requestPayload = {
  // ...
  tool_choice: "auto",  // AI가 자동 결정 (권장)
  // tool_choice: "none",  // 도구 사용 안 함
  // tool_choice: { type: "function", function: { name: "create_file" } },  // 특정 도구 강제
};
```

## 스트리밍 구현

### chat() vs chatStream()

#### chat() - 블로킹 방식

```typescript
async chat(
  messages: GrokMessage[],
  tools?: GrokTool[],
  model?: string,
  searchOptions?: SearchOptions
): Promise<GrokResponse> {
  const response = await this.client.chat.completions.create({
    model: model || this.currentModel,
    messages,
    tools: tools || [],
    tool_choice: tools && tools.length > 0 ? "auto" : undefined,
    temperature: 0.7,
    max_tokens: this.defaultMaxTokens,
    stream: false  // 스트리밍 비활성화
  });

  return response as GrokResponse;
}
```

**특징**:
- 전체 응답이 완료될 때까지 대기
- 간단한 구현
- 헤드리스 모드에 적합

#### chatStream() - 논블로킹 방식

```typescript
async *chatStream(
  messages: GrokMessage[],
  tools?: GrokTool[],
  model?: string,
  searchOptions?: SearchOptions
): AsyncGenerator<any, void, unknown> {
  const stream = await this.client.chat.completions.create({
    model: model || this.currentModel,
    messages,
    tools: tools || [],
    tool_choice: tools && tools.length > 0 ? "auto" : undefined,
    temperature: 0.7,
    max_tokens: this.defaultMaxTokens,
    stream: true  // 스트리밍 활성화
  });

  for await (const chunk of stream) {
    yield chunk;
  }
}
```

**특징**:
- 청크 단위로 실시간 수신
- 빠른 사용자 피드백
- 대화형 모드에 적합

### 스트리밍 청크 구조

```typescript
// 첫 번째 청크 (델타 시작)
{
  id: "chatcmpl-abc123",
  object: "chat.completion.chunk",
  created: 1234567890,
  model: "grok-code-fast-1",
  choices: [{
    index: 0,
    delta: {
      role: "assistant",
      content: ""
    },
    finish_reason: null
  }]
}

// 중간 청크들 (콘텐츠 스트리밍)
{
  choices: [{
    delta: {
      content: "I'll "
    }
  }]
}

{
  choices: [{
    delta: {
      content: "create "
    }
  }]
}

// 도구 호출 청크들
{
  choices: [{
    delta: {
      tool_calls: [{
        index: 0,
        id: "call_123",
        type: "function",
        function: {
          name: "create_file",
          arguments: ""
        }
      }]
    }
  }]
}

{
  choices: [{
    delta: {
      tool_calls: [{
        index: 0,
        function: {
          arguments: '{"path"'
        }
      }]
    }
  }]
}

// 최종 청크
{
  choices: [{
    delta: {},
    finish_reason: "tool_calls"  // 또는 "stop"
  }]
}
```

### 청크 누적 (Accumulation)

GrokAgent에서 청크를 누적하는 로직:

```typescript
private messageReducer(previous: any, item: any): any {
  const reduce = (acc: any, delta: any) => {
    acc = { ...acc };

    for (const [key, value] of Object.entries(delta)) {
      if (acc[key] === undefined || acc[key] === null) {
        // 새로운 필드: 그대로 추가
        acc[key] = value;
      } else if (typeof acc[key] === "string" && typeof value === "string") {
        // 문자열: 연결
        acc[key] += value;
      } else if (Array.isArray(acc[key]) && Array.isArray(value)) {
        // 배열: 재귀적으로 병합
        const accArray = acc[key];
        for (let i = 0; i < value.length; i++) {
          if (!accArray[i]) accArray[i] = {};
          accArray[i] = reduce(accArray[i], value[i]);
        }
      } else if (typeof acc[key] === "object" && typeof value === "object") {
        // 객체: 재귀적으로 병합
        acc[key] = reduce(acc[key], value);
      }
    }

    return acc;
  };

  return reduce(previous, item.choices[0]?.delta || {});
}
```

**사용 예**:
```typescript
let accumulatedMessage = {};

for await (const chunk of stream) {
  accumulatedMessage = this.messageReducer(accumulatedMessage, chunk);

  // 완성된 도구 호출 확인
  if (accumulatedMessage.tool_calls?.length > 0) {
    const hasCompleteTool = accumulatedMessage.tool_calls.some(
      tc => tc.function?.name
    );
    if (hasCompleteTool) {
      // 도구 실행
      await executeTool(accumulatedMessage.tool_calls[0]);
    }
  }
}
```

## 웹 검색 통합

### SearchParameters 인터페이스

```typescript
export interface SearchParameters {
  mode?: "auto" | "on" | "off";
  // sources는 제거됨 (API 기본값 사용)
}

export interface SearchOptions {
  search_parameters?: SearchParameters;
}
```

### search() 메서드

```typescript
async search(
  query: string,
  searchParameters?: SearchParameters
): Promise<GrokResponse> {
  const searchMessage: GrokMessage = {
    role: "user",
    content: query,
  };

  const searchOptions: SearchOptions = {
    search_parameters: searchParameters || { mode: "on" },
  };

  return this.chat([searchMessage], [], undefined, searchOptions);
}
```

### 에이전트에서의 웹 검색 사용

```typescript
// 웹 검색이 필요한 키워드 감지
private shouldUseSearchFor(message: string): boolean {
  const q = message.toLowerCase();
  const keywords = [
    "today", "latest", "news", "trending", "breaking",
    "current", "now", "recent", "update on", "price"
  ];
  return keywords.some(k => q.includes(k));
}

// API 호출 시 검색 옵션 전달
const response = await this.grokClient.chat(
  messages,
  tools,
  undefined,
  this.shouldUseSearchFor(message)
    ? { search_parameters: { mode: "auto" } }
    : { search_parameters: { mode: "off" } }
);
```

**검색 모드**:
- `"auto"`: AI가 필요 시 자동으로 검색
- `"on"`: 항상 검색 수행
- `"off"`: 검색 비활성화

## 에러 처리

### API 에러 캐칭

```typescript
async chat(...): Promise<GrokResponse> {
  try {
    const response = await this.client.chat.completions.create(requestPayload);
    return response as GrokResponse;
  } catch (error: any) {
    throw new Error(`Grok API error: ${error.message}`);
  }
}
```

### 일반적인 에러 타입

#### 1. 인증 에러 (401)

```
Error: Grok API error: Invalid API key
```

**원인**: API 키가 잘못되었거나 만료됨

**해결책**:
```bash
export GROK_API_KEY="올바른_키"
```

#### 2. 연결 에러

```
Error: Grok API error: connect ECONNREFUSED 127.0.0.1:8000
```

**원인**: 로컬 서버가 실행 중이지 않음

**해결책**:
```bash
# vLLM 서버 시작
vllm serve Qwen/Qwen2.5-Coder-32B-Instruct --port 8000
```

#### 3. 타임아웃 에러

```
Error: Grok API error: Request timeout
```

**원인**: 360초(6분) 내에 응답 없음

**해결책**: 더 작은 요청으로 나누거나 타임아웃 증가

#### 4. 토큰 제한 초과

```
Error: Grok API error: Maximum context length exceeded
```

**원인**: 입력 토큰이 모델의 최대 컨텍스트 윈도우 초과

**해결책**: 대화 히스토리 요약 또는 정리

#### 5. 레이트 리밋

```
Error: Grok API error: Rate limit exceeded
```

**원인**: API 요청 횟수 제한 초과

**해결책**: 요청 간격 조절 또는 플랜 업그레이드

## 로컬 모델 호환성

### OpenAI 호환 서버 설정

#### 1. vLLM

```bash
# 설치
pip install vllm

# 서버 실행
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-Coder-32B-Instruct \
  --host 0.0.0.0 \
  --port 8000

# Grok CLI 설정
export GROK_BASE_URL="http://localhost:8000/v1"
export GROK_API_KEY="not-needed"
export GROK_MODEL="Qwen/Qwen2.5-Coder-32B-Instruct"
grok
```

#### 2. Ollama

```bash
# 설치 및 실행
curl -fsSL https://ollama.com/install.sh | sh
ollama serve

# 모델 다운로드
ollama pull qwen2.5-coder:32b

# Grok CLI 설정
export GROK_BASE_URL="http://localhost:11434/v1"
export GROK_API_KEY="ollama"
export GROK_MODEL="qwen2.5-coder:32b"
grok
```

#### 3. LM Studio

```bash
# LM Studio에서 서버 시작 (포트 1234)

# Grok CLI 설정
export GROK_BASE_URL="http://localhost:1234/v1"
export GROK_API_KEY="lm-studio"
export GROK_MODEL="모델이름"
grok
```

### 모델별 설정 최적화

```typescript
// src/grok/client.ts에 추가 가능
const MODEL_CONFIGS: Record<string, {
  maxTokens: number;
  temperature: number;
  contextWindow: number;
}> = {
  "grok-code-fast-1": {
    maxTokens: 1536,
    temperature: 0.7,
    contextWindow: 128000
  },
  "Qwen/Qwen2.5-Coder-32B-Instruct": {
    maxTokens: 4096,
    temperature: 0.3,
    contextWindow: 32768
  },
  "deepseek-ai/DeepSeek-Coder-V2-Instruct": {
    maxTokens: 8192,
    temperature: 0.2,
    contextWindow: 131072
  },
  "gpt-oss/gpt-oss-20b": {
    maxTokens: 2048,
    temperature: 0.7,
    contextWindow: 8192
  }
};

// 생성자에서 사용
constructor(apiKey: string, model?: string, baseURL?: string) {
  // ...
  const config = MODEL_CONFIGS[this.currentModel];
  if (config) {
    this.defaultMaxTokens = config.maxTokens;
  }
}
```

### API 호환성 체크리스트

로컬 모델 서버가 다음을 지원하는지 확인:

- [x] `/v1/chat/completions` 엔드포인트
- [x] `messages` 파라미터 (OpenAI 형식)
- [x] `tools` 파라미터 (Function Calling)
- [x] `stream` 파라미터 (스트리밍)
- [x] 스트리밍 응답 형식 (SSE)

**테스트 방법**:
```bash
# 기본 요청 테스트
curl http://localhost:8000/v1/models

# 채팅 요청 테스트
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-Coder-32B-Instruct",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## 성능 최적화

### 1. 타임아웃 조정

```typescript
// 빠른 응답이 필요한 경우
this.client = new OpenAI({
  apiKey,
  baseURL,
  timeout: 60000  // 1분
});

// 복잡한 작업의 경우
this.client = new OpenAI({
  apiKey,
  baseURL,
  timeout: 600000  // 10분
});
```

### 2. 최대 토큰 수 최적화

```bash
# 빠른 응답
export GROK_MAX_TOKENS=512

# 긴 코드 생성
export GROK_MAX_TOKENS=4096
```

### 3. Temperature 조정

```typescript
// 코드 생성 (결정적)
temperature: 0.2

// 창의적 작업
temperature: 0.9

// 기본 (균형)
temperature: 0.7
```

## 디버깅

### API 요청/응답 로깅

```typescript
async chat(...): Promise<GrokResponse> {
  console.log('API Request:', {
    model: requestPayload.model,
    messageCount: requestPayload.messages.length,
    toolCount: requestPayload.tools?.length || 0
  });

  const response = await this.client.chat.completions.create(requestPayload);

  console.log('API Response:', {
    finishReason: response.choices[0].finish_reason,
    hasToolCalls: !!response.choices[0].message.tool_calls,
    contentLength: response.choices[0].message.content?.length || 0
  });

  return response as GrokResponse;
}
```

### 스트리밍 디버깅

```typescript
async *chatStream(...): AsyncGenerator<any, void, unknown> {
  let chunkCount = 0;

  for await (const chunk of stream) {
    chunkCount++;

    if (process.env.DEBUG) {
      console.log(`Chunk ${chunkCount}:`, JSON.stringify(chunk, null, 2));
    }

    yield chunk;
  }

  console.log(`Total chunks received: ${chunkCount}`);
}
```

## 결론

`GrokClient`는 간단하면서도 강력한 API 클라이언트로:

1. **OpenAI 호환성**: 다양한 모델과 서버 지원
2. **스트리밍**: 실시간 응답으로 빠른 사용자 경험
3. **도구 호출**: Function Calling으로 실제 작업 수행
4. **웹 검색**: 최신 정보 접근
5. **에러 처리**: 안정적인 운영

로컬 모델로의 전환도 단순히 `baseURL`만 변경하면 되므로 매우 유연합니다.
