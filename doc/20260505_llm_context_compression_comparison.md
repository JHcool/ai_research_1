# 各大 LLM 的 Context 壓縮機制比較分析

> 研究日期：2026-05-05  
> 說明：區分「官方文件記載」、「論文公開但機制保留」、「社群推斷」三種可信度層次

---

## 機制全覽（快速對照表）

| 供應商 | 機制 | 類型 | 有損？ | 觸發方式 | 透明度 |
|---|---|---|---|---|---|
| **Anthropic API** | Compaction API | LLM 摘要 | 是 | Token 閾值（可設，預設 150K） | 高（人類可讀摘要） |
| **Anthropic API** | Prompt Caching | KV cache 重用 | 否 | 開發者標記斷點 | 高 |
| **Claude.ai 產品** | 自動摘要 | LLM 摘要 | 是 | Context window 上限 | 無（完全不透明） |
| **OpenAI Assistants** | Thread 截斷 | 中段丟棄 | 是 | `max_prompt_tokens`，自動 | 低（無通知） |
| **OpenAI Responses** | Compaction | 加密 LLM 摘要 | 是 | `compact_threshold`（最低 1K） | 中（有加密 item，但不可讀） |
| **OpenAI o1/o3** | Reasoning budget | 思考 token 預算控制 | 否（預算不是壓縮） | `reasoning_effort` 參數 | 中 |
| **OpenAI o3 stateless** | 加密 Reasoning items | 跨輪推理狀態持久化 | 否 | 每輪，無狀態模式 | 低（加密不可讀） |
| **ChatGPT 產品** | 靜默截斷 | 最舊優先丟棄 | 是 | 32K 產品限制 | 無 |
| **Gemini API** | 明確 Context Caching | KV cache 重用 | 否 | 開發者建立 cache 物件 | 高 |
| **Gemini 2.5+ API** | 隱式 Caching | 自動 KV cache 重用 | 否 | 自動（最低 1K–4K tokens） | 中（usage_metadata 有計數） |
| **Gemini 長文本** | Context Parallelism | 分散式推理 | 否 | 硬體規模決定 | — |
| **LLaMA 3 / vLLM** | PagedAttention + prefix cache | 分頁 KV 重用 | 否 | LRU 驅逐（記憶體滿時） | 中 |
| **llama.cpp** | Rolling window | 前半段丟棄 | 是 | n_ctx 超限 | 無 |
| **Ollama** | 最舊優先靜默丟棄 | 截斷 | 是 | num_ctx 超限 | 無 |
| **Mistral 7B v0.1** | SWA（滑動窗口注意力） | 架構層窗口限制 | 部分 | 每層持續作用 | — |
| **Mistral Large 2+** | Full attention + GQA | KV head 縮減 | 否 | 恆常作用 | — |
| **Cohere Command R+** | 外部 RAG + Rerank | 事前過濾 | 否 | 外部檢索管線 | 高 |
| **DeepSeek（MLA）** | 低秩 KV 投影 | 架構層壓縮 | 近似無損 | 恆常作用 | — |

---

## 1. Anthropic (Claude)

### 1.1 Compaction API（官方文件記載）

**機制（演算法層面）**：

伺服器端摘要管線，整合於 Messages API，需要 `compact-2026-01-12` beta header。

觸發後的執行序列：
1. 偵測 input tokens 超過閾值（預設 150,000）
2. Claude **以同一個模型**生成對話歷史的摘要，包裹在 `<summary></summary>` 標籤中
3. 摘要作為 `compaction` block 在 response 中回傳
4. 後續請求中，compaction block 之前的**所有訊息自動丟棄**（伺服器端永久刪除）
5. 多次壓縮時，後一個 compaction block 取代前一個

**API 參數結構**：
```json
{
  "context_management": {
    "edits": [{
      "type": "compact_20260112",
      "trigger": {"type": "input_tokens", "value": 150000},
      "pause_after_compaction": false,
      "instructions": null
    }]
  }
}
```

