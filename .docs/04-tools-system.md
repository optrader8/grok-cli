# Grok CLI - 도구 시스템 상세 분석

## 목차

1. [개요](#개요)
2. [TextEditorTool](#texteditortool)
3. [BashTool](#bashtool)
4. [SearchTool](#searchtool)
5. [TodoTool](#todotool)
6. [MorphEditorTool](#morpheditortool)
7. [ConfirmationTool](#confirmationtool)
8. [도구 관리 시스템](#도구-관리-시스템)

## 개요

도구(Tool) 시스템은 AI가 실제 작업을 수행할 수 있게 해주는 핵심 인터페이스입니다. 각 도구는 `ToolResult` 인터페이스를 구현하여 일관된 응답을 제공합니다.

**파일 위치**: `src/tools/`

**기본 인터페이스**:
```typescript
interface ToolResult {
  success: boolean;      // 작업 성공 여부
  output?: string;       // 성공 시 결과
  error?: string;        // 실패 시 에러 메시지
}
```

## TextEditorTool

텍스트 파일 읽기, 생성, 편집을 담당합니다.

**파일 위치**: `src/tools/text-editor.ts`

### 1. view() - 파일 읽기

```typescript
async view(
  filePath: string,
  viewRange?: [number, number]
): Promise<ToolResult>
```

**기능**:
- 파일 내용 조회
- 디렉토리 목록 표시
- 행 범위 조회
- 큰 파일 자동 자르기

**사용 예**:
```typescript
// 전체 파일 읽기
const result1 = await textEditor.view("/path/to/file.txt");

// 특정 행 범위 읽기
const result2 = await textEditor.view("/path/to/file.ts", [10, 20]);

// 디렉토리 목록
const result3 = await textEditor.view("/path/to/directory");
```

**반환 예**:
```typescript
// 파일 읽기 성공
{
  success: true,
  output: "Contents of /path/to/file.txt:\n1: line 1\n2: line 2\n... +98 lines"
}

// 디렉토리 읽기
{
  success: true,
  output: "Directory contents of /path:\nfile1.ts\nfile2.js\nsubfolder"
}

// 실패
{
  success: false,
  error: "File or directory not found: /path/to/file.txt"
}
```

**구현 상세**:
```typescript
async view(
  filePath: string,
  viewRange?: [number, number]
): Promise<ToolResult> {
  try {
    const resolvedPath = path.resolve(filePath);

    if (await fs.pathExists(resolvedPath)) {
      const stats = await fs.stat(resolvedPath);

      // 디렉토리인 경우
      if (stats.isDirectory()) {
        const files = await fs.readdir(resolvedPath);
        return {
          success: true,
          output: `Directory contents of ${filePath}:\n${files.join("\n")}`,
        };
      }

      // 파일인 경우
      const content = await fs.readFile(resolvedPath, "utf-8");
      const lines = content.split("\n");

      // 행 범위 조회
      if (viewRange) {
        const [start, end] = viewRange;
        const selectedLines = lines.slice(start - 1, end);
        const numberedLines = selectedLines
          .map((line, idx) => `${start + idx}: ${line}`)
          .join("\n");

        return {
          success: true,
          output: `Lines ${start}-${end} of ${filePath}:\n${numberedLines}`,
        };
      }

      // 자동 자르기 (처음 10줄 표시)
      const totalLines = lines.length;
      const displayLines = totalLines > 10 ? lines.slice(0, 10) : lines;
      const numberedLines = displayLines
        .map((line, idx) => `${idx + 1}: ${line}`)
        .join("\n");
      const additionalLinesMessage =
        totalLines > 10 ? `\n... +${totalLines - 10} lines` : "";

      return {
        success: true,
        output: `Contents of ${filePath}:\n${numberedLines}${additionalLinesMessage}`,
      };
    } else {
      return {
        success: false,
        error: `File or directory not found: ${filePath}`,
      };
    }
  } catch (error: any) {
    return {
      success: false,
      error: `Error viewing ${filePath}: ${error.message}`,
    };
  }
}
```

### 2. create() - 파일 생성

```typescript
async create(
  filePath: string,
  content: string
): Promise<ToolResult>
```

**기능**:
- 새 파일 생성
- 사용자 확인 요청
- Diff 생성 및 표시
- 자동 디렉토리 생성

**사용 예**:
```typescript
const result = await textEditor.create(
  "/path/to/new-file.ts",
  `export function hello() {
  console.log("Hello, World!");
}`
);
```

**확인 시스템**:
```typescript
// 첫 생성 작업: 확인 필요
const result = await textEditor.create("file1.ts", "content");
// → 사용자 확인 대기

// 사용자가 "모두 승인" 클릭
// → 이후 작업은 자동 승인

const result2 = await textEditor.create("file2.ts", "content");
// → 확인 없이 바로 생성
```

### 3. strReplace() - 텍스트 치환

```typescript
async strReplace(
  filePath: string,
  oldStr: string,
  newStr: string,
  replaceAll: boolean = false
): Promise<ToolResult>
```

**기능**:
- 기존 파일 편집
- 문자열 검색 및 치환
- Fuzzy 매칭 (다중 줄 편집)
- 전체 치환 옵션

**사용 예**:
```typescript
// 단일 치환
const result = await textEditor.strReplace(
  "/path/to/file.ts",
  "const greeting = 'Hello';",
  "const greeting = 'Hi';"
);

// 다중 줄 치환
const result2 = await textEditor.strReplace(
  "/path/to/file.ts",
  `function old() {
  return 42;
}`,
  `function new() {
  return 43;
}`
);

// 전체 치환
const result3 = await textEditor.strReplace(
  "/path/to/file.ts",
  "import",
  "require",
  true // replaceAll
);
```

**Fuzzy 매칭**:
```typescript
// 정확한 문자열이 없을 때 유사한 문자열 찾기
const fuzzyResult = this.findFuzzyMatch(content, oldStr);
// → "import React from 'react';" 대신 "import React" 찾음
```

### 4. Diff 생성

```typescript
private generateDiff(
  filePath: string,
  oldContent: string,
  newContent: string
): string
```

**구조**:
```
--- /path/to/file.ts
+++ /path/to/file.ts
@@ -10,5 +10,5 @@
 function hello() {
   console.log("Hello");
-  return 42;
+  return 43;
 }
```

## BashTool

Shell 명령 실행을 담당합니다.

**파일 위치**: `src/tools/bash.ts`

### execute() - 명령 실행

```typescript
async execute(
  command: string,
  timeout: number = 30000
): Promise<ToolResult>
```

**기능**:
- Shell 명령 실행
- 디렉토리 변경
- 타임아웃 관리
- 표준 출력/에러 캡처

**사용 예**:
```typescript
// 파일 목록 조회
const result1 = await bash.execute("ls -la");

// 파일 검색
const result2 = await bash.execute("find . -name '*.ts'");

// 명령 조합
const result3 = await bash.execute("cat file.txt | grep 'pattern'");

// 디렉토리 변경
const result4 = await bash.execute("cd /home/user");
```

**특수 처리**:
```typescript
// cd 명령은 특별히 처리
if (command.startsWith('cd ')) {
  const newDir = command.substring(3).trim();
  process.chdir(newDir);
  this.currentDirectory = process.cwd();
  return {
    success: true,
    output: `Changed directory to: ${this.currentDirectory}`
  };
}
```

**확인 시스템**:
```typescript
// Bash 명령도 확인이 필요
const confirmationResult = await this.confirmationService
  .requestConfirmation({
    operation: 'Run bash command',
    filename: command,
    content: `Command: ${command}\nWorking directory: ${currentDirectory}`
  });

if (!confirmationResult.confirmed) {
  return {
    success: false,
    error: 'Command execution cancelled by user'
  };
}
```

### 유틸리티 메서드

```typescript
// 현재 디렉토리 조회
getCurrentDirectory(): string

// 파일 목록
async listFiles(directory?: string): Promise<ToolResult>

// 파일 검색
async findFiles(pattern: string, directory?: string): Promise<ToolResult>
```

## SearchTool

텍스트 검색과 파일 검색을 통합합니다.

**파일 위치**: `src/tools/search.ts`

### search() - 통합 검색

```typescript
async search(
  query: string,
  options?: {
    searchType?: "text" | "files" | "both";
    includePattern?: string;
    excludePattern?: string;
    caseSensitive?: boolean;
    wholeWord?: boolean;
    regex?: boolean;
    maxResults?: number;
    fileTypes?: string[];
    excludeFiles?: string[];
    includeHidden?: boolean;
  }
): Promise<ToolResult>
```

**기능**:
- 텍스트 검색 (ripgrep 사용)
- 파일 검색 (파일명 기반)
- 통합 검색 결과
- 고급 필터링 옵션

**사용 예**:
```typescript
// 텍스트 검색
const result1 = await search.search(
  "import.*react",
  { searchType: "text", regex: true }
);

// 파일 검색
const result2 = await search.search(
  "*.tsx",
  { searchType: "files" }
);

// 통합 검색
const result3 = await search.search(
  "component",
  { searchType: "both" }
);

// 고급 검색
const result4 = await search.search(
  "function.*callback",
  {
    searchType: "text",
    regex: true,
    caseSensitive: true,
    excludePattern: "node_modules",
    fileTypes: ["ts", "tsx"]
  }
);
```

### Ripgrep 검색

```typescript
private async executeRipgrep(
  query: string,
  options: SearchOptions
): Promise<SearchResult[]>
```

**결과 구조**:
```typescript
interface SearchResult {
  file: string;      // 파일 경로
  line: number;      // 줄 번호
  column: number;    // 열 번호
  text: string;      // 전체 줄 내용
  match: string;     // 매칭된 부분
}
```

**Ripgrep 명령**:
```bash
# 기본 검색
rg "pattern" --json

# 정규식
rg "import.*react" --json -P

# 파일 타입 필터
rg "pattern" -t ts -t tsx --json

# 제외 패턴
rg "pattern" -g "!node_modules" --json
```

### 파일 검색

```typescript
private async findFilesByPattern(
  query: string,
  options: SearchOptions
): Promise<FileSearchResult[]>
```

**알고리즘**: 파일명에서 쿼리와의 유사도 계산
```typescript
interface FileSearchResult {
  path: string;    // 파일 경로
  name: string;    // 파일명
  score: number;   // 유사도 점수 (0-100)
}
```

## TodoTool

작업 관리 및 추적을 담당합니다.

**파일 위치**: `src/tools/todo-tool.ts`

### TodoItem 구조

```typescript
interface TodoItem {
  id: string;                           // 고유 ID
  content: string;                      // 작업 내용
  status: 'pending' | 'in_progress' | 'completed';  // 상태
  priority: 'high' | 'medium' | 'low'; // 우선순위
}
```

### createTodoList() - TODO 리스트 생성

```typescript
async createTodoList(todos: TodoItem[]): Promise<ToolResult>
```

**사용 예**:
```typescript
const result = await todoTool.createTodoList([
  {
    id: "task-1",
    content: "파일 편집",
    status: "in_progress",
    priority: "high"
  },
  {
    id: "task-2",
    content: "테스트 작성",
    status: "pending",
    priority: "medium"
  },
  {
    id: "task-3",
    content: "배포",
    status: "pending",
    priority: "low"
  }
]);
```

**출력 형식**:
```
◐ 파일 편집
  ○ 테스트 작성
  ○ 배포
```

**색상 코딩**:
- 완료(●): 초록색
- 진행중(◐): 청록색
- 보류(○): 기본색

### updateTodoList() - TODO 업데이트

```typescript
async updateTodoList(todos: TodoItem[]): Promise<ToolResult>
```

**사용 예**:
```typescript
// 작업 완료 표시
await todoTool.updateTodoList([
  {
    id: "task-1",
    content: "파일 편집",
    status: "completed",  // 변경됨
    priority: "high"
  }
]);
```

### formatTodoList() - 형식화

```typescript
formatTodoList(): string
```

**구현**:
```typescript
const getCheckbox = (status: string): string => {
  switch (status) {
    case 'completed':  return '●';      // 완료
    case 'in_progress': return '◐';     // 진행중
    case 'pending':    return '○';      // 보류
  }
};

const getStatusColor = (status: string): string => {
  switch (status) {
    case 'completed':  return '\x1b[32m';  // 초록
    case 'in_progress': return '\x1b[36m'; // 청록
    case 'pending':    return '\x1b[37m';  // 기본
  }
};
```

## MorphEditorTool

고속 코드 편집을 위한 도구입니다 (Morph API 사용).

**파일 위치**: `src/tools/morph-editor.ts`

**요구사항**: `MORPH_API_KEY` 환경 변수 필요

### editFile() - 고속 편집

```typescript
async editFile(
  filePath: string,
  edits: Edit[]
): Promise<ToolResult>
```

**Edit 구조**:
```typescript
interface Edit {
  type: "replace" | "delete" | "insert";
  range?: [number, number];           // 편집 범위
  text?: string;                      // 새 텍스트
  lineNumber?: number;                // 줄 번호
}
```

**특징**:
- 고속 처리: 초당 4,500+ 토큰
- 높은 정확도: 98% 이상
- 텍스트 에디터보다 빠름

**사용 예**:
```typescript
const result = await morphEditor.editFile(
  "src/app.ts",
  [
    {
      type: "replace",
      range: [10, 15],
      text: "const newCode = 'updated';"
    },
    {
      type: "insert",
      lineNumber: 20,
      text: "console.log('added');"
    }
  ]
);
```

## ConfirmationTool

사용자 확인 처리를 담당합니다.

**파일 위치**: `src/tools/confirmation-tool.ts`

### 확인 옵션

```typescript
interface ConfirmationOptions {
  operation: string;           // 작업 설명
  filename: string;            // 파일명 또는 명령
  content: string;             // 상세 내용 (Diff 등)
  showVSCodeOpen?: boolean;    // VSCode 열기 옵션 표시 여부
}
```

### 확인 흐름

```
1. 도구가 ConfirmationService.requestConfirmation() 호출
   │
2. ConfirmationDialog UI 표시
   ├─ 작업 설명 표시
   ├─ 파일명 또는 명령 표시
   ├─ Diff 또는 내용 표시
   │
3. 사용자 선택
   ├─ "승인": 작업 진행
   ├─ "거부": 작업 취소
   └─ "모두 승인": 이후 같은 타입 작업은 자동 승인
   │
4. ConfirmationResult 반환
```

### 세션 플래그

```typescript
// 사용자가 "모두 승인" 선택 시 저장
{
  fileOperations: true,    // 파일 작업 자동 승인
  bashCommands: true,      // Bash 자동 승인
  allOperations: true      // 모든 작업 자동 승인
}
```

## 도구 관리 시스템

### GROK_TOOLS 배열

```typescript
// src/grok/tools.ts
export const GROK_TOOLS: GrokTool[] = [
  VIEW_FILE_TOOL,
  CREATE_FILE_TOOL,
  STR_REPLACE_EDITOR_TOOL,
  EDIT_FILE_TOOL,
  BASH_TOOL,
  SEARCH_TOOL,
  CREATE_TODO_TOOL,
  UPDATE_TODO_TOOL
];
```

### 도구 정의 형식

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

### getAllGrokTools() - 도구 목록

```typescript
export async function getAllGrokTools(): Promise<GrokTool[]> {
  const tools: GrokTool[] = [...GROK_TOOLS];

  // MCP 도구 추가
  const mcpTools = await addMCPToolsToGrokTools(tools);

  return mcpTools;
}
```

### MCP 도구 통합

```typescript
async function addMCPToolsToGrokTools(
  tools: GrokTool[]
): Promise<GrokTool[]> {
  const manager = getMCPManager();
  const mcpTools = manager.getTools();

  // MCP 도구를 GrokTool 형식으로 변환
  const formattedMCPTools = mcpTools.map(mcpTool => ({
    type: "function",
    function: {
      name: mcpTool.name,
      description: mcpTool.description,
      parameters: mcpTool.inputSchema
    }
  }));

  return [...tools, ...formattedMCPTools];
}
```

## 도구 실행 흐름

```
┌──────────────────────────────────────┐
│ AI가 도구 호출 결정                   │
└──────────────────┬───────────────────┘
                   │
         ┌─────────▼──────────┐
         │ GrokToolCall 생성   │
         │ (함수명 + 인자)    │
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────────────┐
         │ executeTool() 호출          │
         │ (GrokAgent에서)            │
         └─────────┬──────────────────┘
                   │
         ┌─────────▼──────────────────┐
         │ 도구 선택                   │
         │ (switch문으로)             │
         └────┬─────────────┬────────┬┘
              │             │        │
        ┌─────▼──┐   ┌─────▼──┐   ┌─▼────────┐
        │TextEditor│   │Bash   │   │其他工具│
        └─────┬──┘   └─────┬──┘   └────┬────┘
              │             │         │
        ┌─────▼──────────────▼─────────▼┐
        │ 사용자 확인 (필요 시)          │
        └─────┬──────────────────────────┘
              │
        ┌─────▼──────────────────────────┐
        │ 도구 실행                       │
        │ (실제 작업 수행)               │
        └─────┬──────────────────────────┘
              │
        ┌─────▼──────────────────────────┐
        │ ToolResult 반환                │
        │ {success, output/error}       │
        └─────┬──────────────────────────┘
              │
        ┌─────▼──────────────────────────┐
        │ API에 도구 결과 전송            │
        │ (role: "tool" 메시지)          │
        └──────────────────────────────────┘
```

## 도구 추가 가이드

새로운 도구를 추가하려면:

### 1. 도구 클래스 구현

```typescript
// src/tools/my-tool.ts
export class MyTool {
  async execute(args: any): Promise<ToolResult> {
    try {
      // 구현
      return { success: true, output: "결과" };
    } catch (error: any) {
      return { success: false, error: error.message };
    }
  }
}
```

### 2. 도구 정의 추가

```typescript
// src/grok/tools.ts
const MY_TOOL: GrokTool = {
  type: "function",
  function: {
    name: "my_tool",
    description: "도구 설명",
    parameters: {
      type: "object",
      properties: {
        arg1: { type: "string" }
      },
      required: ["arg1"]
    }
  }
};

export const GROK_TOOLS = [
  // ...
  MY_TOOL
];
```

### 3. 에이전트에서 실행

```typescript
// src/agent/grok-agent.ts
private async executeTool(toolCall: GrokToolCall): Promise<ToolResult> {
  switch (toolCall.function.name) {
    case "my_tool":
      return await this.myTool.execute(args);
    // ...
  }
}
```

## 결론

도구 시스템의 주요 특징:

1. **일관성**: 모든 도구가 동일한 ToolResult 인터페이스 사용
2. **안전성**: 사용자 확인 시스템으로 위험한 작업 보호
3. **확장성**: 새로운 도구 추가가 간단함
4. **통합성**: MCP를 통한 외부 도구 통합 지원
5. **사용성**: 직관적인 API와 명확한 에러 메시지
