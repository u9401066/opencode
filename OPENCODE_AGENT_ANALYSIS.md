# 🤖 OpenCode Agent 架構研究分析

> **作者**: u9401066  
> **日期**: 2026-01-15  
> **來源**: [sst/opencode](https://github.com/sst/opencode)

---

## 📋 目錄

1. [專案概述](#專案概述)
2. [Vercel AI SDK 深度解析](#vercel-ai-sdk-深度解析)
3. [Agent 架構設計](#agent-架構設計)
4. [核心元件分析](#核心元件分析)
5. [完整執行流程實例](#完整執行流程實例)
6. [工具系統詳解](#工具系統詳解)
7. [權限系統](#權限系統)
8. [MCP 整合](#mcp-整合)
9. [Token 管理與 Compaction](#token-管理與-compaction)
10. [Meme 圖解](#meme-圖解)
11. [技術亮點](#技術亮點)
12. [結論](#結論)

---

## 專案概述

OpenCode 是一個開源的 AI 編程助手，類似 Claude Code / Cursor，採用 **Agent Loop** 架構實現自主編程能力。

### 技術棧
| 技術 | 用途 | 說明 |
|------|------|------|
| **Bun** | Runtime | 高效能 JavaScript 執行環境 |
| **TypeScript** | 語言 | 強型別開發 |
| **Vercel AI SDK** | AI 整合 | 統一多家 LLM Provider 介面 |
| **Zod** | Schema 驗證 | 執行時型別檢查 |
| **Namespace 模式** | 架構 | 模組化組織程式碼 |

---

## Vercel AI SDK 深度解析

OpenCode 的核心 AI 能力建立在 **Vercel AI SDK** 之上，這是一個統一多家 LLM Provider 的抽象層。

### 為什麼選擇 Vercel AI SDK？

```
┌─────────────────────────────────────────────────────────────────┐
│                    傳統做法 (痛苦)                               │
│                                                                 │
│   OpenAI API ──┐                                               │
│                │    每個 Provider 都要寫不同的程式碼            │
│   Anthropic ───┼──▶ 不同的請求格式、回應結構、錯誤處理         │
│                │    維護成本極高 😭                             │
│   Google ──────┘                                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Vercel AI SDK (優雅)                           │
│                                                                 │
│   OpenAI API ──┐                                               │
│                │    ┌─────────────────┐    ┌──────────────┐   │
│   Anthropic ───┼──▶ │  AI SDK 統一層  │──▶ │ streamText() │   │
│                │    └─────────────────┘    └──────────────┘   │
│   Google ──────┘           │                                   │
│   Bedrock ─────┘           └── 一套程式碼，支援 20+ Providers   │
└─────────────────────────────────────────────────────────────────┘
```

### 支援的 Provider 列表

OpenCode 內建支援以下 Provider（來自 `provider.ts`）：

| Provider | SDK 套件 | 說明 |
|----------|----------|------|
| Anthropic | `@ai-sdk/anthropic` | Claude 系列模型 |
| OpenAI | `@ai-sdk/openai` | GPT-4o, o1, o3 等 |
| Google | `@ai-sdk/google` | Gemini 系列 |
| Amazon Bedrock | `@ai-sdk/amazon-bedrock` | AWS 托管模型 |
| Azure OpenAI | `@ai-sdk/azure` | Azure 版 OpenAI |
| Groq | `@ai-sdk/groq` | 快速推理 |
| Mistral | `@ai-sdk/mistral` | Mistral AI |
| xAI | `@ai-sdk/xai` | Grok 模型 |
| DeepSeek | `@ai-sdk/deepseek` | DeepSeek 模型 |
| Cerebras | `@ai-sdk/cerebras` | 高速推理 |
| Fireworks | `@ai-sdk/fireworks` | 開源模型托管 |
| Together | `@ai-sdk/togetherai` | 開源模型托管 |
| OpenRouter | `@openrouter/ai-sdk-provider` | 多模型路由 |
| Ollama | `ollama-ai-provider` | 本地模型 |

### 核心函數：`streamText()`

所有 LLM 呼叫最終都通過 `llm.ts` 中的 `LLM.stream()` 函數：

```typescript
// packages/opencode/src/session/llm.ts (簡化版)
import { streamText } from "ai"

export namespace LLM {
  export async function* stream(input: StreamInput) {
    const { messages, model, tools, system, providerOptions } = input
    
    // 🔑 核心：使用 Vercel AI SDK 的 streamText
    const stream = streamText({
      model,                    // LanguageModelV2 介面
      messages,                 // 對話歷史
      tools,                    // 可用工具定義
      system,                   // 系統提示詞
      providerOptions,          // Provider 特定選項
      experimental_telemetry: { isEnabled: true },
      maxSteps: 1,              // 單步執行
    })
    
    // 串流處理每個事件
    for await (const value of stream.fullStream) {
      yield value  // text-delta, tool-call, tool-result, etc.
    }
  }
}
```

### Provider 載入機制

```typescript
// provider.ts 中的動態載入
const BUNDLED_PROVIDERS = {
  "@ai-sdk/anthropic": () => import("@ai-sdk/anthropic"),
  "@ai-sdk/openai": () => import("@ai-sdk/openai"),
  "@ai-sdk/google": () => import("@ai-sdk/google"),
  // ... 更多
}

// 取得 Language Model
async function getLanguage(providerID: string, modelID: string) {
  const sdk = await getSDK(providerID)  // 動態載入 SDK
  const create = sdk.default || sdk[Object.keys(sdk)[0]]
  const provider = create({ /* options */ })
  return provider(modelID)  // 回傳 LanguageModelV2
}
```

---

## Agent 架構設計

### 整體架構圖

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER INPUT                                   │
│                    "幫我重構這段程式碼"                               │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      SESSION LAYER                                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐  │
│  │  prompt.ts  │───▶│ processor.ts│───▶│       llm.ts            │  │
│  │   (入口)    │    │  (Stream)   │    │   (AI SDK 封裝)         │  │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       AGENT LAYER                                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  build  │ │  plan   │ │ general │ │ explore │ │  task   │       │
│  │ (主要)  │ │ (規劃)  │ │ (通用)  │ │ (探索)  │ │ (子代理)│       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       TOOL LAYER                                     │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │
│  │ bash │ │ read │ │ write│ │ edit │ │ grep │ │ glob │ │search│   │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    PERMISSION LAYER                                  │
│         ┌─────────────────────────────────────────┐                 │
│         │   allow / deny / ask (per tool/pattern) │                 │
│         └─────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 核心元件分析

### 1. Agent 定義 (`agent.ts`)

```typescript
// Agent 資訊結構
interface AgentInfo {
  name: string;              // 代理名稱
  description?: string;      // 用途說明
  mode: "subagent" | "primary" | "all";
  permission: PermissionRuleset;
  model?: { providerID, modelID };
  prompt?: string;           // 系統提示詞
  temperature?: number;
  steps?: number;            // 最大執行步數
}
```

### 內建 Agent 一覽

| Agent | Mode | 功能 | 權限特點 |
|-------|------|------|----------|
| `build` | primary | 主要編碼執行 | 完整工具存取 |
| `plan` | primary | 只讀規劃模式 | 禁止 edit/write |
| `general` | subagent | 複雜任務研究 | 無 TODO 權限 |
| `explore` | subagent | Codebase 探索 | 只有讀取工具 |
| `compaction` | hidden | Token 壓縮 | 全部禁止 |
| `title` | hidden | 產生標題 | 全部禁止 |

### 2. Session Loop (`prompt.ts`)

```
┌──────────────────────────────────────────────────────┐
│                    MAIN LOOP                          │
│  ┌─────────────────────────────────────────────────┐ │
│  │  while (true) {                                 │ │
│  │    1. 取得最後 user message                     │ │
│  │    2. 檢查是否需要 compaction                   │ │
│  │    3. 選擇 agent + 準備 tools                   │ │
│  │    4. 呼叫 LLM (streamText)                     │ │
│  │    5. 處理 tool calls                          │ │
│  │    6. 決定: continue / stop / compact          │ │
│  │  }                                             │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

### 3. Tool 系統 (`tool.ts`)

```typescript
Tool.define("read", {
  description: "讀取檔案內容",
  parameters: z.object({
    filePath: z.string(),
    offset: z.number().optional(),
    limit: z.number().optional(),
  }),
  execute: async (args, ctx) => {
    // 1. 權限檢查
    await ctx.ask({ permission: "read", patterns: [args.filePath] });
    // 2. 執行操作
    const content = await Bun.file(args.filePath).text();
    // 3. 回傳結果
    return { title: "Read file", output: content, metadata: {} };
  }
});
```

---

## 完整執行流程實例

讓我們追蹤一個真實請求從輸入到完成的完整流程：

### 場景：用戶要求「幫我在 utils.ts 加一個 formatDate 函數」

```
📝 用戶輸入: "幫我在 utils.ts 加一個 formatDate 函數"
```

### Step 1: Session 接收請求

```typescript
// prompt.ts - SessionPrompt.loop()
async function* loop(input: LoopInput) {
  const { sessionID } = input
  
  // 取得對話歷史
  const messages = await Message.list({ sessionID })
  const lastUserMessage = messages.findLast(m => m.role === "user")
  // → "幫我在 utils.ts 加一個 formatDate 函數"
```

### Step 2: 選擇 Agent 與準備工具

```typescript
  // 根據設定選擇 agent（預設是 "build"）
  const agent = Agent.get("build")
  
  // 解析該 agent 可用的工具
  const tools = await resolveTools(agent, sessionID)
  // → { read, write, edit, bash, grep, glob, task, ... }
```

### Step 3: 建構系統提示詞

```typescript
  // system.ts - 組合完整的 system prompt
  const systemPrompt = `
    你是 OpenCode，一個 AI 編程助手。
    
    當前工作目錄: /home/user/project
    作業系統: Linux
    Shell: bash
    
    可用工具: read, write, edit, bash, grep, glob...
    
    ${agent.prompt}  // agent 特定指示
  `
```

### Step 4: 呼叫 LLM (streamText)

```typescript
  // llm.ts - 使用 Vercel AI SDK
  const stream = LLM.stream({
    model: await Provider.getLanguage("anthropic", "claude-sonnet-4-20250514"),
    messages: [
      { role: "system", content: systemPrompt },
      { role: "user", content: "幫我在 utils.ts 加一個 formatDate 函數" }
    ],
    tools: toolDefinitions,
  })
```

### Step 5: 處理 LLM 回應串流

```typescript
  // processor.ts - SessionProcessor
  for await (const event of stream) {
    switch (event.type) {
      case "text-delta":
        // 💬 AI 正在思考...
        yield { type: "text", content: event.textDelta }
        // → "好的，讓我先讀取 utils.ts 的內容..."
        break
        
      case "tool-call":
        // 🔧 AI 決定呼叫工具
        yield { type: "tool-call", tool: event.toolName, args: event.args }
        // → { tool: "read", args: { filePath: "utils.ts" } }
        break
    }
  }
```

### Step 6: 執行工具呼叫

```typescript
  // 當收到 tool-call 事件
  if (event.type === "tool-call") {
    const tool = ToolRegistry.get(event.toolName)  // 取得 read 工具
    
    // 🔐 權限檢查
    const permission = await PermissionNext.check({
      tool: "read",
      patterns: ["utils.ts"],
      ruleset: agent.permission,
    })
    
    if (permission === "ask") {
      // 詢問用戶是否允許
      yield { type: "permission-request", tool: "read", file: "utils.ts" }
      // 等待用戶回應...
    }
    
    // ✅ 執行工具
    const result = await tool.execute(event.args, ctx)
    // → { output: "export function add(a, b) { return a + b }\n..." }
  }
```

### Step 7: 工具結果回傳給 LLM

```typescript
  // 將工具結果加入對話
  messages.push({
    role: "tool",
    content: result.output,
    toolCallId: event.toolCallId,
  })
  
  // 繼續 LLM 對話（下一輪）
  // AI 看到 utils.ts 內容後，決定使用 edit 工具
```

### Step 8: AI 執行編輯

```typescript
  // AI 的第二次工具呼叫
  event = {
    type: "tool-call",
    toolName: "edit",
    args: {
      filePath: "utils.ts",
      oldString: "export function add(a, b) { return a + b }",
      newString: `export function add(a, b) { return a + b }

export function formatDate(date: Date): string {
  return date.toISOString().split('T')[0]
}`
    }
  }
  
  // 執行 edit 工具
  const editResult = await editTool.execute(event.args, ctx)
  // → { success: true, diff: "..." }
```

### Step 9: 完成與回應

```typescript
  // AI 確認完成
  yield {
    type: "text",
    content: "✅ 已在 utils.ts 中加入 formatDate 函數！\n\n" +
             "```typescript\n" +
             "export function formatDate(date: Date): string {\n" +
             "  return date.toISOString().split('T')[0]\n" +
             "}\n```"
  }
  
  // Loop 結束條件：AI 沒有更多工具呼叫
  return { status: "completed" }
}
```

### 完整流程圖

```
┌──────────────────────────────────────────────────────────────────────┐
│                        完整執行流程                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  👤 User: "幫我在 utils.ts 加一個 formatDate 函數"                    │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 📋 SessionPrompt.loop()                                         ││
│  │    ├── 載入對話歷史                                              ││
│  │    ├── 選擇 Agent (build)                                       ││
│  │    └── 準備工具列表                                              ││
│  └─────────────────────────────────────────────────────────────────┘│
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 🤖 LLM.stream() [Round 1]                                       ││
│  │    └── AI: "讓我先讀取檔案..." → tool_call: read("utils.ts")    ││
│  └─────────────────────────────────────────────────────────────────┘│
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 🔐 Permission Check                                             ││
│  │    └── read + utils.ts → ✅ allowed                             ││
│  └─────────────────────────────────────────────────────────────────┘│
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 🔧 Tool Execute: read                                           ││
│  │    └── return file content                                      ││
│  └─────────────────────────────────────────────────────────────────┘│
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 🤖 LLM.stream() [Round 2]                                       ││
│  │    └── AI 看到內容 → tool_call: edit(utils.ts, ...)             ││
│  └─────────────────────────────────────────────────────────────────┘│
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 🔧 Tool Execute: edit                                           ││
│  │    └── 修改檔案，加入 formatDate 函數                            ││
│  └─────────────────────────────────────────────────────────────────┘│
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 🤖 LLM.stream() [Round 3]                                       ││
│  │    └── AI: "✅ 完成！" (無更多 tool calls)                       ││
│  └─────────────────────────────────────────────────────────────────┘│
│       │                                                              │
│       ▼                                                              │
│  ✅ Session 完成                                                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 工具系統詳解

### Tool.define() 核心介面

每個工具都透過 `Tool.define()` 函數定義，這是 OpenCode 工具系統的核心：

```typescript
// packages/opencode/src/tool/tool.ts
export namespace Tool {
  export function define<TArgs extends z.ZodType>(
    name: string,
    config: {
      description: string;           // 給 LLM 看的工具說明
      parameters: TArgs;             // Zod schema 定義參數
      execute: (
        args: z.infer<TArgs>,       // 解析後的參數
        ctx: ToolContext            // 執行上下文
      ) => Promise<ToolResult>;
    }
  ): ToolDefinition
}

// ToolResult 回傳結構
interface ToolResult {
  title?: string;      // 執行標題 (顯示用)
  output: string;      // 主要輸出內容
  metadata?: any;      // 額外資訊 (檔案路徑、行數等)
}
```

### 內建工具實作範例

#### 1. Read Tool (讀取檔案)

```typescript
// packages/opencode/src/tool/read.ts
Tool.define("read", {
  description: `
    讀取檔案內容。支援 offset 和 limit 參數進行分段讀取。
    對於大檔案，建議分段讀取避免 token 過多。
  `,
  parameters: z.object({
    filePath: z.string().describe("要讀取的檔案路徑"),
    offset: z.number().optional().describe("起始行數 (1-based)"),
    limit: z.number().optional().describe("最大讀取行數"),
  }),
  async execute(args, ctx) {
    // 請求權限
    await ctx.ask({ permission: "read", patterns: [args.filePath] })
    
    // 讀取檔案
    const file = Bun.file(args.filePath)
    const content = await file.text()
    
    // 處理 offset/limit
    const lines = content.split("\n")
    const start = (args.offset ?? 1) - 1
    const end = args.limit ? start + args.limit : lines.length
    const result = lines.slice(start, end).join("\n")
    
    return {
      title: `Read ${args.filePath}`,
      output: result,
      metadata: { lines: lines.length, path: args.filePath }
    }
  }
})
```

#### 2. Edit Tool (編輯檔案)

```typescript
// packages/opencode/src/tool/edit.ts (簡化版)
Tool.define("edit", {
  description: `
    編輯檔案內容。使用 oldString → newString 替換模式。
    必須提供精確的 oldString 以避免錯誤替換。
  `,
  parameters: z.object({
    filePath: z.string(),
    oldString: z.string().describe("要被替換的原始內容"),
    newString: z.string().describe("替換後的新內容"),
  }),
  async execute(args, ctx) {
    await ctx.ask({ permission: "edit", patterns: [args.filePath] })
    
    const file = Bun.file(args.filePath)
    const content = await file.text()
    
    // 檢查 oldString 是否存在且唯一
    const matches = content.split(args.oldString).length - 1
    if (matches === 0) {
      throw new Error("oldString not found in file")
    }
    if (matches > 1) {
      throw new Error("oldString matches multiple locations")
    }
    
    // 執行替換
    const newContent = content.replace(args.oldString, args.newString)
    await Bun.write(args.filePath, newContent)
    
    return {
      title: `Edit ${args.filePath}`,
      output: generateDiff(args.oldString, args.newString),
      metadata: { path: args.filePath }
    }
  }
})
```

#### 3. Bash Tool (執行命令)

```typescript
// packages/opencode/src/tool/bash.ts (簡化版)
Tool.define("bash", {
  description: `
    執行 shell 命令。可設定 timeout 和工作目錄。
    危險命令會需要用戶確認。
  `,
  parameters: z.object({
    command: z.string().describe("要執行的命令"),
    timeout: z.number().optional().default(30000),
    cwd: z.string().optional(),
  }),
  async execute(args, ctx) {
    // 危險命令檢查
    const dangerous = ["rm -rf", "sudo", "mkfs", "> /dev/"]
    if (dangerous.some(d => args.command.includes(d))) {
      await ctx.ask({ permission: "bash:dangerous", patterns: [args.command] })
    }
    
    // 執行命令
    const proc = Bun.spawn(["bash", "-c", args.command], {
      cwd: args.cwd,
      timeout: args.timeout,
    })
    
    const stdout = await new Response(proc.stdout).text()
    const stderr = await new Response(proc.stderr).text()
    
    return {
      title: `$ ${args.command}`,
      output: stdout + (stderr ? `\n[stderr]\n${stderr}` : ""),
      metadata: { exitCode: proc.exitCode }
    }
  }
})
```

#### 4. Task Tool (呼叫子代理)

```typescript
// packages/opencode/src/tool/task.ts (簡化版)
Tool.define("task", {
  description: `
    派遣子代理執行特定任務。適合：
    - 需要深入研究的問題
    - 可並行處理的獨立任務
    - 需要不同專長的任務
  `,
  parameters: z.object({
    description: z.string().describe("任務描述"),
    agent: z.enum(["general", "explore"]).optional(),
  }),
  async execute(args, ctx) {
    const agentName = args.agent ?? "general"
    const agent = Agent.get(agentName)
    
    // 建立子 session
    const childSession = await Session.create({
      parent: ctx.sessionID,
      agent: agentName,
    })
    
    // 執行子代理 loop
    const result = await SessionPrompt.run({
      sessionID: childSession.id,
      message: args.description,
      maxSteps: agent.steps ?? 10,
    })
    
    return {
      title: `Task: ${args.description.slice(0, 50)}...`,
      output: result.summary,
      metadata: { agent: agentName, steps: result.steps }
    }
  }
})
```

### 工具註冊與發現

```typescript
// packages/opencode/src/tool/registry.ts
export namespace ToolRegistry {
  const tools = new Map<string, ToolDefinition>()
  
  // 註冊工具
  export function register(tool: ToolDefinition) {
    tools.set(tool.name, tool)
  }
  
  // 取得工具
  export function get(name: string): ToolDefinition | undefined {
    return tools.get(name)
  }
  
  // 列出所有工具 (給 LLM 用)
  export function list(): ToolDefinition[] {
    return Array.from(tools.values())
  }
  
  // 轉換成 AI SDK 格式
  export function toAITools(filter?: string[]): Record<string, CoreTool> {
    const result: Record<string, CoreTool> = {}
    for (const tool of tools.values()) {
      if (!filter || filter.includes(tool.name)) {
        result[tool.name] = {
          description: tool.description,
          parameters: tool.parameters,
        }
      }
    }
    return result
  }
}
```

---

## 權限系統

OpenCode 實作了精細的權限控制系統，確保 AI 不會執行未授權的操作。

### 權限規則結構

```typescript
// packages/opencode/src/permission/permission.ts
type PermissionLevel = "allow" | "deny" | "ask"

interface PermissionRuleset {
  [tool: string]: PermissionLevel | {
    [pattern: string]: PermissionLevel
  }
}

// 範例：Plan Agent 的權限設定
const planPermission: PermissionRuleset = {
  "*": "deny",                    // 預設全部拒絕
  read: "allow",                  // 允許讀取
  grep: "allow",                  // 允許搜尋
  glob: "allow",                  // 允許列出檔案
  edit: {
    "*": "deny",                  // 預設不能編輯
    ".opencode/plans/*.md": "allow"  // 只能編輯計畫檔
  }
}
```

### 權限檢查流程

```typescript
// packages/opencode/src/permission/next.ts
export namespace PermissionNext {
  export async function check(input: {
    tool: string;
    patterns: string[];
    ruleset: PermissionRuleset;
    sessionID: string;
  }): Promise<PermissionLevel> {
    const { tool, patterns, ruleset, sessionID } = input
    
    // 1. 檢查工具層級規則
    const toolRule = ruleset[tool] ?? ruleset["*"] ?? "ask"
    
    if (typeof toolRule === "string") {
      return toolRule  // "allow" | "deny" | "ask"
    }
    
    // 2. 檢查 pattern 層級規則
    for (const pattern of patterns) {
      const patternRule = matchPattern(pattern, toolRule)
      if (patternRule === "deny") {
        return "deny"  // 只要有一個 deny 就拒絕
      }
    }
    
    // 3. 檢查 session 暫存的權限
    const cached = await getSessionPermission(sessionID, tool, patterns)
    if (cached) return cached
    
    // 4. 預設詢問用戶
    return "ask"
  }
}
```

### 權限請求處理

```typescript
// 當權限是 "ask" 時的處理流程
async function handleAskPermission(ctx: ToolContext, request: {
  tool: string;
  patterns: string[];
}) {
  // 發送權限請求給前端
  ctx.emit({
    type: "permission-request",
    tool: request.tool,
    patterns: request.patterns,
    options: ["allow", "deny", "allow-always", "deny-always"]
  })
  
  // 等待用戶回應
  const response = await ctx.waitForPermission()
  
  switch (response) {
    case "allow":
      return true  // 本次允許
      
    case "deny":
      throw new PermissionDeniedError(request.tool)
      
    case "allow-always":
      // 儲存到 session，後續相同請求自動允許
      await saveSessionPermission(ctx.sessionID, request.tool, "allow")
      return true
      
    case "deny-always":
      await saveSessionPermission(ctx.sessionID, request.tool, "deny")
      throw new PermissionDeniedError(request.tool)
  }
}
```

### Doom Loop 防護

防止 AI 陷入無限迴圈（重複執行相同操作）：

```typescript
// packages/opencode/src/session/processor.ts
export namespace SessionProcessor {
  // 追蹤最近的工具呼叫
  const recentCalls: Map<string, ToolCall[]> = new Map()
  
  export function detectDoomLoop(
    sessionID: string,
    toolCall: ToolCall
  ): boolean {
    const calls = recentCalls.get(sessionID) ?? []
    
    // 檢查是否連續 3 次相同呼叫
    const lastThree = calls.slice(-3)
    if (lastThree.length === 3) {
      const allSame = lastThree.every(c => 
        c.name === toolCall.name &&
        JSON.stringify(c.args) === JSON.stringify(toolCall.args)
      )
      if (allSame) {
        return true  // 🚨 Doom Loop detected!
      }
    }
    
    // 記錄本次呼叫
    calls.push(toolCall)
    if (calls.length > 10) calls.shift()  // 只保留最近 10 次
    recentCalls.set(sessionID, calls)
    
    return false
  }
}
```

### 權限系統流程圖

```text
┌─────────────────────────────────────────────────────────────────┐
│                     權限檢查流程                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🔧 Tool Call: edit("src/app.ts", ...)                         │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Step 1: 檢查工具規則                                       │ │
│  │   ruleset["edit"] = { "*": "ask", "*.md": "allow" }       │ │
│  └───────────────────────────────────────────────────────────┘ │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Step 2: 匹配 Pattern                                       │ │
│  │   "src/app.ts" matches "*" → "ask"                        │ │
│  └───────────────────────────────────────────────────────────┘ │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Step 3: 檢查 Session 快取                                  │ │
│  │   沒有快取 → 繼續                                          │ │
│  └───────────────────────────────────────────────────────────┘ │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Step 4: 詢問用戶                                           │ │
│  │   "允許編輯 src/app.ts？"                                  │ │
│  │   [允許] [拒絕] [永遠允許] [永遠拒絕]                       │ │
│  └───────────────────────────────────────────────────────────┘ │
│       │                                                         │
│       ▼                                                         │
│  ✅ 用戶選擇「允許」→ 執行工具                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## MCP 整合

**Model Context Protocol (MCP)** 是 Anthropic 提出的標準協議，讓 AI 助手可以連接外部工具和資料源。OpenCode 完整支援 MCP。

### 什麼是 MCP？

```text
┌─────────────────────────────────────────────────────────────────┐
│                    MCP 架構概念                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐         ┌──────────────┐         ┌───────────┐ │
│   │ OpenCode │◄───────►│  MCP Server  │◄───────►│ 外部服務  │ │
│   │  (Host)  │  stdio  │  (Bridge)    │   API   │           │ │
│   └──────────┘         └──────────────┘         └───────────┘ │
│                                                                 │
│   範例 MCP Servers:                                             │
│   • 資料庫查詢 (PostgreSQL, MongoDB)                            │
│   • API 整合 (GitHub, Slack, Notion)                           │
│   • 檔案系統 (Google Drive, Dropbox)                           │
│   • 搜尋引擎 (Brave Search, Tavily)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### MCP 設定 (opencode.json)

```json
{
  "mcp": {
    "servers": {
      "github": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {
          "GITHUB_TOKEN": "${GITHUB_TOKEN}"
        }
      },
      "postgres": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-postgres"],
        "env": {
          "DATABASE_URL": "${DATABASE_URL}"
        }
      },
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/dir"]
      }
    }
  }
}
```

### MCP 整合程式碼

```typescript
// packages/opencode/src/mcp/index.ts (簡化版)
import { Client } from "@modelcontextprotocol/sdk/client"
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio"

export namespace MCP {
  const clients: Map<string, Client> = new Map()
  
  // 啟動 MCP Server
  export async function connect(name: string, config: MCPServerConfig) {
    const transport = new StdioClientTransport({
      command: config.command,
      args: config.args,
      env: { ...process.env, ...config.env }
    })
    
    const client = new Client({ name: `opencode-${name}` })
    await client.connect(transport)
    
    clients.set(name, client)
    return client
  }
  
  // 列出 MCP Server 提供的工具
  export async function listTools(serverName: string) {
    const client = clients.get(serverName)
    if (!client) throw new Error(`MCP server ${serverName} not connected`)
    
    const { tools } = await client.listTools()
    return tools  // [{ name, description, inputSchema }, ...]
  }
  
  // 呼叫 MCP 工具
  export async function callTool(
    serverName: string,
    toolName: string,
    args: any
  ) {
    const client = clients.get(serverName)
    if (!client) throw new Error(`MCP server ${serverName} not connected`)
    
    const result = await client.callTool({ name: toolName, arguments: args })
    return result
  }
}
```

### MCP 工具整合到 Agent

```typescript
// 將 MCP 工具轉換成內部工具格式
async function loadMCPTools(serverName: string): Promise<ToolDefinition[]> {
  const mcpTools = await MCP.listTools(serverName)
  
  return mcpTools.map(mcpTool => ({
    name: `mcp_${serverName}_${mcpTool.name}`,
    description: mcpTool.description,
    parameters: convertJsonSchemaToZod(mcpTool.inputSchema),
    execute: async (args, ctx) => {
      // 呼叫 MCP Server
      const result = await MCP.callTool(serverName, mcpTool.name, args)
      
      return {
        title: `MCP: ${mcpTool.name}`,
        output: JSON.stringify(result, null, 2),
        metadata: { server: serverName }
      }
    }
  }))
}
```

### MCP 使用範例

```text
👤 User: "查一下 GitHub 上 sst/opencode 的最新 PR"

🤖 Agent 思考: 需要使用 GitHub MCP...

🔧 Tool Call: mcp_github_list_pull_requests
   Arguments: { repo: "sst/opencode", state: "open", limit: 5 }
   
📡 MCP Server (GitHub):
   → 呼叫 GitHub API
   → 回傳 PR 列表

🤖 Agent: "以下是 sst/opencode 的最新 5 個 PR:
   1. #234 - Add streaming support
   2. #233 - Fix permission bug
   ..."
```

---

## Token 管理與 Compaction

### 為什麼需要 Compaction？

LLM 有 context window 限制（如 Claude 200K tokens），長對話會超出限制：

```text
┌─────────────────────────────────────────────────────────────────┐
│                    Token 使用量                                  │
│                                                                 │
│  ▲ tokens                                                       │
│  │                                                              │
│  │                           ╭────────────╮                    │
│  │                      ╭────╯            │ ← 接近限制 ⚠️        │
│  │                 ╭────╯                 │                     │
│  │            ╭────╯                      │                     │
│  │       ╭────╯                           │                     │
│  │  ─────╯                                │                     │
│  │                                        │                     │
│  ├────────────────────────────────────────┼────────► 對話輪數    │
│  │                                   🗜️ Compaction!             │
│  │                                        │                     │
│  │                                   ╭────╮ ← 壓縮後            │
│  │                              ─────╯                          │
│  │                                                              │
└─────────────────────────────────────────────────────────────────┘
```

### Compaction 機制

```typescript
// packages/opencode/src/session/compaction.ts (簡化版)
export namespace SessionCompaction {
  
  // 檢查是否需要壓縮
  export function isOverflow(input: {
    messages: Message[];
    model: ModelInfo;
  }): boolean {
    const totalTokens = estimateTokens(input.messages)
    const limit = input.model.contextWindow ?? 128000
    const threshold = limit * 0.8  // 80% 時觸發
    
    return totalTokens > threshold
  }
  
  // 執行壓縮
  export async function compact(input: {
    sessionID: string;
    messages: Message[];
  }): Promise<Message[]> {
    const { sessionID, messages } = input
    
    // 使用專門的 compaction agent 來摘要對話
    const summary = await runCompactionAgent({
      messages,
      instruction: `
        請摘要以下對話，保留：
        1. 用戶的主要目標
        2. 已完成的重要操作
        3. 當前進度和待辦事項
        4. 重要的檔案路徑和程式碼片段
      `
    })
    
    // 建立新的壓縮訊息
    const compactedMessage: Message = {
      role: "user",
      content: `[對話摘要]\n${summary}\n\n[繼續之前的任務]`
    }
    
    // 保留最後幾輪對話 + 摘要
    const recentMessages = messages.slice(-4)
    return [compactedMessage, ...recentMessages]
  }
  
  // 選擇性刪除工具結果 (保留重點)
  export function prune(messages: Message[]): Message[] {
    return messages.map(msg => {
      if (msg.role === "tool" && msg.content.length > 10000) {
        // 截斷過長的工具輸出
        return {
          ...msg,
          content: msg.content.slice(0, 5000) + "\n...[truncated]..."
        }
      }
      return msg
    })
  }
}
```

### Compaction 流程圖

```text
┌─────────────────────────────────────────────────────────────────┐
│                   Compaction 決策流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  每次 LLM 呼叫前:                                                │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ 計算 token 使用量                                          │ │
│  │   totalTokens = estimateTokens(messages)                  │ │
│  └───────────────────────────────────────────────────────────┘ │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ tokens > 80% limit?                                       │ │
│  └───────────────────────────────────────────────────────────┘ │
│       │                                                         │
│       ├── No ──► 繼續正常執行                                    │
│       │                                                         │
│       └── Yes ──┐                                               │
│                 ▼                                               │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ 🗜️ 執行 Compaction                                         │ │
│  │   1. prune() - 截斷長輸出                                  │ │
│  │   2. compact() - 使用 AI 摘要對話                          │ │
│  │   3. 保留最後 4 輪對話                                     │ │
│  └───────────────────────────────────────────────────────────┘ │
│                 │                                               │
│                 ▼                                               │
│  ✅ 使用壓縮後的 messages 繼續                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Meme 圖解

### 🎭 OpenCode Agent 的一天

```
                    ╔═══════════════════════════════════════╗
                    ║     👨‍💻 User: "修好這個 bug"          ║
                    ╚═══════════════════════════════════════╝
                                      │
                                      ▼
    ┌─────────────────────────────────────────────────────────────┐
    │  🤖 Build Agent: "好的，讓我看看..."                         │
    │                                                             │
    │  📖 read → 讀取檔案                                         │
    │  🔍 grep → 搜尋相關程式碼                                   │
    │  🧠 思考中... "啊！找到問題了"                               │
    │  ✏️ edit → 修改程式碼                                       │
    │  🖥️ bash → npm test                                        │
    │                                                             │
    │  ✅ "Bug 已修復！"                                          │
    └─────────────────────────────────────────────────────────────┘

    ════════════════════════════════════════════════════════════════
    
    💡 當問題太複雜時...
    
    ┌─────────────────────────────────────────────────────────────┐
    │  🤖 Build Agent: "這個需要深入研究..."                       │
    │                                                             │
    │  📤 task(explore) → 派出探索子代理                          │
    │      │                                                      │
    │      ├── 🔎 Explore Agent #1: "搜尋 API 端點..."            │
    │      ├── 🔎 Explore Agent #2: "分析資料結構..."             │
    │      └── 🔎 Explore Agent #3: "檢查測試案例..."             │
    │                                                             │
    │  📥 收集結果 → 綜合分析 → 執行修改                          │
    └─────────────────────────────────────────────────────────────┘
```

### 🔄 Doom Loop 防護

```
    ┌──────────────────────────────────────────────────────────┐
    │  🤖 Agent: edit("file.ts", same_args)  ← 第 1 次          │
    │  🤖 Agent: edit("file.ts", same_args)  ← 第 2 次          │
    │  🤖 Agent: edit("file.ts", same_args)  ← 第 3 次          │
    │                                                          │
    │  🚨 DOOM LOOP DETECTED!                                  │
    │                                                          │
    │  ⚠️ "你確定要繼續嗎？這看起來像是無限迴圈..."              │
    │     [允許] [拒絕] [永遠允許]                              │
    └──────────────────────────────────────────────────────────┘
```

### 📊 Token 管理 (Compaction)

```
    Token 使用量
    ▲
    │
    │                         ╭─────╮
    │                    ╭────╯     │  ← 接近限制
    │               ╭────╯         │
    │          ╭────╯              │
    │     ╭────╯                   │
    │ ────╯                        │
    ├──────────────────────────────┼────────► 對話長度
    │                              │
    │                        🗜️ COMPACTION!
    │                              │
    │                              ▼
    │                         ╭────╮  ← 壓縮後
    │                    ─────╯    
```

---

## 技術亮點

### 1. 🎯 精細權限控制

```typescript
// Plan Agent 權限 - 只讀不寫
permission: {
  "*": "deny",
  read: "allow",
  grep: "allow", 
  glob: "allow",
  edit: {
    "*": "deny",
    ".opencode/plans/*.md": "allow"  // 只能編輯計畫檔
  }
}
```

### 2. 🔄 Streaming 處理

```typescript
for await (const value of stream.fullStream) {
  switch (value.type) {
    case "text-delta":     // 即時顯示文字
    case "tool-call":      // 工具呼叫
    case "tool-result":    // 工具結果
    case "reasoning-delta": // 推理過程 (Claude)
  }
}
```

### 3. 🧩 Plugin 系統

```typescript
// 自訂工具 - 放在 tools/*.ts
export default {
  description: "我的自訂工具",
  args: { input: z.string() },
  execute: async (args) => "結果"
}
```

### 4. 🔌 MCP 整合

支援 Model Context Protocol，可以連接外部工具服務：
- 資料庫查詢
- API 呼叫  
- 外部檔案系統

---

## 結論

### OpenCode Agent 架構的設計哲學

1. **安全優先** - 精細的權限系統防止意外操作
2. **可擴展** - Plugin 和 MCP 支援自訂擴展
3. **智能分工** - Subagent 機制處理複雜任務
4. **資源管理** - Compaction 機制管理長對話

### 與其他工具比較

| 特性 | OpenCode | Claude Code | Cursor |
|------|----------|-------------|--------|
| 開源 | ✅ | ❌ | ❌ |
| Agent 分工 | ✅ 多層級 | ✅ 單層 | ❌ |
| 自訂工具 | ✅ Plugin | ❌ | ❌ |
| MCP 支援 | ✅ | ✅ | ❌ |
| 權限控制 | ✅ 精細 | ✅ 基本 | ❌ |

### 學習價值

OpenCode 是學習 AI Agent 架構的絕佳範例：
- 完整的 Agent Loop 實現
- 現代 TypeScript 架構模式
- 實用的工具系統設計
- 生產級的錯誤處理

---

## 參考資源

- [OpenCode GitHub](https://github.com/sst/opencode)
- [Vercel AI SDK](https://sdk.vercel.ai/)
- [Model Context Protocol](https://modelcontextprotocol.io/)

---

*本分析由 u9401066 於 2026-01-15 完成*
