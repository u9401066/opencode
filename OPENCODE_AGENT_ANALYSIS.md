# 🤖 OpenCode Agent 架構研究分析

> **作者**: u9401066  
> **日期**: 2026-01-15  
> **來源**: [sst/opencode](https://github.com/sst/opencode)

---

## 📋 目錄

1. [專案概述](#專案概述)
2. [Agent 架構設計](#agent-架構設計)
3. [核心元件分析](#核心元件分析)
4. [工具系統](#工具系統)
5. [Meme 圖解](#meme-圖解)
6. [技術亮點](#技術亮點)
7. [結論](#結論)

---

## 專案概述

OpenCode 是一個開源的 AI 編程助手，類似 Claude Code / Cursor，採用 **Agent Loop** 架構實現自主編程能力。

### 技術棧
- **Runtime**: Bun + TypeScript
- **AI SDK**: Vercel AI SDK
- **Schema**: Zod validation
- **架構模式**: Namespace-based modules

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

## 工具系統

### 完整工具清單

```
📁 檔案操作          🔍 搜尋工具          🌐 網路工具
├── read            ├── grep            ├── websearch
├── write           ├── glob            └── webfetch
├── edit            └── codesearch
└── multiedit

🖥️ 系統工具          🤖 Agent 工具        📋 其他
├── bash            ├── task            ├── question
└── lsp             └── skill           ├── todoread
                                        └── todowrite
```

### Tool Context 介面

```typescript
interface ToolContext {
  sessionID: string;
  messageID: string;
  agent: string;
  abort: AbortSignal;
  callID?: string;
  
  // 更新執行狀態
  metadata(input: { title?: string; metadata?: any }): void;
  
  // 請求權限
  ask(input: { permission: string; patterns: string[] }): Promise<void>;
}
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
