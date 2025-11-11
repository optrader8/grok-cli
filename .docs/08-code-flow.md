# Grok CLI - 완전한 코드 흐름 분석

## 목차

1. [개요](#개요)
2. [프로그램 시작 흐름](#프로그램-시작-흐름)
3. [대화형 모드 (Interactive Mode)](#대화형-모드-interactive-mode)
4. [헤드리스 모드 (Headless Mode)](#헤드리스-모드-headless-mode)
5. [메시지 처리 흐름](#메시지-처리-흐름)
6. [파일 작업 흐름 (확인 시스템)](#파일-작업-흐름-확인-시스템)
7. [스트리밍 응답 흐름](#스트리밍-응답-흐름)
8. [에러 처리 흐름](#에러-처리-흐름)
9. [MCP 도구 실행 흐름](#mcp-도구-실행-흐름)
10. [전체 통신 다이어그램](#전체-통신-다이어그램)

## 개요

Grok CLI의 전체 코드 흐름은 다음과 같은 단계로 나뉩니다:

1. **초기화**: 프로그램 시작 및 설정 로드
2. **모드 선택**: 대화형 또는 헤드리스 모드
3. **메시지 처리**: 사용자 입력 → AI 응답
4. **도구 실행**: 필요시 도구 호출 및 실행
5. **결과 반환**: 사용자에게 결과 표시

## 프로그램 시작 흐름

### 1단계: CLI 진입점

```
$ grok
  │
  ├─ src/index.ts 실행
  │  ├─ Commander.js로 인자 파싱
  │  ├─ --api-key, --base-url, --model 등 수집
  │  └─ --prompt가 있으면 헤드리스 모드
  │
  └─ 설정 로드
```

**코드**:
```typescript
// src/index.ts
import { program } from 'commander';
import { ChatInterface } from './ui/app.js';

program
  .name("grok")
  .version("1.0.1")
  .option("-d, --directory <dir>", "Working directory")
  .option("-k, --api-key <key>", "Grok API key")
  .option("-u, --base-url <url>", "API Base URL")
  .option("-m, --model <model>", "Model name")
  .option("-p, --prompt <prompt>", "Headless mode prompt")
  .option("--max-tool-rounds <rounds>", "Max tool rounds", "400")
  .action(async (message, options) => {
    // 옵션 처리
    const apiKey = options.apiKey || process.env.GROK_API_KEY;
    if (!apiKey) {
      console.error("Error: API key not found");
      process.exit(1);
    }

    const baseUrl = options.baseUrl || process.env.GROK_BASE_URL;
    const model = options.model || process.env.GROK_MODEL;
    const maxToolRounds = parseInt(options.maxToolRounds);

    // 에이전트 생성
    const agent = new GrokAgent(apiKey, baseUrl, model, maxToolRounds);

    // 모드 선택
    if (options.prompt) {
      // 헤드리스 모드
      const entries = await agent.processUserMessage(options.prompt);
      entries.forEach(e => console.log(e.content));
    } else {
      // 대화형 모드
      render(
        <ChatInterface agent={agent} initialMessage={message} />
      );
    }
  });

program.parse(process.argv);
```

### 2단계: 설정 로드

```
환경 변수 확인
  │
  ├─ GROK_API_KEY
  ├─ GROK_BASE_URL
  ├─ GROK_MODEL
  ├─ GROK_MAX_TOKENS
  └─ MORPH_API_KEY (선택)
  │
  ↓
명령줄 플래그 확인
  │
  ├─ --api-key
  ├─ --base-url
  ├─ --model
  ├─ --max-tool-rounds
  └─ --prompt (헤드리스)
  │
  ↓
설정 파일 로드
  │
  ├─ ~/.grok/user-settings.json (사용자)
  └─ .grok/settings.json (프로젝트)
  │
  ↓
GrokClient 초기화
  │
  ├─ API 키로 OpenAI 클라이언트 생성
  ├─ Base URL 설정
  └─ 모델 설정
  │
  ↓
GrokAgent 생성
  │
  ├─ 모든 도구 초기화
  ├─ TokenCounter 생성
  ├─ 시스템 프롬프트 로드
  └─ MCP 서버 초기화 (백그라운드)
```

### 3단계: 모드 선택

```
헤드리스 모드 (--prompt 있음)      대화형 모드 (--prompt 없음)
  │                                  │
  ├─ agent.processUserMessage()      ├─ Ink/React 렌더링
  │                                  │
  ├─ ChatEntry[] 반환                ├─ ChatInterface 컴포넌트
  │                                  │
  ├─ 콘솔에 출력                      ├─ 터미널 UI 표시
  │                                  │
  └─ 프로세스 종료                    └─ 사용자 입력 대기
```

## 대화형 모드 (Interactive Mode)

### 전체 흐름

```
프로그램 시작 (--prompt 없음)
  │
  ├─ Ink/React 초기화
  │  ├─ ChatInterface 컴포넌트 렌더링
  │  ├─ 로고 표시 (cfonts)
  │  └─ 입력 필드 준비
  │
  ├─ useInputHandler 훅 시작
  │  ├─ stdin 리스너 설정
  │  ├─ 키 입력 감지
  │  └─ 입력 상태 관리
  │
  ├─ 입력 표시
  │  ├─ ChatHistory 컴포넌트
  │  ├─ ChatInput 컴포넌트
  │  └─ 명령 제안 (선택)
  │
  ├─ 사용자 입력 대기
  │  └─ Enter 키 (메시지 전송)
  │
  ├─ processUserMessageStream() 호출
  │  ├─ 에이전트 루프 시작
  │  └─ 스트리밍 청크 수신
  │
  ├─ UI 실시간 업데이트
  │  ├─ 텍스트 청크 추가
  │  ├─ 도구 호출 표시
  │  └─ 도구 결과 표시
  │
  ├─ 다음 입력 대기
  │  └─ 반복
  │
  └─ Ctrl+C로 종료
```

## 헤드리스 모드 (Headless Mode)

### 전체 흐름

```
프로그램 시작 (--prompt 있음)
  │
  ├─ GrokAgent 생성
  │
  ├─ processUserMessage(prompt) 호출
  │  ├─ 사용자 메시지 추가
  │  ├─ AI에 전송
  │  ├─ 도구 호출 확인
  │  ├─ 도구 실행 (필요시)
  │  └─ ChatEntry[] 반환
  │
  ├─ 결과 콘솔 출력
  │  └─ entries.map(e => console.log(e.content))
  │
  └─ 프로세스 종료
     └─ process.exit(0)
```

## 메시지 처리 흐름

### 블로킹 방식 (processUserMessage)

```
사용자 메시지 입력
  │
  ├─ 1. 사용자 메시지를 messages 배열에 추가
  │
  ├─ 2. grokClient.chat() 호출
  │     │
  │     └─ POST https://api.x.ai/v1/chat/completions
  │        ├─ messages: [system, user, ...]
  │        ├─ tools: [create_file, edit_file, ...]
  │        ├─ tool_choice: "auto"
  │        ├─ temperature: 0.7
  │        ├─ max_tokens: 1536
  │        └─ stream: false
  │
  ├─ 3. API 응답 수신
  │     ├─ 일반 응답: content
  │     └─ 도구 호출: tool_calls[]
  │
  ├─ 4. toolRounds < maxToolRounds?
  │     │
  │     ├─ YES (도구 호출 있음)
  │     │   │
  │     │   ├─ 5. 각 도구 호출 실행
  │     │   │     ├─ 도구 이름으로 분기
  │     │   │     ├─ 도구 메서드 호출
  │     │   │     └─ ToolResult 반환
  │     │   │
  │     │   ├─ 6. 도구 결과를 tool 메시지로 추가
  │     │   │
  │     │   ├─ 7. 도구 결과와 함께 다시 2번으로 (API 호출)
  │     │   │
  │     │   └─ toolRounds++ (증가)
  │     │
  │     └─ NO (도구 호출 없음 또는 최대 라운드)
  │         │
  │         └─ 8. 최종 응답 반환
  │
  └─ ChatEntry[] (전체 히스토리)
```

**코드**:
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
    const tools = await getAllGrokTools();

    // 2. 초기 API 호출
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

      // 4-YES: 도구 호출 있음
      if (
        assistantMessage.tool_calls &&
        assistantMessage.tool_calls.length > 0
      ) {
        toolRounds++;

        // 5. 각 도구 호출 실행
        for (const toolCall of assistantMessage.tool_calls) {
          const result = await this.executeTool(toolCall);

          // 6. 도구 결과 추가
          this.messages.push({
            role: "tool",
            content: result.success ? result.output || "Success" : result.error || "Error",
            tool_call_id: toolCall.id,
          } as any);
        }

        // 7. 다시 API 호출
        currentResponse = await this.grokClient.chat(
          this.messages,
          tools
        );
      } else {
        // 4-NO: 최종 응답
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
    throw error;
  }
}
```

### 스트리밍 방식 (processUserMessageStream)

```
사용자 메시지 입력
  │
  ├─ grokClient.chatStream() 호출
  │  │
  │  └─ GET (스트리밍)
  │     ├─ 첫 청크: role="assistant", content=""
  │     ├─ 텍스트 청크들: content="I'll..."
  │     ├─ 도구 호출 청크: tool_calls=[...]
  │     └─ 마지막 청크: finish_reason
  │
  ├─ 청크 누적 (messageReducer)
  │  └─ 부분 청크를 완전한 객체로 병합
  │
  ├─ for await (const chunk of stream)
  │  │
  │  ├─ TextContent Chunk
  │  │  │
  │  │  └─ yield { type: "content", content: "텍스트" }
  │  │
  │  ├─ Tool Call Detection
  │  │  │
  │  │  ├─ 도구 호출 발견
  │  │  │
  │  │  └─ 도구 실행
  │  │     ├─ executeTool()
  │  │     └─ yield { type: "tool_result", ... }
  │  │
  │  └─ Completion
  │     │
  │     └─ yield { type: "done" }
  │
  └─ AsyncGenerator<StreamingChunk>
```

## 파일 작업 흐름 (확인 시스템)

### 확인 시스템 세부 흐름

```
TextEditorTool.create() 호출
  │
  ├─ 1. 세션 플래그 확인
  │     ├─ fileOperations 승인됨?
  │     │  YES → 5번으로 (파일 작업 실행)
  │     │
  │     └─ NO → 2번으로
  │
  ├─ 2. ConfirmationService.requestConfirmation() 호출
  │     │
  │     └─ ConfirmationOptions 객체 생성
  │        ├─ operation: "Create File"
  │        ├─ filename: "test.ts"
  │        └─ content: "파일 내용 또는 Diff"
  │
  ├─ 3. EventEmitter로 UI에 신호
  │     │
  │     └─ confirmationService.emit('confirmation-requested', options)
  │
  ├─ 4. ChatInterface가 이벤트 리스닝
  │     │
  │     ├─ 이벤트 감지
  │     │
  │     ├─ setConfirmationOptions(options) 호출
  │     │
  │     └─ ConfirmationDialog 렌더링
  │
  ├─ 5. 사용자 입력 대기
  │     │
  │     ├─ A키: 승인
  │     │   └─ confirmationService.confirmOperation(true, false)
  │     │
  │     ├─ R키: 거부
  │     │   └─ confirmationService.confirmOperation(false)
  │     │
  │     └─ A키 (모두): 모두 승인
  │         └─ confirmationService.confirmOperation(true, true)
  │
  ├─ 6. Promise 해결
  │     │
  │     └─ requestConfirmation()의 Promise resolve
  │
  ├─ 7. 결과 확인
  │     │
  │     ├─ confirmed === true?
  │     │  │
  │     │  ├─ YES → 8번으로 (파일 작업)
  │     │  │
  │     │  └─ NO → { success: false, error: "User rejected" }
  │     │
  │     └─ dontAskAgain === true?
  │         │
  │         └─ YES → sessionFlags.fileOperations = true
  │
  ├─ 8. 파일 작업 실행
  │     ├─ fs.writeFile() 또는 fs.readFile()
  │     ├─ 파일 시스템 작업 수행
  │     └─ ToolResult 반환
  │
  ├─ 9. Diff 생성 (필요시)
  │     │
  │     └─ 변경사항을 diff 형식으로 표시
  │
  └─ ToolResult 반환
     ├─ success: true/false
     ├─ output: 작업 결과
     └─ error: 에러 메시지 (실패 시)
```

### 코드 구현

```typescript
// TextEditorTool
async create(filePath: string, content: string): Promise<ToolResult> {
  try {
    const resolvedPath = path.resolve(filePath);

    // 1. 세션 플래그 확인
    const sessionFlags = this.confirmationService.getSessionFlags();
    if (!sessionFlags.fileOperations && !sessionFlags.allOperations) {
      // 2. 확인 요청
      const confirmationResult = await this.confirmationService
        .requestConfirmation({
          operation: 'Create File',
          filename: filePath,
          content: content
        });

      // 7. 결과 확인
      if (!confirmationResult.confirmed) {
        return {
          success: false,
          error: 'File creation cancelled by user'
        };
      }
    }

    // 8. 파일 작업 실행
    await fs.writeFile(resolvedPath, content, 'utf-8');

    // 9. Diff 생성
    const diff = this.generateDiff(filePath, '', content);

    return {
      success: true,
      output: `File created: ${filePath}\n${diff}`
    };
  } catch (error: any) {
    return {
      success: false,
      error: `Error creating file: ${error.message}`
    };
  }
}
```

## 스트리밍 응답 흐름

### 1. 스트리밍 초기화

```typescript
async *processUserMessageStream(message: string)
  : AsyncGenerator<StreamingChunk> {

  // 사용자 메시지 추가
  this.messages.push({ role: "user", content: message });

  const tools = await getAllGrokTools();

  // 스트리밍 API 호출
  for await (const chunk of this.grokClient.chatStream(
    this.messages,
    tools
  )) {
    // 청크 처리 및 yield
  }
}
```

### 2. 청크 누적 (messageReducer)

```typescript
private messageReducer(previous: any, item: any): any {
  const reduce = (acc: any, delta: any) => {
    acc = { ...acc };

    for (const [key, value] of Object.entries(delta)) {
      if (acc[key] === undefined || acc[key] === null) {
        // 새 필드
        acc[key] = value;
      } else if (typeof acc[key] === "string" && typeof value === "string") {
        // 문자열 연결
        acc[key] += value;
      } else if (Array.isArray(acc[key]) && Array.isArray(value)) {
        // 배열 병합
        const accArray = acc[key];
        for (let i = 0; i < value.length; i++) {
          if (!accArray[i]) accArray[i] = {};
          accArray[i] = reduce(accArray[i], value[i]);
        }
      } else if (typeof acc[key] === "object" && typeof value === "object") {
        // 객체 병합
        acc[key] = reduce(acc[key], value);
      }
    }

    return acc;
  };

  return reduce(previous, item.choices[0]?.delta || {});
}
```

**청크 누적 예**:

```
청크 1: { "content": "I " }
  → accumulated = { content: "I " }

청크 2: { "content": "will " }
  → accumulated = { content: "I will " }

청크 3: { "tool_calls": [{ "id": "1", "function": {"name": "create_file", ...} }] }
  → accumulated = { content: "I will ", tool_calls: [{ ... }] }

청크 4: { "tool_calls": [{ "id": "1", "function": {"arguments": "{\"path" }}] }
  → accumulated = { ..., tool_calls: [{ ..., arguments: "{\"path" }] }

청크 5: { "tool_calls": [{ "id": "1", "function": {"arguments": "\":\"..." }}] }
  → accumulated = { ..., tool_calls: [{ ..., arguments: "{\"path\":\"..." }] }
```

### 3. UI 업데이트

```
청크 → messageReducer → 확인 → yield

다음 데이터 타입별 처리:

text 데이터:
  └─ yield { type: "content", content: "텍스트" }

tool_calls 완료 감지:
  ├─ hasCompleteTool = true (function.name 있음)
  ├─ 도구 실행
  └─ yield { type: "tool_calls", toolCalls: [...] }

도구 결과:
  └─ yield { type: "tool_result", toolCall, toolResult }

스트림 종료:
  └─ yield { type: "done" }
```

## 에러 처리 흐름

### API 에러

```
API 호출
  │
  ├─ 성공
  │  └─ 응답 반환
  │
  └─ 실패
     │
     ├─ 401 Unauthorized
     │  └─ "Invalid API key"
     │
     ├─ 429 Too Many Requests
     │  └─ "Rate limit exceeded"
     │
     ├─ 500+ Server Error
     │  └─ "API server error"
     │
     ├─ Timeout (360초)
     │  └─ "Request timeout"
     │
     └─ Network Error
        └─ "Connection failed"
```

### 도구 실행 에러

```
도구 호출
  │
  ├─ 성공
  │  └─ ToolResult { success: true, output }
  │
  └─ 실패
     │
     ├─ 파일 불포함
     │  └─ ToolResult { success: false, error: "File not found" }
     │
     ├─ 권한 없음
     │  └─ ToolResult { success: false, error: "Permission denied" }
     │
     ├─ 형식 오류
     │  └─ ToolResult { success: false, error: "Invalid arguments" }
     │
     └─ 예외 발생
        └─ ToolResult { success: false, error: "Exception: ..." }
```

### 컨텍스트 윈도우 초과

```
메시지 토큰 계산
  │
  ├─ totalTokens > contextLimit?
  │  │
  │  └─ YES
  │     ├─ "Maximum context length exceeded" 에러
  │     │
  │     ├─ 해결책 1: 대화 히스토리 요약
  │     ├─ 해결책 2: 오래된 메시지 제거
  │     └─ 해결책 3: 더 작은 모델로 변경
  │
  └─ NO
     └─ 진행
```

## MCP 도구 실행 흐름

### 완전한 MCP 호출 흐름

```
AI가 MCP 도구 호출
  │
  ├─ 1. 도구 이름: mcp__linear__create_issue
  │
  ├─ 2. executeMCPTool() 호출
  │
  ├─ 3. 도구 이름 파싱
  │     ├─ serverName = "linear"
  │     └─ toolName = "create_issue"
  │
  ├─ 4. MCPManager 획득
  │     └─ const manager = getMCPManager()
  │
  ├─ 5. 서버 연결 확인
  │     ├─ 서버 연결됨?
  │     │  YES → 6번으로
  │     │
  │     └─ NO
  │        ├─ addServer() 호출
  │        ├─ 도구 목록 로드
  │        └─ 6번으로
  │
  ├─ 6. 도구 호출 전송
  │     │
  │     └─ client.callTool({
  │        ├─ name: "create_issue"
  │        └─ arguments: { title, body, ... }
  │     })
  │
  ├─ 7. MCP 서버 처리
  │     ├─ JSON-RPC 요청 수신
  │     ├─ 도구 로직 실행
  │     └─ JSON-RPC 응답 전송
  │
  ├─ 8. 결과 수신
  │     ├─ content: [{ type: "text", text: "..." }]
  │     └─ OR error
  │
  ├─ 9. ToolResult로 변환
  │     ├─ success: true/false
  │     └─ output/error: 결과 메시지
  │
  └─ API에 도구 결과 전송
     └─ { role: "tool", content: "결과" }
```

## 전체 통신 다이어그램

### 시스템 간 통신

```
┌─────────────────────────────────────────────────────────────┐
│                    사용자 (터미널)                          │
└─────────────────────────┬───────────────────────────────────┘
                          │ 키 입력
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Grok CLI (로컬)                          │
│                                                             │
│  ├─ ChatInterface (UI)                                     │
│  ├─ useInputHandler (입력 처리)                            │
│  ├─ GrokAgent (오케스트레이션)                            │
│  │  ├─ TextEditorTool                                     │
│  │  ├─ BashTool                                           │
│  │  ├─ SearchTool                                         │
│  │  ├─ TodoTool                                           │
│  │  └─ ConfirmationTool                                   │
│  └─ GrokClient (API)                                      │
│                                                             │
└─────────────────────────┬───────────────────────────────────┘
                          │ HTTP/gRPC
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                   Grok API (원격)                          │
│              https://api.x.ai/v1/chat/completions         │
│                                                             │
│  메시지 배열 → AI 처리 → 응답 생성                         │
│  (선택: 도구 호출)                                         │
│                                                             │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
    ┌─────────┐      ┌──────────┐    ┌──────────┐
    │ Linear  │      │ GitHub   │    │ 기타 MCP │
    │ (SSE)   │      │ (Stdio)  │    │ 서버     │
    └─────────┘      └──────────┘    └──────────┘

  MCP 프로토콜로 통신 ←→ MCPManager
```

### 메시지 생명주기

```
1. 사용자 입력
   "Create a file called test.ts"

2. ChatEntry 생성
   {
     type: "user",
     content: "Create a file called test.ts",
     timestamp: Date
   }

3. GrokMessage 추가
   { role: "user", content: "Create a file called test.ts" }

4. API 호출
   POST /chat/completions
   {
     messages: [system, user, ...],
     tools: [create_file, edit_file, ...],
     ...
   }

5. API 응답
   {
     choices: [{
       message: {
         role: "assistant",
         tool_calls: [{
           id: "call_123",
           function: {
             name: "create_file",
             arguments: "{\"path\": \"test.ts\", \"content\": \"...\"}"
           }
         }]
       }
     }]
   }

6. 도구 호출
   ExecuteToolCall {
     name: "create_file",
     path: "test.ts",
     content: "..."
   }

7. 도구 실행
   TextEditorTool.create()
   ├─ 확인 요청 (필요시)
   ├─ 파일 작업 실행
   └─ ToolResult 반환

8. 도구 결과 메시지
   {
     role: "tool",
     content: "File created: test.ts",
     tool_call_id: "call_123"
   }

9. 최종 API 호출
   POST /chat/completions
   {
     messages: [system, user, assistant, tool, ...],
     ...
   }

10. 최종 응답
    {
      choices: [{
        message: {
          role: "assistant",
          content: "I've created the file test.ts for you."
        }
      }]
    }

11. ChatEntry 추가
    {
      type: "assistant",
      content: "I've created the file test.ts for you.",
      timestamp: Date
    }

12. UI 업데이트
    ChatHistory에 모든 항목 표시
```

## 결론

Grok CLI의 코드 흐름은 다음과 같은 핵심 원칙을 따릅니다:

1. **이벤트 기반**: 사용자 입력 → 상태 변경 → UI 업데이트
2. **에이전트 루프**: 도구 호출 검출 → 실행 → 결과 전달 → 반복
3. **비동기 처리**: 스트리밍으로 실시간 피드백
4. **안전성**: 사용자 확인 시스템으로 위험 작업 보호
5. **확장성**: MCP를 통한 외부 서비스 통합

이 설계를 통해 Grok CLI는 강력하고 유연하며 사용하기 쉬운 AI 어시스턴트를 제공합니다.