**官方摘要 Prompt（逐字）**：
> "You have written a partial transcript for the initial task above. Please write a summary of the transcript. The purpose of this summary is to provide continuity so you can continue to make progress towards solving the task in a future context...Write down anything that would be helpful, including the state, next steps, learnings etc."

這是**任務導向的前向摘要**，不是對話重現。`instructions` 參數可完全替換此 prompt。

**`pause_after_compaction: true`**：在步驟 2 和 4 之間暫停，以 `stop_reason: "compaction"` 回傳，讓開發者注入額外 context 再繼續——可實現混合策略（摘要 + 保留特定近期訊息）。

**優點**：
- 品質最佳：摘要由主模型本身生成（非小型摘要模型），語義忠實度最高
- 最透明：摘要為人類可讀，開發者可完整審查
- 可客製化：`instructions` 可限制摘要重點（如「保留所有程式碼片段」）
- 支援 ZDR（Zero Data Retention）合規

**缺點**：
- 有損（lossy）：原始歷史不可復原
- 觸發時增加延遲：需額外一次完整 LLM pass
- 最低閾值 50K tokens，細粒度控制有限
- 品質取決於模型的摘要能力，無正式資訊保留保證

### 1.2 Prompt Caching（官方文件記載，非壓縮機制）

KV cache 重用，不是壓縮——模型仍處理完整 context，只省去重計算。

- 開發者以 `cache_control: {type: "ephemeral"}` 標記斷點
- 儲存斷點前所有 attention head 和層的 KV tensor
- 相同前綴的後續請求直接重用，無需重算
- Cache TTL：5 分鐘（每次命中自動更新）
- 最低閾值：Sonnet/Haiku 1,024 tokens；Opus 2,048–4,096 tokens
- 費率：寫入 +25% 溢價；命中讀取約為正常輸入價的 10%（$0.50/M vs $5/M）
- 前綴必須完全相同，差一個 token 即完全 cache miss

### 1.3 Claude.ai 產品 vs. API（官方幫助文件記載）

**重要差異**：

| 面向 | Compaction API | Claude.ai 產品 |
|---|---|---|
| 摘要 prompt | 開發者可自訂 | 固定，不可更改 |
| 觸發通知 | 回傳 compaction block | 用戶不會收到任何通知 |
| 開發者控制鉤 | `pause_after_compaction` | 無 |
| 透明度 | 高 | 零 |
| 計費影響 | 計入用量 | 不計入用量 |

**結論**：Claude.ai 的自動摘要對用戶完全不透明，是目前主要供應商中透明度最低的產品層壓縮實作。

---

## 2. OpenAI (GPT-4o, o1/o3, GPT-4.1)

### 2.1 Responses API Compaction（官方文件記載，2026 年 2 月發布）

最接近 Anthropic Compaction API 的對等功能。

**機制**：
- 啟用方式：請求中加入 `context_management` 參數，設定 `compact_threshold`（最低 1,000 tokens）
- 觸發時：伺服器生成一個**加密的 compaction item**，嵌入 response stream
- 後續請求必須把 compaction item 傳回，伺服器解密後重建狀態

**與 Anthropic 的關鍵差異**：Compaction item 是**加密的，非人類可讀**。OpenAI 描述為「攜帶關鍵先前狀態與推理，但 token 更少」，但不公開摘要 prompt 或格式。

**獨立端點**：`/responses/compact`——接受完整 context，返回已壓縮的版本，適合在長 agent 迴圈前預先壓縮。

**優點**：
- 機密性更高：摘要內容對開發者不可見（隱私場景有優勢）
- 最低觸發閾值低（1K tokens），比 Anthropic 更細粒度
- 支援 ZDR（`store=false`）

**缺點**：
- 透明度最低的「有損摘要」實作：開發者無法審查什麼被保留了
- 無法客製化摘要邏輯
- 如果加密摘要的品質有問題，開發者無從知曉

### 2.2 Assistants API Thread 截斷（官方文件記載）

**兩種截斷策略**：

- **`auto`（預設）**：丟棄對話中段的訊息以符合 `max_prompt_tokens`。保留最近的訊息與最早（通常是 system）訊息，中間段被丟棄。
- **`last_messages`**：明確指定保留最近 N 條訊息的整數值。

