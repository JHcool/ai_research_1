# DeepSeek 近期進展 與 多Agent任務分解理論與實務

> 研究日期：2026-04-29

---

## 第一部分：DeepSeek 的真實改進機制

### 結論先說

**DeepSeek V3 / R1 的性能提升與「多Agent協作」無關。** 經查閱 DeepSeek 官方技術報告（arXiv:2412.19437、arXiv:2501.12948），未找到任何關於 agent swarm、多Agent訓練或多Agent推理的機制。「Agent群」的說法可能源自術語混淆。

### 實際改進機制（四大核心）

#### 1. 混合專家架構（MoE）— DeepSeek-V3
- 671B 總參數，每個 token 只激活 37B
- **auxiliary-loss-free 負載均衡**：相較過去 MoE 的新創，不再需要輔助損失函數
- **多 Token 預測（Multi-Token Prediction）**：同時預測多個未來 token，強化表示學習

> 注意：MoE 的「expert」聽起來像 agent，實際上是**單一模型內的子網路路由層**，非多Agent協作。

#### 2. 多頭潛在注意力（MLA）— DeepSeek-V2 & V3
- 將 KV cache 壓縮進低秩潛在空間
- 有效批次容量提升約 **9.6 倍**（相比標準多頭注意力）
- 純矩陣壓縮技術，與 agent 無關

#### 3. 硬體-演算法協同設計 — DeepSeek-V3
- **FP8 混合精度訓練**：首次在兆參數規模驗證，FLOPS 比 FP16 多一倍
- **DualPipe 演算法**：新型管線並行排程，重疊計算與通訊，減少空泡（pipeline bubble）
- **PTX 級 GPU 優化**：warp 層級專化，最大化 H800 利用率
- 全訓練成本：14.8T tokens，僅用 **278.8 萬 H800 GPU 小時**，且無 loss spike

#### 4. 群組相對策略優化（GRPO）— DeepSeek-R1
- 每個 prompt 生成**一組候選輸出**（group of outputs），使用當前 policy 採樣
- 以規則型 reward（正確性 + 格式）評分
- 以**組內平均作為 baseline** 計算相對優勢，取代 PPO 中獨立的 critic/value 網路
- 結果：單一模型通過純 RL 自發產生自我反思、回溯（「aha moments」）、動態推理長度

> 「GRPO 的 group」= 同一模型的一批採樣輸出，**不是多個獨立 agent**

### 術語混淆來源

| 混淆點 | 實際是什麼 |
|---|---|
| MoE 的「experts」 | 單一模型內的子網路，token 路由選擇 |
| GRPO 的「group」 | 同一模型一次採樣的多個候選答案 |
| DeepSeek 做「agent產品」 | Bloomberg 2025/09 報導 DeepSeek 正在**開發面向用戶的 agent 產品**（非訓練機制） |
| Kimi K2.6「Agent Swarm」 | 競爭對手 Kimi 的功能，不是 DeepSeek |
| 第三方整合 | CrewAI、MetaGPT 等**使用** DeepSeek 模型作為組件，不是 DeepSeek 自身架構 |

