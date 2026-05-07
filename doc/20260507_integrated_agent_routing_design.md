# 整合式 Agent 路由系統設計

> 設計日期：2026-05-07  
> 整合來源：任務分解理論（AOP）、執行拓撲、Claude Code 執行手段、Context 壓縮研究

---

## 設計目標

建立一個可用於類 Claude 系統的 Agent 路由框架，解決以下三個問題：

1. **何時分派**：判斷任務是否需要多 agent，需要哪種拓撲
2. **如何分派**：決定用哪種執行手段，給每個 agent 什麼 context
3. **如何維持品質**：在壓縮/蒸餾過程中不損失任務關鍵資訊

---

## 系統架構概覽

```
用戶輸入
   │
   ▼
┌─────────────────────────────────┐
│  Layer 1：任務分析器             │  ← 判斷複雜度、依賴結構、專業需求
│  Task Analyzer                  │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  Layer 2：拓撲選擇器             │  ← 輸出拓撲類型 + agent 角色定義
│  Topology Selector              │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  Layer 3：Context 蒸餾器         │  ← 為每個 agent 提取最小充分 context
│  Context Curator                │    （獨立於執行器，類 ActiveContext）
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  Layer 4：執行分派器             │  ← 選擇 Claude Code 執行手段
│  Execution Dispatcher           │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  Layer 5：結果聚合器             │  ← 合併 subagent 輸出，更新全域狀態
│  Result Aggregator              │
└─────────────────────────────────┘
```

---

## Layer 1：任務分析器

分析五個維度，決定後續路由：

```
Dimension 1：依賴結構
  INDEPENDENT   → 子任務互不依賴，可全部並行
  SEQUENTIAL    → 有嚴格前後依賴，A 完成才能做 B
  DAG           → 混合型，部分並行、部分有前置依賴
  CYCLIC        → 需要反覆迭代，結果需多輪精煉

Dimension 2：專業化需求
  GENERALIST    → 單一 LLM 能力足夠
  SPECIALIST    → 需要特定工具組或特定角色（如程式分析 vs. 網路搜尋）
  UNKNOWN       → 任務開始時不知道需要什麼專家

Dimension 3：時間敏感度
  SYNC          → 需要立即結果才能繼續
  ASYNC         → 可以在背景執行，完成時通知
  SCHEDULED     → 週期性觸發，與當前對話無關

Dimension 4：狀態需求
  STATELESS     → 每個子任務完全獨立
  STATEFUL      → 需要接續前一個 agent 的推理狀態

Dimension 5：規模
  SMALL         → 1–3 個子任務
  MEDIUM        → 4–10 個子任務
  LARGE         → 10+ 個子任務，需層次化管理
```

---

## Layer 2：拓撲選擇器

### 決策樹

```
任務是否可以單 agent 完成？
└─ 是 → 直接回答，不分派

└─ 否 → 子任務彼此獨立（INDEPENDENT）？
   ├─ 是 → 規模 SMALL/MEDIUM？
   │  ├─ 是 → [Fan-out]
   │  └─ 否（LARGE）→ [Hierarchical Fan-out]
   │
   └─ 否（有依賴）→ 依賴是線性的（SEQUENTIAL）？
      ├─ 是 → [Pipeline]
      │
      └─ 否（DAG）→ 哪個專家未知（UNKNOWN）？
         ├─ 是 → [Handoff / Router]
         │
         └─ 否 → 需要反覆精煉（CYCLIC）？
            ├─ 是 → [Mesh / Group Chat]
            └─ 否 → [DAG Fan-out with gating]
```

### 拓撲定義與適用場景

| 拓撲 | 適用條件 | 風險 |
|---|---|---|
| **Fan-out** | 獨立子任務，同一輸入的多角度分析 | Orchestrator context 瓶頸（>50 結果） |
| **Pipeline** | 嚴格線性依賴，每階段輸出是下一階段輸入 | 延遲累積；一個階段失敗卡整條管線 |
| **Hierarchical** | 10+ agents 或多領域，需要中間管理層 | 每層增加 2–3s 基礎延遲 |
| **Handoff** | 任務開始時不確定需要哪個專家 | 轉交點易丟失關鍵 context |
| **DAG Fan-out with gating** | 混合依賴，部分並行、部分有前置 | 依賴追蹤複雜度高 |
| **Mesh / Group Chat** | 3–8 個 agents 需要共識建立或迭代 | 超過 8 個 agents 組合爆炸 |

---

## Layer 3：Context 蒸餾器（核心）

**設計原則**：與執行器架構分離（ActiveContext 的核心洞察）。蒸餾器是獨立模組，負責在 dispatch 前決定「給這個 agent 什麼」。

### 三區塊模型

每次 dispatch 前，把可用 context 分成三個區塊：

