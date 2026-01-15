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

### 專案結構詳解

```text
packages/opencode/src/
├── agent/                 # Agent 定義層
│   ├── agent.ts          # Agent 類型定義與內建 agents
│   └── index.ts          # 匯出入口
│
├── session/               # Session 管理層 (核心)
│   ├── prompt.ts         # 🔑 主要 Loop 入口
│   ├── processor.ts      # Stream 處理器
│   ├── llm.ts            # LLM 呼叫封裝
│   ├── system.ts         # System Prompt 組合
│   ├── compaction.ts     # Token 壓縮機制
│   └── message.ts        # 訊息儲存管理
│
├── tool/                  # 工具系統層
│   ├── tool.ts           # Tool.define() 核心介面
│   ├── registry.ts       # 工具註冊表
│   ├── bash.ts           # Shell 命令工具
│   ├── read.ts           # 檔案讀取工具
│   ├── write.ts          # 檔案寫入工具
│   ├── edit.ts           # 檔案編輯工具 (含多種 replacer)
│   ├── grep.ts           # 文字搜尋工具
│   ├── glob.ts           # 檔案列表工具
│   ├── task.ts           # 子代理呼叫工具
│   └── ...               # 其他工具
│
├── permission/            # 權限控制層
│   ├── permission.ts     # 權限類型定義
│   └── next.ts           # 權限檢查邏輯
│
├── provider/              # AI Provider 層
│   ├── provider.ts       # 多 Provider 支援
│   └── transform.ts      # 訊息轉換與快取
│
├── mcp/                   # MCP 整合層
│   └── index.ts          # Model Context Protocol 客戶端
│
├── config/                # 設定管理
│   └── config.ts         # opencode.json 解析
│
├── storage/               # 資料持久化
│   ├── sqlite.ts         # SQLite 資料庫
│   └── session.ts        # Session 儲存
│
└── app/                   # 應用入口
    └── index.ts          # CLI / TUI 啟動
```

### Namespace 模式說明

OpenCode 採用 TypeScript Namespace 模式組織程式碼，這是一種函數式風格：

```typescript
// ❌ 傳統 Class 風格
class SessionService {
  static async create() { }
  static async get() { }
  static async list() { }
}

// ✅ OpenCode 的 Namespace 風格
export namespace Session {
  export async function create() { }
  export async function get() { }
  export async function list() { }
}

// 使用方式
import { Session } from "./session"
const session = await Session.create({ ... })
```

**Namespace 的優點：**
1. **Tree-shaking 友善** - 未使用的函數會被移除
2. **無 this 綁定問題** - 純函數更容易測試
3. **清晰的模組邊界** - 每個 Namespace 是獨立單元
4. **IDE 自動完成友善** - `Session.` 後會列出所有可用函數

### 核心資料流

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                         OpenCode 資料流                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  📁 Config Layer                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  opencode.json → Config.load() → 全域設定                        │   │
│  │  • providers: 模型設定                                           │   │
│  │  • mcp: 外部工具服務                                             │   │
│  │  • permissions: 預設權限                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│       │                                                                 │
│       ▼                                                                 │
│  💾 Storage Layer                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  SQLite Database (~/.opencode/data.db)                          │   │
│  │  • sessions: 對話 session 記錄                                   │   │
│  │  • messages: 完整對話歷史                                        │   │
│  │  • permissions: 用戶授權的權限快取                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│       │                                                                 │
│       ▼                                                                 │
│  🔄 Session Layer                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Session.create() → SessionPrompt.loop() → SessionProcessor     │   │
│  │       │                    │                      │              │   │
│  │       │                    ▼                      │              │   │
│  │       │             LLM.stream()                  │              │   │
│  │       │                    │                      │              │   │
│  │       └────────────────────┴──────────────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│       │                                                                 │
│       ▼                                                                 │
│  🔧 Tool Layer                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ToolRegistry.get() → Tool.execute() → PermissionNext.check()   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

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

### Message 轉換層 (`transform.ts`)

不同 Provider 對訊息格式有不同要求，`transform.ts` 負責統一處理：

```typescript
// packages/opencode/src/provider/transform.ts
export namespace ProviderTransform {
  
  // 轉換訊息格式
  export function message(
    messages: CoreMessage[],
    providerID: string
  ): CoreMessage[] {
    return messages.map(msg => {
      // 1. 處理多模態內容 (圖片、檔案)
      if (Array.isArray(msg.content)) {
        msg.content = normalizeContent(msg.content, providerID)
      }
      
      // 2. 應用 Provider 特定的快取策略
      if (providerID.includes("anthropic") || providerID.includes("bedrock")) {
        msg = applyCaching(msg)
      }
      
      return msg
    })
  }
  
  // Anthropic 的 Cache Control (省錢神器)
  function applyCaching(msg: CoreMessage): CoreMessage {
    // Anthropic 支援 prompt caching，可以大幅降低重複內容的成本
    // 對於 system prompt 和工具定義，加上 cache_control
    if (msg.role === "system" || isToolDefinition(msg)) {
      return {
        ...msg,
        experimental_providerMetadata: {
          anthropic: {
            cacheControl: { type: "ephemeral" }
          }
        }
      }
    }
    return msg
  }
}
```

### Provider 特定選項 (`providerOptions`)

```typescript
// 不同 Provider 的特殊設定
export function getProviderOptions(providerID: string, config: ModelConfig) {
  const options: Record<string, any> = {}
  
  // Anthropic: 支援 extended thinking
  if (providerID.includes("anthropic")) {
    if (config.thinking) {
      options.anthropic = {
        thinking: {
          type: "enabled",
          budgetTokens: config.thinkingBudget ?? 10000
        }
      }
    }
  }
  
  // Google Gemini: 思考配置
  if (providerID.includes("google")) {
    if (config.thinking) {
      options.google = {
        thinkingConfig: {
          thinkingBudget: config.thinkingBudget ?? 8000
        }
      }
    }
  }
  
  // OpenAI o1/o3: reasoning effort
  if (providerID.includes("openai") && config.reasoningEffort) {
    options.openai = {
      reasoningEffort: config.reasoningEffort  // "low" | "medium" | "high"
    }
  }
  
  return options
}
```

### 串流事件類型詳解

