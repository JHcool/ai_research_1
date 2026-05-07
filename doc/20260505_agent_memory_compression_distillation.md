# Agent 記憶體系統：壓縮、蒸餾與品質維持

> 研究日期：2026-05-05  
> 關鍵字：context compression, memory distillation, agent memory architecture, context window management

---

## 核心問題：為什麼需要壓縮？

### 兩個根本性的降級效應

**Lost in the Middle**（Liu et al., 2023 / TACL 2024 — [arXiv:2307.03172](https://arxiv.org/abs/2307.03172)）：  
LLM 對 context 的注意力呈 U 型曲線——開頭與結尾的 token 受到最多注意，中間段最少。隨著相關資訊被推向中間，多文件 QA 性能下降 30%+。這是 RoPE 等位置編碼的距離衰減造成的，是架構性問題，無法靠 prompt 工程完全克服。

**Context Rot**（Anthropic engineering blog）：  
Transformer 注意力是二次複雜度——10 萬 token 意味著約 100 億對 token 關係。隨著 context 增長，softmax 正規化使每個 token 的注意力權重縮水，**噪音底線上升而非信號增強**。這是性能梯度式下降，不是懸崖式崩潰。

**2024 年的關鍵發現**（[arXiv:2510.05381](https://arxiv.org/html/2510.05381v1)）：  
即使模型能完美檢索所有相關證據，**長 context 下推理能力本身也會降級**。問題不只是檢索失敗——模型在高負荷 context 下的推理品質本身就會下滑。

---

## Claude Code 的壓縮機制（Compaction）

**這是目前最詳細的官方公開機制說明。**

### 觸發條件與執行流程

API 層面以 `compact-2026-01-12` beta header 暴露，觸發閾值預設為 **150,000 tokens**（最低可設 50,000）：

1. API 偵測到 input tokens 超過閾值
2. Claude 生成整段對話歷史的摘要
3. 摘要放入 `compaction` block 回傳
4. **後續請求中，compaction block 之前的所有訊息自動丟棄**
5. 對話從摘要繼續往前

`pause_after_compaction: true` 參數可在步驟 3 和 4 之間暫停，讓開發者審查或注入額外 context 後再繼續。

### 官方摘要 Prompt（逐字）

> "You have written a partial transcript for the initial task above. Please write a summary of the transcript. The purpose of this summary is to provide continuity so you can continue to make progress towards solving the task in a future context, where the raw history above may not be accessible and will be replaced with this summary. Write down anything that would be helpful, including the state, next steps, learnings etc. You must wrap your summary in a `<summary></summary>` block."

**關鍵**：這是**任務導向的前向摘要**，不是對話的忠實重現。模型被要求保留「當前狀態、下一步、學到的事」——不是「對話發生了什麼」。`instructions` 參數可完全替換這個 prompt。

### 保留 vs. 丟棄

| 保留 | 丟棄 |
|---|---|
| 架構決策 | 已使用的工具輸出 |
| 未解決的 bug | 已處理的結果 |
| 實作細節與依賴 | 重複內容 |
| 當前任務狀態 | 不再需要的中間推理 |

Claude Code 具體做法：壓縮 context + 保留最近存取的 5 個檔案。

> 參考：[Anthropic Compaction API 文件](https://platform.claude.com/docs/en/build-with-claude/compaction)

---

## 記憶體架構類型分類

### 四層記憶分類（來源：[Experience Compression Spectrum — arXiv:2604.15877](https://arxiv.org/abs/2604.15877)）

| 記憶類型 | 位置 | 內容 | 存取方式 |
|---|---|---|---|
| **Working Memory** | In-context | 當前任務狀態、近期觀察 | 直接注意力 |
| **Episodic Memory** | 外部（可搜尋） | 過去互動記錄、事件日誌 | 語義搜尋 / 時間序列 |
| **Semantic/Declarative Memory** | 外部（可搜尋） | 事實知識、用戶偏好、實體 | Embedding 相似度 |
| **Procedural Memory** | 外部或 system prompt | 行為規則、工作流、技能 | 注入 system prompt |

### 壓縮層級光譜（同上論文）

| 層級 | 類型 | 壓縮比 | 範例內容 |
|---|---|---|---|
| L0 | Raw trace | 1:1 | 完整互動日誌 |
| L1 | Episodic memory | 5–20× | 結構化事件摘要、事實 |
| L2 | Procedural skill | 50–500× | 可重用工作流、程式碼模板 |
| L3 | Declarative rule | 1,000×+ | 抽象原則、行為指南 |

**當前缺口**：目前的 22+ 個系統都只支援固定層級，沒有任何系統能自動做跨層級壓縮（episodic → skill → rule 的向上提升）。L3 幾乎完全缺席。

---

## 主要系統實作解析

### MemGPT / Letta — OS 分頁隱喻

**論文**：[arXiv:2310.08560](https://arxiv.org/abs/2310.08560)

**三層架構**：

| 層級 | 類比 | 內容 | 存取 |
|---|---|---|---|
| **Core Memory** | RAM | `human` block（用戶資訊）+ `persona` block（agent 自我描述），固定大小，**永遠在 context 中** | 直接，但有容量限制 |
| **Recall Memory** | 可搜尋的對話歷史 | 完整對話存入外部資料庫 | 主動搜尋函式呼叫 |
| **Archival Memory** | 長期文件庫 | 已處理過的知識，agent 主動寫入 | 明確搜尋工具呼叫 |

**核心機制**：Agent 通過函式呼叫自行控制記憶移動：
- `core_memory_replace(block, old, new)` — 改寫永久在場的區塊
- `archival_memory_insert("fact")` — 推送到長期儲存
- `archival_memory_search("query")` — 把相關歸檔事實拉進 context

沒有自動壓縮觸發——agent 自己管理認知預算。Core Memory 空間填滿時，agent 必須決定如何改寫濃縮，這是模型內在的有損壓縮。

---

### MemoryOS — Heat 驅動的分層淘汰

**論文**：[arXiv:2506.06326](https://arxiv.org/abs/2506.06326)（EMNLP 2025 Oral）

**三層 + 淘汰公式**：

- **STM（短期）**：對話頁（query + response + timestamp），FIFO 滿了 → 最舊頁移到 MTM
- **MTM（中期）**：同主題頁聚成 segment，LLM 摘要壓縮；滿了 → Heat 最低的 segment 刪除（**永久遺失**）
- **LPM（長期）**：用戶固定屬性 + 動態知識庫。Heat 高的 MTM 內容可晉升到 LPM。

**Heat 公式（這就是「決定保留什麼」的核心）**：

```
Heat = α·N_visit + β·L_interaction + γ·R_recency
R_recency = exp(-Δt/μ)
```

- `N_visit`：被檢索次數
- `L_interaction`：segment 包含的對話頁數（體積代理）
- `R_recency`：時間指數衰減——近期內容 Heat 高

淘汰時 Heat 最低的先出。**沒有語義重要性評分**——只有頻率、體積、近期性三個代理指標。

**成效**：在 LoCoMo benchmark 上 F1 +49%、BLEU-1 +46%（vs. GPT-4o-mini 基線）。

---

### Mem0 — 事實萃取 + 圖記憶

**論文**：[arXiv:2504.19413](https://arxiv.org/abs/2504.19413)（2025 年 4 月）

**不存原始對話，只萃取事實**：

每次對話輪次，系統建構 context 窗：
```
P = (對話摘要 S, 近 m 條訊息, m_{t-1}, m_t)
```
傳給 LLM 萃取候選事實 `Ω = {ω₁, ω₂...ωₙ}`。

**四操作更新策略**：對每個新事實，取 top-10 語義相似的已有記憶，LLM 判斷選其一：

| 操作 | 條件 | 說明 |
|---|---|---|
| **ADD** | 無等效記憶 | 新建條目 |
| **UPDATE** | 有但可補充 | 擴增既有記憶 |
| **DELETE** | 新資訊與既有矛盾 | 刪除舊的 |
| **NOOP** | 已知或無關 | 不變 |

舊事實不會被直接覆蓋，保留時間戳，維持時序 context。

**圖記憶（Mem0^g）**：  
可選擴展，記憶建模為有向標記圖 `G=(V,E,L)`，節點是實體，邊是關係三元組 `(主體, 關係, 客體)`。檢索時雙模：實體中心（沿圖邊遍歷）+ 語義三元組（embedding 相似）。

**非同步摘要器**：獨立背景程序定期更新對話摘要 `S`，不阻塞主萃取管線。

**成效**：比完整 context 方法 response time -91%、prompt tokens -80%，任務連貫性維持。

---

### LangMem / LangGraph Memory — 三層架構

**來源**：[langchain-ai.github.io/langmem](https://langchain-ai.github.io/langmem/)

**三層**：
- **Core layer**：無狀態純函式。`Memory Manager`（決定 insert/update/delete/consolidate）+ `Prompt Optimizer`（基於對話軌跡優化 system prompt）
- **Storage layer**：加持久化，自動存到 LangGraph `BaseStore`
- **Agent layer**：對 agent 暴露 `create_manage_memory_tool`、`create_search_memory_tool`，讓 agent 自主操控

**兩種記憶形成模式**：
- **Hot-path（同步）**：每次對話前更新。立即反映關鍵 context，但增加延遲、增加 agent 決策複雜度
- **Background（非同步）**：對話後在背景執行。無延遲影響，適合摘要型記憶

**相關性評分**：語義相似度 × 重要性 × 強度（recency + frequency 函式）。

---

### ACON — 學習型壓縮指南優化

**論文**：[arXiv:2510.00615](https://arxiv.org/abs/2510.00615)（2025 年 10 月）

**核心洞察：壓縮指南應從失敗中學習，而非手工設計。**

**雙壓縮策略（分開處理）**：
- **互動歷史壓縮**：閾值 `T_hist = 4096 tokens`（最佳值），超過觸發摘要
- **觀測壓縮**：閾值 `T_obs = 1024 tokens`（最佳值），壓縮單次工具輸出

**壓縮指南優化迴圈**：

```
1. 收集配對軌跡：同一任務下
   (a) 完整 context 的 agent 成功
   (b) 壓縮 context 的 agent 失敗
2. Optimizer LLM 分析差距：壓縮丟了什麼資訊導致失敗？
3. 更新壓縮指南（自然語言 prompt）以保留那類資訊
4. 交替優化兩個目標：
   - 最大化效用：不失敗需要保留什麼？
   - 最大化壓縮：成功案例中實際用到了什麼？去掉沒用到的
```

**蒸餾部署**：大模型壓縮器（遵循優化後指南）的行為可以蒸餾進小模型（supervised learning），「在蒸餾進小模型後保留 95%+ 的準確率」。

**成效**：peak token -26–54%，小型 LM 長任務性能 +46%。

---

### Focus Agent — 主動式 Context 壓縮

**論文**：[arXiv:2601.07190](https://arxiv.org/abs/2601.07190)（2026 年 1 月）

**Agent 自行決定何時壓縮，不依賴外部觸發。**

**兩個原語**：
- `start_focus("調查 X")` — 宣告子任務範圍，打 checkpoint
- `complete_focus("發現 Y，繼續做 Z")` — 觸發 checkpoint 以來的內容整合

執行 `complete_focus` 時：
1. Agent 生成結構化摘要：嘗試了什麼、學到了什麼、結果如何
2. 摘要追加到 context 頂部的**知識區塊（Knowledge block）**
3. **checkpoint 之間的所有原始訊息全部刪除**

效果是「鋸齒形」context：探索期增長，整合期塌縮，知識區塊持續累積。

**關鍵發現**：被動 prompting 每任務只觸發 1–2 次壓縮，節省微乎其微。需要**積極 prompting**（「每 10–15 個工具呼叫壓縮一次」）才能達到每任務 6.0 次壓縮、token 節省 22.7%，且任務成功率不變（60% SWE-bench，與未壓縮基線相同）。

---

### ActiveContext — RL 訓練的 Context 策展器

**論文**：[arXiv:2604.11462](https://arxiv.org/abs/2604.11462)（2025 年 4 月）

**這是目前最理論嚴謹的方法，打破了「壓縮必然犧牲品質」的假設。**

**架構嚴格分離**：
- **TaskExecutor**：凍結的前沿模型（Gemini/GPT-4o 等），**不訓練**，接收策展後的記憶 + 當前觀測，輸出行動
- **ContextCurator**：可訓練的輕量 7B 模型（Qwen-2.5-7B-Instruct 初始化），每一步接收 `(當前記憶, 最新觀測, 前一行動)` 並自回歸地**改寫整個工作記憶**

**RL 訓練信號**：TaskExecutor 凍結，所以任務成功/失敗的變異完全歸因於 ContextCurator 的記憶決策。使用 **MT-GRPO（多輪 GRPO）** + KL 散度正規化（β=0.001）訓練。

| 測試 | 完整 Context | ActiveContext | Token 變化 |
|---|---|---|---|
| WebArena (Gemini-3.0-flash) | 36.4% | **41.2%** | -8.8% |
| DeepSearch (Gemini-2.5-flash) | 33.4% | **41.5%** | **-79%**（34K→7.3K）|

**為什麼壓縮後性能反而提升**：長 context 包含「干擾資訊」，會主動誤導推理。好的壓縮移除的不是信號而是噪音——7B RL 策展器「媲美 GPT-4o 的 context 管理能力」。

---

### QUITO-X — 資訊瓶頸理論應用於壓縮

**論文**：[arXiv:2408.10497](https://arxiv.org/abs/2408.10497)（EMNLP 2025 Findings）

**這是「應該保留多少」的形式化理論答案。**

傳統方法用 self-information（困惑度）或 attention weights 評分——這些指標與查詢無關。QUITO-X 的核心洞察：

**壓縮應最大化與查詢的互資訊，同時最小化與完整 context 的互資訊。**

形式化：最大化 `I(Z; Q)`（壓縮 context Z 與查詢 Q 的互資訊）同時最小化 `I(Z; C)`（Z 與完整 context C 的互資訊）。這是信息瓶頸（IB）問題：找 C 關於 Q 的最小充分統計量。

實作用 FLAN-T5-small（80M 參數）的 cross-attention 機制，以查詢條件互資訊評分每個 context token。

**設計意涵**：「正確的」壓縮 context 是以最少 bits 保留最多任務相關資訊的那個。Token 基於與當前任務的關係保留或丟棄，而非基於無條件重要性。

---

### A-MEM — Zettelkasten 網狀記憶

**論文**：[arXiv:2502.12110](https://arxiv.org/abs/2502.12110)（2025 年 2 月）

新增記憶時，不只是存入，而是建立結構化筆記：
```
{contextual_description, keywords, tags, links, content}
```

**並分析與現有記憶的語義連結，建立雙向連結**。最特別之處：新記憶的加入可以**觸發既有記憶的更新**——之前的記憶可因新資訊而修訂描述與屬性。

記憶是活的互連網絡，不是靜態條目庫。

---

## 多 Agent 系統中的蒸餾問題

### 現有做法

來源：[Anthropic multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)

大規模場景（Orchestrator context 接近 200K tokens）的策略：
- Orchestrator 把計劃存到外部 Memory
- Subagent 完成階段後把成果存外部，只回傳**輕量引用（reference）**，不傳完整轉錄
- Orchestrator 只在 context 裡保留引用和指標，按需拉取

### 三種 Context 傳遞策略與權衡

| 策略 | Context 大小 | Token 效率 | 資訊損失 | 延遲 |
|---|---|---|---|---|
| **結構化物件** | 200–500 tokens | 最高 | 無（確定性選擇） | 最低 |
| **摘要交接** | 500–2000 tokens | 中 | 有（隨機性損失）| +500ms–1.5s |
| **層次生成** | 每層各自管理 | 高（按層隔離） | 按層隔離控制 | 每層加 2–3s |

### 形式理論現況

多 Agent 間 context 選擇**沒有像 IB 理論那樣成熟的正式框架**。最接近的應用：

在每個 handoff 點，把已完成的工作壓縮為關於下一個 agent 任務的最小充分統計量——最大化 `I(compressed_result; next_task_objective)`，同時最小化 `I(compressed_result; full_context)`。

**Orchestrator 瓶頸問題**：多個 subagent 時，orchestrator 必須持有所有結果才能合成——天生的瓶頸。最佳實踐是成果外部化 + 引用傳遞。

---

## 品質衡量指標

| 指標類型 | 說明 | 使用在 |
|---|---|---|
| **任務性能指標** | 壓縮前後的任務成功率差距 | ACON, Focus, ActiveContext |
| **Token 效率指標** | Peak token、總消耗、每任務成本 | 所有系統 |
| **資訊理論指標** | I(Z;Q) 互資訊 | QUITO-X |
| **對話品質指標** | GPT-4 評審 1–100 分（連貫性、一致性） | Recursive Summarization |
| **RAG 指標** | 忠實度（有無失真）+ context recall（相關內容保留率）| 檢索評測，可套用到壓縮評測 |

---

## 設計建議（針對 Multi-Agent Route 系統）

1. **按內容類型選壓縮層級，不用單一策略**  
   L1（事實，5–20×）用於追蹤發生了什麼；L2（程序，50–500×）用於重複工作流；L3（規則，1000×+）用於抽象限制。

2. **架構上把壓縮器與執行器分開**  
   ActiveContext 的核心洞察：「過濾噪音」和「從過濾後內容推理」是正交目標。輕量壓縮器（7B RL 訓練）可以媲美前沿模型的壓縮品質，成本遠低。這對應路由設計——在每個 subagent dispatch 之前加一個輕量 context curation 模組。

3. **互動歷史與環境觀測分開壓縮，用不同閾值**  
   ACON 的發現：互動歷史閾值 4096 tokens，單次觀測閾值 1024 tokens。混在一起用同一策略會丟失觀測資訊且歷史壓縮不足。

4. **Subagent context 用結構化物件，不用散文摘要**  
   200–500 token 結構化物件確定性保留任務關鍵欄位。散文摘要引入隨機性損失且增加延遲。只有面向人類的輸出或長階段交接才用散文摘要。

5. **在階段邊界主動壓縮，而非在 token 限制時被動觸發**  
   Focus Agent 的發現：被動壓縮（等 context 填滿）每任務只壓縮 1–2 次。**在自然任務邊界主動壓縮**（每 10–15 個工具呼叫）才能達到 6× 壓縮次數與最佳節省，品質不變。在 orchestrator 層面：subagent 完成階段後立即壓縮成果，不要等所有 subagent 都完成再一起處理。

6. **用失敗驅動的指南優化改善壓縮 prompt**  
   ACON 的對比失敗分析迴圈可直接應用：在代表性任務上有無壓縮各跑一遍，找出因資訊損失導致的失敗模式，針對那些資訊類別修改壓縮指令。這是不動模型權重的最有效品質改善方式。

7. **架構決策和未解問題永遠保留，已使用的工具輸出可以丟棄**  
   Anthropic 的明確指導：已使用在推理中的工具呼叫結果可從歷史中剪掉。決策、未解依賴、當前任務狀態應逐字保留在摘要中。

---

## 關鍵論文索引

| 論文 | 重點貢獻 | 連結 |
|---|---|---|
| MemGPT / Letta | OS 分頁三層架構，agent 自主記憶管理 | [arXiv:2310.08560](https://arxiv.org/abs/2310.08560) |
| Lost in the Middle | U 型注意力曲線，中間段性能衰減 | [arXiv:2307.03172](https://arxiv.org/abs/2307.03172) |
| A-MEM | Zettelkasten 網狀記憶，動態連結與更新 | [arXiv:2502.12110](https://arxiv.org/abs/2502.12110) |
| Mem0 | 四操作事實萃取 + 圖記憶 | [arXiv:2504.19413](https://arxiv.org/abs/2504.19413) |
| MemoryOS | Heat 公式驅動分層淘汰 | [arXiv:2506.06326](https://arxiv.org/abs/2506.06326) |
| ACON | 失敗驅動壓縮指南優化 + 蒸餾 | [arXiv:2510.00615](https://arxiv.org/abs/2510.00615) |
| Focus Agent | 主動式自我壓縮，鋸齒形 context | [arXiv:2601.07190](https://arxiv.org/abs/2601.07190) |
| ActiveContext | RL 訓練策展器，壓縮後性能反而提升 | [arXiv:2604.11462](https://arxiv.org/abs/2604.11462) |
| Experience Compression Spectrum | L0–L3 壓縮層級分類學 | [arXiv:2604.15877](https://arxiv.org/abs/2604.15877) |
| QUITO-X | 資訊瓶頸理論應用，形式化最優壓縮 | [arXiv:2408.10497](https://arxiv.org/abs/2408.10497) |
| Recursive Summarization | 遞歸摘要基線方法 | [arXiv:2308.15022](https://arxiv.org/abs/2308.15022) |
| Anthropic Compaction API | Claude Code 壓縮機制官方文件 | [platform.claude.com/docs](https://platform.claude.com/docs/en/build-with-claude/compaction) |
| Anthropic Multi-Agent System | 大規模多 Agent 的 context 策略 | [anthropic.com/engineering](https://www.anthropic.com/engineering/multi-agent-research-system) |