**透明度**：無通知機制。開發者只能靠比對 `usage` 計數或觀察到 context 遺失才知道截斷發生了。

**無摘要**：純截斷，不生成任何摘要替代丟失的內容。

**優點**：零額外延遲，零額外成本。

**缺點**：最原始的有損處理。中段截斷對任務連續性的破壞是不可預測的——取決於中段內容的重要性。

### 2.3 ChatGPT 產品（社群推斷）

**靜默截斷**：ChatGPT 介面將 GPT-4o 限制在約 32K tokens（儘管 API 支援 128K，GPT-4.1 支援 1M）。超出後靜默丟棄最舊訊息，**無任何通知**。沒有自動摘要介入。

這意味著 ChatGPT 的有效對話記憶大約是 API 能力的 25%，且用戶不知道何時失憶。

### 2.4 o1/o3/o4 推理模型的特殊機制（官方文件記載）

**推理 token 的 context 佔用**：

- 推理 token 在推理時生成，**不在 API response 中回傳**，但**佔用 context window** 並**按 output token 計費**
- 複雜任務可能生成數千至數萬個推理 token
- `reasoning_effort` 參數（`none/minimal/low/medium/high/xhigh`）控制分配的推理 token 數量——這是**預算控制，不是壓縮**

**加密推理 items（o3/o4-mini 限定）**：

- Responses API 可透過 `include: ["reasoning.encrypted_content"]` 回傳加密推理 item
- 後續輪次傳回加密 item，讓模型存取先前推理狀態
- `store=false` 模式：加密內容在記憶體中解密使用，從不寫入磁碟，新推理 token 立即重新加密回傳
- **這不是壓縮**，是推理狀態的跨輪持久化（解決隱私合規問題）

---

## 3. Google (Gemini 1.5/2.0/2.5)

### 3.1 長文本架構：Context Parallelism（論文公開，機制保留）

Gemini 的策略與其他供應商從根本上不同：**做超大 context window，而非做壓縮**。

**Context Parallelism 基礎設施**（arXiv:2411.01783）：

處理百萬 token 的核心是分散式推理——跨多個 GPU 分配 KV cache 與注意力計算：

| 算法 | 流通的資料 | 最優使用場景 |
|---|---|---|
| **Pass-KV** | KV tensor 在 GPU ring 中環流 | 完整預填充（prefill）場景 |
| **Pass-Q** | Query embedding 環流 | 解碼期 + prefix cache 命中率高時 |

切換閾值判斷式：`T/(T+P) ≤ 2·(NKV/NH)`  
（T = 新 tokens，P = 已快取 tokens，NKV = KV heads，NH = query heads）

**實測數據**：
- 128 H100 上 405B 模型，1M token prefill：**77 秒**，並行化效率 93%
- 128K context：**3.8 秒**

**這是無損的分散式計算，不是壓縮**。完整 context 被完整處理。

**注意**：Gemini 1.5 的 99%+ recall（≤1M tokens）的具體架構機制，Google 未公開。論文描述了結果，但「一系列重大架構變更」的細節被視為商業機密。

### 3.2 明確 Context Caching（官方文件記載）

開發者主動管理的 KV cache 重用：

- 把大型前綴（system prompt、長文件、影片、音訊）送至 `/cachedContents` API
- Google 存儲為命名 cache 物件，可設 TTL（預設 1 小時）
- 後續請求引用 cache object；快取 token 享 **75% 折扣**（Gemini 2.5 為 90%）
- 最低閾值：Gemini 2.5 Flash 1,024 tokens；Gemini 2.5 Pro 4,096 tokens
- **限制**：快取內容只能作為 prompt 前綴，不能出現在對話中間

**這是無損效率優化，不是壓縮。**

### 3.3 隱式 Caching（Gemini 2.5+，官方文件記載，2025 年 5 月）

**對比 Anthropic prompt caching 的最大差異**：Gemini 不需要開發者標記斷點，**基礎設施自動識別跨請求的共同前綴**並應用快取折扣。

