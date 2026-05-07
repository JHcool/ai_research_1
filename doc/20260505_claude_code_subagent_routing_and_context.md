# Claude Code Subagent 路由機制、執行拓撲與 Context 提取

> 研究日期：2026-05-05  
> 來源：系統提示詞洩漏分析 + 當前可觀察的 Claude Code system prompt + Anthropic 官方文件

---

## Q1：Claude Code 如何決定調用 Subagent？

### 結論

**純 LLM 語義判斷，無關鍵字觸發，無硬編碼路由。**

### 機制

Claude Code 的 routing 分兩層，都是 LLM 讀取自然語言做語義比對：

**層一：Agent 工具的 description**  
Agent tool 的 description 欄位直接說明「何時使用哪種 subagent」。LLM 讀取當前任務，對照 description，判斷是否符合。這是**概率性**的，不是確定性的——官方文件和洩漏分析都確認 auto-invoke 不可靠，Claude 常自行處理而不觸發 subagent。

**層二：Skill 的 `TRIGGER when:` 條件**  
例如 `claude-api` skill 的條件：
```
TRIGGER when: code imports `anthropic`/`@anthropic-ai/sdk`; user asks for the Claude API...
SKIP: file imports `openai`/other-provider SDK
```
這些文字**被載入 LLM context**，LLM 語義判斷是否符合——是概率性的，不是 regex 或 pattern matching。

**什麼是硬編碼的**：只有工具的參數 enum（`subagent_type` 只接受特定值）和工具可用性 gating。路由決策本身全是 LLM。

> 最可靠的觸發方式：用戶明確輸入 `/skill-name` 或在 prompt 裡明確提到 agent 名稱。Auto-invoke 準確率不穩定。

### 參考來源
- [Claude Code system prompt 洩漏 (asgeirtj)](https://github.com/asgeirtj/system_prompts_leaks/blob/main/Anthropic/claude-code.md)
- [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)
- [How Claude Code Builds a System Prompt — dbreunig](https://www.dbreunig.com/2026/04/04/how-claude-code-builds-a-system-prompt.html)
- [Claude Code Source Leak Analysis — Alex Kim](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/)
- [Tracing Claude Code's LLM Traffic — George Sung](https://medium.com/@georgesung/tracing-claude-codes-llm-traffic-agentic-loop-sub-agents-tool-use-prompts-7796941806f5)

---

## Q2：執行拓撲 vs. Claude Code 的分派手段

Claude Code 提供的執行手段不只有「spawn subagent」：

| 執行手段 | 怎麼用 | 對應拓撲 | 特性 |
|---|---|---|---|
| `Agent(foreground)` | 單次 Agent call，等待結果 | **Sequential / Pipeline** | 阻塞，結果影響下一步 |
| 同一訊息多個 `Agent` call | 一次發多個 Agent tool call | **Fan-out / Concurrent** | 真並行，互相獨立 |
| `Agent(run_in_background=true)` | 加 background 旗標 | **Async 背景** | 非阻塞，完成時通知 |
| `SendMessage(to: agentId)` | 向現有 agent 繼續對話 | **Handoff / Stateful** | 有記憶的接力，同一 agent 延續 |
| `Agent(isolation: "worktree")` | 獨立 git worktree | **隔離並行** | 程式碼變更互不污染，完成後回報 branch |
| `CronCreate` | 建立 cron 排程 | **Scheduled / Recurring** | 定時觸發遠端 agent |
| `Bash(run_in_background=true)` | shell 背景執行 | **Process 層並行** | 非 agent，但可與 agent 協調 |
| `TaskCreate / TaskUpdate` | 任務追蹤原語 | **協調層（非執行）** | 跨 agent 共享進度狀態 |

### 拓撲選擇決策樹

```
任務是否需要等待結果才能繼續？
├── 是 → foreground Agent（Sequential/Pipeline）
│         → 多個 foreground 可鏈式接力（Handoff）
│
└── 否 → 子任務彼此獨立？
          ├── 是 → 同一訊息多個 Agent（Fan-out）
          │         run_in_background=true（Async Fan-out）
          │
          └── 否（有依賴但不需立即結果）
                → background Agent + TaskCreate 追蹤
                  worktree 隔離（程式碼修改場景）
```

---

## Q3：Claude Code 如何決定給 Subagent 的 Context？

### 核心原則（直接引用 Claude Code system prompt）

> *"Brief the agent like a smart colleague who just walked into the room — it hasn't seen this conversation, doesn't know what you've tried, doesn't understand why this task matters."*

**反模式（Never delegate understanding）**：
> 不要寫「based on your findings, fix the bug」。這是把理解推給 agent。應寫出包含 file path、line number、具體要改什麼的 prompt，證明 orchestrator 已理解。

### 「必須知道的事」的 5 個維度

| 維度 | 內容 | 為什麼必須給 |
|---|---|---|
| **1. 具體目標** | 這個 subagent 的任務是什麼 | 沒有目標，agent 漂移 |
| **2. 任務邊界** | 明確「不」調查什麼 | 防止與其他 agent 重複（AOP 非冗餘性原則） |
| **3. 已排除的路徑** | Orchestrator 已嘗試或排除的方向 | 避免 agent 浪費 token 重做 |
| **4. 輸出格式** | 要求的結構（JSON / 列表 / 程式碼等） | 方便 orchestrator 聚合 |
| **5. 工具指引** | 使用哪些工具、何時停止 | 控制 token 消耗和執行深度 |

**不應放入的**：完整對話歷史、與此任務無關的背景、orchestrator 自己的推理過程。

### 不同拓撲下的 Context 分派策略

```
Fan-out 場景：
  → 每個 subagent 只得到「自己的 slice」
  → 共同的背景知識 → 全部給
  → 各自的任務邊界 → 必須明確互斥（否則重複）

Pipeline 場景：
  → 每個 stage 得到上一 stage 的輸出 + 原始目標
  → 不需要完整的前序推理過程，只需最終結果

Hierarchical 場景：
  → Sub-orchestrator 得到子領域目標 + 全域約束
  → 葉子 agent 得到原子任務 + 輸出格式要求
  → 越底層 context 越窄、越具體
```

---

## 路由系統設計建議

1. **Routing 決策**：用 tool description 的語義判斷，不要做 keyword matching。給 orchestrator LLM 一張「agent 能力表」讓它自己判斷。

2. **Fan-out 的 prompt 生成**：在 dispatch 前做「任務劃分驗證」——檢查每對子任務是否違反 AOP 的非冗餘性（overlap）和完整性（coverage）。

3. **Context 蒸餾**：維護「shared facts」區塊（所有 agent 都需要）和「task-specific facts」區塊（只給對應 agent），dispatch 時組合拼接而不是傳整個對話。

4. **Token 最佳化**：越深層的 agent，context 應越窄。Orchestrator 讀全局，leaf agent 只讀自己那塊。
