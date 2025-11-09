# API 통합

이 문서는 Grok CLI가 통합하는 외부 API 및 서비스에 대해 설명합니다.

## 📋 목차

1. [X.AI Grok API](#1-xai-grok-api)
2. [Morph API](#2-morph-api)
3. [Model Context Protocol (MCP)](#3-model-context-protocol-mcp)
4. [Ripgrep](#4-ripgrep)
5. [기타 OpenAI 호환 API](#5-기타-openai-호환-api)

---

## 1. X.AI Grok API

### 개요

X.AI의 Grok API는 OpenAI 호환 인터페이스를 제공하는 대화형 AI 서비스입니다.

**기본 정보**:
- **엔드포인트**: `https://api.x.ai/v1`
- **인증**: Bearer Token (API Key)
- **프로토콜**: HTTPS/REST
- **형식**: JSON

### 사용 가능한 모델

```typescript
const GROK_MODELS = [
  'grok-code-fast-1',      // 코딩 특화, 빠른 응답
  'grok-4-latest',         // 최신 Grok 4 모델
  'grok-3-latest',         // Grok 3 모델
  'grok-3-fast',           // Grok 3 빠른 버전
  'grok-vision-latest'     // 비전 기능 포함
];
```

### API 키 획득

1. X.AI 플랫폼 계정 생성: https://x.ai
2. 대시보드에서 API 키 생성
3. 환경 변수 또는 설정 파일에 저장

```bash
export GROK_API_KEY=xai-your-api-key-here
```

### 기본 사용법

```typescript
import OpenAI from 'openai';

const client = new OpenAI({
  apiKey: process.env.GROK_API_KEY,
  baseURL: 'https://api.x.ai/v1'
});

const response = await client.chat.completions.create({
  model: 'grok-code-fast-1',
  messages: [
    { role: 'user', content: 'Hello, Grok!' }
  ]
});

console.log(response.choices[0].message.content);
```

### 주요 기능

#### 1. Function Calling (도구 호출)

```typescript
const response = await client.chat.completions.create({
  model: 'grok-code-fast-1',
  messages: [
    { role: 'user', content: 'What files are in the src/ directory?' }
  ],
  tools: [
    {
      type: 'function',
      function: {
        name: 'view_file',
        description: 'View file or directory contents',
        parameters: {
          type: 'object',
          properties: {
            path: {
              type: 'string',
              description: 'File or directory path'
            }
          },
          required: ['path']
        }
      }
    }
  ]
});

// 도구 호출 확인
if (response.choices[0].finish_reason === 'tool_calls') {
  const toolCall = response.choices[0].message.tool_calls[0];
  console.log('Tool:', toolCall.function.name);
  console.log('Args:', JSON.parse(toolCall.function.arguments));
}
```

#### 2. Streaming (스트리밍)

```typescript
const stream = await client.chat.completions.create({
  model: 'grok-code-fast-1',
  messages: [{ role: 'user', content: 'Write a Python function' }],
  stream: true
});

for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content || '';
  process.stdout.write(content);
}
```

#### 3. Web Search (Grok 전용 기능)

```typescript
const response = await client.chat.completions.create({
  model: 'grok-code-fast-1',
  messages: [
    { role: 'user', content: 'What are the latest TypeScript features?' }
  ],
  // @ts-ignore - Grok 전용 파라미터
  search: {
    enabled: true
  }
});
```

### 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|--------|------|
| `model` | string | 필수 | 사용할 모델명 |
| `messages` | array | 필수 | 대화 메시지 배열 |
| `temperature` | number | 0.7 | 창의성 (0-2) |
| `max_tokens` | number | 1536 | 최대 생성 토큰 수 |
| `top_p` | number | 1.0 | 핵 샘플링 |
| `frequency_penalty` | number | 0 | 반복 억제 (-2 to 2) |
| `presence_penalty` | number | 0 | 주제 다양성 (-2 to 2) |
| `stream` | boolean | false | 스트리밍 활성화 |
| `tools` | array | - | 사용 가능한 도구 목록 |
| `search` | object | - | 웹 검색 설정 (Grok 전용) |

### 에러 처리

```typescript
try {
  const response = await client.chat.completions.create({
    model: 'grok-code-fast-1',
    messages: [{ role: 'user', content: 'Hello' }]
  });
} catch (error) {
  if (error.status === 401) {
    console.error('Invalid API key');
  } else if (error.status === 429) {
    console.error('Rate limit exceeded');
  } else if (error.status === 500) {
    console.error('Server error');
  } else {
    console.error('Unknown error:', error.message);
  }
}
```

### 요금 및 제한

- **토큰 제한**: 모델별로 다름 (일반적으로 128k 컨텍스트)
- **요청 제한**: API 키별 RPM/TPM 제한
- **요금**: 사용량 기반 (X.AI 가격 페이지 참조)

### Grok CLI에서의 활용

```typescript
// src/grok/client.ts
export function createGrokClient(config: ClientConfig): OpenAI {
  return new OpenAI({
    apiKey: config.apiKey,
    baseURL: config.baseURL || 'https://api.x.ai/v1',
    defaultHeaders: {
      'User-Agent': 'grok-cli/0.0.33'
    }
  });
}

// src/agent/grok-agent.ts
const response = await this.client.chat.completions.create({
  model: this.config.model,
  messages: this.conversationHistory,
  tools: this.tools,
  max_tokens: this.config.maxTokens || 1536,
  stream: true
});
```

---

## 2. Morph API

### 개요

Morph API는 고속 코드 편집을 위한 특화된 AI 서비스입니다.

**기본 정보**:
- **엔드포인트**: `https://api.morphllm.com/v1`
- **인증**: Bearer Token (API Key)
- **모델**: `morph-v3-large`
- **특징**: 4,500+ tokens/sec 편집 속도

### API 키 획득

1. Morph 웹사이트 방문
2. API 키 신청 및 획득
3. 환경 변수에 설정 (선택사항)

```bash
export MORPH_API_KEY=morph-your-api-key-here
```

### Fast Apply 편집 형식

Morph API는 특별한 편집 형식을 사용합니다:

```typescript
const prompt = `
<instruction>
Fix the null pointer error in the getUserData function
</instruction>

<code>
function getUserData(id) {
  const user = users.find(u => u.id === id);
  return user.name;  // Null pointer error if user not found
}
</code>

<update>
function getUserData(id) {
  const user = users.find(u => u.id === id);
  if (!user) {
    throw new Error(\`User not found: \${id}\`);
  }
  return user.name;
}
</update>
`;
```

### 사용법

```typescript
import OpenAI from 'openai';

const morphClient = new OpenAI({
  apiKey: process.env.MORPH_API_KEY,
  baseURL: 'https://api.morphllm.com/v1'
});

const response = await morphClient.chat.completions.create({
  model: 'morph-v3-large',
  messages: [
    {
      role: 'user',
      content: `
<instruction>Add input validation</instruction>
<code>${code}</code>
      `.trim()
    }
  ]
});

// <update> 태그에서 수정된 코드 추출
const updated = extractUpdateTag(response.choices[0].message.content);
```

### Grok CLI에서의 활용

```typescript
// src/tools/morph-editor.ts
export async function editFileWithMorph(args: {
  path: string;
  instructions: string;
}): Promise<ToolResult> {
  // Morph API 키 확인
  if (!process.env.MORPH_API_KEY) {
    // 일반 편집으로 폴백
    return await strReplaceEditor(args);
  }

  const code = fs.readFileSync(args.path, 'utf-8');

  const prompt = `
<instruction>
${args.instructions}
</instruction>

<code>
${code}
</code>
  `.trim();

  const response = await morphClient.chat.completions.create({
    model: 'morph-v3-large',
    messages: [{ role: 'user', content: prompt }]
  });

  const updated = extractUpdateTag(response.choices[0].message.content);

  // 파일 업데이트
  fs.writeFileSync(args.path, updated, 'utf-8');

  return {
    output: `Fast Apply edit completed for ${args.path}`
  };
}
```

### 장점 vs 일반 편집

| 항목 | Fast Apply (Morph) | 일반 편집 |
|------|-------------------|----------|
| 속도 | 4,500+ tokens/sec | ~100 tokens/sec |
| 정확도 | 높음 | 중간 |
| 형식 | 전체 파일 | 부분 교체 |
| 비용 | API 키 필요 | 무료 |
| 폴백 | 자동 | N/A |

---

## 3. Model Context Protocol (MCP)

### 개요

MCP는 AI 애플리케이션이 외부 도구 및 데이터 소스에 접근할 수 있도록 하는 개방형 프로토콜입니다.

**공식 사이트**: https://modelcontextprotocol.io

### 지원 전송 방식

#### 1. Stdio (Standard Input/Output)

로컬 프로세스로 MCP 서버를 실행합니다.

```bash
# 서버 추가
grok mcp add myserver -t stdio -c "bun" -a "server.js"

# 설정 예시
{
  "myserver": {
    "transport": {
      "type": "stdio",
      "command": "bun",
      "args": ["server.js"]
    }
  }
}
```

**구현**:
```typescript
import { spawn } from 'child_process';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';

const process = spawn('bun', ['server.js']);

const transport = new StdioClientTransport({
  reader: process.stdout,
  writer: process.stdin
});
```

#### 2. SSE (Server-Sent Events)

HTTP 스트리밍을 사용하는 원격 MCP 서버입니다.

```bash
# 서버 추가
grok mcp add linear -t sse -u "https://mcp.linear.app/sse"

# 설정 예시
{
  "linear": {
    "transport": {
      "type": "sse",
      "url": "https://mcp.linear.app/sse"
    }
  }
}
```

**구현**:
```typescript
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js';

const transport = new SSEClientTransport({
  url: 'https://mcp.linear.app/sse'
});
```

#### 3. HTTP

REST API 기반 MCP 서버입니다.

```bash
# 서버 추가
grok mcp add api -t http -u "https://api.example.com/mcp"
```

### MCP 서버 구현 예시

```typescript
// server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new Server(
  {
    name: 'my-mcp-server',
    version: '1.0.0'
  },
  {
    capabilities: {
      tools: {}
    }
  }
);

// 도구 등록
server.setRequestHandler('tools/list', async () => {
  return {
    tools: [
      {
        name: 'get_weather',
        description: 'Get weather for a location',
        inputSchema: {
          type: 'object',
          properties: {
            location: {
              type: 'string',
              description: 'City name'
            }
          },
          required: ['location']
        }
      }
    ]
  };
});

// 도구 실행
server.setRequestHandler('tools/call', async (request) => {
  if (request.params.name === 'get_weather') {
    const { location } = request.params.arguments;
    const weather = await fetchWeather(location);
    return {
      content: [
        {
          type: 'text',
          text: `Weather in ${location}: ${weather}`
        }
      ]
    };
  }
});

// 서버 시작
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 실행

```bash
# 서버 실행
bun run server.ts

# Grok CLI에 추가
grok mcp add weather -t stdio -c "bun" -a "run" -a "server.ts"

# 사용
grok "What's the weather in San Francisco?"
```

### 도구 명명 규칙

MCP 도구는 다음 형식으로 AI에 노출됩니다:

```
mcp__<server-name>__<tool-name>
```

예시:
- `mcp__linear__create_issue`
- `mcp__github__create_pr`
- `mcp__weather__get_forecast`

### 인기 MCP 서버

```bash
# Linear (이슈 관리)
grok mcp add linear -t sse -u "https://mcp.linear.app/sse"

# GitHub
grok mcp add github -t stdio -c "npx" -a "-y" -a "@modelcontextprotocol/server-github"

# Filesystem
grok mcp add fs -t stdio -c "npx" -a "-y" -a "@modelcontextprotocol/server-filesystem"

# Slack
grok mcp add slack -t stdio -c "npx" -a "-y" -a "@modelcontextprotocol/server-slack"
```

### Grok CLI의 MCP 통합

```typescript
// src/mcp/client.ts
export class MCPManager extends EventEmitter {
  async initializeServer(name: string, config: MCPServerConfig) {
    const transport = createTransport(config.transport);
    const client = new Client({ name: 'grok-cli' }, { capabilities: { tools: {} } });

    await client.connect(transport);

    // 도구 로드
    const tools = await client.listTools();
    for (const tool of tools.tools) {
      this.tools.set(`mcp__${name}__${tool.name}`, {
        serverName: name,
        originalName: tool.name,
        schema: tool.inputSchema
      });
    }
  }

  async executeTool(serverName: string, toolName: string, args: any) {
    const server = this.servers.get(serverName);
    const result = await server.client.callTool({
      name: toolName,
      arguments: args
    });
    return { output: JSON.stringify(result.content) };
  }
}
```

### MCP 관리 명령어

```bash
# 서버 추가
grok mcp add <name> -t <type> [options]

# 서버 목록
grok mcp list

# 서버 테스트
grok mcp test <name>

# 서버 제거
grok mcp remove <name>
```

---

## 4. Ripgrep

### 개요

Ripgrep(rg)은 초고속 텍스트 검색 도구로, Grok CLI의 검색 기능에 사용됩니다.

**공식 사이트**: https://github.com/BurntSushi/ripgrep

### 설치

```bash
# macOS
brew install ripgrep

# Ubuntu/Debian
sudo apt install ripgrep

# Fedora
sudo dnf install ripgrep

# Windows (Scoop)
scoop install ripgrep

# Windows (Chocolatey)
choco install ripgrep
```

### 기본 사용법

```bash
# 텍스트 검색
rg "pattern" /path/to/search

# JSON 출력 (Grok CLI가 사용)
rg --json "pattern" /path

# 대소문자 무시
rg -i "pattern" /path

# Glob 패턴
rg --glob "*.ts" "pattern" /path

# 제외 패턴
rg --glob "!node_modules/" "pattern" /path
```

### Grok CLI에서의 활용

```typescript
// src/tools/search.ts
import { spawn } from 'child_process';

async function searchTextContent(
  query: string,
  path: string,
  options: SearchOptions
): Promise<TextResult[]> {
  const args = [
    '--json',
    '--max-count', '50',
    '--max-depth', '10'
  ];

  if (!options.case_sensitive) {
    args.push('--ignore-case');
  }

  if (!options.regex) {
    args.push('--fixed-strings');
  }

  if (options.glob) {
    args.push('--glob', options.glob);
  }

  args.push(
    '--glob', '!node_modules/',
    '--glob', '!.git/',
    query,
    path
  );

  const rg = spawn('rg', args);

  // JSON 출력 파싱
  rg.stdout.on('data', (data) => {
    const lines = data.toString().split('\n');
    for (const line of lines) {
      const match = JSON.parse(line);
      if (match.type === 'match') {
        results.push({
          path: match.data.path.text,
          line: match.data.line_number,
          content: match.data.lines.text
        });
      }
    }
  });
}
```

### 성능 특성

- **속도**: grep보다 5-10배 빠름
- **대용량**: GB 단위 코드베이스도 빠르게 검색
- **스마트**: .gitignore 자동 인식
- **병렬**: 멀티코어 활용

### 제한사항

- 시스템에 설치 필요 (Node.js 패키지 아님)
- 바이너리 파일 검색 제한적

---

## 5. 기타 OpenAI 호환 API

Grok CLI는 OpenAI 호환 인터페이스를 제공하는 모든 서비스와 호환됩니다.

### OpenAI

```bash
export GROK_API_KEY=sk-your-openai-key
export GROK_BASE_URL=https://api.openai.com/v1
export GROK_MODEL=gpt-4-turbo

grok "Hello, GPT-4!"
```

### OpenRouter

다양한 모델을 하나의 API로 제공합니다.

```bash
export GROK_API_KEY=sk-or-your-key
export GROK_BASE_URL=https://openrouter.ai/api/v1
export GROK_MODEL=anthropic/claude-3-opus

grok "Hello, Claude!"
```

**사용 가능 모델**:
- `anthropic/claude-3-opus`
- `anthropic/claude-3-sonnet`
- `google/gemini-pro`
- `meta-llama/llama-3-70b-instruct`

### Groq

초고속 추론을 제공합니다.

```bash
export GROK_API_KEY=gsk-your-groq-key
export GROK_BASE_URL=https://api.groq.com/openai/v1
export GROK_MODEL=llama3-70b-8192

grok "Hello, Llama!"
```

**사용 가능 모델**:
- `llama3-70b-8192`
- `llama3-8b-8192`
- `mixtral-8x7b-32768`
- `gemma-7b-it`

### Together AI

```bash
export GROK_API_KEY=your-together-key
export GROK_BASE_URL=https://api.together.xyz/v1
export GROK_MODEL=meta-llama/Llama-3-70b-chat-hf
```

### 로컬 모델 (Ollama, LM Studio)

```bash
# Ollama
export GROK_BASE_URL=http://localhost:11434/v1
export GROK_MODEL=llama3
export GROK_API_KEY=dummy  # 필수는 아니지만 설정 필요

# LM Studio
export GROK_BASE_URL=http://localhost:1234/v1
export GROK_MODEL=local-model
export GROK_API_KEY=lm-studio
```

### 커스텀 설정

```typescript
// 사용자 설정에 모델 추가
const settings = settingsManager.getUserSettings();
settings.models.push('custom-model-name');
settings.baseURL = 'https://custom-api.com/v1';
settingsManager.saveUserSettings(settings);
```

---

## 🔐 보안 고려사항

### API 키 보호

```bash
# ✅ 환경 변수 사용
export GROK_API_KEY=xai-...

# ✅ 설정 파일 (권한 0o600)
~/.grok/user-settings.json

# ❌ 코드에 하드코딩
const apiKey = 'xai-123...';  # 절대 금지!

# ❌ Git에 커밋
git add .env  # .gitignore 확인!
```

### 네트워크 보안

```typescript
// HTTPS 강제
const client = new OpenAI({
  baseURL: 'https://api.x.ai/v1'  // http:// 사용 금지
});

// 타임아웃 설정
const response = await client.chat.completions.create({
  ...params,
  timeout: 30000  // 30초
});
```

### 에러 로깅

```typescript
try {
  const response = await client.chat.completions.create(params);
} catch (error) {
  // ✅ 에러 로그 (API 키 제외)
  console.error('API request failed:', {
    status: error.status,
    message: error.message
  });

  // ❌ 전체 에러 로그 (API 키 노출 위험)
  console.error('Error:', error);
}
```

---

## 📊 API 사용 모니터링

### 토큰 카운팅

```typescript
import { encoding_for_model } from 'tiktoken';

const encoding = encoding_for_model('grok-code-fast-1');
const tokens = encoding.encode(text);
console.log(`Tokens: ${tokens.length}`);
```

### 비용 추정

```typescript
// 대략적인 비용 계산
const inputTokens = 1000;
const outputTokens = 500;
const inputCost = inputTokens * 0.00001;  # 모델별 가격
const outputCost = outputTokens * 0.00003;
const totalCost = inputCost + outputCost;
```

### 요청 로깅

```typescript
// 요청/응답 로그
console.log({
  timestamp: new Date().toISOString(),
  model: 'grok-code-fast-1',
  input_tokens: 1000,
  output_tokens: 500,
  latency_ms: 2500
});
```

---

## 🎓 추가 리소스

- [X.AI API 문서](https://docs.x.ai)
- [OpenAI API 참조](https://platform.openai.com/docs/api-reference)
- [MCP 공식 문서](https://modelcontextprotocol.io)
- [Ripgrep 가이드](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md)