```
┌──────────────────────────────────────────┐
│  SHARED_FACTS（所有 agent 都需要的）      │
│  - 原始用戶目標                           │
│  - 全域約束（格式、語言、安全限制等）      │
│  - 已做的決策（不能推翻的）               │
│  - 未解決的依賴（需要 agent 注意的）      │
│  大小限制：100–300 tokens                 │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│  TASK_SPECIFIC（只給該 agent 的）         │
│  - 這個 agent 的具體目標                  │
│  - 任務邊界（明確「不做什麼」）           │
│  - 已排除的路徑（避免重做）               │
│  - 輸出 schema（格式要求）               │
│  大小限制：100–400 tokens                 │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│  TOOL_GUIDANCE（工具使用指引）            │
│  - 使用哪些工具                           │
│  - 何時停止（終止條件）                  │
│  - 不要使用哪些工具                       │
│  大小限制：50–100 tokens                  │
└──────────────────────────────────────────┘
```

**總計：250–800 tokens per agent brief**（vs. 傳整個對話的 5,000–20,000 tokens）

### 各拓撲的蒸餾規則

```
Fan-out 拓撲：
  SHARED_FACTS = 全域目標 + 全域約束
  TASK_SPECIFIC = 該 agent 的子任務 + 「不調查」的邊界
  → 最重要：任務邊界必須互斥（AOP 非冗餘性）
  → 最重要：所有 agent 的 TASK_SPECIFIC 聯集必須覆蓋完整目標（AOP 完整性）

Pipeline 拓撲：
  Stage N 的 SHARED_FACTS = 原始目標 + 全域約束
  Stage N 的 TASK_SPECIFIC = Stage N-1 的【輸出結果】（不是 N-1 的推理過程）
  → 傳遞的是成品，不是過程

Hierarchical 拓撲：
  Sub-orchestrator 的 brief = 子領域目標 + 全域約束 + 管理的 agents 清單
  Leaf agent 的 brief = 原子任務 + 輸出 schema + 工具指引
  → 越底層 brief 越窄越具體

Handoff 拓撲：
  目標 agent 的 TASK_SPECIFIC 必須包含：
    - 已嘗試什麼 + 為什麼需要轉交
    - 下一步的具體目標
  → 轉交點是資訊最容易遺失的地方，必須明確交代「為什麼在這裡接棒」
```

### 品質驗證（AOP 三原則檢查）

在正式 dispatch 前，對所有子任務的集合做：

```python
def validate_decomposition(subtasks, agents):
    # 1. 可解性：每個子任務都有能力解決它的 agent
    for task in subtasks:
        assert any(agent.can_solve(task) for agent in agents), \
            f"No agent can solve: {task}"
    
    # 2. 完整性：子任務聯集覆蓋原始問題
    covered = union(task.scope for task in subtasks)
    assert covers(covered, original_problem), \
        "Decomposition incomplete"
    
    # 3. 非冗餘性：子任務不重疊
    for i, ti in enumerate(subtasks):
        for j, tj in enumerate(subtasks):
            if i != j:
                assert not overlaps(ti.scope, tj.scope), \
                    f"Tasks {i} and {j} overlap"
```

---

## Layer 4：執行分派器

### Claude Code 執行手段對應表

| 拓撲 | 執行手段 | 使用方式 |
|---|---|---|
| Fan-out（並行） | 同一訊息多個 `Agent` call | 一次發出所有 Agent tool call，Claude Code 自動並行執行 |
| Fan-out（背景） | `Agent(run_in_background=true)` | 非阻塞，完成時通知；適合不需要立即結果的研究任務 |
| Pipeline | 序列 `Agent` calls | 等待前一個結果再發下一個 |
| Hierarchical | 巢狀 `Agent` calls | 中間 orchestrator agent 自己再 spawn subagents |
| Handoff | `SendMessage(to: agentId)` | 向現有 agent 繼續對話，保留其推理狀態 |
| 程式碼隔離 | `Agent(isolation: "worktree")` | 互不污染的 git branch，完成後回報 |
| 定期任務 | `CronCreate` | 排程遠端 agent，與當前對話解耦 |
| 狀態追蹤 | `TaskCreate/TaskUpdate` | 跨 agent 共享進度，不是執行手段而是協調原語 |

### 並行執行注意事項

```
LangGraph / Claude Code 並行執行的原子性：
  一個 superstep 中，若其中一個 agent 失敗
  → 整個 superstep 的所有並行 agent 都失敗

設計對策：
  1. 每個並行 agent 的任務必須是獨立可重試的
  2. 關鍵路徑的 agent 先序列執行，確認成功後再 fan-out 並行部分
  3. 使用 TaskCreate 記錄各 agent 的完成狀態，方便局部重試
```

---

## Layer 5：結果聚合器

### 聚合策略對應拓撲

| 拓撲 | 聚合方法 | 說明 |
|---|---|---|
| Fan-out | **LLM synthesis** | Orchestrator 讀取所有子結果，生成統一輸出 |
| Pipeline | **Structured relay** | 最終階段的輸出即最終答案 |
| Hierarchical | **Hierarchical fusion** | 各層向上回報，逐層摘要合併 |
| Handoff | **Final agent output** | 最後接手的 agent 的輸出即答案 |
| Mesh/Group Chat | **Voting / consensus** | 多輪後提取共識，或 LLM 判斷最終答案 |