- 預設啟用，對所有 Gemini 2.5+ 模型生效
- Gemini 2.5 折扣 90%（vs. Anthropic 的讀取 90%）
- 回傳的 `usage_metadata.cached_content_token_count` 提供透明度

### 3.4 無應用層截斷或摘要

Gemini API **不提供**任何等同於 Anthropic Compaction API 或 OpenAI Assistants 截斷的功能。超出 context window → API 直接回傳錯誤，無自動降級。

Google 的策略哲學：**用 2M context window 跳過 RAG 工作流**——context 夠大，壓縮和分塊是不必要的。

---

## 4. Meta (LLaMA 3.x)

Meta 在模型層面提供架構優化，但不提供任何 API 層的壓縮管理——context 管理完全交給推理框架與應用開發者。

### 4.1 LLaMA 3 模型層架構（官方文件記載）

**Grouped Query Attention (GQA)**：
- LLaMA 3.1 70B：64 個 query heads，8 個 KV heads → KV cache 記憶體需求減少 **8 倍**（vs. 標準 MHA）
- LLaMA 3.1 8B：相同比例
- **無滑動窗口注意力**：LLaMA 3 使用全注意力覆蓋 128K window

**KV cache 記憶體估算（FP16）**：
| 模型 | 128K context | 加 FP8 量化後 |
|---|---|---|
| LLaMA 3.1 70B | ~40 GB / 用戶 | ~20 GB |
| LLaMA 3.1 8B | ~16 GB / 用戶 | ~8 GB |

Meta 在 API 層沒有發布任何摘要或壓縮機制。

### 4.2 vLLM（主流生產推理框架）

**PagedAttention（論文/文件記載）**：

KV cache 分成固定大小的**物理 block**（預設 16 tokens）。Block 以頁表間接存取，可分散在 GPU RAM 非連續位置，記憶體碎片化 < 4%。

**自動前綴快取（APC）**：

- 以 block 內容的 hash 值去重
- 相同前綴的請求重用已存的物理 block
- agent 迴圈 / 多租戶 SaaS / 長文件 Q&A 場景命中率 60–85%
- 這是推理層的 KV 重用，類比 Anthropic prompt caching，但在開源推理框架層實現

**KV cache 驅逐策略**：
- 當物理記憶體耗盡，以 LRU 驅逐 `reference_count == 0` 的 block
- **無內容感知驅逐**：不看 attention score，不知道哪個 block 語義上更重要

**KV cache 量化**：`kv_cache_dtype="fp8_e4m3"` 可減少 50% cache 記憶體，品質影響輕微。

### 4.3 llama.cpp

**Rolling window（預設）**：

超出 `n_ctx` 時，丟棄前半段非保留 token，靜默截斷，無摘要。

**RoPE 擴展（位置外推，非壓縮）**：

```bash
--rope-scaling yarn --rope-scale 4 --yarn-orig-ctx 32768
```

YaRN、LongRoPE 等技術讓模型能處理超過訓練長度的序列，但這是位置編碼的外推，不是壓縮。品質隨長度增加會有不同程度的降級。

### 4.4 Ollama

最差的透明度設計：超出 `num_ctx`（預設 4,096）時，**靜默丟棄最舊訊息，無任何警告或錯誤**。應用開發者若未手動設定 `num_ctx`，大量訊息會無聲消失。

---

## 5. Mistral

### 5.1 Mistral 7B v0.1 — 滑動窗口注意力（SWA）（論文記載：arXiv:2310.06825）

**機制**：每個注意力層只關注前 4,096 個 tokens（非完整序列），使用 rolling buffer。

- 線性計算複雜度：O(seq_len × window_size) vs. O(seq_len²)
- 理論有效注意力跨度：4,096 × 32 層 = 131,072 tokens（通過層間資訊遞傳）
- KV cache 固定大小：只需 sliding_window 個 token 的 cache

**SWA 在 v0.2 中被移除**：Mistral 7B v0.2 以 32K context + 全注意力 + `rope_theta = 1e6` 取代 SWA。這說明 SWA 的計算效益在大規模下不抵品質損失。

### 5.2 Mistral Large 2（123B，128K context）