```typescript
// Vercel AI SDK 的 fullStream 會產生以下事件類型
type StreamEvent = 
  | { type: "text-delta"; textDelta: string }           // 文字片段
  | { type: "tool-call"; toolName: string; args: any }  // 工具呼叫開始
  | { type: "tool-result"; result: any }                // 工具執行結果
  | { type: "reasoning-delta"; textDelta: string }      // 推理過程 (Claude)
  | { type: "finish"; usage: TokenUsage }               // 完成
  | { type: "error"; error: Error }                     // 錯誤

// OpenCode 的處理方式
for await (const event of stream.fullStream) {
  switch (event.type) {
    case "text-delta":
      // 即時串流顯示給用戶
      yield { type: "part", part: { type: "text", text: event.textDelta } }
      break
      
    case "reasoning-delta":
      // Claude 的思考過程 (可選擇是否顯示)
      if (showReasoning) {
        yield { type: "part", part: { type: "reasoning", text: event.textDelta } }
      }
      break
      
    case "tool-call":
      // 記錄工具呼叫，準備執行
      pendingToolCalls.push({
        id: event.toolCallId,
        name: event.toolName,
        args: event.args,
      })
      break
      
    case "finish":
      // 記錄 token 使用量
      tokenUsage = event.usage
      break
  }
}
```

---

## Agent 架構設計

### 整體架構圖

```text
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

### Agent 定義的完整實現

```typescript
// packages/opencode/src/agent/agent.ts (完整版)
import { z } from "zod"

// Agent 資訊的完整 Schema
export const AgentInfoSchema = z.object({
  name: z.string(),
  description: z.string().optional(),
  
  // mode 決定 agent 在哪裡可用
  mode: z.enum([
    "primary",   // 主要 agent，用戶可直接選用
    "subagent",  // 子代理，只能被 task tool 呼叫
    "all",       // 兩者皆可
  ]),
  
  // 權限規則集
  permission: z.record(z.union([
    z.enum(["allow", "deny", "ask"]),
    z.record(z.enum(["allow", "deny", "ask"]))
  ])),
  
  // 可選的模型覆寫
  model: z.object({
    providerID: z.string(),
    modelID: z.string(),
  }).optional(),
  
  // 系統提示詞
  prompt: z.string().optional(),
  
  // LLM 參數
  temperature: z.number().min(0).max(2).optional(),
  maxTokens: z.number().optional(),
  
  // 執行限制
  steps: z.number().optional(),  // 最大工具呼叫次數
  timeout: z.number().optional(), // 超時時間 (ms)
})

export type AgentInfo = z.infer<typeof AgentInfoSchema>
```

### 內建 Agent 詳細設定

```typescript
// packages/opencode/src/agent/agent.ts

export namespace Agent {
  // 內建 agents 定義
  const BUILTIN_AGENTS: Record<string, AgentInfo> = {
    
    // 🔨 Build Agent - 主要編碼執行者
    build: {
      name: "build",
      description: "主要編碼 agent，具備完整的檔案操作和命令執行能力",
      mode: "primary",
      permission: {
        "*": "ask",  // 預設詢問
        read: "allow",
        grep: "allow",
        glob: "allow",
      },
      prompt: `
你是一個專業的程式設計師。你的任務是幫助用戶編寫、修改和除錯程式碼。

工作原則：
1. 在修改檔案前，先閱讀相關程式碼了解上下文
2. 使用 grep 和 glob 搜尋相關檔案
3. 小步驟修改，每次只改動必要的部分
4. 修改後執行測試確認功能正常
5. 遇到複雜問題時，使用 task 派遣子代理深入研究
      `,
      temperature: 0,  // 編碼任務使用低溫度
    },
    
    // 📋 Plan Agent - 只讀規劃模式
    plan: {
      name: "plan",
      description: "規劃模式，只能讀取和分析，不能修改檔案",
      mode: "primary",
      permission: {
        "*": "deny",       // 預設拒絕
        read: "allow",
        grep: "allow",
        glob: "allow",
        codesearch: "allow",
        edit: {
          "*": "deny",
          ".opencode/plans/*.md": "allow",  // 只能編輯計畫檔
        },
        write: {
          "*": "deny",
          ".opencode/plans/*.md": "allow",
        },
      },
      prompt: `
你是一個技術規劃師。你的任務是分析程式碼並提出改進方案。

工作原則：
1. 深入理解現有程式碼結構
2. 識別問題和改進機會
3. 將計畫寫入 .opencode/plans/ 目錄
4. 不要直接修改程式碼，只提供建議
      `,
    },
    
    // 🔬 General Agent - 複雜任務研究
    general: {
      name: "general",
      description: "通用子代理，適合需要深入研究的複雜任務",
      mode: "subagent",
      permission: {
        "*": "ask",
        read: "allow",
        grep: "allow",
        glob: "allow",
        bash: "ask",
        edit: "ask",
        write: "ask",
        todoread: "deny",   // 子代理不能存取主 TODO
        todowrite: "deny",
      },
      steps: 20,  // 限制執行步數
      prompt: `
你是一個研究助手。你的任務是深入調查特定問題並回報發現。

工作原則：
1. 專注於指派的任務
2. 完整收集所需資訊
3. 清晰整理發現結果
4. 不要偏離主題
      `,
    },
    
    // 🔍 Explore Agent - Codebase 探索
    explore: {
      name: "explore",
      description: "探索子代理，只有讀取權限，適合快速了解程式碼",
      mode: "subagent",
      permission: {
        "*": "deny",       // 極度限制
        read: "allow",
        grep: "allow",
        glob: "allow",
        codesearch: "allow",
      },
      steps: 10,
      prompt: `
你是一個程式碼探索器。你的任務是快速理解程式碼結構。

工作原則：
1. 使用 glob 了解目錄結構
2. 使用 grep 搜尋關鍵字
3. 閱讀重要檔案
4. 整理成清晰的摘要
      `,
    },
    
    // 🗜️ Compaction Agent - Token 壓縮 (隱藏)
    compaction: {
      name: "compaction",
      description: "內部使用，壓縮對話歷史",
      mode: "all",  // 但實際上是隱藏的
      permission: {
        "*": "deny",  // 不需要任何工具
      },
      prompt: `
你的任務是摘要對話內容。保留：
1. 用戶的主要目標和需求
2. 已完成的重要操作
3. 當前進度和待辦事項
4. 重要的檔案路徑和程式碼片段
5. 任何錯誤或問題的上下文

輸出格式要清晰、結構化，方便後續對話使用。
      `,
      temperature: 0,
    },
    
    // 📝 Title Agent - 產生對話標題 (隱藏)
    title: {
      name: "title",
      description: "內部使用，產生對話標題",
      mode: "all",
      permission: { "*": "deny" },
      maxTokens: 50,
      prompt: "根據對話內容產生一個簡短的標題（5-10個字）",
    },
  }
  
