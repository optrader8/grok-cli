# Grok CLI - UI/UX 구조 상세 분석

## 목차

1. [개요](#개요)
2. [Ink/React 아키텍처](#inkréact-아키텍처)
3. [핵심 컴포넌트](#핵심-컴포넌트)
4. [입력 처리 훅](#입력-처리-훅)
5. [스트리밍 UI 업데이트](#스트리밍-ui-업데이트)
6. [확인 대화상자](#확인-대화상자)
7. [스타일링 및 테마](#스타일링-및-테마)
8. [성능 최적화](#성능-최적화)

## 개요

Grok CLI의 UI는 Ink와 React를 사용하여 터미널에서 동적 인터페이스를 구성합니다. React의 선언적 패러다임을 터미널 환경에 적용합니다.

**파일 위치**: `src/ui/`

**핵심 기술**:
- **Ink**: React를 터미널에서 실행
- **React**: UI 상태 관리
- **cfonts**: 로고 렌더링
- **Styled Components**: 터미널 스타일링

## Ink/React 아키텍처

### Ink의 기본 개념

```typescript
import { render } from 'ink';
import { Box, Text } from 'ink';

// React 컴포넌트를 터미널에 렌더링
const App = () => (
  <Box flexDirection="column" padding={1}>
    <Text color="green">Hello, World!</Text>
  </Box>
);

render(<App />);
```

### Ink의 기본 컴포넌트

| 컴포넌트 | 역할 | 속성 |
|---------|------|------|
| Box | 레이아웃 | flexDirection, padding, margin, etc. |
| Text | 텍스트 출력 | color, bold, dimColor, etc. |
| Spacer | 여백 | flex, width, height |

### React Hooks와의 통합

```typescript
// 상태 관리
const [input, setInput] = useState("");
const [isProcessing, setIsProcessing] = useState(false);
const [chatHistory, setChatHistory] = useState([]);

// 이펙트
useEffect(() => {
  console.clear();
}, []);

// 커스텀 훅
const { input, cursorPosition } = useInputHandler({...});
```

## 핵심 컴포넌트

### 1. ChatInterface - 메인 인터페이스

**파일**: `src/ui/components/chat-interface.tsx`

**책임**:
- 전체 UI 오케스트레이션
- 채팅 히스토리 관리
- 사용자 입력 처리
- 확인 대화상자 표시

**구조**:
```typescript
function ChatInterfaceWithAgent({
  agent,
  initialMessage,
}: {
  agent: GrokAgent;
  initialMessage?: string;
}) {
  // 상태
  const [chatHistory, setChatHistory] = useState<ChatEntry[]>([]);
  const [isProcessing, setIsProcessing] = useState(false);
  const [processingTime, setProcessingTime] = useState(0);
  const [tokenCount, setTokenCount] = useState(0);
  const [isStreaming, setIsStreaming] = useState(false);
  const [confirmationOptions, setConfirmationOptions] =
    useState<ConfirmationOptions | null>(null);

  // 입력 처리
  const { input, cursorPosition, showCommandSuggestions, ... } =
    useInputHandler({
      agent,
      chatHistory,
      setChatHistory,
      setIsProcessing,
      setIsStreaming,
      ...
    });

  // UI 렌더링
  return (
    <Box flexDirection="column" paddingX={2}>
      <ChatHistory entries={chatHistory} />
      {isProcessing && <LoadingSpinner />}
      {showCommandSuggestions && <CommandSuggestions />}
      {showModelSelection && <ModelSelection />}
      <ChatInput input={input} cursorPosition={cursorPosition} />
      {confirmationOptions && <ConfirmationDialog />}
    </Box>
  );
}
```

### 2. ChatHistory - 대화 히스토리

**파일**: `src/ui/components/chat-history.tsx`

**책임**:
- ChatEntry 배열 렌더링
- 사용자/AI 메시지 구분
- 도구 호출 및 결과 표시
- 스크롤 관리

**구조**:
```typescript
interface ChatHistoryProps {
  entries: ChatEntry[];
  isStreaming?: boolean;
}

function ChatHistory({ entries, isStreaming }: ChatHistoryProps) {
  return (
    <Box flexDirection="column">
      {entries.map((entry, idx) => (
        <Box key={idx} flexDirection="column" marginBottom={1}>
          {entry.type === "user" && (
            <Text color="cyan" bold>
              You: {entry.content}
            </Text>
          )}
          {entry.type === "assistant" && (
            <Text color="green">
              Grok: {entry.content}
            </Text>
          )}
          {entry.type === "tool_call" && (
            <Text color="yellow">
              Tool: {entry.toolCall?.function.name}
            </Text>
          )}
          {entry.type === "tool_result" && (
            <Text color="blue">
              Result: {entry.content}
            </Text>
          )}
        </Box>
      ))}
    </Box>
  );
}
```

### 3. ChatInput - 입력 필드

**파일**: `src/ui/components/chat-input.tsx`

**책임**:
- 사용자 입력 표시
- 커서 위치 표시
- 명령어 자동완성 표시
- 입력 히스토리 관리

**구조**:
```typescript
interface ChatInputProps {
  input: string;
  cursorPosition: number;
  isProcessing?: boolean;
}

function ChatInput({ input, cursorPosition, isProcessing }: ChatInputProps) {
  return (
    <Box>
      <Text color="gray">{">"}</Text>
      <Text>
        {input.slice(0, cursorPosition)}
        {!isProcessing && <Text inverse>_</Text>}
        {input.slice(cursorPosition)}
      </Text>
    </Box>
  );
}
```

**커서 렌더링**:
```
> hello wo|rld
     └── cursorPosition = 8
         표시되는 문자: "hello wo" + inverse("_") + "rld"
```

### 4. LoadingSpinner - 로딩 표시

**파일**: `src/ui/components/loading-spinner.tsx`

**책임**:
- 처리 중 상태 표시
- 애니메이션 구현
- 경과 시간 표시

**구조**:
```typescript
const frames = ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏'];

function LoadingSpinner({ isActive, elapsedTime }: LoadingSpinnerProps) {
  const [frameIndex, setFrameIndex] = useState(0);

  useEffect(() => {
    if (!isActive) return;

    const interval = setInterval(() => {
      setFrameIndex(prev => (prev + 1) % frames.length);
    }, 80);

    return () => clearInterval(interval);
  }, [isActive]);

  if (!isActive) return null;

  return (
    <Text color="cyan">
      {frames[frameIndex]} Processing... ({formatTime(elapsedTime)})
    </Text>
  );
}
```

### 5. ConfirmationDialog - 확인 대화상자

**파일**: `src/ui/components/confirmation-dialog.tsx`

**책임**:
- 사용자 확인 요청 표시
- 선택지 제공 (승인/거부/모두 승인)
- Diff 렌더링
- VSCode 열기 옵션

**구조**:
```typescript
interface ConfirmationDialogProps {
  options: ConfirmationOptions;
  onConfirm: (confirmed: boolean, dontAskAgain?: boolean) => void;
}

function ConfirmationDialog({ options, onConfirm }: ConfirmationDialogProps) {
  const [selectedOption, setSelectedOption] = useState(0);

  return (
    <Box flexDirection="column" borderStyle="round" borderColor="yellow">
      <Text bold>{options.operation}</Text>
      <Text>{options.filename}</Text>

      {/* Diff 렌더링 */}
      <DiffRenderer content={options.content} />

      {/* 선택지 */}
      <Box>
        <Text
          color={selectedOption === 0 ? "green" : "gray"}
          onClick={() => onConfirm(true)}
        >
          [A]pprove
        </Text>
        <Text color="gray"> </Text>
        <Text
          color={selectedOption === 1 ? "red" : "gray"}
          onClick={() => onConfirm(false)}
        >
          [R]eject
        </Text>
        <Text color="gray"> </Text>
        <Text
          color={selectedOption === 2 ? "green" : "gray"}
          onClick={() => onConfirm(true, true)}
        >
          [A]pprove All
        </Text>
      </Box>
    </Box>
  );
}
```

### 6. CommandSuggestions - 명령 자동완성

**파일**: `src/ui/components/command-suggestions.tsx`

**책임**:
- 입력한 명령어와 일치하는 명령 제안
- 선택 상태 표시
- 상위 3개 제안

**구조**:
```typescript
function CommandSuggestions({
  suggestions,
  selectedIndex,
}: CommandSuggestionsProps) {
  return (
    <Box flexDirection="column">
      {suggestions.slice(0, 3).map((suggestion, idx) => (
        <Text
          key={idx}
          color={idx === selectedIndex ? "green" : "gray"}
          dimColor={idx !== selectedIndex}
        >
          {suggestion}
        </Text>
      ))}
    </Box>
  );
}
```

### 7. ModelSelection - 모델 선택

**파일**: `src/ui/components/model-selection.tsx`

**책임**:
- 사용 가능한 모델 목록 표시
- 현재 선택된 모델 하이라이트
- 모델 변경 처리

**구조**:
```typescript
function ModelSelection({
  models,
  selectedIndex,
  onSelect,
}: ModelSelectionProps) {
  return (
    <Box flexDirection="column" borderStyle="round">
      {models.map((model, idx) => (
        <Text
          key={idx}
          color={idx === selectedIndex ? "green" : "gray"}
          bold={idx === selectedIndex}
        >
          {idx === selectedIndex ? "▶ " : "  "} {model}
        </Text>
      ))}
    </Box>
  );
}
```

### 8. DiffRenderer - 변경사항 표시

**파일**: `src/ui/components/diff-renderer.tsx`

**책임**:
- 파일 변경사항 시각화
- 추가/삭제/수정 줄 구분
- 문법 강조 (선택)

**구조**:
```typescript
function DiffRenderer({ content }: DiffRendererProps) {
  const lines = content.split('\n');

  return (
    <Box flexDirection="column" marginY={1}>
      {lines.map((line, idx) => {
        if (line.startsWith('+')) {
          return <Text key={idx} color="green">{line}</Text>;
        } else if (line.startsWith('-')) {
          return <Text key={idx} color="red">{line}</Text>;
        } else if (line.startsWith('@@')) {
          return <Text key={idx} color="cyan">{line}</Text>;
        } else {
          return <Text key={idx}>{line}</Text>;
        }
      })}
    </Box>
  );
}
```

### 9. MCPStatus - MCP 서버 상태

**파일**: `src/ui/components/mcp-status.tsx`

**책임**:
- 연결된 MCP 서버 표시
- 서버별 도구 수 표시
- 상태 아이콘 표시

**구조**:
```typescript
function MCPStatus({ servers }: MCPStatusProps) {
  return (
    <Box flexDirection="column">
      <Text bold color="blue">MCP Servers:</Text>
      {servers.map(server => (
        <Text key={server.name}>
          {server.connected ? "✓" : "✗"} {server.name}
          ({server.tools.length} tools)
        </Text>
      ))}
    </Box>
  );
}
```

## 입력 처리 훅

### useInputHandler - 메인 입력 처리

**파일**: `src/hooks/use-input-handler.ts`

**책임**:
- 사용자 키보드 입력 감지
- 입력 상태 관리
- 메시지 전송 처리
- 명령 제안 생성

**사용 예**:
```typescript
const {
  input,
  cursorPosition,
  showCommandSuggestions,
  selectedCommandIndex,
  showModelSelection,
  selectedModelIndex,
  commandSuggestions,
  availableModels,
  autoEditEnabled,
} = useInputHandler({
  agent,
  chatHistory,
  setChatHistory,
  setIsProcessing,
  setIsStreaming,
  setTokenCount,
  setProcessingTime,
  processingStartTime,
  isProcessing,
  isStreaming,
  isConfirmationActive: !!confirmationOptions,
});
```

**키 처리**:
```typescript
// Enter: 메시지 전송
// Up/Down: 입력 히스토리 탐색
// Ctrl+K: 명령어 모드 토글
// Ctrl+M: 모델 선택 토글
// Tab: 명령 제안 자동완성
// Escape: 선택 모드 취소
```

### useInputHistory - 입력 히스토리

**파일**: `src/hooks/use-input-history.ts`

**책임**:
- 이전 입력 저장
- 히스토리 탐색 (Up/Down)
- 현재 입력 임시 저장

**구현**:
```typescript
function useInputHistory() {
  const [history, setHistory] = useState<string[]>([]);
  const [historyIndex, setHistoryIndex] = useState(-1);
  const [currentInput, setCurrentInput] = useState("");

  const addToHistory = (input: string) => {
    setHistory(prev => [...prev, input]);
  };

  const goToPrevious = () => {
    if (historyIndex > 0) {
      setHistoryIndex(prev => prev - 1);
    }
  };

  const goToNext = () => {
    if (historyIndex < history.length) {
      setHistoryIndex(prev => prev + 1);
    }
  };

  return { history, historyIndex, addToHistory, goToPrevious, goToNext };
}
```

### useEnhancedInput - 향상된 입력

**파일**: `src/hooks/use-enhanced-input.ts`

**책임**:
- 명령어 제안 생성
- 자동완성
- 구문 강조 (선택)

**명령어 제안**:
```typescript
const suggestions = [
  "/help - Show help",
  "/clear - Clear history",
  "/model - Change model",
  "/settings - Show settings",
];

// 입력에 따라 필터링
const filtered = suggestions.filter(s =>
  s.toLowerCase().includes(input.toLowerCase())
);
```

## 스트리밍 UI 업데이트

### 실시간 업데이트 흐름

```
사용자 입력 제출
  │
  ├─ processUserMessageStream() 호출 (에이전트)
  │
  ├─ for await (const chunk of stream) {
  │    if (chunk.type === "content") {
  │      // 텍스트 실시간 추가
  │      setChatHistory(prev => [
  │        ...prev,
  │        { type: "assistant", content: chunk.content, ... }
  │      ]);
  │    } else if (chunk.type === "tool_calls") {
  │      // 도구 호출 표시
  │      setChatHistory(prev => [
  │        ...prev,
  │        { type: "tool_call", toolCalls: chunk.toolCalls, ... }
  │      ]);
  │    } else if (chunk.type === "tool_result") {
  │      // 도구 결과 표시
  │      // ...
  │    }
  │  }
  │
  └─ 완료 (처리 완료)
```

### 상태 업데이트

```typescript
for await (const chunk of agent.processUserMessageStream(message)) {
  switch (chunk.type) {
    case "content":
      // 텍스트 청크 추가
      setChatHistory(prev => {
        const last = prev[prev.length - 1];
        if (last?.type === "assistant" && last?.isStreaming) {
          // 기존 assistant 메시지 업데이트
          return [
            ...prev.slice(0, -1),
            {
              ...last,
              content: (last.content || "") + chunk.content
            }
          ];
        }
        // 새 메시지 추가
        return [
          ...prev,
          {
            type: "assistant",
            content: chunk.content,
            timestamp: new Date(),
            isStreaming: true
          }
        ];
      });
      break;

    case "token_count":
      setTokenCount(chunk.tokenCount || 0);
      break;

    case "done":
      setIsProcessing(false);
      break;
  }
}
```

## 확인 대화상자

### 확인 흐름

```
도구가 사용자 확인 필요
  │
  ├─ ConfirmationService.requestConfirmation() 호출
  │
  ├─ confirmation-requested 이벤트 발생
  │
  ├─ ChatInterface가 이벤트 리스닝
  │
  ├─ setConfirmationOptions() 호출
  │
  ├─ ConfirmationDialog 렌더링
  │
  ├─ 사용자 입력 대기
  │  ├─ A: 승인
  │  ├─ R: 거부
  │  └─ A: 모두 승인
  │
  ├─ confirmationService.confirmOperation() 호출
  │
  └─ Promise 해결 (도구 계속 실행)
```

### 구현

```typescript
// 에이전트에서
const confirmationResult = await confirmationService
  .requestConfirmation({
    operation: "Create File",
    filename: "test.ts",
    content: fileDiff
  });

if (!confirmationResult.confirmed) {
  return { success: false, error: "User rejected" };
}

// UI에서
useEffect(() => {
  const handleConfirmationRequested = (options) => {
    setConfirmationOptions(options);
  };

  confirmationService.on('confirmation-requested',
    handleConfirmationRequested);

  return () => {
    confirmationService.off('confirmation-requested',
      handleConfirmationRequested);
  };
}, []);

// 사용자 응답 처리
const handleConfirmation = (confirmed, dontAskAgain) => {
  confirmationService.confirmOperation(confirmed, dontAskAgain);
  setConfirmationOptions(null);
};
```

## 스타일링 및 테마

### 색상 사용

```typescript
// 역할별 색상
const colors = {
  user: "cyan",           // 사용자 입력
  assistant: "green",     // AI 응답
  tool: "yellow",         // 도구 실행
  toolResult: "blue",     // 도구 결과
  error: "red",           // 에러
  warning: "yellow",      // 경고
  success: "green",       // 성공
  info: "blue",           // 정보
};
```

### 로고 렌더링

```typescript
import cfonts from 'cfonts';

const logoOutput = cfonts.render("GROK", {
  font: "3d",
  align: "left",
  colors: ["magenta", "gray"],
  gradient: ["magenta", "cyan"],
  transitionGradient: true,
});

console.log(logoOutput.string);
```

### Box 스타일링

```typescript
<Box
  flexDirection="column"
  paddingX={2}
  paddingY={1}
  borderStyle="round"
  borderColor="cyan"
  marginBottom={1}
>
  <Text bold color="green">Title</Text>
  <Text>Content</Text>
</Box>
```

## 성능 최적화

### 1. 메모이제이션

```typescript
const ChatHistory = React.memo(({ entries }) => {
  return (
    <Box flexDirection="column">
      {entries.map((entry) => (
        <ChatEntry key={entry.id} entry={entry} />
      ))}
    </Box>
  );
});
```

### 2. 스크롤 관리

```typescript
const scrollRef = useRef<any>();

useEffect(() => {
  // 새 메시지 추가 시 자동 스크롤
  if (scrollRef.current) {
    scrollRef.current.scrollToBottom?.();
  }
}, [chatHistory]);
```

### 3. 렌더링 최적화

```typescript
// 불필요한 리렌더링 방지
const memoizedAgent = useMemo(() => agent, [agent]);
const memoizedHistory = useMemo(() => chatHistory, [chatHistory]);
```

### 4. 히스토리 크기 제한

```typescript
// 오래된 메시지 제거
const MAX_HISTORY = 100;

if (chatHistory.length > MAX_HISTORY) {
  setChatHistory(prev => prev.slice(-MAX_HISTORY));
}
```

## 결론

UI 시스템의 주요 특징:

1. **React 기반**: 선언적 UI 관리
2. **실시간 업데이트**: 스트리밍으로 즉각적 피드백
3. **사용자 경험**: 직관적인 입력 및 확인
4. **확장성**: 컴포넌트 기반 아키텍처
5. **성능**: 메모이제이션 및 최적화
