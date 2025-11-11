# Grok CLI - 에이전트 시스템 상세 분석

## 목차

1. [개요](#개요)
2. [GrokAgent 클래스](#grokagentt-클래스)
3. [에이전트 루프 메커니즘](#에이전트-루프-메커니즘)
4. [메시지 처리 방식](#메시지-처리-방식)
5. [도구 실행 오케스트레이션](#도구-실행-오케스트레이션)
6. [토큰 카운팅](#토큰-카운팅)
7. [중단 메커니즘](#중단-메커니즘)
8. [채팅 히스토리 관리](#채팅-히스토리-관리)
9. [MCP 통합](#mcp-통합)

## 개요

`GrokAgent`는 Grok CLI의 핵심 엔진으로, 사용자 메시지 처리, AI 응답 관리, 도구 실행을 담당합니다. EventEmitter를 상속받아 이벤트 기반 아키텍처를 지원합니다.

**파일 위치**: `src/agent/grok-agent.ts`

**핵심 책임**:
- 사용자 메시지 처리 (스트리밍/블로킹)
- 도구 호출 감지 및 실행
- 문맥 관리 (대화 히스토리)
- 토큰 카운팅
- MCP 도구 통합

## GrokAgent 클래스

### 클래스 구조

```typescript
export class GrokAgent extends EventEmitter {
  // 의존성
  private grokClient: GrokClient;                    // API 클라이언트
  private textEditor: TextEditorTool;                // 파일 편집
  private morphEditor: MorphEditorTool | null;       // 고속 편집
  private bash: BashTool;                            // 명령 실행
  private todoTool: TodoTool;                        // TODO 관리
  private confirmationTool: ConfirmationTool;        // 확인 도구
  private search: SearchTool;                        // 검색 도구

  // 상태
  private chatHistory: ChatEntry[] = [];             // 채팅 히스토리
  private messages: GrokMessage[] = [];              // API 메시지
  private tokenCounter: TokenCounter;                // 토큰 카운터
  private abortController: AbortController | null;   // 중단 제어
  private mcpInitialized: boolean = false;           // MCP 초기화 여부
  private maxToolRounds: number;                     // 최대 도구 라운드

  // 메서드
  constructor(apiKey, baseURL?, model?, maxToolRounds?)
  async processUserMessage(message): Promise<ChatEntry[]>
  async *processUserMessageStream(message): AsyncGenerator<StreamingChunk>
  private async executeTool(toolCall): Promise<ToolResult>
  private async executeMCPTool(toolCall): Promise<ToolResult>
  setModel(model): void
  abortCurrentOperation(): void
}
```

### 생성자

```typescript
constructor(
  apiKey: string,
  baseURL?: string,
  model?: string,
  maxToolRounds?: number
) {
  super();

  // 설정 관리자에서 저장된 모델 로드
  const manager = getSettingsManager();
  const savedModel = manager.getCurrentModel();
  const modelToUse = model || savedModel || "grok-code-fast-1";

  // 최대 라운드 설정 (기본값: 400)
  this.maxToolRounds = maxToolRounds || 400;

  // 도구 초기화
  this.grokClient = new GrokClient(apiKey, modelToUse, baseURL);
  this.textEditor = new TextEditorTool();
  this.morphEditor = process.env.MORPH_API_KEY ? new MorphEditorTool() : null;
  this.bash = new BashTool();
  this.todoTool = new TodoTool();
  this.confirmationTool = new ConfirmationTool();
  this.search = new SearchTool();
  this.tokenCounter = createTokenCounter(modelToUse);

  // MCP 초기화
  this.initializeMCP();

  // 커스텀 지침 로드
  const customInstructions = loadCustomInstructions();

  // 시스템 메시지 초기화
  this.messages.push({
    role: "system",
    content: `You are Grok CLI...${customInstructions ? `\n\nCUSTOM INSTRUCTIONS:\n${customInstructions}` : ""}...`
  });
}
```

**초기화 단계**:

1. **모델 결정**: 설정된 모델 또는 기본값 사용
2. **도구 초기화**: 모든 도구 인스턴스 생성
3. **MCP 서버**: 백그라운드에서 비동기 초기화
4. **커스텀 지침**: 프로젝트별 지침 로드
5. **시스템 프롬프트**: 초기 메시지에 지침 추가

## 에이전트 루프 메커니즘

### 에이전트 루프 흐름도

```
1. 사용자 메시지 입력
   ▼
2. 사용자 메시지를 messages 배열에 추가
   ▼
3. grokClient.chat() 호출
   ├─ messages (전체 대화 히스토리)
   ├─ tools (사용 가능한 도구)
   └─ searchOptions (웹 검색 옵션)
   ▼
4. AI 응답 수신
   ├─ 일반 텍스트 응답
   └─ tool_calls 배열 (도구 호출)
   ▼
5. 도구 호출이 있는가?
   │
   ├─ YES (toolRounds < maxToolRounds)
   │  ▼
   │  6. 각 도구 호출 실행
   │     ├─ TextEditorTool (파일 작업)
   │     ├─ BashTool (명령 실행)
   │     ├─ SearchTool (검색)
   │     ├─ TodoTool (TODO)
   │     ├─ MCP 도구
   │     └─ ToolResult 반환
   │  ▼
   │  7. 도구 결과를 tool 메시지로 추가
   │  ▼
   │  8. 다시 3번으로 (AI에 도구 결과 전송)
   │
   └─ NO
      ▼
      9. 최종 응답 반환
```

### 코드 구현

```typescript
async processUserMessage(message: string): Promise<ChatEntry[]> {
  // 1. 사용자 메시지 추가
  const userEntry: ChatEntry = {
    type: "user",
    content: message,
    timestamp: new Date(),
  };
  this.chatHistory.push(userEntry);
  this.messages.push({ role: "user", content: message });

  const newEntries: ChatEntry[] = [userEntry];
  let toolRounds = 0;

  try {
    // 2. 사용 가능한 도구 목록 가져오기
    const tools = await getAllGrokTools();

    // 3. 초기 API 호출
    let currentResponse = await this.grokClient.chat(
      this.messages,
      tools,
      undefined,
      this.isGrokModel() && this.shouldUseSearchFor(message)
        ? { search_parameters: { mode: "auto" } }
        : { search_parameters: { mode: "off" } }
    );

    // 4. 에이전트 루프
    while (toolRounds < this.maxToolRounds) {
      const assistantMessage = currentResponse.choices[0]?.message;

      if (!assistantMessage) {
        throw new Error("No response from Grok");
      }

      // 5. 도구 호출 확인
      if (
        assistantMessage.tool_calls &&
        assistantMessage.tool_calls.length > 0
      ) {
        toolRounds++;

        // 6. 도구 호출 메시지 추가
        const assistantEntry: ChatEntry = {
          type: "assistant",
          content: assistantMessage.content || "Using tools to help you...",
          timestamp: new Date(),
          toolCalls: assistantMessage.tool_calls,
        };
        this.chatHistory.push(assistantEntry);
        newEntries.push(assistantEntry);

        // API 메시지에도 추가
        this.messages.push({
          role: "assistant",
          content: assistantMessage.content || "",
          tool_calls: assistantMessage.tool_calls,
        } as any);

        // 7. 각 도구 호출 실행
        for (const toolCall of assistantMessage.tool_calls) {
          const result = await this.executeTool(toolCall);

          // 도구 결과를 메시지에 추가
          this.messages.push({
            role: "tool",
            content: result.success ? result.output || "Success" : result.error || "Error",
            tool_call_id: toolCall.id,
          } as any);
        }

        // 8. 다시 API 호출 (도구 결과 포함)
        currentResponse = await this.grokClient.chat(
          this.messages,
          tools
        );
      } else {
        // 9. 최종 응답
        const finalEntry: ChatEntry = {
          type: "assistant",
          content: assistantMessage.content || "Task completed",
          timestamp: new Date(),
        };
        this.chatHistory.push(finalEntry);
        newEntries.push(finalEntry);
        break;
      }
    }

    return newEntries;
  } catch (error: any) {
    // 에러 처리
    throw error;
  }
}
```

## 메시지 처리 방식

### 1. 블로킹 방식 (processUserMessage)

```typescript
async processUserMessage(message: string): Promise<ChatEntry[]>
```

**특징**:
- 전체 응답이 완료될 때까지 대기
- `ChatEntry[]` 배열 반환 (모든 히스토리)
- 헤드리스 모드에 적합
- 간단한 사용 방식

**사용 예**:
```typescript
const entries = await agent.processUserMessage("파일을 읽어줘");
console.log(entries); // 모든 교환 내역
```

**장점**:
- 구현이 간단
- 전체 결과를 한 번에 얻을 수 있음
- 순차 처리 가능

**단점**:
- 사용자 피드백 지연
- 느린 응답 시간
- 실시간 UI 업데이트 어려움

### 2. 스트리밍 방식 (processUserMessageStream)

```typescript
async *processUserMessageStream(
  message: string
): AsyncGenerator<StreamingChunk>
```

**특징**:
- 비동기 제너레이터 사용
- 청크 단위로 실시간 전송
- 대화형 모드에 적합
- 빠른 피드백

**청크 타입**:
```typescript
type StreamingChunk =
  | { type: "content"; content: string }                    // 텍스트
  | { type: "tool_calls"; toolCalls: GrokToolCall[] }      // 도구 호출
  | { type: "tool_result"; toolCall; toolResult: ToolResult } // 도구 결과
  | { type: "token_count"; tokenCount: number }             // 토큰 수
  | { type: "done" }                                        // 완료
```

**사용 예**:
```typescript
for await (const chunk of agent.processUserMessageStream(message)) {
  if (chunk.type === "content") {
    // 실시간 텍스트 표시
    console.log(chunk.content);
  } else if (chunk.type === "tool_calls") {
    // 도구 실행 표시
    console.log(`도구 실행: ${chunk.toolCalls.map(t => t.function.name).join(", ")}`);
  } else if (chunk.type === "tool_result") {
    // 도구 결과 표시
    console.log(`결과: ${chunk.toolResult.success ? "성공" : "실패"}`);
  }
}
```

**장점**:
- 실시간 피드백
- 빠른 사용자 경험
- 대화형 UI 지원
- 점진적 응답 처리

**단점**:
- 구현이 복잡
- 청크별 처리 필요
- 동기화 관리 필요

### ChatEntry 타입

```typescript
export interface ChatEntry {
  type: "user" | "assistant" | "tool_result" | "tool_call";
  content: string;
  timestamp: Date;
  toolCalls?: GrokToolCall[];      // assistant가 도구를 호출할 때
  toolCall?: GrokToolCall;          // 개별 도구 호출
  toolResult?: {                    // 도구 결과
    success: boolean;
    output?: string;
    error?: string;
  };
  isStreaming?: boolean;
}
```

## 도구 실행 오케스트레이션

### 도구 실행 흐름

```typescript
private async executeTool(toolCall: GrokToolCall): Promise<ToolResult> {
  const { function: func } = toolCall;
  const { name, arguments: argsStr } = func;

  try {
    // 1. JSON 인자 파싱
    const args = JSON.parse(argsStr || "{}");

    // 2. 도구별 실행
    switch (name) {
      case "view_file":
        return await this.textEditor.view(args.path, args.view_range);

      case "create_file":
        return await this.textEditor.create(args.path, args.content);

      case "str_replace_editor":
        return await this.textEditor.strReplace(
          args.path,
          args.old_str,
          args.new_str,
          args.replace_all
        );

      case "edit_file":
        if (!this.morphEditor) {
          return {
            success: false,
            error: "Morph API not configured"
          };
        }
        return await this.morphEditor.editFile(args.path, args.edits);

      case "bash":
        return await this.bash.execute(args.command, args.timeout);

      case "search":
        return await this.search.search(args.query, args.options);

      case "create_todo_list":
        return await this.todoTool.createTodoList(args.todos);

      case "update_todo_list":
        return await this.todoTool.updateTodoList(args.todos);

      default:
        // MCP 도구인지 확인
        if (name.startsWith("mcp__")) {
          return await this.executeMCPTool(toolCall);
        }
        return {
          success: false,
          error: `Unknown tool: ${name}`
        };
    }
  } catch (error: any) {
    return {
      success: false,
      error: `Tool execution error: ${error.message}`
    };
  }
}
```

### MCP 도구 실행

```typescript
private async executeMCPTool(
  toolCall: GrokToolCall
): Promise<ToolResult> {
  try {
    const toolName = toolCall.function.name;
    const args = JSON.parse(toolCall.function.arguments || "{}");

    // MCP 도구 호출
    const result = await callMCPTool(toolName, args);

    return result;
  } catch (error: any) {
    return {
      success: false,
      error: `MCP tool error: ${error.message}`
    };
  }
}
```

### 도구별 책임

| 도구 | 책임 | 입력 | 출력 |
|-----|------|-----|-----|
| TextEditor | 파일 읽기/생성/편집 | 경로, 콘텐츠 | 파일 내용 또는 변경 결과 |
| Bash | Shell 명령 실행 | 명령어 | 표준 출력 |
| Search | 텍스트/파일 검색 | 쿼리 | 검색 결과 |
| Todo | TODO 리스트 관리 | 할일 항목 | 형식화된 리스트 |
| Morph | 고속 코드 편집 | 파일, 편집 | 편집 결과 |
| Confirmation | 사용자 확인 | 작업 세부사항 | 사용자 결정 |

## 토큰 카운팅

### TokenCounter 클래스

```typescript
class TokenCounter {
  private encoder: Tiktoken;

  constructor(model: string = 'gpt-4') {
    try {
      // 특정 모델의 인코딩 가져오기
      this.encoder = encoding_for_model(model as any);
    } catch {
      // 기본값: GPT-4 스타일의 인코딩
      this.encoder = get_encoding('cl100k_base');
    }
  }

  // 문자열의 토큰 수 계산
  countTokens(text: string): number {
    if (!text) return 0;
    return this.encoder.encode(text).length;
  }

  // 메시지 배열의 토큰 수 계산
  countMessageTokens(messages: Array<{
    role: string;
    content: string | null;
    tool_calls?: any[];
  }>): number {
    let totalTokens = 0;

    for (const message of messages) {
      // 메시지당 기본 토큰
      totalTokens += 3;

      // 콘텐츠 토큰
      if (message.content && typeof message.content === 'string') {
        totalTokens += this.countTokens(message.content);
      }

      // 역할 토큰
      if (message.role) {
        totalTokens += this.countTokens(message.role);
      }

      // 도구 호출 토큰
      if (message.tool_calls) {
        totalTokens += this.countTokens(JSON.stringify(message.tool_calls));
      }
    }

    // 응답 준비 토큰
    totalTokens += 3;

    return totalTokens;
  }

  // 스트리밍 콘텐츠 토큰 추정
  estimateStreamingTokens(accumulatedContent: string): number {
    return this.countTokens(accumulatedContent);
  }
}
```

### 사용 예

```typescript
// 초기화
const tokenCounter = createTokenCounter("grok-code-fast-1");

// 단일 문자열 카운팅
const textTokens = tokenCounter.countTokens("안녕하세요");  // ~2-3

// 메시지 배열 카운팅
const messageTokens = tokenCounter.countMessageTokens([
  { role: "user", content: "파일을 만들어줘" },
  { role: "assistant", content: "파일을 만들겠습니다" }
]);

// 스트리밍 추정
const streamingTokens = tokenCounter.estimateStreamingTokens(
  accumulatedContent
);
```

### 토큰 계산 규칙

```
메시지 토큰 = 모든 메시지의 (기본 3토큰 + 콘텐츠 + 역할 + 도구호출) + 응답준비 3토큰

예:
- 메시지 1: role=3 + "user"=1 + "안녕"=2 + 기본3 = 9
- 메시지 2: role=3 + "assistant"=1 + "안녕하세요"=2 + 기본3 = 9
- 응답준비: 3
- 합계: 9 + 9 + 3 = 21
```

### 모델별 인코딩

```typescript
// 인코딩 선택 (우선순위)
1. 특정 모델의 인코딩 (encoding_for_model)
   - "grok-4-latest" → cl100k_base (GPT-4)
   - "gpt-3.5-turbo" → cl100k_base

2. 기본: cl100k_base
   - GPT-4, GPT-3.5 호환
   - 대부분의 최신 모델 호환
```

## 중단 메커니즘

### AbortController 사용

```typescript
class GrokAgent {
  private abortController: AbortController | null = null;

  async processUserMessage(message: string): Promise<ChatEntry[]> {
    // 새 AbortController 생성
    this.abortController = new AbortController();

    try {
      // 메시지 처리
      // ...
    } finally {
      // 정리
      this.abortController = null;
    }
  }

  abortCurrentOperation(): void {
    if (this.abortController) {
      // 신호 전송 (API 호출 취소 등)
      this.abortController.abort();
      console.log("Operation aborted");
    }
  }
}
```

### 중단 신호 전파

```typescript
// 1. 에이전트에 전달
agent.abortCurrentOperation();

// 2. AbortController 신호
this.abortController?.signal.addEventListener('abort', () => {
  // 실행 중인 작업 중단
});

// 3. API 호출 취소
const signal = this.abortController?.signal;
if (signal?.aborted) {
  throw new Error('Operation aborted');
}
```

## 채팅 히스토리 관리

### 두 가지 히스토리 유지

```typescript
class GrokAgent {
  // 1. 채팅 히스토리: UI 표시용
  private chatHistory: ChatEntry[] = [];

  // 2. API 메시지: API 호출용
  private messages: GrokMessage[] = [];
}
```

### ChatHistory vs Messages

```
ChatHistory (UI용)                    Messages (API용)
┌─────────────────────────┐          ┌──────────────────────┐
│ type: "user"            │          │ role: "user"         │
│ content: "..."          │    ──→   │ content: "..."       │
│ timestamp: Date         │          │                      │
└─────────────────────────┘          └──────────────────────┘

┌─────────────────────────┐          ┌──────────────────────┐
│ type: "assistant"       │          │ role: "assistant"    │
│ content: "..."          │    ──→   │ content: "..."       │
│ toolCalls: [...]        │          │ tool_calls: [...]    │
│ timestamp: Date         │          │                      │
└─────────────────────────┘          └──────────────────────┘

┌─────────────────────────┐          ┌──────────────────────┐
│ type: "tool_result"     │          │ role: "tool"         │
│ content: "..."          │    ──→   │ content: "..."       │
│ toolResult: {...}       │          │ tool_call_id: "..."  │
│ timestamp: Date         │          │                      │
└─────────────────────────┘          └──────────────────────┘
```

### 히스토리 추가 순서

```typescript
// 1. 사용자 메시지
this.chatHistory.push(userEntry);
this.messages.push({ role: "user", content: message });

// 2. AI 응답 (도구 호출)
this.chatHistory.push(assistantEntry);
this.messages.push({
  role: "assistant",
  content: assistantMessage.content || "",
  tool_calls: assistantMessage.tool_calls,
});

// 3. 도구 결과
this.messages.push({
  role: "tool",
  content: result.output,
  tool_call_id: toolCall.id,
});
```

## MCP 통합

### MCP 초기화

```typescript
private async initializeMCP(): Promise<void> {
  // 백그라운드에서 비동기 초기화
  Promise.resolve().then(async () => {
    try {
      const config = loadMCPConfig();
      if (config.servers.length > 0) {
        await initializeMCPServers();
      }
    } catch (error) {
      console.warn("MCP initialization failed:", error);
    } finally {
      this.mcpInitialized = true;
    }
  });
}
```

### MCP 도구 추가

```typescript
// 도구 정의에 MCP 도구 추가
const tools = await getAllGrokTools();
// tools 배열에 MCP 도구도 포함됨

// API 호출 시 MCP 도구도 함께 전송
const response = await this.grokClient.chat(
  this.messages,
  tools, // MCP 도구 포함
  undefined,
  searchOptions
);
```

### MCP 도구 실행

```typescript
// MCP 도구 이름 형식: mcp__{serverName}__{toolName}
if (toolName.startsWith("mcp__")) {
  return await this.executeMCPTool(toolCall);
}

// MCP 도구 실행
private async executeMCPTool(toolCall: GrokToolCall): Promise<ToolResult> {
  const result = await callMCPTool(
    toolCall.function.name,
    JSON.parse(toolCall.function.arguments || "{}")
  );
  return result;
}
```

## 모델 변경

### setModel 메서드

```typescript
setModel(model: string): void {
  this.grokClient.setModel(model);

  // 토큰 카운터 업데이트
  this.tokenCounter = createTokenCounter(model);

  // 설정 저장
  const manager = getSettingsManager();
  manager.updateProjectSetting('model', model);
}
```

### 모델 변경 효과

```
1. API 클라이언트 모델 변경
   └─ 다음 chat() 호출부터 새 모델 사용

2. 토큰 카운터 업데이트
   └─ 새 모델의 토큰 계산 방식 적용

3. 프로젝트 설정 저장
   └─ 다음 세션에 동일 모델 사용
```

## 웹 검색 통합

### 검색 활성화 조건

```typescript
private shouldUseSearchFor(message: string): boolean {
  const q = message.toLowerCase();
  const keywords = [
    "today", "latest", "news", "trending", "breaking",
    "current", "now", "recent", "update on", "price"
  ];

  // 키워드 감지 또는 날짜 패턴 감지
  return keywords.some(k => q.includes(k)) || /(20\d{2})/.test(q);
}
```

### 검색 옵션 전달

```typescript
const response = await this.grokClient.chat(
  this.messages,
  tools,
  undefined,
  this.isGrokModel() && this.shouldUseSearchFor(message)
    ? { search_parameters: { mode: "auto" } }  // 검색 활성화
    : { search_parameters: { mode: "off" } }   // 검색 비활성화
);
```

## 성능 최적화

### 1. 도구 라운드 제한

```typescript
// 무한 루프 방지
let toolRounds = 0;
const maxToolRounds = 400; // 기본값

while (toolRounds < maxToolRounds) {
  toolRounds++;
  // ...
}
```

### 2. 병렬 도구 실행

```typescript
// 현재: 순차 실행
for (const toolCall of assistantMessage.tool_calls) {
  const result = await this.executeTool(toolCall);
}

// 최적화 예 (병렬 실행)
const results = await Promise.all(
  assistantMessage.tool_calls.map(tc => this.executeTool(tc))
);
```

### 3. 토큰 캐싱

```typescript
// 문맥 윈도우 초과 시 오래된 메시지 제거
if (this.tokenCounter.countMessageTokens(this.messages) > contextLimit) {
  // 시스템 메시지는 유지
  const systemMessages = this.messages.filter(m => m.role === 'system');
  const otherMessages = this.messages.filter(m => m.role !== 'system');

  // 오래된 메시지 제거
  this.messages = [
    ...systemMessages,
    ...otherMessages.slice(-MAX_HISTORY_MESSAGES)
  ];
}
```

## 에러 처리

### 에러 타입별 처리

```typescript
try {
  const response = await this.grokClient.chat(...);
  // ...
} catch (error: any) {
  if (error.message.includes("API key")) {
    // API 키 에러
    return {
      success: false,
      error: "Invalid API key"
    };
  } else if (error.message.includes("timeout")) {
    // 타임아웃 에러
    return {
      success: false,
      error: "Request timed out"
    };
  } else if (error.message.includes("context")) {
    // 컨텍스트 윈도우 초과
    return {
      success: false,
      error: "Conversation history too long"
    };
  } else {
    // 기타 에러
    throw error;
  }
}
```

## 결론

GrokAgent는 다음을 통해 강력한 AI 어시스턴트를 구현합니다:

1. **에이전트 루프**: 도구 호출과 결과의 반복적 처리
2. **이중 메모리**: ChatHistory(UI)와 Messages(API)의 분리
3. **도구 오케스트레이션**: 다양한 도구의 통합 실행
4. **토큰 관리**: 효율적인 컨텍스트 윈도우 관리
5. **스트리밍 지원**: 실시간 피드백을 위한 비동기 처리
6. **MCP 통합**: 외부 서비스와의 원활한 연동