  // 取得 agent
  export function get(name: string): AgentInfo {
    const agent = BUILTIN_AGENTS[name]
    if (!agent) throw new Error(`Agent "${name}" not found`)
    return agent
  }
  
  // 列出可用 agents
  export function list(mode?: "primary" | "subagent"): AgentInfo[] {
    return Object.values(BUILTIN_AGENTS).filter(a => {
      if (a.name === "compaction" || a.name === "title") return false
      if (!mode) return true
      return a.mode === mode || a.mode === "all"
    })
  }
}
```

---

## 核心元件分析

### 1. Session 管理 (`session/`)

Session 是 OpenCode 的核心概念，代表一次完整的對話互動。

```typescript
// packages/opencode/src/session/session.ts
export namespace Session {
  // Session 資料結構
  export interface SessionData {
    id: string;                    // UUID
    title?: string;                // 自動產生的標題
    agent: string;                 // 使用的 agent 名稱
    parentID?: string;             // 父 session ID (子代理用)
    createdAt: Date;
    updatedAt: Date;
    status: "active" | "completed" | "error";
    metadata: {
      totalTokens: number;         // 累計 token 使用量
      totalCost: number;           // 累計費用
      toolCalls: number;           // 工具呼叫次數
    };
  }
  
  // 建立新 session
  export async function create(input: {
    agent?: string;
    parent?: string;
  }): Promise<SessionData> {
    const session: SessionData = {
      id: crypto.randomUUID(),
      agent: input.agent ?? "build",
      parentID: input.parent,
      createdAt: new Date(),
      updatedAt: new Date(),
      status: "active",
      metadata: { totalTokens: 0, totalCost: 0, toolCalls: 0 },
    }
    
    // 儲存到 SQLite
    await Storage.sessions.insert(session)
    return session
  }
  
  // 取得 session
  export async function get(id: string): Promise<SessionData | null> {
    return Storage.sessions.findOne({ id })
  }
  
  // 列出所有 sessions
  export async function list(options?: {
    limit?: number;
    offset?: number;
  }): Promise<SessionData[]> {
    return Storage.sessions.find({
      orderBy: { updatedAt: "desc" },
      limit: options?.limit ?? 50,
      offset: options?.offset ?? 0,
    })
  }
}
```

### 2. Message 儲存 (`session/message.ts`)

```typescript
// packages/opencode/src/session/message.ts
export namespace Message {
  // 訊息類型 (符合 Vercel AI SDK)
  export type MessageRole = "user" | "assistant" | "tool" | "system"
  
  export interface MessageData {
    id: string;
    sessionID: string;
    role: MessageRole;
    content: string | ContentPart[];  // 支援多模態
    toolCalls?: ToolCall[];           // assistant 的工具呼叫
    toolCallId?: string;              // tool 訊息對應的呼叫 ID
    createdAt: Date;
    metadata?: {
      model?: string;
      tokens?: { input: number; output: number };
      duration?: number;
    };
  }
  
  // 新增訊息
  export async function create(input: {
    sessionID: string;
    role: MessageRole;
    content: string | ContentPart[];
    toolCalls?: ToolCall[];
    toolCallId?: string;
  }): Promise<MessageData> {
    const message: MessageData = {
      id: crypto.randomUUID(),
      ...input,
      createdAt: new Date(),
    }
    await Storage.messages.insert(message)
    return message
  }
  
  // 取得 session 的所有訊息
  export async function list(input: {
    sessionID: string;
  }): Promise<MessageData[]> {
    return Storage.messages.find({
      where: { sessionID: input.sessionID },
      orderBy: { createdAt: "asc" },
    })
  }
  
  // 轉換成 AI SDK 格式
  export function toAIMessages(messages: MessageData[]): CoreMessage[] {
    return messages.map(msg => ({
      role: msg.role,
      content: msg.content,
      ...(msg.toolCalls && { toolCalls: msg.toolCalls }),
      ...(msg.toolCallId && { toolCallId: msg.toolCallId }),
    }))
  }
}
```

### 3. Session Loop 完整實現 (`session/prompt.ts`)

這是 OpenCode 的心臟，控制整個 Agent 執行流程：

```typescript
// packages/opencode/src/session/prompt.ts (完整版)
export namespace SessionPrompt {
  
  export interface LoopInput {
    sessionID: string;
    signal?: AbortSignal;      // 取消信號
    maxSteps?: number;         // 最大步數限制
  }
  
  export interface LoopOutput {
    type: "part" | "complete" | "error" | "permission";
    part?: StreamPart;
    error?: Error;
    permission?: PermissionRequest;
  }
  