### 參考文獻
- [DeepSeek-R1 論文 (arXiv:2501.12948)](https://arxiv.org/abs/2501.12948)
- [DeepSeek-V3 Technical Report (arXiv:2412.19437)](https://arxiv.org/abs/2412.19437)
- [Martin Fowler DeepSeek 技術綜覽](https://martinfowler.com/articles/deepseek-papers.html)

---

## 第二部分：多 Agent 任務分解的理論與實務

### 背景：為什麼需要任務分解？

當單一 agent 面臨以下情況時，多 agent 系統才有必要：
- prompt 長度或複雜度超過 context window
- 任務需要不同專業工具組合
- 需要平行加速（獨立子任務可同時進行）
- 安全邊界要求（不同 agent 有不同權限）

**不要過早引入多 agent**：先用單一 agent + 工具，當單 agent 明顯失敗時再分解。

---

### 任務分解的三大正確性原則（AOP Framework）

來源：[Agent-Oriented Planning (arXiv:2410.02189)](https://arxiv.org/html/2410.02189v2)

1. **可解性（Solvability）**：每個子任務必須有至少一個 agent 能獨立完整解決。生成無法被任何 agent 解決的子任務是最主要的失敗原因。

2. **完整性（Completeness）**：所有子任務的聯集必須覆蓋原始問題。分解過程中的資訊遺失會使最終答案不完整。

3. **非冗餘性（Non-Redundancy）**：子任務不可重疊。重疊導致 agent 重複勞動，產生衝突輸出難以聚合。

> 違反以上三原則的「agent 大雜燴」系統，錯誤放大可達 **17 倍**。  
> 來源：[Why Your Multi-Agent System is Failing](https://towardsdatascience.com/why-your-multi-agent-system-is-failing-escaping-the-17x-error-trap-of-the-bag-of-agents/)

---

### 分解策略類型

| 策略 | 說明 | 適用時機 |
|---|---|---|
| **LLM 驅動（zero-shot）** | Orchestrator LLM 自由生成任務拆分 | 新穎、多變的任務 |
| **LLM 驅動（few-shot）** | 示例引導拆分格式 | 輸出結構有要求時 |
| **程式化 / SOP** | 固定邏輯拆分已知任務類型 | 重複、可預測的流程 |
| **HTN（層次任務網路）** | 抽象目標逐步精化為可執行步驟 | 含可重用子程序的複雜流程 |
| **DAG 動態生成** | 任務建模為帶依賴邊的節點圖，執行時生成 | 有明確前置條件的混合串/並行任務 |
| **限制式分解（ACONIC）** | 將任務建模為可滿足性問題；樹寬衡量複雜度 | 需要正確性保證的形式推理任務 |

---

### 執行拓撲類型

| 拓撲 | 結構 | 是否並行 | 控制方式 |
|---|---|---|---|
| **順序 / Pipeline** | 固定線性階段，每階段基於前一階段 | 否 | 確定性 |
| **並發 / Fan-out** | 同一輸入分發給多個 agent 同時執行 | 是 | Scatter-Gather |
| **層次（Hierarchical）** | 督導者與工作者組成的樹狀結構 | 層內部分並行 | 集中式 |
| **Handoff / 路由** | 控制動態轉交給更合適的 agent | 否（單一活躍） | 動態 |
| **Swarm** | 去中心化，共享狀態，無中央協調者 | 是 | 湧現式 |
| **Mesh** | 小型團隊內的明確 Peer-to-Peer 連接 | 部分 | 點對點 |
| **Magentic / 任務台帳** | Manager 動態建立與修正任務列表 | 部分 | 自適應 |

---

### 拓撲選擇決策表（Microsoft Azure，2026）

| 使用情況 | 推薦模式 |
|---|---|
| 明確線性依賴，無並行可能 | Sequential |
| 同一輸入的獨立分析，對延遲敏感 | Concurrent |
| 共識構建、迭代驗證（腦力激盪） | Group Chat |
| 開始時不知道需要哪個專家 | Handoff |
| 開放式、無預先解決路徑 | Magentic（任務台帳） |

來源：[Azure AI Agent Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)

---

### Anthropic 生產系統的實際數據

來源：[Anthropic 多 Agent 研究系統部落格](https://www.anthropic.com/engineering/multi-agent-research-system)

使用 Opus 4 作為 Orchestrator + Sonnet 4 作為 subagent，性能超越單一 Opus 4 達 **90.2%**。

| 查詢複雜度 | Agent 數量 | 每個 Agent 的工具呼叫數 |
|---|---|---|
| 簡單事實查詢 | 1 個 | 3–10 |
| 直接比較分析 | 2–4 個 | 每個 10–15 |
| 複雜研究任務 | 10+ 個 | 每個 15+ |

**每個 subagent 的任務說明必須包含**：
1. 具體、明確的目標
2. 明確的任務邊界（**不**調查什麼）
3. 要求的輸出格式
4. 使用哪些工具及何時停止的指引

**最主要的失敗模式**：模糊的分解（如「研究半導體短缺」）導致 agent 重複搜尋相同內容。

---

### 主要框架的分解實作方式

#### LangGraph（LangChain）
- **分解策略**：圖拓撲。多個傳入依賴的節點自動並行（fan-out = 並行 superstep）
- **關鍵 API**：**Send API** — 執行時動態 map-reduce，返回 `Send(node, state)` 列表，LangGraph 全部並行執行
- **聚合**：Reducer 函數合併並行狀態寫入
- **注意**：並行失敗是原子性的——一個分支失敗，整個 superstep 的所有分支都失敗
- **實測加速**：137x（61s → 0.45s）
- 文件：https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/

#### AutoGen（Microsoft）
- **分解策略四種模式**：(1) 函數式規劃器生成 3-5 子任務，(2) group chat 自動發言選擇，(3) group chat 自定確定性路由，(4) AutoBuild 動態生成 agent
- **角色類型**：`AssistantAgent`、`UserProxyAgent`、`ToolAgent`、`GroupChatManager`
- 文件：https://microsoft.github.io/autogen/0.2/docs/topics/task_decomposition/

#### CrewAI
- **分解策略**：Task 物件模型。每個 `Task` 有：描述、預期輸出、指派 agent、依賴 context、工具、async 旗標
- **執行模式**：Sequential（線性）、Hierarchical（manager 委派）、Async（`async_execution=True`）
- **依賴管理**：`context` 屬性明確引用前置 task 的輸出
- **聚合**：`TaskOutput` 物件，支援 raw / JSON / Pydantic 格式
- 文件：https://docs.crewai.com/en/concepts/tasks

#### MetaGPT
- **分解策略**：SOP 編碼，模擬軟體公司角色：PM → Architect → Project Manager → Engineer → QA
- **通訊**：傳遞**結構化文件**（PRD、系統設計、任務列表、程式碼），而非自然語言對話
- **發佈訂閱**：共享訊息池，agent 只消費相關訊息類型
- **性能**：Pass@1 85.9–87.7%（HumanEval）
- 論文：https://arxiv.org/abs/2308.00352

#### MegaAgent（ACL 2025）
- **分解策略**：全自主、無 SOP 的遞迴分解。Boss → Admin → Ordinary，admin 按需招募子 agent
- **時間複雜度**：O(log n) vs. 順序的 O(n)
- **依賴管理**：組內 chat（協作）+ 組間 chat via admin（跨組依賴）
- **規模**：最多 590 個 agent 的國家政策模擬
- 論文：https://arxiv.org/abs/2408.09955

#### AgentOrchestra（arXiv:2506.12508，GAIA SOTA 89.04%）
- **分解策略**：Planning agent + TEA 協議。todo 工具追蹤每個子任務（id、描述、參數、優先級、狀態、結果）
- **四種專業 subagent**：Deep Researcher（網路搜尋）、Browser Use Agent（網頁互動）、Deep Analyzer（多模態推理）、Tool Manager（動態建立新工具）
- **聚合**：Planning agent 匯總回饋，動態更新計劃，呼叫 done 工具標示完成
- 論文：https://arxiv.org/abs/2506.12508

---

### 依賴管理的五種模式

| 模式 | 說明 | 使用框架 |
|---|---|---|
| **順序閘控** | B 只在 A 的輸出落入共享狀態後才啟動 | LangGraph reducers, CrewAI context |
| **動態重規劃** | 每個子任務完成後根據實際結果修正剩餘任務 | TDAG, AgentOrchestra, MegaAgent |
| **發佈-訂閱** | Agent 訂閱相關訊息類型，只消費所需 | MetaGPT 共享訊息池 |
| **Contract Net** | Orchestrator 廣播任務，agent 競標，贏家被指派 | 古典 HMAS 方法 |
| **黑板 / 共享狀態** | 所有 agent 讀寫同一資料結構，Reducer 解衝突 | LangGraph state, Swarm context |

---

### 結果聚合的七種模式

| 聚合方法 | 最適用 | 使用在 |
|---|---|---|
| **LLM 合成** | 研究、分析、開放式任務 | Anthropic 系統, AgentOrchestra |
| **投票 / 多數決** | 分類、二元決策 | DyLAN, Group Chat |
| **加權合併** | 帶分數的推薦 | 含品質權重的並發協調 |
| **結構化接力** | 線性管線，前一輸出為下一輸入 | CrewAI sequential, MetaGPT |
| **Pydantic 驗證** | 可靠的結構化輸出 | CrewAI `output_pydantic`, AutoGen reflection |
| **層次融合** | 大規模層次系統 | MegaAgent, HMAS |
| **黑板整合** | 鬆耦合團隊 | Swarm 架構 |

---

### 主要失敗模式

1. **Agent 大雜燴（Bag of Agents）**：無結構拓撲 → 錯誤放大 17 倍
2. **模糊分解**：子任務沒有 agent 能解決（違反可解性）
3. **完整性缺口**：分解丟失原始問題資訊
4. **冗餘子任務**：多個 agent 調查同一角度，產生衝突答案
5. **Context Window 瓶頸**：Orchestrator-Worker 超過 50 個子結果時，orchestrator 成為限制
6. **原子性失敗傳播**：LangGraph 並行 superstep 一個分支失敗，全部失敗
7. **層次延遲堆疊**：每層至少 2-3 秒，4 層層次系統有 10 秒基礎延遲

---

### 五種模式的決策矩陣（拓撲選擇）

| 任務特性 | 使用模式 | 注意事項 |
|---|---|---|
| 獨立子任務，無需跨 agent 通訊 | Orchestrator-Worker | 超過 50 個結果時 orchestrator context 瓶頸 |
| 有清晰階段邊界的固定序列 | Pipeline | 5 個階段最少 10+ 秒 |
| 3–8 個緊密協作者迭代共享成果 | Mesh | 超過 8 個 agent 組合爆炸 |
| 大型未知問題空間需要探索 | Swarm | 可觀察性差，難以強制排序 |
| 20+ agent 的多領域企業規模 | Hierarchical | 每層最少 2-3 秒延遲 |

---

## 學術調查論文（2024-2026 精選）

| 論文 | 核心貢獻 | 連結 |
|---|---|---|
| LLM Multi-Agent Survey (IJCAI 2024) | 三種推理框架分類：多階段串行、集體決策、自我精煉 | [arXiv:2402.01680](https://arxiv.org/abs/2402.01680) |
| LLM-MAS 近期進展 (2025/01) | 分類應用：任務解決、模擬、評估；指出依賴管理仍未解決 | [arXiv:2412.17481](https://arxiv.org/abs/2412.17481) |
| LLM Multi-Agent 挑戰 (2025/05) | 四大對齊挑戰；7 種單 agent 分解方法分類 | [arXiv:2402.03578](https://arxiv.org/html/2402.03578v2) |
| HMAS 分類學 (2025) | 五軸特徵化 + 六種協調機制 | [arXiv:2508.12683](https://arxiv.org/html/2508.12683) |
| 多 Agent 協調架構與協議 (2026/01) | 三層架構：專業 agent + 協調層 + 通訊協議（MCP+A2A） | [arXiv:2601.13671](https://arxiv.org/html/2601.13671v1) |
| AOP 框架 (2024/10) | 可解性、完整性、非冗餘性三原則 | [arXiv:2410.02189](https://arxiv.org/html/2410.02189v2) |
| MetaGPT (ICLR 2024 Oral) | SOP 角色編碼，結構化文件通訊 | [arXiv:2308.00352](https://arxiv.org/abs/2308.00352) |
| MegaAgent (ACL 2025) | 自主遞迴分解，590 agent 規模驗證 | [arXiv:2408.09955](https://arxiv.org/abs/2408.09955) |
| AgentOrchestra (2026/01) | TEA 協議，GAIA SOTA 89.04% | [arXiv:2506.12508](https://arxiv.org/abs/2506.12508) |
| TDAG (2025) | DAG 結構 + 動態重規劃（注意：仍為順序執行） | [arXiv:2402.10178](https://arxiv.org/abs/2402.10178) |
| DyLAN (COLM 2024) | 動態 agent 團隊選擇，mid-execution 淘汰低效 agent | [arXiv:2310.02170](https://arxiv.org/abs/2310.02170) |
| DAG 分解與動態策略 (2024) | 顯式提及分解策略可依複雜度自適應，提升準確率 | [arXiv:2410.22457](https://arxiv.org/html/2410.22457v1) |
| ACONIC 限制式分解 (2025) | 以滿足性問題建模任務，樹寬衡量複雜度 | [arXiv:2510.07772](https://arxiv.org/html/2510.07772v1) |
