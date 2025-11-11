# Grok CLI - MCP 통합 상세 분석

## 목차

1. [개요](#개요)
2. [MCP 프로토콜 이해](#mcp-프로토콜-이해)
3. [전송 타입](#전송-타입)
4. [MCPManager 클래스](#mcpmanager-클래스)
5. [서버 관리](#서버-관리)
6. [도구 통합](#도구-통합)
7. [설정 및 초기화](#설정-및-초기화)
8. [실제 예제](#실제-예제)

## 개요

Model Context Protocol (MCP)은 LLM과 외부 서비스를 연결하는 표준 프로토콜입니다. Grok CLI는 MCP를 통해 Linear, GitHub 등의 서비스와 통합할 수 있습니다.

**파일 위치**: `src/mcp/`

**주요 파일**:
- `client.ts`: MCPManager 클래스
- `transports.ts`: 전송 계층 (stdio, HTTP, SSE)
- `config.ts`: 설정 로드/저장

**핵심 기능**:
- 다중 MCP 서버 관리
- 다양한 전송 방식 지원
- 도구 자동 추가
- 동적 서버 추가/제거

## MCP 프로토콜 이해

### MCP 아키텍처

```
┌──────────────┐                    ┌──────────────┐
│ Grok CLI     │  ←────MCP────→    │ MCP Server   │
│ (Client)     │  ←────────────→    │ (Linear,     │
│              │  ←────────────→    │  GitHub...)  │
└──────────────┘                    └──────────────┘

통신 방식:
- 요청/응답 기반
- 서버가 제공하는 도구 자동 탐색
- 도구 호출 및 결과 전송
```

### MCP 메시지 형식

```json
// 도구 목록 요청
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}

// 도구 목록 응답
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "create_issue",
        "description": "Create a new issue",
        "inputSchema": {
          "type": "object",
          "properties": {
            "title": { "type": "string" },
            "body": { "type": "string" }
          },
          "required": ["title"]
        }
      }
    ]
  }
}

// 도구 호출
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "create_issue",
    "arguments": {
      "title": "New issue",
      "body": "Issue details"
    }
  }
}

// 도구 결과
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Issue created: #123"
      }
    ]
  }
}
```

### 프로토콜 명령

| 메서드 | 설명 | 사용처 |
|--------|------|--------|
| tools/list | 사용 가능한 도구 목록 | 서버 초기화 |
| tools/call | 도구 호출 | 도구 실행 |
| resources/list | 리소스 목록 (선택) | 특정 서버 |
| resources/read | 리소스 읽기 (선택) | 특정 서버 |

## 전송 타입

### 1. Stdio 전송

**특징**:
- 로컬 프로세스와의 통신
- 자식 프로세스로 실행
- 가장 일반적인 방식

**설정**:
```typescript
{
  type: "stdio",
  command: "npx",
  args: ["@github/mcp-server"],
  env?: {
    "GITHUB_TOKEN": "..."
  }
}
```

**구현**:
```typescript
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const transport = new StdioClientTransport({
  command: "npx",
  args: ["@github/mcp-server"],
  env: {
    ...process.env,
    GITHUB_TOKEN: "..."
  }
});
```

### 2. HTTP 전송

**특징**:
- 원격 서버와의 통신
- RESTful API 기반
- CORS 지원

**설정**:
```typescript
{
  type: "http",
  url: "http://localhost:3000"
}
```

**구현**:
```typescript
import { HTTPClientTransport } from "@modelcontextprotocol/sdk/client/http.js";

const transport = new HTTPClientTransport({
  url: "http://localhost:3000"
});
```

### 3. SSE 전송

**특징**:
- Server-Sent Events 기반
- 단방향 스트리밍 + 양방향 요청
- 웹 기반 서버에 적합

**설정**:
```typescript
{
  type: "sse",
  url: "https://mcp.linear.app/sse"
}
```

**구현**:
```typescript
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js";

const transport = new SSEClientTransport({
  url: "https://mcp.linear.app/sse"
});
```

### TransportConfig 타입

```typescript
export type TransportConfig =
  | {
      type: "stdio";
      command: string;
      args?: string[];
      env?: Record<string, string>;
    }
  | {
      type: "http";
      url: string;
    }
  | {
      type: "sse";
      url: string;
    };
```

## MCPManager 클래스

### 클래스 구조

```typescript
export class MCPManager extends EventEmitter {
  // 상태
  private clients: Map<string, Client> = new Map();     // 서버별 클라이언트
  private transports: Map<string, MCPTransport> = new Map(); // 전송 계층
  private tools: Map<string, MCPTool> = new Map();      // 도구 캐시

  // 메서드
  async addServer(config: MCPServerConfig): Promise<void>
  async removeServer(serverName: string): Promise<void>
  getTool(toolName: string): MCPTool | undefined
  getTools(): MCPTool[]
  async callTool(toolName: string, args: any): Promise<CallToolResult>
}
```

### addServer - 서버 추가

```typescript
async addServer(config: MCPServerConfig): Promise<void> {
  try {
    // 1. 전송 계층 생성
    let transportConfig = config.transport;
    if (!transportConfig && config.command) {
      // 레거시 지원
      transportConfig = {
        type: 'stdio',
        command: config.command,
        args: config.args,
        env: config.env
      };
    }

    if (!transportConfig) {
      throw new Error('Transport configuration is required');
    }

    // 2. 전송 객체 생성
    const transport = createTransport(transportConfig);
    this.transports.set(config.name, transport);

    // 3. MCP 클라이언트 생성
    const client = new Client(
      {
        name: "grok-cli",
        version: "1.0.0"
      },
      {
        capabilities: {
          tools: {}
        }
      }
    );

    this.clients.set(config.name, client);

    // 4. 연결
    const sdkTransport = await transport.connect();
    await client.connect(sdkTransport);

    // 5. 도구 목록 로드
    const toolsResult = await client.listTools();

    // 6. 도구 등록
    for (const tool of toolsResult.tools) {
      const mcpTool: MCPTool = {
        name: `mcp__${config.name}__${tool.name}`,
        description: tool.description || `Tool from ${config.name}`,
        inputSchema: tool.inputSchema,
        serverName: config.name
      };
      this.tools.set(mcpTool.name, mcpTool);
    }

    // 7. 이벤트 발생
    this.emit('serverAdded', config.name, toolsResult.tools.length);
  } catch (error) {
    this.emit('serverError', config.name, error);
    throw error;
  }
}
```

### removeServer - 서버 제거

```typescript
async removeServer(serverName: string): Promise<void> {
  // 1. 도구 제거
  for (const [toolName, tool] of this.tools.entries()) {
    if (tool.serverName === serverName) {
      this.tools.delete(toolName);
    }
  }

  // 2. 클라이언트 종료
  const client = this.clients.get(serverName);
  if (client) {
    await client.close();
    this.clients.delete(serverName);
  }

  // 3. 전송 종료
  const transport = this.transports.get(serverName);
  if (transport) {
    await transport.disconnect();
    this.transports.delete(serverName);
  }

  this.emit('serverRemoved', serverName);
}
```

### callTool - 도구 호출

```typescript
async callTool(
  toolName: string,
  args: any
): Promise<CallToolResult> {
  // 1. 도구 찾기
  const tool = this.tools.get(toolName);
  if (!tool) {
    throw new Error(`Tool not found: ${toolName}`);
  }

  // 2. 서버 찾기
  const client = this.clients.get(tool.serverName);
  if (!client) {
    throw new Error(`Server not connected: ${tool.serverName}`);
  }

  // 3. 도구 호출
  const result = await client.callTool({
    name: tool.name,
    arguments: args
  });

  return result;
}
```

## 서버 관리

### 전송 팩토리

```typescript
// src/mcp/transports.ts
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { HTTPClientTransport } from "@modelcontextprotocol/sdk/client/http.js";
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js";

export function createTransport(config: TransportConfig): MCPTransport {
  switch (config.type) {
    case 'stdio':
      return new StdioTransport(config);
    case 'http':
      return new HTTPTransport(config);
    case 'sse':
      return new SSETransport(config);
    default:
      throw new Error(`Unknown transport type: ${(config as any).type}`);
  }
}
```

### MCPTransport 인터페이스

```typescript
interface MCPTransport {
  connect(): Promise<any>;          // SDK Transport 객체 반환
  disconnect(): Promise<void>;      // 연결 해제
}

// Stdio 구현
class StdioTransport implements MCPTransport {
  constructor(private config: StdioTransportConfig) {}

  async connect(): Promise<any> {
    return new StdioClientTransport({
      command: this.config.command,
      args: this.config.args,
      env: this.config.env ? { ...process.env, ...this.config.env } : undefined
    });
  }

  async disconnect(): Promise<void> {
    // 프로세스 종료
  }
}

// HTTP 구현
class HTTPTransport implements MCPTransport {
  constructor(private config: HTTPTransportConfig) {}

  async connect(): Promise<any> {
    return new HTTPClientTransport({
      url: this.config.url
    });
  }

  async disconnect(): Promise<void> {
    // HTTP 연결 종료
  }
}

// SSE 구현
class SSETransport implements MCPTransport {
  constructor(private config: SSETransportConfig) {}

  async connect(): Promise<any> {
    return new SSEClientTransport({
      url: this.config.url
    });
  }

  async disconnect(): Promise<void> {
    // SSE 연결 종료
  }
}
```

## 도구 통합

### MCPTool 타입

```typescript
export interface MCPTool {
  name: string;                  // mcp__{serverName}__{toolName}
  description: string;           // 도구 설명
  inputSchema: any;              // JSON Schema
  serverName: string;            // 소속 서버
}
```

### GrokTool로 변환

```typescript
function convertMCPToolToGrokTool(mcpTool: MCPTool): GrokTool {
  return {
    type: "function",
    function: {
      name: mcpTool.name,
      description: mcpTool.description,
      parameters: mcpTool.inputSchema
    }
  };
}
```

### 도구 목록에 추가

```typescript
async function addMCPToolsToGrokTools(
  tools: GrokTool[]
): Promise<GrokTool[]> {
  const manager = getMCPManager();
  const mcpTools = manager.getTools();

  // MCP 도구를 GrokTool로 변환
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

### 에이전트에서 호출

```typescript
private async executeMCPTool(
  toolCall: GrokToolCall
): Promise<ToolResult> {
  try {
    const toolName = toolCall.function.name;
    const args = JSON.parse(toolCall.function.arguments || "{}");

    // MCP 도구 호출
    const manager = getMCPManager();
    const result = await manager.callTool(toolName, args);

    // 결과를 ToolResult로 변환
    const content = result.content
      ?.map(c => {
        if (c.type === 'text') return c.text;
        // 기타 타입 처리
        return '';
      })
      .join('\n') || '';

    return {
      success: true,
      output: content
    };
  } catch (error: any) {
    return {
      success: false,
      error: `MCP tool error: ${error.message}`
    };
  }
}
```

## 설정 및 초기화

### loadMCPConfig - 설정 로드

```typescript
export function loadMCPConfig(): MCPConfig {
  const manager = getSettingsManager();
  const projectSettings = manager.loadProjectSettings();
  const servers = projectSettings.mcpServers
    ? Object.values(projectSettings.mcpServers)
    : [];
  return { servers };
}
```

### saveMCPConfig - 설정 저장

```typescript
export function saveMCPConfig(config: MCPConfig): void {
  const manager = getSettingsManager();
  const mcpServers: Record<string, MCPServerConfig> = {};

  for (const server of config.servers) {
    mcpServers[server.name] = server;
  }

  manager.updateProjectSetting('mcpServers', mcpServers);
}
```

### initializeMCPServers - 서버 초기화

```typescript
export async function initializeMCPServers(): Promise<void> {
  const config = loadMCPConfig();
  const manager = getMCPManager();

  for (const server of config.servers) {
    try {
      await manager.addServer(server);
      console.log(`MCP server initialized: ${server.name}`);
    } catch (error) {
      console.warn(`Failed to initialize MCP server ${server.name}:`, error);
    }
  }
}
```

### 에이전트 초기화

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

## 실제 예제

### 예 1: Linear 통합

#### 설정

```bash
# .grok/settings.json에 추가
{
  "mcpServers": {
    "linear": {
      "name": "linear",
      "transport": {
        "type": "sse",
        "url": "https://mcp.linear.app/sse"
      }
    }
  }
}
```

#### 사용

```
사용자: "Linear에서 최근 이슈 찾아줘"

→ AI가 mcp__linear__list_issues 도구 호출
→ MCPManager가 Linear 서버에 도구 호출 전송
→ Linear가 이슈 목록 반환
→ 사용자에게 표시
```

### 예 2: GitHub 통합

#### 설정

```bash
# .grok/settings.json
{
  "mcpServers": {
    "github": {
      "name": "github",
      "transport": {
        "type": "stdio",
        "command": "npx",
        "args": ["@github/mcp-server"],
        "env": {
          "GITHUB_TOKEN": "ghp_..."
        }
      }
    }
  }
}
```

#### 사용

```typescript
// 프로세스로 MCP 서버 실행
// ├─ parent: grok-cli
// └─ child: @github/mcp-server
//    ├─ stdin: JSON-RPC 요청
//    └─ stdout: JSON-RPC 응답
```

### 예 3: 커스텀 MCP 서버

#### 서버 생성

```typescript
// custom-mcp-server.ts
import http from 'http';

const server = http.createServer((req, res) => {
  if (req.url === '/api/tools/call') {
    // 도구 호출 처리
    const toolName = req.headers['x-tool-name'];
    const result = handleToolCall(toolName);
    res.writeHead(200);
    res.end(JSON.stringify(result));
  }
});

server.listen(3000);
```

#### 설정

```bash
# .grok/settings.json
{
  "mcpServers": {
    "custom": {
      "name": "custom",
      "transport": {
        "type": "http",
        "url": "http://localhost:3000"
      }
    }
  }
}
```

### 예 4: MCP 서버 추가 (CLI)

```bash
# 명령줄로 MCP 서버 추가
grok mcp add linear \
  --transport sse \
  --url "https://mcp.linear.app/sse"

grok mcp add github \
  --transport stdio \
  --command "npx" \
  --args "@github/mcp-server" \
  --env "GITHUB_TOKEN=ghp_..."

grok mcp add custom \
  --transport http \
  --url "http://localhost:3000"
```

### 예 5: MCP 도구 사용

```
사용자: "새 GitHub 이슈를 생성하고 Linear에 링크 추가해줘"

AI 응답:
1. 도구 호출: mcp__github__create_issue
   - title: "새로운 기능 개발"
   - body: "..."

2. 도구 호출: mcp__linear__create_issue
   - title: "새로운 기능 개발"
   - github_link: "https://github.com/..."

결과: GitHub 이슈 생성 + Linear 이슈 생성 + 연결됨
```

## 보안

### API 키 관리

```bash
# 환경 변수로 관리
export GITHUB_TOKEN="ghp_..."
export LINEAR_API_KEY="lin_..."

# 또는 .grok/settings.json (개발용만)
{
  "mcpServers": {
    "github": {
      "env": {
        "GITHUB_TOKEN": "ghp_..."
      }
    }
  }
}

# .gitignore에 추가
.grok/settings.json
```

### 권한 관리

```typescript
// 서버 접근 제어 (필요시)
if (!user.hasPermission('use_mcp_servers')) {
  throw new Error('Permission denied');
}
```

## 에러 처리

### 서버 연결 실패

```typescript
try {
  await manager.addServer(config);
} catch (error) {
  console.error(`Failed to connect to ${config.name}:`, error);
  // 사용자에게 알림
}
```

### 도구 호출 실패

```typescript
try {
  const result = await manager.callTool(toolName, args);
} catch (error) {
  return {
    success: false,
    error: `Tool execution failed: ${error.message}`
  };
}
```

## 결론

MCP 통합의 주요 특징:

1. **표준 프로토콜**: Model Context Protocol 준수
2. **다양한 전송 방식**: Stdio, HTTP, SSE 지원
3. **자동 도구 탐색**: 서버에서 제공하는 도구 자동 인식
4. **동적 통합**: 런타임에 서버 추가/제거
5. **안전한 실행**: 에러 처리 및 권한 관리