  // 🔑 主要執行 Loop
  export async function* loop(input: LoopInput): AsyncGenerator<LoopOutput> {
    const { sessionID, signal, maxSteps = 50 } = input
    
    let stepCount = 0
    let shouldContinue = true
    
    while (shouldContinue && stepCount < maxSteps) {
      // 檢查取消信號
      if (signal?.aborted) {
        yield { type: "error", error: new Error("Aborted") }
        return
      }
      
      stepCount++
      
      // ═══════════════════════════════════════════════════════════
      // Step 1: 載入對話歷史和設定
      // ═══════════════════════════════════════════════════════════
      const session = await Session.get(sessionID)
      if (!session) throw new Error("Session not found")
      
      const messages = await Message.list({ sessionID })
      const agent = Agent.get(session.agent)
      
      // ═══════════════════════════════════════════════════════════
      // Step 2: 檢查是否需要 Compaction
      // ═══════════════════════════════════════════════════════════
      const model = await Provider.getModelInfo(session.agent)
      if (SessionCompaction.isOverflow({ messages, model })) {
        yield { type: "part", part: { type: "status", status: "compacting" } }
        
        const compactedMessages = await SessionCompaction.compact({
          sessionID,
          messages,
        })
        
        // 更新資料庫中的訊息
        await Message.replaceAll(sessionID, compactedMessages)
        messages.length = 0
        messages.push(...compactedMessages)
      }
      
      // ═══════════════════════════════════════════════════════════
      // Step 3: 準備工具
      // ═══════════════════════════════════════════════════════════
      const tools = await resolveTools(agent, sessionID)
      const aiTools = ToolRegistry.toAITools(Object.keys(tools))
      
      // ═══════════════════════════════════════════════════════════
      // Step 4: 建構 System Prompt
      // ═══════════════════════════════════════════════════════════
      const systemPrompt = await System.build({
        agent,
        cwd: process.cwd(),
        env: process.env,
      })
      
      // ═══════════════════════════════════════════════════════════
      // Step 5: 呼叫 LLM
      // ═══════════════════════════════════════════════════════════
      const llmStream = LLM.stream({
        model: await Provider.getLanguage(agent.model),
        messages: Message.toAIMessages(messages),
        tools: aiTools,
        system: systemPrompt,
        providerOptions: Provider.getOptions(agent),
      })
      
      // ═══════════════════════════════════════════════════════════
      // Step 6: 處理 LLM 回應
      // ═══════════════════════════════════════════════════════════
      const pendingToolCalls: ToolCall[] = []
      let assistantContent = ""
      
      for await (const event of llmStream) {
        switch (event.type) {
          case "text-delta":
            assistantContent += event.textDelta
            yield { type: "part", part: { type: "text", text: event.textDelta } }
            break
            
          case "reasoning-delta":
            yield { type: "part", part: { type: "reasoning", text: event.textDelta } }
            break
            
          case "tool-call":
            pendingToolCalls.push({
              id: event.toolCallId,
              name: event.toolName,
              args: event.args,
            })
            yield { 
              type: "part", 
              part: { type: "tool-call", name: event.toolName, args: event.args }
            }
            break
        }
      }
      
      // 儲存 assistant 訊息
      if (assistantContent || pendingToolCalls.length > 0) {
        await Message.create({
          sessionID,
          role: "assistant",
          content: assistantContent,
          toolCalls: pendingToolCalls,
        })
      }
      
      // ═══════════════════════════════════════════════════════════
      // Step 7: 執行工具呼叫
      // ═══════════════════════════════════════════════════════════
      if (pendingToolCalls.length === 0) {
        // 沒有工具呼叫，對話結束
        shouldContinue = false
        yield { type: "complete" }
        break
      }
      
      for (const toolCall of pendingToolCalls) {
        // Doom Loop 檢測
        if (SessionProcessor.detectDoomLoop(sessionID, toolCall)) {
          yield {
            type: "permission",
            permission: {
              type: "doom-loop",
              tool: toolCall.name,
              message: "偵測到可能的無限迴圈，是否繼續？",
            }
          }
          // 等待用戶確認...
        }
        
        // 取得工具
        const tool = tools[toolCall.name]
        if (!tool) {
          await Message.create({
            sessionID,
            role: "tool",
            content: `Error: Tool "${toolCall.name}" not found`,
            toolCallId: toolCall.id,
          })
          continue
        }
        
        // 權限檢查
        const permission = await PermissionNext.check({
          tool: toolCall.name,
          patterns: extractPatterns(toolCall.args),
          ruleset: agent.permission,
          sessionID,
        })
        
        if (permission === "deny") {
          await Message.create({
            sessionID,
            role: "tool",
            content: `Permission denied for ${toolCall.name}`,
            toolCallId: toolCall.id,
          })
          continue
        }
        
        if (permission === "ask") {
          yield {
            type: "permission",
            permission: {
              type: "tool",
              tool: toolCall.name,
              patterns: extractPatterns(toolCall.args),
            }
          }
          // 等待用戶回應...
        }
        
        // 執行工具
        try {
          yield { type: "part", part: { type: "tool-start", name: toolCall.name } }
          
          const result = await tool.execute(toolCall.args, {
            sessionID,
            messageID: "", // 會在執行時設定
            agent: agent.name,
            abort: signal ?? new AbortController().signal,
            callID: toolCall.id,
            metadata: (update) => {
              yield { type: "part", part: { type: "tool-update", ...update } }
            },
            ask: async (req) => {
              // 工具內部的權限請求
            },
          })
          
          // 儲存工具結果
          await Message.create({
            sessionID,
            role: "tool",
            content: result.output,
            toolCallId: toolCall.id,
          })
          
          yield { 
            type: "part", 
            part: { type: "tool-result", name: toolCall.name, result } 
          }
          
        } catch (error) {
          await Message.create({
            sessionID,
            role: "tool",
            content: `Error: ${error.message}`,
            toolCallId: toolCall.id,
          })
          
          yield { 
            type: "part", 
            part: { type: "tool-error", name: toolCall.name, error } 
          }
        }
      }
      
      // 繼續下一輪 Loop
    }
    
    // 達到最大步數
    if (stepCount >= maxSteps) {
      yield { 
        type: "error", 
        error: new Error(`Reached maximum steps limit (${maxSteps})`) 
      }
    }
  }
}
```

### 4. System Prompt 組合 (`session/system.ts`)

```typescript
// packages/opencode/src/session/system.ts
export namespace System {
  