### Orchestrator 瓶頸對策

當子 agent 數量超過 10 個，直接讓 orchestrator 讀所有結果會打爆其 context window。解法：

```
策略 A：成果外部化 + 引用傳遞
  Subagent 完成後 → 結果存入外部儲存（檔案/資料庫）
  → 只傳輕量 reference 回 orchestrator
  → Orchestrator 依需要拉取，不一次全裝入 context

策略 B：中間層合併
  把 10 個 subagent 的結果先送給 3 個中間 aggregator
  每個 aggregator 合併 3–4 個結果
  → Orchestrator 只需整合 3 個中間結果
```

---

## 壓縮整合：維持長對話品質

### 主動壓縮觸發點（Focus Agent 原則）

不等 context 滿才壓縮，在自然的**階段邊界**主動壓縮：

```
觸發壓縮的事件：
  ✓ 一個 subagent 的主要任務完成後（立即壓縮其結果）
  ✓ Pipeline 的一個 stage 完成後
  ✓ Orchestrator context 超過 60% 閾值時（預防性，不等到 100%）
  ✗ 不要等到 context 滿才壓縮（被動壓縮效率低 6 倍）
```

### 壓縮時保留 vs. 丟棄

```
永遠保留（逐字保留）：
  - 架構決策 + 決策理由
  - 未解決的依賴和問題
  - 當前任務狀態（已完成什麼 / 下一步是什麼）
  - 關鍵約束（安全、格式、範圍限制）

可以丟棄：
  - 已納入推理的工具呼叫輸出原文
  - 失敗嘗試的細節（保留：「嘗試了 X，因為 Y 失敗」，丟掉：X 的完整輸出）
  - 中間計算步驟

結構化保留格式（不要散文摘要）：
  {
    "goal": "...",
    "decisions_made": [...],
    "open_issues": [...],
    "completed_phases": [...],
    "next_step": "..."
  }
```

### 壓縮品質優化迴圈（ACON 原則）

```
1. 對典型任務，有壓縮和無壓縮各跑一次
2. 找出因壓縮導致失敗的案例
3. 分析：哪類資訊被壓縮丟失導致失敗？
4. 更新壓縮 prompt 以保留那類資訊
5. 重複

→ 這是不動模型權重就能提升壓縮品質的最直接方法
```

---

## 完整決策流程（一次路由的全過程）

```
輸入：用戶任務描述

Step 1 [任務分析]
  → 評估五個維度（依賴、專業化、時間、狀態、規模）
  → 輸出：維度向量

Step 2 [拓撲選擇]
  → 根據維度向量查決策樹
  → 輸出：拓撲類型 + agent 角色清單

Step 3 [AOP 驗證]
  → 檢查可解性、完整性、非冗餘性
  → 失敗 → 重新分解直到通過

Step 4 [Context 蒸餾]
  → 為每個 agent 組合 SHARED_FACTS + TASK_SPECIFIC + TOOL_GUIDANCE
  → 驗證每份 brief 不超過 800 tokens

Step 5 [執行分派]
  → 查拓撲→執行手段對應表
  → 選擇 foreground/background/worktree/cron
  → 發出 Agent tool call（並行時同一訊息中）

Step 6 [結果聚合]
  → 按拓撲選擇聚合方法
  → 超過 10 個子結果 → 外部化 + 引用傳遞
  → 觸發壓縮（如在階段邊界）
  → 更新全域任務狀態

Step 7 [輸出]
  → 回傳用戶
  → 更新 context 狀態供下一輪使用
```

---

## 設計取捨說明

| 設計選擇 | 取捨 |
|---|---|
| Context 蒸餾器與執行器分離 | 增加一次 LLM pass 但大幅減少每個 subagent 的 context 大小，整體 token 成本降低 |
| 結構化物件取代散文摘要 | 確定性保留，但需要前期定義 schema；散文更靈活但有隨機損失 |
| 主動壓縮 vs. 被動壓縮 | 主動壓縮需要明確設計觸發點，但效率高 6 倍（Focus Agent 實驗） |
| AOP 驗證增加延遲 | 但可防止 17 倍的錯誤放大（bag-of-agents 效應） |
| Fan-out 並行的原子性失敗 | 並行快但一個失敗全失敗；Pipeline 慢但局部可重試 |

---

## 參考來源整合

| 設計決策 | 來源 |
|---|---|
| AOP 三原則 | arXiv:2410.02189 |
| 17 倍錯誤放大 | Towards Data Science: Why Your Multi-Agent System is Failing |
| ActiveContext 架構分離 | arXiv:2604.11462 |
| 主動壓縮觸發 | arXiv:2601.07190 (Focus Agent) |
| 結構化物件 vs. 散文 | Anthropic multi-agent engineering blog |
| Orchestrator 瓶頸對策 | Anthropic production pattern (200K token 外部化策略) |
| ACON 品質優化迴圈 | arXiv:2510.00615 |
| 執行手段對應 | Claude Code system prompt (本對話可觀察) |