- **全注意力**（已確認，非 SWA）
- **GQA**：48 個 attention heads，8 個 KV heads → KV cache 減少 **6 倍**（vs. 全 MHA）
- Flash Attention 推理優化
- **API 層無壓縮功能**：context 超限直接報錯，開發者自行管理

### 5.3 Mistral Large 3（2025，675B total / 41B active，MoE）

256K context window。無公開的 context 管理機制文件。

---

## 6. 其他值得關注的方法

### 6.1 Cohere Command R+ — RAG-First 架構

**根本哲學不同：不壓縮，而是在裝入 context 之前就過濾掉不相關的內容。**

- 128K context window，104B 參數
- 模型通過 SFT + 偏好優化，原生支援 grounded generation（從檢索片段生成並產生 inline citation）
- **Rerank API**：獨立的 cross-encoder 模型對檢索到的 chunk 重排序，只把最相關的幾個 chunk 送進 context window

**優點**：
- 完全無損：不壓縮，只選擇最相關的
- 長文件場景比 1M token window 更具成本效益
- Citation 可審計，生成可溯源

**缺點**：
- 需要完整的 RAG 基礎設施（向量資料庫 + 索引 + 檢索管線）
- 增加檢索延遲
- 對話歷史（非文件）的處理仍是標準截斷

### 6.2 xAI Grok

- Grok 4.1：256K tokens；Grok 4.1 Fast：聲稱 2M tokens
- **「智慧記憶」（smart memory）**：超出 window 時使用滑動窗口設計，丟棄最舊 token
- **可信度低**：這是 xAI 的行銷描述，無技術論文或 API 文件確認具體機制。社群推斷為標準滑動窗口截斷，可能加上 attention score 優先級排序，但未證實

### 6.3 DeepSeek — Multi-Head Latent Attention (MLA)（論文記載：arXiv:2405.04434）

目前架構層 KV cache 壓縮最先進的生產部署方案，現已被多個推理框架採用。

**機制**：

標準 MHA/GQA 快取的是完整解析度的 K/V tensor。MLA 改為快取**低秩壓縮後的潛在表示**，在注意力計算時通過 up-projection 重建可用的 K/V：

```
標準 MHA：快取 K/V（高維，每層、每頭完整儲存）
MLA：快取 latent vector（低秩，上投影時重建 K/V）
```

**效果**：
- 相比 DeepSeek 67B（MHA 基線）：KV cache 減少 **93.3%**
- 吞吐量提升 **5.76 倍**
- 結合 FP8：KV cache 從 135 GB 降至 8 GB（**17 倍縮減**）
- MLA 性能等同於或優於 GQA，而壓縮比遠更高

**這是架構層的近似無損壓縮**——latent vector 在數值精度範圍內忠實重建 K/V，不是語義有損的摘要或截斷。現在由 vLLM、SGLang 等框架支援。

---

## 深度比較分析

### 四類機制的本質差異

```
1. LLM 摘要型（Anthropic Compaction、OpenAI Responses Compaction）
   優點：任務連續性最佳，智能選擇保留什麼
   缺點：有損（原始歷史不可恢復），觸發時增加延遲與成本

2. 截斷型（OpenAI Assistants、ChatGPT 產品、llama.cpp、Ollama）
   優點：零額外延遲，零額外成本
   缺點：最粗暴的有損處理，破壞任務連續性，透明度極低

3. KV Cache 重用型（Anthropic Prompt Caching、Gemini Caching）
   優點：完全無損，大幅降低重複前綴的延遲與成本
   缺點：不是真正的壓縮，不解決 context window 超限問題

4. 架構層型（DeepSeek MLA、GQA、SWA）
   優點：恆常作用，近無損，無 API 層開發者負擔
   缺點：需要模型層實作，現有模型無法後期套用，開發者無法控制
```

### 透明度詳細比較

