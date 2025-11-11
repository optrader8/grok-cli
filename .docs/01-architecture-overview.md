# Grok CLI - 아키텍처 개요

## 목차

1. [소개](#소개)
2. [고수준 아키텍처](#고수준-아키텍처)
3. [핵심 레이어](#핵심-레이어)
4. [데이터 흐름](#데이터-흐름)
5. [컴포넌트 간 통신](#컴포넌트-간-통신)
6. [확장성 고려사항](#확장성-고려사항)

## 소개

Grok CLI는 터미널 기반의 AI 코딩 어시스턴트로, Grok API를 활용하여 자연어로 파일 편집, 명령 실행, 코드 생성 등의 작업을 수행합니다. 이 문서는 전체 시스템의 아키텍처를 상세히 설명합니다.

### 핵심 설계 원칙

1. **모듈화**: 각 컴포넌트는 독립적으로 동작하며 명확한 책임을 가집니다
2. **확장성**: OpenAI 호환 API를 통해 다양한 모델 지원
3. **사용자 중심**: 직관적인 터미널 UI와 실시간 피드백
4. **안전성**: 파일 작업 전 사용자 확인 시스템
5. **비동기 처리**: 스트리밍 응답과 논블로킹 I/O

## 고수준 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                         Grok CLI                                │
│                                                                 │
│  ┌────────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  CLI Interface │  │  UI Layer    │  │  Settings        │  │
│  │  (Commander)   │  │  (Ink/React) │  │  Management      │  │
│  └────────┬───────┘  └──────┬───────┘  └─────────┬────────┘  │
│           │                  │                    │            │
│           └──────────────────┴────────────────────┘            │
│                              │                                 │
│                    ┌─────────▼─────────┐                      │
│                    │   Grok Agent      │                      │
│                    │  (Core Orchestrator)                     │
│                    └─────────┬─────────┘                      │
│                              │                                 │
│         ┌────────────────────┼────────────────────┐           │
│         │                    │                    │           │
│    ┌────▼────┐         ┌─────▼─────┐      ┌──────▼──────┐   │
│    │  Grok   │         │   Tools   │      │     MCP     │   │
│    │ Client  │         │  System   │      │   Manager   │   │
│    └────┬────┘         └─────┬─────┘      └──────┬──────┘   │
│         │                    │                    │           │
│         │              ┌─────▼─────┐              │           │
│         │              │ TextEditor│              │           │
│         │              │   Bash    │              │           │
│         │              │   Search  │              │           │
│         │              │   Todo    │              │           │
│         │              │   Morph   │              │           │
│         │              └───────────┘              │           │
│         │                                         │           │
└─────────┼─────────────────────────────────────────┼───────────┘
          │                                         │
          │                                         │
    ┌─────▼──────┐                          ┌──────▼──────┐
    │  Grok API  │                          │ MCP Servers │
    │ (OpenAI    │                          │  (Linear,   │
    │ Compatible)│                          │  GitHub...) │
    └────────────┘                          └─────────────┘
```

## 핵심 레이어

### 1. Presentation Layer (프레젠테이션 레이어)

#### 1.1 CLI Interface (`src/index.ts`)

**책임**:
- 명령줄 인자 파싱 (`commander` 사용)
- 환경 변수 로딩
- 애플리케이션 초기화
- 모드 선택 (대화형 vs 헤드리스)

**주요 기능**:
```typescript
// 대화형 모드
grok

// 헤드리스 모드
grok --prompt "파일을 읽어줘"

// Git 통합
grok git commit-and-push

// MCP 관리
grok mcp add linear --transport sse --url "..."
```

**코드 구조**:
```typescript
program
  .name("grok")
  .version("1.0.1")
  .option("-d, --directory <dir>", "작업 디렉토리 설정")
  .option("-k, --api-key <key>", "Grok API 키")
  .option("-u, --base-url <url>", "API Base URL")
  .option("-m, --model <model>", "AI 모델")
  .option("-p, --prompt <prompt>", "헤드리스 모드 프롬프트")
  .option("--max-tool-rounds <rounds>", "최대 도구 실행 라운드", "400")
  .action(async (message, options) => {
    // 옵션 처리 및 에이전트 초기화
  });
```

#### 1.2 UI Layer (`src/ui/`)

**책임**:
- 터미널 UI 렌더링 (Ink + React)
- 사용자 입력 처리
- 실시간 피드백 표시
- 채팅 히스토리 관리

**주요 컴포넌트**:

| 컴포넌트 | 파일 | 역할 |
|---------|------|-----|
| ChatInterface | `chat-interface.tsx` | 메인 채팅 인터페이스 |
| ChatHistory | `chat-history.tsx` | 대화 히스토리 렌더링 |
| ChatInput | `chat-input.tsx` | 사용자 입력 필드 |
| LoadingSpinner | `loading-spinner.tsx` | 로딩 애니메이션 |
| ConfirmationDialog | `confirmation-dialog.tsx` | 파일 작업 확인 대화상자 |
| ModelSelection | `model-selection.tsx` | 모델 선택 UI |
| MCPStatus | `mcp-status.tsx` | MCP 서버 상태 표시 |
| CommandSuggestions | `command-suggestions.tsx` | 명령 자동완성 |

**Ink 기반 아키텍처**:
```typescript
// React 컴포넌트로 터미널 UI 구성
function ChatInterface({ agent, initialMessage }: ChatInterfaceProps) {
  const [chatHistory, setChatHistory] = useState<ChatEntry[]>([]);
  const [isProcessing, setIsProcessing] = useState(false);

  return (
    <Box flexDirection="column" paddingX={2}>
      <ChatHistory entries={chatHistory} />
      <ChatInput input={input} />
      <LoadingSpinner isActive={isProcessing} />
    </Box>
  );
}
```

### 2. Application Layer (애플리케이션 레이어)

#### 2.1 Grok Agent (`src/agent/grok-agent.ts`)

**책임**:
- 사용자 메시지 처리 오케스트레이션
- AI 응답 관리 (스트리밍/일반)
- 도구 호출 실행
- 대화 히스토리 유지
- 토큰 카운팅

**핵심 메서드**:

```typescript
class GrokAgent extends EventEmitter {
  // 메시지 처리 (논블로킹, 스트리밍)
  async *processUserMessageStream(message: string): AsyncGenerator<StreamingChunk>

  // 메시지 처리 (블로킹, 전체 응답)
  async processUserMessage(message: string): Promise<ChatEntry[]>

  // 도구 실행
  private async executeTool(toolCall: GrokToolCall): Promise<ToolResult>

  // MCP 도구 실행
  private async executeMCPTool(toolCall: GrokToolCall): Promise<ToolResult>

  // 모델 변경
  setModel(model: string): void

  // 현재 작업 중단
  abortCurrentOperation(): void
}
```

**에이전트 루프**:
```typescript
while (toolRounds < maxToolRounds) {
  // 1. AI에게 메시지 전송 + 사용 가능한 도구 목록
  const response = await grokClient.chat(messages, tools);

  // 2. 도구 호출이 있는가?
  if (response.tool_calls?.length > 0) {
    // 3. 각 도구 실행
    for (const toolCall of response.tool_calls) {
      const result = await executeTool(toolCall);
      messages.push({ role: "tool", content: result.output });
    }

    // 4. 다시 AI에게 전송 (도구 결과 포함)
    toolRounds++;
  } else {
    // 5. 도구 호출 없음 → 최종 응답
    break;
  }
}
```

#### 2.2 Settings Management (`src/utils/settings-manager.ts`)

**책임**:
- 사용자 설정 관리 (`~/.grok/user-settings.json`)
- 프로젝트 설정 관리 (`.grok/settings.json`)
- 설정 우선순위 처리
- API 키 및 Base URL 관리

**설정 계층 구조**:
```
환경 변수 (GROK_API_KEY, GROK_BASE_URL, GROK_MODEL)
  ↓
명령줄 플래그 (--api-key, --base-url, --model)
  ↓
프로젝트 설정 (.grok/settings.json)
  ↓
사용자 설정 (~/.grok/user-settings.json)
  ↓
시스템 기본값
```

### 3. Domain Layer (도메인 레이어)

#### 3.1 Grok Client (`src/grok/client.ts`)

**책임**:
- OpenAI SDK 래퍼
- API 통신 관리
- 스트리밍 응답 처리
- 오류 처리

**주요 메서드**:
```typescript
class GrokClient {
  // 일반 채팅 (블로킹)
  async chat(
    messages: GrokMessage[],
    tools?: GrokTool[],
    model?: string,
    searchOptions?: SearchOptions
  ): Promise<GrokResponse>

  // 스트리밍 채팅 (논블로킹)
  async *chatStream(
    messages: GrokMessage[],
    tools?: GrokTool[],
    model?: string,
    searchOptions?: SearchOptions
  ): AsyncGenerator<any, void, unknown>

  // 웹 검색
  async search(
    query: string,
    searchParameters?: SearchParameters
  ): Promise<GrokResponse>
}
```

**OpenAI SDK 사용**:
```typescript
constructor(apiKey: string, model?: string, baseURL?: string) {
  this.client = new OpenAI({
    apiKey,
    baseURL: baseURL || "https://api.x.ai/v1",
    timeout: 360000,
  });
}
```

#### 3.2 Tools System (`src/tools/`)

**책임**:
- 실제 도구 구현
- 파일 시스템 작업
- Bash 명령 실행
- 검색 및 TODO 관리

**도구 목록**:

| 도구 | 파일 | 기능 |
|-----|------|-----|
| TextEditor | `text-editor.ts` | 파일 읽기/생성/편집 |
| Bash | `bash.ts` | Shell 명령 실행 |
| Search | `search.ts` | 파일/텍스트 검색 |
| TodoTool | `todo-tool.ts` | TODO 리스트 관리 |
| MorphEditor | `morph-editor.ts` | 고속 코드 편집 (Morph API) |
| Confirmation | `confirmation-tool.ts` | 사용자 확인 처리 |

**도구 인터페이스**:
```typescript
interface ToolResult {
  success: boolean;
  output?: string;
  error?: string;
}

// 예: TextEditorTool
class TextEditorTool {
  async view(filePath: string, viewRange?: [number, number]): Promise<ToolResult>
  async create(filePath: string, content: string): Promise<ToolResult>
  async strReplace(filePath: string, oldStr: string, newStr: string): Promise<ToolResult>
}
```

#### 3.3 MCP Integration (`src/mcp/`)

**책임**:
- Model Context Protocol 서버 관리
- 외부 도구 통합 (Linear, GitHub 등)
- 다양한 전송 방식 지원 (stdio, HTTP, SSE)

**주요 컴포넌트**:
```typescript
// MCP 설정 로딩
function loadMCPConfig(): MCPConfig

// MCP 서버 초기화
async function initializeMCPServers(): Promise<void>

// MCP 도구 호출
async function callMCPTool(toolName: string, args: any): Promise<ToolResult>
```

### 4. Infrastructure Layer (인프라 레이어)

#### 4.1 Hooks (`src/hooks/`)

**React 커스텀 훅**:
- `use-input-handler.ts`: 사용자 입력 처리
- `use-input-history.ts`: 입력 히스토리 관리
- `use-enhanced-input.ts`: 향상된 입력 기능

#### 4.2 Utils (`src/utils/`)

**유틸리티 모듈**:
- `token-counter.ts`: 토큰 카운팅 (`tiktoken`)
- `confirmation-service.ts`: 확인 대화상자 관리
- `custom-instructions.ts`: 프로젝트별 커스텀 지침 로딩
- `text-utils.ts`: 텍스트 처리 유틸리티

## 데이터 흐름

### 일반 메시지 처리 흐름

```
1. 사용자 입력
   │
   ▼
2. ChatInterface (UI)
   │ useInputHandler hook
   ▼
3. GrokAgent.processUserMessageStream()
   │
   ├─▶ 4. GrokClient.chatStream()
   │      │
   │      └─▶ 5. OpenAI SDK
   │             │
   │             └─▶ 6. Grok API (또는 로컬 모델)
   │                    │
   │                    ▼
   │                 7. 스트리밍 응답
   │
   ├─▶ 8. 도구 호출 필요?
   │      │
   │      ├─ Yes ─▶ 9. executeTool()
   │      │            │
   │      │            ├─▶ TextEditorTool
   │      │            ├─▶ BashTool
   │      │            ├─▶ SearchTool
   │      │            └─▶ MCP Tools
   │      │                 │
   │      │                 ▼
   │      │              10. ToolResult
   │      │                 │
   │      │                 └─▶ 다시 3번으로 (최대 400 라운드)
   │      │
   │      └─ No ─▶ 11. 최종 응답
   │
   ▼
12. UI 업데이트 (실시간 스트리밍)
```

### 파일 작업 흐름 (확인 시스템 포함)

```
1. AI가 파일 작업 요청 (create_file/str_replace_editor)
   │
   ▼
2. TextEditorTool.create() 또는 strReplace()
   │
   ├─▶ 3. 세션 플래그 확인
   │      │
   │      ├─ 이미 승인됨 ─▶ 5. 파일 작업 실행
   │      │
   │      └─ 미승인 ─▶ 4. ConfirmationService.requestConfirmation()
   │                    │
   │                    ▼
   │                 4.1 ConfirmationDialog 표시 (UI)
   │                    │
   │                    ├─ 사용자 승인 ─▶ 5. 파일 작업 실행
   │                    │                     │
   │                    │                     ▼
   │                    │                  6. diff 생성
   │                    │                     │
   │                    │                     ▼
   │                    │                  7. ToolResult 반환
   │                    │
   │                    └─ 사용자 거부 ─▶ 8. ToolResult (error)
   │
   ▼
9. AI에게 결과 전달
```

### 스트리밍 청크 타입

```typescript
type StreamingChunk =
  | { type: "content"; content: string }          // 텍스트 콘텐츠
  | { type: "tool_calls"; toolCalls: GrokToolCall[] }  // 도구 호출
  | { type: "tool_result"; toolCall: GrokToolCall; toolResult: ToolResult }  // 도구 결과
  | { type: "token_count"; tokenCount: number }   // 토큰 카운트 업데이트
  | { type: "done" }                              // 완료
```

## 컴포넌트 간 통신

### 1. EventEmitter 패턴

**GrokAgent**:
```typescript
class GrokAgent extends EventEmitter {
  // 이벤트 발행 없음 (현재 구현에서는 제너레이터 사용)
}
```

**ConfirmationService**:
```typescript
class ConfirmationService extends EventEmitter {
  emit('confirmation-requested', options)  // 확인 요청
  emit('confirmation-resolved', result)     // 확인 완료
}
```

### 2. Generator 패턴 (스트리밍)

```typescript
async *processUserMessageStream(message: string): AsyncGenerator<StreamingChunk> {
  // AI 응답 스트리밍
  for await (const chunk of stream) {
    yield { type: "content", content: chunk.delta.content };
  }

  // 도구 호출
  yield { type: "tool_calls", toolCalls: [...] };

  // 도구 결과
  yield { type: "tool_result", toolCall, toolResult };

  // 완료
  yield { type: "done" };
}
```

### 3. Promise 패턴 (확인 시스템)

```typescript
class ConfirmationService {
  private pendingPromise: {
    resolve: (result: ConfirmationResult) => void;
    reject: (error: Error) => void;
  } | null = null;

  async requestConfirmation(options: ConfirmationOptions): Promise<ConfirmationResult> {
    return new Promise((resolve, reject) => {
      this.pendingPromise = { resolve, reject };
      this.emit('confirmation-requested', options);
    });
  }

  confirmOperation(confirmed: boolean, dontAskAgain?: boolean): void {
    if (this.pendingPromise) {
      this.pendingPromise.resolve({ confirmed, dontAskAgain });
      this.pendingPromise = null;
    }
  }
}
```

## 확장성 고려사항

### 1. 새로운 모델 추가

OpenAI 호환 API를 사용하므로 새로운 모델 추가가 매우 쉽습니다:

```bash
# 로컬 모델 서버 실행
vllm serve Qwen/Qwen2.5-Coder-32B-Instruct --port 8000

# Grok CLI 설정
export GROK_BASE_URL="http://localhost:8000/v1"
export GROK_MODEL="Qwen/Qwen2.5-Coder-32B-Instruct"
grok
```

**설정 파일로 관리**:
```json
// ~/.grok/user-settings.json
{
  "baseURL": "http://localhost:8000/v1",
  "defaultModel": "Qwen/Qwen2.5-Coder-32B-Instruct",
  "models": [
    "Qwen/Qwen2.5-Coder-32B-Instruct",
    "deepseek-ai/DeepSeek-Coder-V2-Instruct"
  ]
}
```

### 2. 새로운 도구 추가

**단계**:

1. 도구 클래스 구현 (`src/tools/my-tool.ts`):
```typescript
export class MyTool {
  async execute(args: MyToolArgs): Promise<ToolResult> {
    // 구현
    return { success: true, output: "결과" };
  }
}
```

2. 도구 정의 추가 (`src/grok/tools.ts`):
```typescript
const MY_TOOL: GrokTool = {
  type: "function",
  function: {
    name: "my_tool",
    description: "도구 설명",
    parameters: {
      type: "object",
      properties: {
        arg1: { type: "string", description: "인자 설명" }
      },
      required: ["arg1"]
    }
  }
};

export const GROK_TOOLS = [
  // ...기존 도구들
  MY_TOOL
];
```

3. 에이전트에서 도구 실행 추가 (`src/agent/grok-agent.ts`):
```typescript
private async executeTool(toolCall: GrokToolCall): Promise<ToolResult> {
  switch (toolCall.function.name) {
    // ...기존 케이스들
    case "my_tool":
      return await this.myTool.execute(args);
  }
}
```

### 3. MCP 서버 추가

MCP (Model Context Protocol)를 통해 외부 서비스와 쉽게 통합할 수 있습니다:

```bash
# Linear 통합
grok mcp add linear --transport sse --url "https://mcp.linear.app/sse"

# GitHub 통합
grok mcp add github --transport stdio --command "npx" --args "@github/mcp-server"

# 커스텀 MCP 서버
grok mcp add my-server --transport http --url "http://localhost:3000"
```

**설정 저장**:
```json
// .grok/settings.json
{
  "mcpServers": {
    "linear": {
      "name": "linear",
      "transport": "sse",
      "url": "https://mcp.linear.app/sse"
    }
  }
}
```

### 4. UI 커스터마이징

Ink를 사용하므로 React 컴포넌트로 UI를 쉽게 커스터마이징할 수 있습니다:

```typescript
// 새로운 컴포넌트 추가
function MyCustomComponent() {
  return (
    <Box borderStyle="round" borderColor="cyan">
      <Text>커스텀 콘텐츠</Text>
    </Box>
  );
}

// ChatInterface에 통합
<Box flexDirection="column">
  <ChatHistory entries={chatHistory} />
  <MyCustomComponent />  {/* 추가 */}
  <ChatInput input={input} />
</Box>
```

## 성능 최적화

### 1. 스트리밍 응답

모든 AI 응답은 스트리밍으로 처리되어 빠른 피드백을 제공합니다:

```typescript
for await (const chunk of agent.processUserMessageStream(message)) {
  if (chunk.type === "content") {
    // 실시간으로 UI 업데이트
    updateUI(chunk.content);
  }
}
```

### 2. 토큰 카운팅 최적화

실시간 토큰 카운팅을 통해 컨텍스트 윈도우를 효율적으로 관리합니다:

```typescript
// 스트리밍 중 토큰 추정
const currentTokens = tokenCounter.estimateStreamingTokens(accumulatedContent);

// 정확한 토큰 카운팅 (필요시)
const exactTokens = tokenCounter.countMessageTokens(messages);
```

### 3. 비동기 처리

모든 I/O 작업은 비동기로 처리되어 블로킹을 방지합니다:

```typescript
// 파일 작업
await fs.readFile(path, 'utf-8');
await fs.writeFile(path, content);

// API 호출
await grokClient.chat(messages);

// Bash 명령
await bashTool.execute(command);
```

## 보안 고려사항

### 1. API 키 관리

- API 키는 환경 변수 또는 설정 파일에 저장
- 설정 파일은 `0o600` 권한으로 보호
- Git에서 `.env` 파일 제외

### 2. 파일 작업 확인

모든 파일 작업은 사용자 확인을 거칩니다:

```typescript
// 첫 번째 작업: 확인 요청
const result = await confirmationService.requestConfirmation({
  operation: "Write",
  filename: "example.js",
  content: diff
});

// 사용자가 "모두 승인" 선택시: 이후 작업은 자동 승인
if (result.dontAskAgain) {
  confirmationService.setSessionFlag('fileOperations', true);
}
```

### 3. Bash 명령 실행

사용자가 명시적으로 요청한 명령만 실행하며, 위험한 명령에 대한 경고를 제공합니다.

## 테스트 전략

### 1. 단위 테스트

각 도구와 유틸리티는 독립적으로 테스트 가능합니다:

```typescript
// TextEditorTool 테스트
test('파일 생성', async () => {
  const editor = new TextEditorTool();
  const result = await editor.create('test.txt', 'content');
  expect(result.success).toBe(true);
});
```

### 2. 통합 테스트

에이전트와 도구 간 통합을 테스트합니다:

```typescript
test('메시지 처리 및 도구 실행', async () => {
  const agent = new GrokAgent(apiKey);
  const entries = await agent.processUserMessage('파일을 만들어줘');
  expect(entries.some(e => e.type === 'tool_result')).toBe(true);
});
```

### 3. E2E 테스트

전체 시스템을 실제 사용 시나리오로 테스트합니다.

## 결론

Grok CLI는 모듈화되고 확장 가능한 아키텍처를 가지고 있어:

1. **새로운 모델 추가가 쉬움** (OpenAI 호환 API)
2. **도구 확장이 간단함** (플러그인 방식)
3. **외부 서비스 통합이 용이함** (MCP)
4. **UI 커스터마이징이 직관적** (React/Ink)

이러한 설계 덕분에 로컬 모델로의 전환, 새로운 기능 추가, 성능 최적화 등이 모두 용이합니다.
