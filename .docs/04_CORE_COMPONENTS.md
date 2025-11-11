# 핵심 컴포넌트

이 문서는 Grok CLI의 핵심 소스 파일들을 상세히 분석합니다.

## 📑 목차

1. [메인 엔트리 포인트](#1-메인-엔트리-포인트)
2. [Grok Agent](#2-grok-agent)
3. [Grok API Client](#3-grok-api-client)
4. [Tools - 파일 편집](#4-tools---파일-편집)
5. [Tools - 검색](#5-tools---검색)
6. [Tools - Bash](#6-tools---bash)
7. [MCP 통합](#7-mcp-통합)
8. [설정 관리](#8-설정-관리)
9. [UI 컴포넌트](#9-ui-컴포넌트)
10. [확인 서비스](#10-확인-서비스)

---

## 1. 메인 엔트리 포인트

### src/index.ts (463 lines)

앱의 진입점이자 CLI 설정을 담당합니다.

#### 주요 기능

```typescript
#!/usr/bin/env node

import { Command } from 'commander';
import { render } from 'ink';
import App from './ui/app.js';

const program = new Command();

program
  .name('grok')
  .description('Grok CLI - AI-powered terminal assistant')
  .version('0.0.33')
  .option('-k, --api-key <key>', 'Grok API key')
  .option('-u, --base-url <url>', 'API base URL')
  .option('-m, --model <model>', 'AI model to use')
  .option('-d, --directory <path>', 'Working directory')
  .option('-p, --prompt <text>', 'Headless mode prompt')
  .option('--max-tool-rounds <n>', 'Max tool rounds', '400')
  .argument('[message...]', 'Initial message')
  .action(async (message, options) => {
    // CLI 로직
  });
```

#### API 키 로드 우선순위

```typescript
// 1. CLI 인자
const apiKey = options.apiKey ||
  // 2. 환경 변수
  process.env.GROK_API_KEY ||
  // 3. 사용자 설정
  settingsManager.getUserSettings().apiKey;

if (!apiKey) {
  console.error('Error: GROK_API_KEY is required');
  process.exit(1);
}
```

#### 모드 선택

```typescript
// 1. Git 명령 모드
if (program.args[0] === 'git') {
  await handleGitCommand(program.args[1]);
  return;
}

// 2. 헤드리스 모드
if (options.prompt) {
  await runHeadlessMode(options.prompt, config);
  return;
}

// 3. 대화형 모드 (기본)
render(<App config={config} initialMessage={message} />);
```

#### 시그널 핸들링

```typescript
// SIGTERM 우아한 종료
process.on('SIGTERM', () => {
  console.log('\nReceived SIGTERM, shutting down...');
  process.exit(0);
});

// 미처리 예외
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
  process.exit(1);
});
```

#### MCP 명령 등록

```typescript
// grok mcp add <name> [options]
program
  .command('mcp')
  .command('add')
  .argument('<name>', 'Server name')
  .option('-t, --transport <type>', 'Transport type')
  .option('-c, --command <cmd>', 'Command to run')
  .option('-a, --args <args...>', 'Command arguments')
  .option('-u, --url <url>', 'Server URL')
  .action(async (name, options) => {
    await addMCPServer(name, options);
  });
```

---

## 2. Grok Agent

### src/agent/grok-agent.ts (781 lines)

AI 에이전트의 핵심 로직을 담당하는 가장 중요한 파일입니다.

#### 클래스 구조

```typescript
export class GrokAgent {
  private client: OpenAI;
  private tools: Tool[];
  private conversationHistory: Message[] = [];
  private mcpManager: MCPManager;
  private confirmationService: ConfirmationService;
  private abortController?: AbortController;

  constructor(config: AgentConfig) {
    this.client = createGrokClient(config);
    this.tools = this.initializeTools();
    this.mcpManager = MCPManager.getInstance();
    this.confirmationService = ConfirmationService.getInstance();
  }

  async processMessage(message: string): Promise<string> { }
  async processMessageStream(message: string): AsyncGenerator<string> { }
  private async executeToolCall(toolCall: ToolCall): Promise<ToolResult> { }
}
```

#### 도구 실행 루프

```typescript
async processMessage(message: string): Promise<string> {
  // 1. 메시지를 히스토리에 추가
  this.conversationHistory.push({
    role: 'user',
    content: message
  });

  let round = 0;
  const maxRounds = 400;

  while (round < maxRounds) {
    // 2. AI에게 요청
    const response = await this.client.chat.completions.create({
      model: this.config.model,
      messages: this.conversationHistory,
      tools: this.tools
    });

    const choice = response.choices[0];

    // 3. 종료 조건 확인
    if (choice.finish_reason === 'stop') {
      return choice.message.content;
    }

    // 4. 도구 호출 실행
    if (choice.message.tool_calls) {
      for (const toolCall of choice.message.tool_calls) {
        const result = await this.executeToolCall(toolCall);

        // 결과를 히스토리에 추가
        this.conversationHistory.push({
          role: 'tool',
          tool_call_id: toolCall.id,
          content: JSON.stringify(result)
        });
      }
    }

    round++;
  }

  throw new Error('Max tool rounds reached');
}
```

#### 스트리밍 처리

```typescript
async *processMessageStream(message: string) {
  this.conversationHistory.push({
    role: 'user',
    content: message
  });

  let round = 0;
  const maxRounds = 400;

  while (round < maxRounds) {
    const stream = await this.client.chat.completions.create({
      model: this.config.model,
      messages: this.conversationHistory,
      tools: this.tools,
      stream: true  // 스트리밍 활성화
    });

    let currentMessage = '';
    let currentToolCalls: ToolCall[] = [];

    // 스트림 청크 처리
    for await (const chunk of stream) {
      const delta = chunk.choices[0]?.delta;

      // 텍스트 콘텐츠
      if (delta?.content) {
        currentMessage += delta.content;
        yield delta.content;  // 즉시 출력
      }

      // 도구 호출
      if (delta?.tool_calls) {
        // 도구 호출 정보 축적
        for (const toolCall of delta.tool_calls) {
          if (!currentToolCalls[toolCall.index]) {
            currentToolCalls[toolCall.index] = {
              id: toolCall.id || '',
              type: 'function',
              function: { name: '', arguments: '' }
            };
          }

          // 함수명
          if (toolCall.function?.name) {
            currentToolCalls[toolCall.index].function.name = toolCall.function.name;

            // 함수명 확인 즉시 준비 시작
            yield `\n[Tool: ${toolCall.function.name}]\n`;
          }

          // 인자 축적
          if (toolCall.function?.arguments) {
            currentToolCalls[toolCall.index].function.arguments +=
              toolCall.function.arguments;
          }
        }
      }
    }

    // 스트림 종료 후 도구 실행
    if (currentToolCalls.length > 0) {
      for (const toolCall of currentToolCalls) {
        const result = await this.executeToolCall(toolCall);
        yield `\n[Result: ${result.output}]\n`;

        this.conversationHistory.push({
          role: 'tool',
          tool_call_id: toolCall.id,
          content: JSON.stringify(result)
        });
      }
    } else {
      // 도구 호출 없으면 종료
      break;
    }

    round++;
  }
}
```

#### 도구 실행

```typescript
private async executeToolCall(toolCall: ToolCall): Promise<ToolResult> {
  const { name, arguments: argsStr } = toolCall.function;
  const args = JSON.parse(argsStr);

  // MCP 도구
  if (name.startsWith('mcp__')) {
    const [, serverName, toolName] = name.split('__');
    return await this.mcpManager.executeTool(serverName, toolName, args);
  }

  // 빌트인 도구
  switch (name) {
    case 'view_file':
      return await viewFile(args);

    case 'create_file':
      // 사용자 확인 필요
      const approved = await this.confirmationService.requestConfirmation({
        tool: 'create_file',
        args,
        description: `Create file: ${args.path}`
      });

      if (!approved) {
        return { output: 'User rejected the operation' };
      }

      return await createFile(args);

    case 'str_replace_editor':
      return await strReplaceEditor(args);

    case 'bash':
      return await executeBash(args);

    case 'search':
      return await search(args);

    case 'create_todo_list':
      return await createTodoList(args);

    case 'update_todo_list':
      return await updateTodoList(args);

    default:
      return { output: `Unknown tool: ${name}` };
  }
}
```

#### 웹 검색 자동 활성화

```typescript
// Grok 모델에서만 웹 검색 지원
private shouldEnableWebSearch(message: string): boolean {
  if (!this.config.model.includes('grok')) {
    return false;
  }

  // 실시간 정보가 필요한 쿼리 감지
  const webSearchKeywords = [
    'latest', 'recent', 'current', 'today',
    'news', 'update', 'price', 'weather'
  ];

  return webSearchKeywords.some(keyword =>
    message.toLowerCase().includes(keyword)
  );
}

// API 호출 시 적용
const response = await this.client.chat.completions.create({
  model: this.config.model,
  messages: this.conversationHistory,
  tools: this.tools,
  // @ts-ignore (Grok 전용 파라미터)
  search: this.shouldEnableWebSearch(message) ? {
    enabled: true
  } : undefined
});
```

#### 시스템 프롬프트

```typescript
private getSystemPrompt(): string {
  const basePrompt = `You are Grok, an AI coding assistant...`;

  // 커스텀 지침 추가
  const customInstructions = loadCustomInstructions();

  if (customInstructions) {
    return `${basePrompt}\n\n# Custom Instructions\n\n${customInstructions}`;
  }

  return basePrompt;
}

// 메시지 히스토리 시작 시 추가
this.conversationHistory = [
  {
    role: 'system',
    content: this.getSystemPrompt()
  }
];
```

#### 취소 처리

```typescript
abort(): void {
  if (this.abortController) {
    this.abortController.abort();
  }
}

// API 호출 시 AbortController 사용
const response = await this.client.chat.completions.create({
  model: this.config.model,
  messages: this.conversationHistory,
  tools: this.tools,
  signal: this.abortController?.signal
});
```

---

## 3. Grok API Client

### src/grok/client.ts (154 lines)

OpenAI 호환 API 클라이언트 래퍼입니다.

#### 클라이언트 생성

```typescript
import OpenAI from 'openai';

export function createGrokClient(config: ClientConfig): OpenAI {
  return new OpenAI({
    apiKey: config.apiKey,
    baseURL: config.baseURL || 'https://api.x.ai/v1',
    defaultHeaders: {
      'User-Agent': 'grok-cli/0.0.33'
    }
  });
}
```

#### 채팅 완성 (논스트리밍)

```typescript
export async function createCompletion(
  client: OpenAI,
  options: CompletionOptions
): Promise<ChatCompletion> {
  return await client.chat.completions.create({
    model: options.model,
    messages: options.messages,
    tools: options.tools,
    max_tokens: options.maxTokens || 1536,
    temperature: options.temperature || 0.7,
    // Grok 전용 파라미터
    // @ts-ignore
    search: options.enableWebSearch ? { enabled: true } : undefined
  });
}
```

#### 채팅 완성 (스트리밍)

```typescript
export async function createCompletionStream(
  client: OpenAI,
  options: CompletionOptions
): Promise<AsyncIterable<ChatCompletionChunk>> {
  return await client.chat.completions.create({
    ...options,
    stream: true  // 스트리밍 활성화
  });
}
```

#### 다중 제공자 지원

```typescript
// X.AI Grok
const grokClient = createGrokClient({
  apiKey: 'xai-...',
  baseURL: 'https://api.x.ai/v1'
});

// OpenAI
const openaiClient = createGrokClient({
  apiKey: 'sk-...',
  baseURL: 'https://api.openai.com/v1'
});

// OpenRouter
const openrouterClient = createGrokClient({
  apiKey: 'sk-or-...',
  baseURL: 'https://openrouter.ai/api/v1'
});

// Groq
const groqClient = createGrokClient({
  apiKey: 'gsk_...',
  baseURL: 'https://api.groq.com/openai/v1'
});
```

---

## 4. Tools - 파일 편집

### src/tools/text-editor.ts (669 lines)

파일 편집의 핵심 로직을 구현합니다.

#### view_file - 파일 보기

```typescript
export async function viewFile(args: {
  path: string;
  start_line?: number;
  end_line?: number;
}): Promise<ToolResult> {
  const { path, start_line, end_line } = args;

  // 파일 존재 확인
  if (!fs.existsSync(path)) {
    return { output: `Error: File not found: ${path}` };
  }

  // 디렉토리인 경우
  if (fs.statSync(path).isDirectory()) {
    const entries = fs.readdirSync(path, { withFileTypes: true });
    const listing = entries.map(entry => {
      const type = entry.isDirectory() ? '[DIR]' : '[FILE]';
      return `${type} ${entry.name}`;
    }).join('\n');

    return { output: `Directory listing for ${path}:\n${listing}` };
  }

  // 파일 읽기
  const content = fs.readFileSync(path, 'utf-8');
  const lines = content.split('\n');

  // 라인 범위 적용
  const start = start_line || 1;
  const end = end_line || lines.length;
  const selectedLines = lines.slice(start - 1, end);

  // 라인 번호 추가
  const numberedLines = selectedLines.map((line, i) => {
    const lineNum = start + i;
    return `${lineNum}| ${line}`;
  }).join('\n');

  return {
    output: `File: ${path} (lines ${start}-${end})\n${numberedLines}`
  };
}
```

#### create_file - 파일 생성

```typescript
export async function createFile(args: {
  path: string;
  content: string;
}): Promise<ToolResult> {
  const { path, content } = args;

  // 파일이 이미 존재하는 경우
  if (fs.existsSync(path)) {
    return { output: `Error: File already exists: ${path}` };
  }

  // 디렉토리 생성
  const dir = dirname(path);
  if (!fs.existsSync(dir)) {
    fs.mkdirSync(dir, { recursive: true });
  }

  // 파일 쓰기
  fs.writeFileSync(path, content, 'utf-8');

  return { output: `Created file: ${path} (${content.length} chars)` };
}
```

#### str_replace_editor - 텍스트 교체

```typescript
export async function strReplaceEditor(args: {
  path: string;
  old_str: string;
  new_str: string;
}): Promise<ToolResult> {
  const { path, old_str, new_str } = args;

  // 파일 읽기
  const content = fs.readFileSync(path, 'utf-8');

  // Undo를 위해 백업
  if (!undoStack.has(path)) {
    undoStack.set(path, []);
  }
  undoStack.get(path)!.push(content);

  // 정확한 매치 시도
  let newContent = content.replace(old_str, new_str);

  // 매치 실패 시 퍼지 매칭
  if (newContent === content) {
    const fuzzyMatch = findFuzzyMatch(content, old_str);
    if (fuzzyMatch) {
      newContent = content.replace(fuzzyMatch, new_str);
    } else {
      return {
        output: `Error: Could not find exact match for old_str in ${path}`
      };
    }
  }

  // 파일 쓰기
  fs.writeFileSync(path, newContent, 'utf-8');

  // Diff 생성
  const diff = generateDiff(content, newContent);

  return {
    output: `File ${path} updated successfully.\n\n${diff}`
  };
}
```

#### 퍼지 매칭

```typescript
// 다중 라인 문자열의 공백 차이 무시
function findFuzzyMatch(content: string, target: string): string | null {
  const normalizeWhitespace = (str: string) =>
    str.replace(/\s+/g, ' ').trim();

  const normalizedTarget = normalizeWhitespace(target);
  const lines = content.split('\n');

  for (let i = 0; i < lines.length; i++) {
    for (let j = i + 1; j <= lines.length; j++) {
      const candidate = lines.slice(i, j).join('\n');
      if (normalizeWhitespace(candidate) === normalizedTarget) {
        return candidate;
      }
    }
  }

  return null;
}
```

#### Diff 생성

```typescript
function generateDiff(oldContent: string, newContent: string): string {
  const oldLines = oldContent.split('\n');
  const newLines = newContent.split('\n');

  // 간단한 diff 알고리즘
  const hunks: Hunk[] = [];
  let i = 0, j = 0;

  while (i < oldLines.length || j < newLines.length) {
    if (i < oldLines.length && j < newLines.length && oldLines[i] === newLines[j]) {
      // 동일한 라인
      i++;
      j++;
    } else {
      // 변경 감지
      const hunk: Hunk = {
        oldStart: i,
        oldCount: 0,
        newStart: j,
        newCount: 0,
        lines: []
      };

      // 삭제된 라인
      while (i < oldLines.length && !newLines.includes(oldLines[i])) {
        hunk.lines.push(`- ${oldLines[i]}`);
        hunk.oldCount++;
        i++;
      }

      // 추가된 라인
      while (j < newLines.length && !oldLines.includes(newLines[j])) {
        hunk.lines.push(`+ ${newLines[j]}`);
        hunk.newCount++;
        j++;
      }

      if (hunk.lines.length > 0) {
        hunks.push(hunk);
      }
    }
  }

  // 3줄 컨텍스트 추가
  const diffLines = hunks.map(hunk => {
    const context = [];

    // 앞 컨텍스트
    for (let i = Math.max(0, hunk.oldStart - 3); i < hunk.oldStart; i++) {
      context.push(`  ${oldLines[i]}`);
    }

    context.push(...hunk.lines);

    // 뒤 컨텍스트
    const endLine = hunk.oldStart + hunk.oldCount;
    for (let i = endLine; i < Math.min(oldLines.length, endLine + 3); i++) {
      context.push(`  ${oldLines[i]}`);
    }

    return `@@ -${hunk.oldStart},${hunk.oldCount} +${hunk.newStart},${hunk.newCount} @@\n` +
      context.join('\n');
  }).join('\n\n');

  // 요약
  const addedLines = hunks.reduce((sum, h) => sum + h.newCount, 0);
  const removedLines = hunks.reduce((sum, h) => sum + h.oldCount, 0);

  return `Summary: +${addedLines} -${removedLines}\n\n${diffLines}`;
}
```

#### undo_edit - 실행 취소

```typescript
const undoStack = new Map<string, string[]>();

export async function undoEdit(args: { path: string }): Promise<ToolResult> {
  const { path } = args;

  const stack = undoStack.get(path);
  if (!stack || stack.length === 0) {
    return { output: `No edits to undo for ${path}` };
  }

  const previousContent = stack.pop()!;
  fs.writeFileSync(path, previousContent, 'utf-8');

  return { output: `Undid last edit to ${path}` };
}
```

---

## 5. Tools - 검색

### src/tools/search.ts (444 lines)

Ripgrep 기반 통합 검색을 구현합니다.

#### 통합 검색 (Cursor 스타일)

```typescript
export async function search(args: {
  query: string;
  path?: string;
  case_sensitive?: boolean;
  regex?: boolean;
  glob?: string;
  max_results?: number;
}): Promise<ToolResult> {
  const {
    query,
    path = process.cwd(),
    case_sensitive = false,
    regex = false,
    glob,
    max_results = 50
  } = args;

  // 텍스트 검색 + 파일명 검색 병렬 실행
  const [textResults, fileResults] = await Promise.all([
    searchTextContent(query, path, { case_sensitive, regex, glob }),
    searchFileNames(query, path, { glob })
  ]);

  // 결과 병합 및 정렬
  const allResults = [...textResults, ...fileResults]
    .sort((a, b) => b.score - a.score)
    .slice(0, max_results);

  // 출력 형식
  const output = allResults.map(result => {
    if (result.type === 'text') {
      return `${result.path}:${result.line}\n  ${result.content}`;
    } else {
      return `[FILE] ${result.path}`;
    }
  }).join('\n\n');

  return {
    output: `Found ${allResults.length} results:\n\n${output}`
  };
}
```

#### 텍스트 검색 (Ripgrep)

```typescript
import { spawn } from 'child_process';

async function searchTextContent(
  query: string,
  path: string,
  options: SearchOptions
): Promise<TextResult[]> {
  const args = [
    '--json',  // JSON 출력
    '--max-count', '50',
    '--max-depth', '10'
  ];

  // 대소문자 구분
  if (!options.case_sensitive) {
    args.push('--ignore-case');
  }

  // 정규식
  if (!options.regex) {
    args.push('--fixed-strings');
  }

  // Glob 패턴
  if (options.glob) {
    args.push('--glob', options.glob);
  }

  // 기본 제외 패턴
  args.push(
    '--glob', '!node_modules/',
    '--glob', '!.git/',
    '--glob', '!dist/',
    '--glob', '!build/'
  );

  args.push(query, path);

  return new Promise((resolve, reject) => {
    const rg = spawn('rg', args);
    let output = '';

    rg.stdout.on('data', (data) => {
      output += data.toString();
    });

    rg.on('close', (code) => {
      if (code === 1) {
        // 결과 없음 (정상)
        resolve([]);
        return;
      }

      if (code !== 0) {
        reject(new Error(`ripgrep exited with code ${code}`));
        return;
      }

      // JSON 파싱
      const results: TextResult[] = [];
      const lines = output.trim().split('\n');

      for (const line of lines) {
        try {
          const data = JSON.parse(line);

          if (data.type === 'match') {
            results.push({
              type: 'text',
              path: data.data.path.text,
              line: data.data.line_number,
              content: data.data.lines.text.trim(),
              score: 1.0
            });
          }
        } catch (e) {
          // JSON 파싱 실패 무시
        }
      }

      resolve(results);
    });
  });
}
```

#### 파일명 검색

```typescript
import { readdir } from 'fs/promises';
import { join } from 'path';

async function searchFileNames(
  query: string,
  basePath: string,
  options: { glob?: string }
): Promise<FileResult[]> {
  const results: FileResult[] = [];
  const maxDepth = 10;

  async function walk(dir: string, depth: number) {
    if (depth > maxDepth) return;

    const entries = await readdir(dir, { withFileTypes: true });

    for (const entry of entries) {
      // 제외 디렉토리
      if (entry.isDirectory()) {
        if (['node_modules', '.git', 'dist', 'build'].includes(entry.name)) {
          continue;
        }

        await walk(join(dir, entry.name), depth + 1);
      }

      // 파일명 매칭
      const fullPath = join(dir, entry.name);
      const score = fuzzyScore(entry.name, query);

      if (score > 0.3) {
        results.push({
          type: 'file',
          path: fullPath,
          score
        });
      }
    }
  }

  await walk(basePath, 0);
  return results;
}
```

#### 퍼지 스코어링

```typescript
function fuzzyScore(filename: string, query: string): number {
  const lowerFilename = filename.toLowerCase();
  const lowerQuery = query.toLowerCase();

  // 정확한 매치
  if (lowerFilename === lowerQuery) {
    return 1.0;
  }

  // 포함 여부
  if (lowerFilename.includes(lowerQuery)) {
    return 0.8;
  }

  // 연속 문자 매칭
  let queryIndex = 0;
  for (const char of lowerFilename) {
    if (char === lowerQuery[queryIndex]) {
      queryIndex++;
      if (queryIndex === lowerQuery.length) {
        return 0.6;
      }
    }
  }

  // 매치 실패
  return 0;
}
```

---

## 6. Tools - Bash

### src/tools/bash.ts (86 lines)

Bash 명령 실행을 담당합니다.

#### 기본 구현

```typescript
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

export async function executeBash(args: {
  command: string;
}): Promise<ToolResult> {
  const { command } = args;

  // cd 명령 특수 처리
  if (command.startsWith('cd ')) {
    const dir = command.slice(3).trim();
    try {
      process.chdir(dir);
      return {
        output: `Changed directory to ${process.cwd()}`
      };
    } catch (error) {
      return {
        output: `Error changing directory: ${error.message}`
      };
    }
  }

  // 일반 명령 실행
  try {
    const { stdout, stderr } = await execAsync(command, {
      cwd: process.cwd(),
      timeout: 30000,  // 30초 타임아웃
      maxBuffer: 1024 * 1024  // 1MB 버퍼
    });

    const output = stdout || stderr || 'Command completed';
    return { output };
  } catch (error) {
    return {
      output: `Error: ${error.message}\nExit code: ${error.code}`
    };
  }
}
```

#### 안전성 검증

```typescript
// 사용자 확인 필요 (ConfirmationService에서 처리)
const dangerousCommands = [
  'rm -rf',
  'sudo',
  'dd',
  'mkfs',
  'format'
];

function isDangerous(command: string): boolean {
  return dangerousCommands.some(dangerous =>
    command.includes(dangerous)
  );
}

// Agent에서 실행 전 확인
if (isDangerous(args.command)) {
  const approved = await confirmationService.requestConfirmation({
    tool: 'bash',
    args,
    description: `Execute: ${args.command}`,
    warning: 'This command may be dangerous'
  });

  if (!approved) {
    return { output: 'User rejected the operation' };
  }
}
```

---

## 7. MCP 통합

### src/mcp/client.ts (170 lines)

Model Context Protocol 클라이언트를 관리합니다.

#### MCPManager 클래스

```typescript
import { EventEmitter } from 'events';
import { Client } from '@modelcontextprotocol/sdk';

export class MCPManager extends EventEmitter {
  private static instance: MCPManager;
  private servers: Map<string, MCPServer> = new Map();
  private tools: Map<string, MCPTool> = new Map();

  static getInstance(): MCPManager {
    if (!MCPManager.instance) {
      MCPManager.instance = new MCPManager();
    }
    return MCPManager.instance;
  }

  async initializeAll(): Promise<void> {
    const config = loadMCPConfig();

    for (const [name, serverConfig] of Object.entries(config.mcpServers)) {
      try {
        await this.initializeServer(name, serverConfig);
      } catch (error) {
        console.error(`Failed to initialize MCP server ${name}:`, error);
      }
    }
  }

  async initializeServer(name: string, config: MCPServerConfig): Promise<void> {
    // 전송 생성
    const transport = createTransport(config.transport);

    // 클라이언트 생성
    const client = new Client({
      name: 'grok-cli',
      version: '0.0.33'
    }, {
      capabilities: {
        tools: {}
      }
    });

    // 연결
    await client.connect(transport);

    // 서버 등록
    this.servers.set(name, {
      name,
      client,
      transport,
      config
    });

    // 도구 로드
    const tools = await client.listTools();
    for (const tool of tools.tools) {
      // mcp__<server>__<tool> 형식으로 등록
      const fullName = `mcp__${name}__${tool.name}`;
      this.tools.set(fullName, {
        serverName: name,
        originalName: tool.name,
        schema: tool.inputSchema,
        description: tool.description
      });
    }

    this.emit('server-connected', name);
  }

  async executeTool(
    serverName: string,
    toolName: string,
    args: any
  ): Promise<ToolResult> {
    const server = this.servers.get(serverName);
    if (!server) {
      throw new Error(`Server not found: ${serverName}`);
    }

    try {
      const result = await server.client.callTool({
        name: toolName,
        arguments: args
      });

      return {
        output: JSON.stringify(result.content)
      };
    } catch (error) {
      return {
        output: `Error calling tool: ${error.message}`
      };
    }
  }

  getTools(): MCPTool[] {
    return Array.from(this.tools.values());
  }

  async disconnect(serverName: string): Promise<void> {
    const server = this.servers.get(serverName);
    if (server) {
      await server.client.close();
      this.servers.delete(serverName);

      // 해당 서버의 도구 제거
      for (const [key, tool] of this.tools.entries()) {
        if (tool.serverName === serverName) {
          this.tools.delete(key);
        }
      }

      this.emit('server-disconnected', serverName);
    }
  }
}
```

### src/mcp/transports.ts

#### Stdio Transport

```typescript
import { spawn } from 'child_process';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';

export function createStdioTransport(config: {
  command: string;
  args: string[];
}): StdioClientTransport {
  const process = spawn(config.command, config.args);

  return new StdioClientTransport({
    reader: process.stdout,
    writer: process.stdin
  });
}
```

#### SSE Transport

```typescript
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js';

export function createSSETransport(config: {
  url: string;
}): SSEClientTransport {
  return new SSEClientTransport({
    url: config.url
  });
}
```

---

## 8. 설정 관리

### src/utils/settings-manager.ts (326 lines)

사용자 및 프로젝트 설정을 관리합니다.

#### SettingsManager 클래스

```typescript
import fs from 'fs-extra';
import os from 'os';
import { join } from 'path';

export class SettingsManager {
  private static instance: SettingsManager;
  private userSettingsPath: string;
  private projectSettingsPath: string;
  private userSettings?: UserSettings;
  private projectSettings?: ProjectSettings;

  private constructor() {
    this.userSettingsPath = join(os.homedir(), '.grok', 'user-settings.json');
    this.projectSettingsPath = join(process.cwd(), '.grok', 'settings.json');
  }

  static getInstance(): SettingsManager {
    if (!SettingsManager.instance) {
      SettingsManager.instance = new SettingsManager();
    }
    return SettingsManager.instance;
  }

  getUserSettings(): UserSettings {
    if (!this.userSettings) {
      this.userSettings = this.loadUserSettings();
    }
    return this.userSettings;
  }

  getProjectSettings(): ProjectSettings {
    if (!this.projectSettings) {
      this.projectSettings = this.loadProjectSettings();
    }
    return this.projectSettings;
  }

  private loadUserSettings(): UserSettings {
    if (!fs.existsSync(this.userSettingsPath)) {
      return this.createDefaultUserSettings();
    }

    try {
      const data = fs.readFileSync(this.userSettingsPath, 'utf-8');
      return JSON.parse(data);
    } catch (error) {
      console.error('Error loading user settings:', error);
      return this.createDefaultUserSettings();
    }
  }

  private loadProjectSettings(): ProjectSettings {
    if (!fs.existsSync(this.projectSettingsPath)) {
      return {};
    }

    try {
      const data = fs.readFileSync(this.projectSettingsPath, 'utf-8');
      return JSON.parse(data);
    } catch (error) {
      console.error('Error loading project settings:', error);
      return {};
    }
  }

  private createDefaultUserSettings(): UserSettings {
    const defaults: UserSettings = {
      models: [
        'grok-code-fast-1',
        'grok-4-latest',
        'grok-3-latest'
      ]
    };

    fs.ensureDirSync(join(os.homedir(), '.grok'));
    fs.writeFileSync(
      this.userSettingsPath,
      JSON.stringify(defaults, null, 2),
      'utf-8'
    );

    // 권한 설정 (소유자만 읽기/쓰기)
    fs.chmodSync(this.userSettingsPath, 0o600);

    return defaults;
  }

  saveUserSettings(settings: UserSettings): void {
    fs.ensureDirSync(join(os.homedir(), '.grok'));
    fs.writeFileSync(
      this.userSettingsPath,
      JSON.stringify(settings, null, 2),
      'utf-8'
    );
    fs.chmodSync(this.userSettingsPath, 0o600);
    this.userSettings = settings;
  }

  saveProjectSettings(settings: ProjectSettings): void {
    fs.ensureDirSync(join(process.cwd(), '.grok'));
    fs.writeFileSync(
      this.projectSettingsPath,
      JSON.stringify(settings, null, 2),
      'utf-8'
    );
    this.projectSettings = settings;
  }

  getCurrentModel(): string {
    // 우선순위: 프로젝트 > 사용자 기본값 > 시스템 기본값
    return this.projectSettings?.model ||
      this.userSettings?.defaultModel ||
      'grok-code-fast-1';
  }

  setCurrentModel(model: string): void {
    const projectSettings = this.getProjectSettings();
    projectSettings.model = model;
    this.saveProjectSettings(projectSettings);
  }

  getApiKey(): string | undefined {
    // 환경 변수 우선
    return process.env.GROK_API_KEY || this.userSettings?.apiKey;
  }

  getBaseURL(): string {
    return process.env.GROK_BASE_URL ||
      this.userSettings?.baseURL ||
      'https://api.x.ai/v1';
  }
}
```

---

## 9. UI 컴포넌트

### src/ui/components/chat-interface.tsx (416 lines)

메인 UI 컴포넌트입니다.

#### 기본 구조

```typescript
import React, { useState, useEffect } from 'react';
import { Box, Text } from 'ink';
import ChatHistory from './chat-history.js';
import ChatInput from './chat-input.js';
import ConfirmationDialog from './confirmation-dialog.js';
import LoadingSpinner from './loading-spinner.js';
import MCPStatus from './mcp-status.js';
import { useInputHandler } from '../../hooks/use-input-handler.js';

interface Props {
  config: AgentConfig;
  initialMessage?: string;
}

export default function ChatInterface({ config, initialMessage }: Props) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [tokenCount, setTokenCount] = useState(0);
  const [confirmationData, setConfirmationData] = useState(null);
  const [autoEditMode, setAutoEditMode] = useState(false);

  const { sendMessage, abort } = useInputHandler({
    config,
    onMessageUpdate: (message) => {
      setMessages(prev => [...prev, message]);
    },
    onStreamChunk: (chunk) => {
      // 실시간 스트리밍 업데이트
      setMessages(prev => {
        const last = prev[prev.length - 1];
        if (last && last.role === 'assistant') {
          return [
            ...prev.slice(0, -1),
            { ...last, content: last.content + chunk }
          ];
        }
        return [...prev, { role: 'assistant', content: chunk }];
      });
    },
    onTokenCount: (count) => {
      setTokenCount(count);
    },
    onLoadingChange: (loading) => {
      setIsLoading(loading);
    }
  });

  // 확인 서비스 구독
  useEffect(() => {
    const confirmationService = ConfirmationService.getInstance();

    confirmationService.on('confirmation-needed', (data, callback) => {
      setConfirmationData({ data, callback });
    });

    return () => {
      confirmationService.removeAllListeners('confirmation-needed');
    };
  }, []);

  // 초기 메시지
  useEffect(() => {
    if (initialMessage) {
      sendMessage(initialMessage);
    }
  }, []);

  return (
    <Box flexDirection="column" height="100%">
      {/* 헤더 */}
      <Box borderStyle="round" borderColor="cyan">
        <Box flexDirection="column" width="100%" padding={1}>
          <Text bold color="cyan">Grok CLI v0.0.33</Text>
          <Text dimColor>Model: {config.model}</Text>
          <MCPStatus />
        </Box>
      </Box>

      {/* 메시지 히스토리 */}
      <Box flexGrow={1} flexDirection="column" overflowY="auto">
        <ChatHistory messages={messages} />
      </Box>

      {/* 로딩 인디케이터 */}
      {isLoading && (
        <LoadingSpinner tokenCount={tokenCount} />
      )}

      {/* 확인 대화상자 */}
      {confirmationData && (
        <ConfirmationDialog
          data={confirmationData.data}
          onApprove={() => {
            confirmationData.callback(true);
            setConfirmationData(null);
          }}
          onReject={() => {
            confirmationData.callback(false);
            setConfirmationData(null);
          }}
        />
      )}

      {/* 입력 필드 */}
      <ChatInput
        onSubmit={sendMessage}
        onCancel={abort}
        autoEditMode={autoEditMode}
        onToggleAutoEdit={() => setAutoEditMode(!autoEditMode)}
      />
    </Box>
  );
}
```

---

## 10. 확인 서비스

### src/utils/confirmation-service.ts

사용자 확인을 관리하는 서비스입니다.

```typescript
import { EventEmitter } from 'events';

export class ConfirmationService extends EventEmitter {
  private static instance: ConfirmationService;
  private dontAskAgainFlags: Set<string> = new Set();

  static getInstance(): ConfirmationService {
    if (!ConfirmationService.instance) {
      ConfirmationService.instance = new ConfirmationService();
    }
    return ConfirmationService.instance;
  }

  async requestConfirmation(data: ConfirmationData): Promise<boolean> {
    // "다시 묻지 않기" 플래그 확인
    const flagKey = `${data.tool}`;
    if (this.dontAskAgainFlags.has(flagKey)) {
      return true;
    }

    return new Promise((resolve) => {
      this.emit('confirmation-needed', data, (approved: boolean, dontAskAgain: boolean) => {
        if (approved && dontAskAgain) {
          this.dontAskAgainFlags.add(flagKey);
        }
        resolve(approved);
      });
    });
  }

  clearDontAskAgainFlags(): void {
    this.dontAskAgainFlags.clear();
  }
}
```

---

이 문서는 주요 컴포넌트들의 핵심 로직을 다룹니다. 더 자세한 구현은 해당 소스 파일을 참조하세요.