  export async function build(input: {
    agent: AgentInfo;
    cwd: string;
    env: NodeJS.ProcessEnv;
  }): Promise<string> {
    const { agent, cwd, env } = input
    
    const parts: string[] = []
    
    // 基本身份
    parts.push(`You are OpenCode, an AI coding assistant.`)
    
    // 環境資訊
    parts.push(`
## Environment
- Working Directory: ${cwd}
- Operating System: ${process.platform}
- Shell: ${env.SHELL ?? "unknown"}
- Node Version: ${process.version}
- Current Time: ${new Date().toISOString()}
`)
    
    // Agent 特定指示
    if (agent.prompt) {
      parts.push(`## Instructions\n${agent.prompt}`)
    }
    
    // 工具使用指南
    parts.push(`
## Tool Usage Guidelines
1. Always read files before modifying them
2. Use grep to search for patterns across files
3. Use glob to discover file structure
4. Make small, focused changes
5. Run tests after modifications
6. Use task to delegate complex subtasks
`)
    
    // 安全提醒
    parts.push(`
## Safety Guidelines
- Never execute destructive commands without user confirmation
- Be careful with rm, sudo, and other dangerous operations
- Always verify file paths before writing
`)
    
    return parts.join("\n\n")
  }
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

### 完整內建工具清單

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        OpenCode 完整工具清單                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  📁 檔案操作工具                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ read      │ 讀取檔案內容，支援分段讀取 (offset/limit)            │   │
│  │ write     │ 建立新檔案或完全覆寫現有檔案                         │   │
│  │ edit      │ 精確替換檔案中的特定內容 (oldString → newString)     │   │
│  │ multiedit │ 批次編輯，一次替換多處內容                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  🔍 搜尋工具                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ grep       │ 文字/正則搜尋，支援多檔案搜尋                       │   │
│  │ glob       │ 列出符合 pattern 的檔案 (如 **/*.ts)                │   │
│  │ codesearch │ 語意化程式碼搜尋，找到相關的類別/函數              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  🖥️ 系統工具                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ bash      │ 執行 shell 命令，支援 timeout 和 cwd                 │   │
│  │ lsp       │ 呼叫 Language Server Protocol 取得型別資訊           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  🤖 Agent 工具                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ task      │ 派遣子代理執行特定任務                               │   │
│  │ skill     │ 載入預定義的技能腳本                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  🌐 網路工具                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ websearch │ 網頁搜尋 (需要設定 API key)                          │   │
│  │ webfetch  │ 抓取網頁內容並轉換成 Markdown                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  📋 其他工具                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ question  │ 向用戶提問並等待回應                                 │   │
│  │ todoread  │ 讀取專案的 TODO 列表                                 │   │
│  │ todowrite │ 更新專案的 TODO 列表                                 │   │
│  │ memory    │ 讀寫長期記憶 (跨 session 保存)                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 更多工具實作範例

#### 5. Grep Tool (文字搜尋)

```typescript
// packages/opencode/src/tool/grep.ts
Tool.define("grep", {
  description: `
    在檔案中搜尋文字或正則表達式。
    可以搜尋單一檔案或整個目錄。
    返回匹配的行和上下文。
  `,
  parameters: z.object({
    pattern: z.string().describe("搜尋模式 (文字或正則)"),
    path: z.string().describe("檔案或目錄路徑"),
    isRegex: z.boolean().optional().default(false),
    caseSensitive: z.boolean().optional().default(true),
    context: z.number().optional().default(2).describe("顯示匹配行前後的行數"),
    maxResults: z.number().optional().default(50),
  }),
  async execute(args, ctx) {
    const { pattern, path, isRegex, caseSensitive, context, maxResults } = args
    
    // 建立正則表達式
    const flags = caseSensitive ? "g" : "gi"
    const regex = isRegex ? new RegExp(pattern, flags) : new RegExp(escapeRegex(pattern), flags)
    
    // 收集所有要搜尋的檔案
    const files = await collectFiles(path)
    const results: SearchResult[] = []
    
    for (const file of files) {
      const content = await Bun.file(file).text()
      const lines = content.split("\n")
      
      for (let i = 0; i < lines.length; i++) {
        if (regex.test(lines[i])) {
          // 取得上下文行
          const startLine = Math.max(0, i - context)
          const endLine = Math.min(lines.length - 1, i + context)
          
          results.push({
            file,
            line: i + 1,
            content: lines[i],
            context: lines.slice(startLine, endLine + 1).join("\n"),
          })
          
          if (results.length >= maxResults) break
        }
      }
      
      if (results.length >= maxResults) break
    }
    
    // 格式化輸出
    const output = results.map(r => 
      `${r.file}:${r.line}\n${r.context}\n`
    ).join("\n---\n")
    
    return {
      title: `Grep: ${pattern}`,
      output: output || "No matches found",
      metadata: { matches: results.length, pattern }
    }
  }
})
```

#### 6. Glob Tool (檔案列表)

```typescript
// packages/opencode/src/tool/glob.ts
import { Glob } from "bun"

Tool.define("glob", {
  description: `
    列出符合 pattern 的檔案。
    使用標準 glob 語法：
    - * 匹配任意字元
    - ** 匹配任意層級目錄
    - ? 匹配單一字元
    - [abc] 匹配括號內的字元
  `,
  parameters: z.object({
    pattern: z.string().describe("Glob pattern (如 **/*.ts)"),
    cwd: z.string().optional().describe("起始目錄"),
    ignore: z.array(z.string()).optional().describe("要忽略的 patterns"),
  }),
  async execute(args, ctx) {
    const { pattern, cwd = process.cwd(), ignore = [] } = args
    
    // 預設忽略
    const defaultIgnore = [
      "node_modules/**",
      ".git/**",
      "dist/**",
      "build/**",
      "*.min.js",
    ]
    
    const allIgnore = [...defaultIgnore, ...ignore]
    
    // 使用 Bun 的 Glob
    const glob = new Glob(pattern)
    const files: string[] = []
    
    for await (const file of glob.scan({ cwd, absolute: true })) {
      // 檢查是否應該忽略
      const shouldIgnore = allIgnore.some(ig => {
        const ignoreGlob = new Glob(ig)
        return ignoreGlob.match(file)
      })
      
      if (!shouldIgnore) {
        files.push(file)
      }
    }
    
    // 排序並格式化
    files.sort()
    
    // 生成樹狀結構
    const tree = buildFileTree(files, cwd)
    
    return {
      title: `Glob: ${pattern}`,
      output: tree,
      metadata: { count: files.length, pattern }
    }
  }
})