| 層次 | 供應商 / 方案 | 開發者能看到什麼 |
|---|---|---|
| **最高** | Anthropic Compaction API | 完整的人類可讀摘要 block，可審查、可客製化 |
| **中** | OpenAI Responses Compaction | 加密 item（知道發生了，但看不到內容） |
| **中** | Gemini Implicit Caching | `usage_metadata.cached_content_token_count` 計數 |
| **低** | OpenAI Assistants 截斷 | 無通知，只能靠 usage 差異推斷 |
| **無** | ChatGPT 產品 | 靜默截斷，用戶不知道 |
| **無** | Claude.ai 產品 | 靜默摘要，用戶不知道 |
| **無** | Ollama | 靜默截斷，開發者不知道 |

### 延遲與成本影響

| 機制 | 觸發延遲 | 成本影響 |
|---|---|---|
| LLM 摘要（Compaction） | +數秒（完整 LLM pass） | +摘要 token 成本；後續 turn 省 token |
| KV cache 重用 | 寫入 +25%；命中 -90% | 重複前綴場景大幅節省 |
| Context Parallelism（Gemini） | 1M token prefill ~77s（128 H100） | 極高硬體成本，但可服務大客戶 |
| 截斷 | 零 | 零 |
| GQA/MLA（架構層） | 零（推理時間縮短） | 降低 VRAM 需求，間接降低硬體成本 |
| RAG + Rerank（Cohere） | +檢索延遲（數百毫秒） | 外部向量庫成本 |

### 對 Agent 系統的適用性

| 場景 | 推薦方案 | 原因 |
|---|---|---|
| 長對話 Agent 任務連續性 | Anthropic Compaction API | 唯一可客製化的 LLM 摘要，透明度最高 |
| 大型文件一次性 Q&A | Gemini 2M context + Implicit Caching | 無損，自動快取，無需壓縮 |
| 多租戶 SaaS 高並發 | vLLM + PagedAttention + APC | 分頁管理 + 前綴共享，記憶體效率最高 |
| 企業長文件知識庫 | Cohere Command R+ RAG | 避免把無關內容放入 context |
| 需要隱私合規的推理 | OpenAI o3 加密 reasoning items | 推理狀態跨輪持久化，且從不明文存儲 |
| 本地部署 / edge | llama.cpp + YaRN | 唯一支援 context 長度外推的本地選項 |

---

## 可信度聲明

| 資訊 | 來源類型 |
|---|---|
| Anthropic Compaction API 細節 | 官方 API 文件 |
| OpenAI Responses/Assistants 截斷策略 | 官方 API 文件 |
| Gemini Implicit Caching | 官方文件 + Google 部落格 |
| Context Parallelism 算法 | Google 研究論文（arXiv:2411.01783） |
| Gemini 1.5 長文本架構機制 | 論文公開結果，架構**保留** |
| LLaMA 3 GQA + vLLM PagedAttention | 官方模型卡 + vLLM 文件 |
| DeepSeek MLA | 官方技術報告（arXiv:2405.04434） |
| ChatGPT 產品 32K 截斷 | 社群觀察 + 已知 API/產品差距推斷 |
| Grok「智慧記憶」機制 | 行銷描述，無技術文件，社群推斷 |
| Claude.ai 自動摘要 | Anthropic 幫助文件（功能描述，無技術細節） |

---

## 關鍵論文與文件

| 資源 | 連結 |
|---|---|
| Anthropic Compaction API | https://platform.claude.com/docs/en/build-with-claude/compaction |
| OpenAI Responses Compaction | https://developers.openai.com/api/docs/guides/compaction |
| OpenAI Assistants 截斷 | https://platform.openai.com/docs/assistants/deep-dive |
| Gemini Context Caching | https://ai.google.dev/gemini-api/docs/caching |
| Gemini Context Parallelism | https://arxiv.org/html/2411.01783v1 |
| Gemini 1.5 Technical Report | https://arxiv.org/abs/2403.05530 |
| vLLM PagedAttention | https://docs.vllm.ai/en/stable/design/prefix_caching/ |
| DeepSeek-V2 MLA | https://arxiv.org/html/2405.04434v4 |
| Mistral 7B SWA | https://arxiv.org/abs/2310.06825 |
| TransMLA（MLA 遷移） | https://arxiv.org/abs/2502.07864 |