// 輔助函數：生成檔案樹
function buildFileTree(files: string[], root: string): string {
  const lines: string[] = []
  const dirs = new Map<string, string[]>()
  
  for (const file of files) {
    const relative = file.replace(root + "/", "")
    const parts = relative.split("/")
    const dir = parts.slice(0, -1).join("/") || "."
    const name = parts[parts.length - 1]
    
    if (!dirs.has(dir)) dirs.set(dir, [])
    dirs.get(dir)!.push(name)
  }
  
  for (const [dir, files] of [...dirs].sort()) {
    lines.push(`📁 ${dir}/`)
    for (const file of files.sort()) {
      lines.push(`   └── ${file}`)
    }
  }
  
  return lines.join("\n")
}
```

#### 7. Write Tool (寫入檔案)

```typescript
// packages/opencode/src/tool/write.ts
Tool.define("write", {
  description: `
    建立新檔案或完全覆寫現有檔案。
    如果目錄不存在會自動建立。
    注意：這會完全覆寫檔案，如果只想修改部分內容請使用 edit。
  `,
  parameters: z.object({
    filePath: z.string().describe("檔案路徑"),
    content: z.string().describe("檔案內容"),
  }),
  async execute(args, ctx) {
    const { filePath, content } = args
    
    // 請求權限
    await ctx.ask({ permission: "write", patterns: [filePath] })
    
    // 檢查檔案是否已存在
    const exists = await Bun.file(filePath).exists()
    
    // 確保目錄存在
    const dir = path.dirname(filePath)
    await fs.promises.mkdir(dir, { recursive: true })
    
    // 寫入檔案
    await Bun.write(filePath, content)
    
    // 統計資訊
    const lines = content.split("\n").length
    const bytes = Buffer.byteLength(content, "utf8")
    
    return {
      title: exists ? `Overwrite ${filePath}` : `Create ${filePath}`,
      output: `${exists ? "Overwrote" : "Created"} ${filePath}\n` +
              `Lines: ${lines}, Bytes: ${bytes}`,
      metadata: { path: filePath, lines, bytes, created: !exists }
    }
  }
})
```

#### 8. MultiEdit Tool (批次編輯)

```typescript
// packages/opencode/src/tool/multiedit.ts
Tool.define("multiedit", {
  description: `
    批次編輯檔案，一次執行多個替換操作。
    每個替換都必須精確匹配。
    適合需要在多處進行相關修改的情況。
  `,
  parameters: z.object({
    filePath: z.string().describe("檔案路徑"),
    edits: z.array(z.object({
      oldString: z.string(),
      newString: z.string(),
    })).describe("替換操作列表"),
  }),
  async execute(args, ctx) {
    const { filePath, edits } = args
    
    await ctx.ask({ permission: "edit", patterns: [filePath] })
    
    let content = await Bun.file(filePath).text()
    const results: { success: boolean; old: string; error?: string }[] = []
    
    for (const edit of edits) {
      const matches = content.split(edit.oldString).length - 1
      
      if (matches === 0) {
        results.push({ 
          success: false, 
          old: edit.oldString.slice(0, 50),
          error: "Not found" 
        })
        continue
      }
      
      if (matches > 1) {
        results.push({ 
          success: false, 
          old: edit.oldString.slice(0, 50),
          error: `Multiple matches (${matches})` 
        })
        continue
      }
      
      content = content.replace(edit.oldString, edit.newString)
      results.push({ success: true, old: edit.oldString.slice(0, 50) })
    }
    
    // 只有全部成功才寫入
    const allSuccess = results.every(r => r.success)
    if (allSuccess) {
      await Bun.write(filePath, content)
    }
    
    // 輸出結果
    const output = results.map((r, i) => 
      `${i + 1}. ${r.success ? "✅" : "❌"} ${r.old}... ${r.error ?? ""}`
    ).join("\n")
    
    return {
      title: `MultiEdit ${filePath}`,
      output: allSuccess 
        ? `Successfully applied ${edits.length} edits\n${output}`
        : `Failed - no changes made\n${output}`,
      metadata: { 
        success: allSuccess, 
        total: edits.length,
        successful: results.filter(r => r.success).length 
      }
    }
  }
})
```

### Tool Context 完整介面

```typescript
// packages/opencode/src/tool/tool.ts
interface ToolContext {
  // 識別資訊
  sessionID: string;           // 所屬 session
  messageID: string;           // 所屬訊息
  agent: string;               // 執行的 agent
  callID?: string;             // 工具呼叫 ID
  
  // 控制
  abort: AbortSignal;          // 取消信號
  
  // 更新執行狀態 (即時回報給前端)
  metadata(input: {
    title?: string;            // 更新標題
    status?: string;           // 狀態描述
    progress?: number;         // 進度 (0-100)
    metadata?: any;            // 額外資訊
  }): void;
  
  // 請求權限
  ask(input: {
    permission: string;        // 權限類型
    patterns: string[];        // 相關 patterns (檔案路徑等)
    message?: string;          // 給用戶的說明
  }): Promise<void>;
  
  // 讀取設定
  config<T>(key: string): T | undefined;
  
  // 發送事件給前端
  emit(event: ToolEvent): void;
}

// 工具事件類型
type ToolEvent = 
  | { type: "progress"; value: number }
  | { type: "log"; message: string }
  | { type: "file-change"; path: string; action: "create" | "modify" | "delete" }
  | { type: "command-output"; stdout: string; stderr: string }
```
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

### 1. 🎯 精細權限控制系統

OpenCode 的權限系統是其最大亮點之一，支援工具級和 Pattern 級的細粒度控制：

```typescript
// 範例：不同 Agent 的權限配置

// Build Agent - 完整權限但需確認
const buildPermission = {
  "*": "ask",              // 預設詢問
  read: "allow",           // 讀取自動允許
  grep: "allow",
  glob: "allow",
  bash: {
    "*": "ask",
    "npm test": "allow",   // 測試命令自動允許
    "npm run *": "allow",
    "git status": "allow",
    "rm -rf *": "deny",    // 危險命令永遠拒絕
  },
  edit: {
    "*": "ask",
    "*.test.ts": "allow",  // 測試檔案自動允許
    "package.json": "ask", // 敏感檔案需確認
  }
}

// Plan Agent - 嚴格只讀
const planPermission = {
  "*": "deny",
  read: "allow",
  grep: "allow",
  glob: "allow",
  edit: {
    "*": "deny",
    ".opencode/plans/*.md": "allow"  // 只能編輯計畫檔
  }
}

// Explore Agent - 最小權限
const explorePermission = {
  "*": "deny",
  read: "allow",
  grep: "allow",
  glob: "allow",
  codesearch: "allow",
}
```

### 2. 🔄 非同步串流處理架構

OpenCode 使用 Generator 函數實現優雅的串流處理：

```typescript
// 串流處理的核心模式
export async function* processStream(input: Input): AsyncGenerator<Output> {
  // 1. 初始化
  yield { type: "start", timestamp: Date.now() }
  
  // 2. 處理 LLM 串流
  for await (const event of llmStream) {
    switch (event.type) {
      case "text-delta":
        // 即時顯示，低延遲
        yield { type: "text", content: event.textDelta }
        break
        
      case "reasoning-delta":
        // Claude 的思考過程
        yield { type: "reasoning", content: event.textDelta }
        break
        
      case "tool-call":
        // 工具呼叫開始
        yield { type: "tool-start", name: event.toolName }
        
        // 執行工具（可能是另一個 generator）
        for await (const progress of executeTool(event)) {
          yield { type: "tool-progress", ...progress }
        }
        
        yield { type: "tool-end", name: event.toolName }
        break
    }
  }
  
  // 3. 完成
  yield { type: "complete", timestamp: Date.now() }
}

// 使用方式 - 前端可以即時消費
for await (const event of processStream(input)) {
  ui.update(event)  // 即時更新 UI
}
```

### 3. 🧩 Plugin 系統設計

OpenCode 支援用戶自訂工具，放在專案的 `tools/` 目錄：

```typescript
// tools/deploy.ts - 自訂部署工具
import { z } from "zod"

export default {
  name: "deploy",
  description: "部署應用到指定環境",
  
  parameters: z.object({
    environment: z.enum(["staging", "production"]),
    version: z.string().optional(),
  }),
  
  // 權限提示
  permission: {
    type: "ask",
    message: "此操作將部署到 {environment} 環境，是否繼續？"
  },
  
  async execute(args, ctx) {
    const { environment, version } = args
    
    // 更新執行狀態
    ctx.metadata({ status: "Building..." })
    
    // 執行部署命令
    const buildResult = await ctx.runCommand("npm run build")
    
    ctx.metadata({ status: "Deploying...", progress: 50 })
    
    const deployResult = await ctx.runCommand(
      `deploy --env ${environment} ${version ? `--version ${version}` : ""}`
    )
    
    return {
      title: `Deploy to ${environment}`,
      output: `Successfully deployed!\n${deployResult.stdout}`,
      metadata: { environment, version }
    }
  }
}
```

**載入流程：**

```typescript
// packages/opencode/src/tool/plugin.ts
export namespace Plugin {
  export async function loadUserTools(projectDir: string) {
    const toolsDir = path.join(projectDir, "tools")
    if (!await fs.exists(toolsDir)) return []
    
    const files = await glob("*.ts", { cwd: toolsDir })
    const tools: ToolDefinition[] = []
    
    for (const file of files) {
      const module = await import(path.join(toolsDir, file))
      const tool = module.default
      
      // 驗證工具定義
      if (tool && tool.name && tool.execute) {
        tools.push(tool)
        ToolRegistry.register(tool)
      }
    }
    
    return tools
  }
}
```

### 4. 🔌 MCP 整合的完整實現

```typescript
// packages/opencode/src/mcp/client.ts
export namespace MCPClient {
  const connections: Map<string, Client> = new Map()
  
  // 連接到 MCP Server
  export async function connect(config: MCPServerConfig): Promise<void> {
    const { name, command, args, env } = config
    
    // 建立傳輸層
    const transport = new StdioClientTransport({
      command,
      args,
      env: { ...process.env, ...resolveEnv(env) }
    })
    
    // 建立客戶端
    const client = new Client({
      name: `opencode-${name}`,
      version: "1.0.0"
    }, {
      capabilities: {
        tools: {},
        resources: {},
        prompts: {},
      }
    })
    
    // 連接
    await client.connect(transport)
    connections.set(name, client)
    
    // 監聽工具列表變化
    client.onToolListChanged(async () => {
      await refreshTools(name)
    })
  }
  
  // 取得所有 MCP 工具
  export async function getAllTools(): Promise<MCPTool[]> {
    const allTools: MCPTool[] = []
    
    for (const [serverName, client] of connections) {
      const { tools } = await client.listTools()
      
      for (const tool of tools) {
        allTools.push({
          server: serverName,
          name: tool.name,
          description: tool.description,
          inputSchema: tool.inputSchema,
        })
      }
    }
    
    return allTools
  }
  
  // 呼叫 MCP 工具
  export async function callTool(
    serverName: string,
    toolName: string,
    args: Record<string, unknown>
  ): Promise<MCPToolResult> {
    const client = connections.get(serverName)
    if (!client) throw new Error(`MCP server "${serverName}" not connected`)
    
    const result = await client.callTool({
      name: toolName,
      arguments: args
    })
    
    // 處理不同類型的回應
    if (result.isError) {
      throw new Error(result.content[0]?.text ?? "Unknown error")
    }
    
    return {
      content: result.content.map(c => {
        if (c.type === "text") return c.text
        if (c.type === "image") return `[Image: ${c.mimeType}]`
        return JSON.stringify(c)
      }).join("\n"),
      metadata: result.metadata
    }
  }
}
```

### 5. 🗜️ 智能 Compaction 策略

```typescript
// packages/opencode/src/session/compaction.ts
export namespace SessionCompaction {
  
  // 多層級壓縮策略
  export async function smartCompact(input: {
    messages: Message[];
    model: ModelInfo;
    preserveRecent: number;  // 保留最近幾輪
  }): Promise<Message[]> {
    const { messages, model, preserveRecent } = input
    const totalTokens = estimateTokens(messages)
    const limit = model.contextWindow ?? 128000
    
    // 層級 1: 截斷長工具輸出
    if (totalTokens > limit * 0.6) {
      messages = pruneToolOutputs(messages, {
        maxLength: 5000,
        keepStructure: true,  // 保留 JSON 結構
      })
    }
    
    // 層級 2: 移除舊的工具呼叫細節
    if (totalTokens > limit * 0.7) {
      messages = collapseOldToolCalls(messages, {
        keepRecent: preserveRecent * 2,
        summarize: true,
      })
    }
    
    // 層級 3: AI 摘要
    if (totalTokens > limit * 0.8) {
      const oldMessages = messages.slice(0, -preserveRecent)
      const recentMessages = messages.slice(-preserveRecent)
      
      const summary = await summarizeWithAI({
        messages: oldMessages,
        instruction: `
          摘要這段對話，保留：
          1. 用戶的原始目標
          2. 已完成的重要步驟
          3. 遇到的問題和解決方案
          4. 當前狀態和待辦事項
          5. 重要的檔案路徑和程式碼
        `
      })
      
      return [
        { role: "user", content: `[對話摘要]\n${summary}` },
        ...recentMessages
      ]
    }
    
    return messages
  }
}
```

### 6. 🛡️ 錯誤恢復機制

```typescript
// packages/opencode/src/session/recovery.ts
export namespace SessionRecovery {
  
  // 工具執行失敗時的恢復策略
  export async function handleToolError(
    error: Error,
    toolCall: ToolCall,
    ctx: ToolContext
  ): Promise<RecoveryAction> {
    
    // 1. 可重試的錯誤
    if (isRetryable(error)) {
      return {
        action: "retry",
        delay: calculateBackoff(ctx.retryCount),
        maxRetries: 3
      }
    }
    
    // 2. 權限錯誤 - 請求用戶確認
    if (error instanceof PermissionDeniedError) {
      return {
        action: "ask-permission",
        tool: toolCall.name,
        patterns: error.patterns
      }
    }
    
    // 3. 檔案不存在 - 提供建議
    if (error.code === "ENOENT") {
      const suggestions = await findSimilarFiles(error.path)
      return {
        action: "suggest",
        message: `File not found: ${error.path}`,
        suggestions
      }
    }
    
    // 4. 無法恢復 - 回報給 AI
    return {
      action: "report",
      error: error.message,
      context: `Tool "${toolCall.name}" failed`
    }
  }
}
```

---

## 結論

### OpenCode Agent 架構的設計哲學

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                     OpenCode 設計哲學                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  🔐 安全優先 (Security First)                                           │
│  ├── 精細的權限系統防止意外操作                                         │
│  ├── Doom Loop 檢測防止無限迴圈                                         │
│  ├── 危險命令需要明確確認                                               │
│  └── Session 級別的權限快取                                             │
│                                                                         │
│  🧩 可擴展性 (Extensibility)                                            │
│  ├── Plugin 系統支援自訂工具                                            │
│  ├── MCP 協議連接外部服務                                               │
│  ├── 自訂 Agent 配置                                                    │
│  └── Provider 抽象支援多 LLM                                            │
│                                                                         │
│  🤖 智能分工 (Smart Delegation)                                         │
│  ├── Subagent 機制處理複雜任務                                          │
│  ├── Explore Agent 快速了解 codebase                                    │
│  ├── General Agent 深入研究問題                                         │
│  └── 並行執行多個子任務                                                 │
│                                                                         │
│  📊 資源管理 (Resource Management)                                      │
│  ├── Token 使用量追蹤                                                   │
│  ├── 智能 Compaction 壓縮對話                                           │
│  ├── 串流處理減少記憶體使用                                             │
│  └── 費用追蹤和統計                                                     │
│                                                                         │
│  🔄 容錯能力 (Fault Tolerance)                                          │
│  ├── 工具失敗自動重試                                                   │
│  ├── 優雅的錯誤恢復                                                     │
│  ├── Session 狀態持久化                                                 │
│  └── 中斷後可繼續執行                                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 與其他工具比較

| 特性 | OpenCode | Claude Code | Cursor | Aider |
| ------ | ---------- | ------------- | -------- | ------- |
| 開源 | ✅ | ❌ | ❌ | ✅ |
| Agent 分工 | ✅ 多層級 | ✅ 單層 | ❌ | ❌ |
| 自訂工具 | ✅ Plugin | ❌ | ❌ | ❌ |
| MCP 支援 | ✅ | ✅ | ❌ | ❌ |
| 權限控制 | ✅ 精細 | ✅ 基本 | ❌ | ✅ 基本 |
| 多 Provider | ✅ 20+ | ❌ Claude only | ✅ | ✅ |
| 本地模型 | ✅ Ollama | ❌ | ❌ | ✅ |
| TUI 介面 | ✅ | ✅ | ❌ GUI | ✅ |
| Web UI | ✅ | ❌ | ✅ | ❌ |
| 上下文壓縮 | ✅ 智能 | ✅ | ✅ | ✅ |

### 程式碼統計

```text
packages/opencode/src/
├── agent/         ~200 行    # Agent 定義
├── session/       ~1500 行   # Session 管理 (核心)
├── tool/          ~2000 行   # 工具系統
├── permission/    ~400 行    # 權限控制
├── provider/      ~800 行    # AI Provider
├── mcp/           ~500 行    # MCP 整合
├── config/        ~300 行    # 設定管理
├── storage/       ~400 行    # 資料持久化
└── app/           ~600 行    # 應用入口

總計: ~6700+ 行 TypeScript
```

### 學習價值

OpenCode 是學習現代 AI Agent 架構的絕佳範例：

**1. 架構設計模式**
- Namespace 模式的函數式設計
- Generator 串流處理
- 插件化架構

**2. AI 整合技術**
- Vercel AI SDK 的使用
- 多 Provider 抽象
- Function Calling 實現

**3. 工程實踐**
- TypeScript + Zod 型別安全
- SQLite 持久化
- 錯誤處理和恢復

**4. 安全設計**
- 權限系統設計
- 輸入驗證
- 危險操作防護

### 延伸閱讀建議

1. **深入 Vercel AI SDK**: 了解更多串流處理和工具呼叫的細節
2. **MCP 規範**: 學習如何開發自己的 MCP Server
3. **Agent 設計模式**: 研究 ReAct、CoT 等 Agent 架構
4. **LLM 應用安全**: 學習 prompt injection 防護

---

## 參考資源

### 官方資源
- [OpenCode GitHub](https://github.com/sst/opencode)
- [OpenCode 文件](https://opencode.ai/docs)
- [Vercel AI SDK](https://sdk.vercel.ai/)
- [Model Context Protocol](https://modelcontextprotocol.io/)

### 相關閱讀
- [Building AI Agents with TypeScript](https://sdk.vercel.ai/docs/ai-sdk-core)
- [MCP Server 開發指南](https://modelcontextprotocol.io/docs/server/building)
- [Claude Function Calling](https://docs.anthropic.com/claude/docs/function-calling)

### 社群
- [OpenCode Discord](https://discord.gg/opencode)
- [SST Discord](https://discord.gg/sst)

---

**本分析由 u9401066 於 2026-01-15 完成**

文件版本: v2.0 (深度擴充版)
總行數: 2500+ 行
涵蓋主題: 專案結構、Vercel AI SDK、Agent 系統、Session Loop、工具系統、權限控制、MCP 整合、Token 管理
