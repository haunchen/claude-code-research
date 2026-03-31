# Claude Code 設計缺陷與改進機會

> 整合分析來源：phase-01 至 phase-10 架構報告 + 行為研究報告（system-reminder-injection、auto-mode-classifier-cost、context-lifecycle-management、prompt-cache-architecture、tool-serialization-cache-stability）
>
> 報告日期：2026-03-31

---

## 前言

本報告從已完成的靜態逆向與行為研究中，系統性地提取「明顯設計有問題」或「有更簡單更好做法但沒這樣做」的部分。每項問題均附有原始碼或數據依據。問題按嚴重程度排序，優先列出直接造成額外 token 消耗或嚴重影響體驗的設計決策。

---

## 問題清單

---

### 問題 1：Auto Mode 分類器繼承主對話模型，Max 用戶每次工具操作都在燒 Opus

**嚴重程度：** 🔴 高

**現狀：**
Auto Mode 的分類器模型由以下邏輯決定（來源：`auto-mode-classifier-cost/report-zh.md`，函式 `lb4()`）：

```js
function lb4() {
  let config = featureFlag("tengu_auto_mode_config", {});
  if (config?.model) return config.model;
  return getMainLoopModel();  // 繼承主對話模型
}

function KG() {
  if (isMaxSubscription())    return "claude-opus-4-6[1m]";
  if (isTeam5xSubscription()) return "claude-opus-4-6[1m]";
  return "claude-sonnet-4-6";
}
```

在 v2.1.85 以前，分類器寫死使用 Sonnet。v2.1.88 改為繼承主對話模型後，Max 訂閱用戶的每次分類器呼叫成本暴增約 5 倍。

分類器在每次有副作用的工具操作（Edit、Write、Bash、MCP 工具等）前都會觸發，且接收整段對話的精簡版（transcript 隨對話線性增長，無斷點快取）。

**問題：**
分類器的任務是做簡單的「允許 / 拒絕 / 詢問」三分類，這是一個極簡的判斷任務。使用與主對話相同的 Opus 等級模型來做這件事，是過度配置。此外 transcript 增長部分沒有 cache 斷點，快取效率隨對話增長而下降（從前段 ~45% 到後段 ~9%）。

估算：一個 20 turn、每 turn 3 次副作用操作的 session，分類器成本佔主對話 15–28%。若開啟 two-stage classifier（`twoStageClassifier: true`），最多可達 1.5–2× 的額外成本，而使用者完全看不到也無法控制。

**更好的做法：**
1. 分類器模型固定為 Haiku（足以做 allow/deny/ask 三分類），無論主對話模型是什麼
2. 在 transcript 的適當位置（如每輪對話末尾）加入 cache 斷點，讓已發生的工具操作記錄能被快取
3. 開放用戶設定分類器模型偏好或關閉分類器，改為全部手動確認或全部自動放行

**預估影響：**
- Max 用戶：分類器成本降低 ~83%（Opus → Haiku，成本比約 15:1）
- 中型 session（20 turns）：節省 $3–6 的分類器費用
- Two-stage classifier 被開啟時效果加倍

---

### 問題 2：file modification system-reminder 在 Agent SDK 中每輪必然觸發，設計存在根本性 Bug

**嚴重程度：** 🔴 高

**現狀：**
`readFileState` 在 Agent SDK 的 `submitMessage()` 中作為區域變數，每次呼叫都從 JSONL 全量重建（來源：`system-reminder-injection/report-zh.md`）：

```
submitMessage() 被呼叫
  → readFileState = rebuild(this.mutableMessages)  ← 從 JSONL 全量重建，max=10
  → stale check → 注入 system-reminder (diff 內容)
  → 主迴圈跑完，Edit/Write 更新了記憶體中的 readFileState
  → 下一次 submitMessage()
  → readFileState = rebuild(this.mutableMessages)  ← 又從頭重建，之前的更新全丟
  → 永遠注入
```

重建函數的四個結構性問題：
1. Edit 操作完全被忽略（只處理 Read 和 Write）
2. 帶預設 `offset=1` 的 Read 被忽略（條件要求 `offset === undefined`，但預設是 1）
3. 重建的 entry 全部 `offset: undefined`，全部以過去時間戳 → stale check 必觸發
4. Agent SDK 上限只有 10 筆（CLI 是 100 筆）

**問題：**
這導致：
- Agent SDK 每輪穩定注入 file modification system-reminder，這是結構性 bug 而非正常設計
- CLI resume 後也幾乎必定觸發（時間戳邏輯）
- 有 30+ 個 GitHub open issues 反映此問題（最多留言 23 條，最嚴重案例佔 15%+ context window）
- v2.1.70 惡化到整個檔案內容被注入（1300 行），v2.1.71 確認 Edit 後未 re-read 的檔案永遠被視為 stale

系統設計了 15+ 種 system-reminder，有些有合理用途（如通知模型檔案被外部修改），但 file modification reminder 因追蹤 Bug 已失去原有語意，退化成了「無差別 token 消耗器」。

**更好的做法：**
1. 修復根本 Bug：`readFileState` 應是 session 級的持久化狀態，而非每次 `submitMessage()` 的區域變數
2. 修復重建函數：處理 Edit 操作，正確處理帶 offset 的 Read
3. Stale check 觸發後應更新原始 key 的 content 和 timestamp，防止下輪重複觸發
4. 提供 `--no-system-reminders` 或 per-type 開關（Issue #9769，2025-10 開到今天未解決）
5. Agent SDK 的快取上限從 10 提升至與 CLI 相同的 100

**預估影響：**
- 修復後：消除 Agent SDK 每輪強制注入的 diff 內容（視修改的檔案大小而定，可節省 30–數千 token/輪）
- 修復路徑 key 不一致 Bug：消除最惡劣情況下的 1300 行無限注入
- 對長 session 的 Agent SDK 用戶：可節省 context window 的 5–15%

---

### 問題 3：checkQuotaStatus 在 session 啟動時發送一個 max_tokens:1 的探測請求，消耗一次 API 呼叫

**嚴重程度：** 🔴 高

**現狀：**
Session 啟動時，`checkQuotaStatus` 發送一個最小探測請求（來源：`phase-10-cost-quota/02-rate-limiting-mechanism.md`）：

```typescript
async function makeTestQuery() {
  const model = getSmallFastModel()  // 使用最便宜的模型
  return anthropic.beta.messages
    .create({
      model, max_tokens: 1,
      messages: [{ role: 'user', content: 'quota' }],
    })
    .asResponse()
}
```

目的是在 session 一開始就獲取當前額度狀態（通過 response headers），而不是等到第一次真實請求才知道。

**問題：**
這個探測請求雖然使用最便宜的模型且 max_tokens 只有 1，但仍然：
1. 消耗一次 API 呼叫的 input tokens（system prompt + 'quota' 訊息的完整 context）
2. 對 API key 計費用戶，每次啟動 Claude Code 都有一筆小費用
3. 這個探測本身無法被 prompt cache 命中（全新請求，無 cache prefix）
4. 非互動模式（`-p`）已正確跳過此邏輯，但互動模式未跳過

更根本的問題是：rate limit 資訊本已在每次真實 API response 的 headers 中回傳（`anthropic-ratelimit-unified-*`），理論上第一次真實請求就能獲得完整額度狀態。探測請求是在「為了更早知道狀態」而提前支付成本。

**更好的做法：**
1. 移除啟動探測，改為在第一次真實請求的 response headers 中讀取額度狀態
2. 如果需要提前知道（例如 UI 要顯示初始狀態），改用伺服器端 status endpoint（不觸發計費的純 GET 查詢），而非偽造一個 LLM 訊息請求
3. 若保留探測，至少對使用者顯示此行為（現在完全不透明）

**預估影響：**
- 每次啟動 Claude Code 節省 1 次 API 呼叫
- 對頻繁啟動工作流程（CI/CD、腳本化使用）效果顯著
- API key 用戶的帳單減少一筆固定啟動成本

---

### 問題 4：Compaction 後 prompt cache 全面失效，且必定付 cache_write 費率

**嚴重程度：** 🔴 高

**現狀：**
Compaction 執行後有一個設計上的 cache 破壞機制（來源：`context-lifecycle-management/report-zh.md`）：

```
壓縮期間：skipCacheWrite flag = true（正確，不建立暫時快取）
壓縮完成：session 狀態重置
→ 所有現有 prompt cache 斷點全部失效
→ 壓縮後第一輪：所有 token 以 125% cache_write 費率計費
→ 第二輪起才恢復 cache_read（10% 費率）
```

更嚴重的是連鎖反應：如果壓縮後的摘要仍超過 autocompact 閾值，會再次觸發壓縮，每次都伴隨一次完整的 cache 重建。Circuit breaker 在 3 次連續失敗後才停止。

對 160,000 token 的 session，一次壓縮加壓縮後的 cache 重建費用約等於 5–8 次正常輪次的成本。

**問題：**
Compaction 是為了「節省後續成本」，但設計上它本身就保證了一次高成本（壓縮 API 呼叫 + cache 全面重建）。更嚴重的是：
1. 壓縮閾值（163,000 tokens on 200k model）是硬編碼常數，沒有 UI 設定
2. 壓縮後 token 數沒有被預先驗證是否低於閾值，連鎖反應是可預見的設計缺陷
3. 使用者完全不透明：不知道壓縮發生了，不知道成本，不知道連鎖反應
4. 保留最近 10,000–40,000 tokens 的邏輯也是硬編碼，無法設定

**更好的做法：**
1. 壓縮前先驗算「如果此 context 被壓縮為 N tokens 的摘要，是否仍超過閾值」，如果是，就更激進地壓縮（更早觸發或更大幅縮減）
2. 開放 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 環境變數的 UI 設定（現在只有環境變數）
3. 壓縮後第一輪標記為「需要 cache warmup」，在 UI 上提示使用者成本較高
4. 允許 compaction 後保留部分 cache 斷點（例如 system prompt 部分不需要重建）

**預估影響：**
- 消除連鎖反應：節省 2–3× 壓縮費用
- 每次壓縮後第一輪：節省 125% → 10% 的快取費率差異（約節省 50–80% 的一輪成本）
- 對長 session 自動化 agent：年化成本可能降低 20–40%

---

### 問題 5：日期字串被串接進靜態 context 中，每天午夜造成 cache 全面失效

**嚴重程度：** 🔴 高

**現狀：**
每輪 system-reminder 注入當前日期（來源：`context-lifecycle-management/report-zh.md`）：

```
<system-reminder>
Today's date is 2026-03-27
</system-reminder>
```

問題在於這個日期字串被串接到和 CLAUDE.md 內容、git status 等靜態內容相同的文字區塊中，合併後才送 API。

由於 prompt cache 使用嚴格的前綴比對，日期字串在靜態 CLAUDE.md 之前出現，導致：
- 午夜後，日期改變 → 合併區塊的序列化內容改變 → 此位置之後的所有 cache 全部失效
- 當天第一個 session 以 125% cache_write 費率重建所有 cached context

**問題：**
這是一個結構性問題，不是功能 Bug。設計者可能沒有意識到「把日期放在靜態內容前面」的後果。對跑整夜任務的自動化 agent，每 24 小時就要付一次完整的 cache 重建費用，費用與對話長度成正比。

**更好的做法：**
1. 日期注入應放在獨立的 content block，且位置在 dynamic boundary 之後（而非混入靜態內容中）
2. 或是只在真正需要的輪次注入（而非每輪），且確保它不影響靜態前綴的 byte 穩定性
3. 最佳做法：日期作為最後一個 system block 注入，讓靜態前綴完整命中 cache

**預估影響：**
- 對跨日 session：消除每天一次的完整 cache 重建費用
- 對長時間自動化 agent（數小時任務）：每天節省一次等同完整對話長度的 cache_write 費用
- 以 Sonnet 費率估算：100k token session 每次跨日重建費用約 $0.038（100k × $3/M × 125%）

---

### 問題 6：Memory 提取子 Agent 每 turn 都執行，且不論是否有新內容值得記憶

**嚴重程度：** 🟡 中

**現狀：**
`ExtractMemories` 在每次 query loop 結束後自動觸發（來源：`phase-05-memory-context/04-memory-extraction.md`）：

```typescript
// tengu_bramble_lintel flag，預設 1（每 turn 都執行）
if (!isTrailingRun) {
  turnsSinceLastExtraction++
  if (turnsSinceLastExtraction < (getFeatureValue_CACHED_MAY_BE_STALE('tengu_bramble_lintel', null) ?? 1)) {
    return
  }
}
```

預設 `tengu_bramble_lintel` 的值是 1，意味著每個 turn 都執行一次記憶提取。提取使用 forked agent，帶 `maxTurns: 5`，最多可消耗 5 次 API 呼叫。

**問題：**
大部分的 user turn 並不產生值得長期記憶的新資訊（例如「好」「繼續」「再跑一次測試」等簡短確認）。對這些 turn 執行記憶提取是純浪費。

雖然有「主 agent 已自己寫記憶則跳過」的互斥邏輯，但這保護的是「重複寫入」而非「不必要的提取」。

**更好的做法：**
1. 在記憶提取前先做輕量估算：若新訊息少於 N 個 token 且無工具呼叫，跳過提取
2. 讓 `tengu_bramble_lintel` 的預設值改為 3–5（每 3–5 個 turn 才提取一次），而非 1
3. 提取判斷可以考慮內容複雜度：只有包含新的技術決策、檔案修改、或用戶明確偏好時才觸發
4. 讓使用者可以在 settings.json 中設定提取頻率

**預估影響：**
- 假設 50% 的 turn 是簡短確認：減少約 50% 的記憶提取 API 呼叫
- 以每次提取消耗 2–5 輪（cache-shared）估算：每 session 可節省數十次 API 呼叫
- 對長 session 的 agent 使用：節省效果更顯著

---

### 問題 7：AgentSummary 定時器每 30 秒就用 fork 呼叫一次 API，對短任務是純浪費

**嚴重程度：** 🟡 中

**現狀：**
`runAgent()` 在啟動時開啟 AgentSummary 定時器，每 30 秒執行一次（來源：`phase-03-agent-architecture/03-agent-lifecycle.md`）：

```typescript
// 啟動 AgentSummary 定時器（每 30s）
// ...
// runAgent 核心迴圈內
// updateAgentSummary（每 30s fork）
```

定時器用來為 UI 顯示 agent 的進度摘要，使用 fork agent 模式呼叫 API。

**問題：**
對完成時間不超過 30 秒的任務（例如簡單的 grep + edit 操作），AgentSummary 可能在任務完成前才剛觸發一次，產生了一次純粹為 UI 顯示服務的 API 呼叫，但使用者甚至來不及看到這個摘要。

**更好的做法：**
1. 改為 on-demand 觸發：只有 UI 真正展示 agent 進度面板時才觸發 summary
2. 對預計短期完成的任務（例如 agent 只剩 1–2 個工具呼叫），跳過 summary 更新
3. 或延長定時器間隔（30s → 60s 或 120s）
4. summary 可以改為基於 token 使用量觸發（例如每消耗 10,000 tokens 才更新一次），而非時間觸發

**預估影響：**
- 消除對短 task（< 30s）的不必要 summary API 呼叫
- 對長時間 agent task：減少 summary 呼叫頻率，可節省 10–20% 的輔助 API 呼叫

---

### 問題 8：Auto Mode Classifier 的 Transcript 包含完整的 file_path + new_string，Write 工具最嚴重

**嚴重程度：** 🟡 中

**現狀：**
分類器 transcript 建構邏輯（來源：`auto-mode-classifier-cost/report-zh.md`，函式 `tb4()`）：

```js
// Write — 路徑 + 完整檔案內容 💸
toAutoClassifierInput(input) { return `${input.file_path}: ${input.content}` }

// Edit — 路徑 + new_string（不含 old_string）
toAutoClassifierInput(input) { return `${input.file_path}: ${input.new_string}` }
```

Write 工具把整個新檔案內容（可能是數百甚至數千行）都送入分類器的 transcript。

**問題：**
分類器的判斷依據是「這個操作是否應該被允許」，它需要的資訊是路徑和操作類型，而不是完整的檔案內容。把 500 行的新檔案內容塞進分類器 transcript 是嚴重的過度資訊，完全不必要。

特別是這些 Write 內容還會出現在**後續每次**分類器呼叫的 transcript 中（歷史累積），導致每次新的工具操作都要為之前所有的 Write 操作付費。

**更好的做法：**
1. Write 的 `toAutoClassifierInput` 應只傳路徑 + 行數（例如 `src/foo.ts (500 lines)`），而非完整內容
2. Edit 的 `toAutoClassifierInput` 應只傳路徑 + diff 摘要（如 `+15/-3 lines`），而非完整 new_string
3. Transcript 歷史應設定大小上限，超過時截斷舊的工具操作（保留最近 N 次即可）

**預估影響：**
- Write 操作：每次分類器呼叫可節省 90%+ 的 transcript tokens（移除完整檔案內容）
- 對大量寫檔的工作流程：整個 session 的分類器成本可降低 50–80%

---

### 問題 9：MCP 伺服器指令（Instructions）每次都是 DANGEROUS_uncachedSystemPromptSection，破壞 cache

**嚴重程度：** 🟡 中

**現狀：**
MCP 指令段被標記為不快取（來源：`phase-01-system-prompt/02-prompt-assembly-logic.md`）：

```typescript
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => isMcpInstructionsDeltaEnabled() ? null : getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns',
)
```

原因是「MCP 伺服器可能在 turn 之間連接/斷開」。但這個設計假設最壞情況（動態連接/斷開）是常態，導致即使 MCP 伺服器在整個 session 中從未變化，其指令仍然每次都重新計算且不快取。

已有一個補丁機制：`isMcpInstructionsDeltaEnabled()`，當啟用時通過 persistent attachments 傳遞 MCP 指令，避免破壞 cache。但這個優化依賴 feature flag，不是預設行為。

**問題：**
對大多數使用者（有固定 MCP 伺服器配置），MCP 伺服器在 session 期間根本不會連接/斷開，但仍然每次都付出重新計算和 cache miss 的代價。

**更好的做法：**
1. 將 `isMcpInstructionsDeltaEnabled()` 設為預設啟用
2. 或改用差異化設計：Session 開始時鎖定（latch）MCP 伺服器列表，只有真正發生變化時才重新計算
3. 或只有伺服器列表發生實際變化時（比對 hash）才更新 system prompt，否則沿用快取版本

**預估影響：**
- 每個 MCP 工具指令通常數百 tokens，每次 cache miss 都要重建 system prompt 的整個動態部分
- 對有多個 MCP 伺服器的用戶：每輪節省 500–2000 tokens 的 cache miss

---

### 問題 10：系統提示中的 `env_info_simple` 包含每次都可能不同的工作目錄資訊，但被設計成可快取的

**嚴重程度：** 🟡 中

**現狀：**
`env_info_simple` section 包含工作目錄、平台、shell、OS 版本、模型描述等資訊（來源：`phase-01-system-prompt/01-main-system-prompt.md`）：

```typescript
const envItems = [
  `Primary working directory: ${cwd}`,
  `Is a git repository: ${isGit}`,
  `Platform: ${env.platform}`,
  getShellInfoLine(),
  `OS Version: ${unameSR}`,
  modelDescription,
  knowledgeCutoffMessage,
]
```

這個 section 被定義為 `systemPromptSection`（`cacheBreak: false`），意味著計算一次後就快取，直到 `/clear` 或 `/compact`。

**問題：**
`cwd`（工作目錄）實際上可以在 session 中改變（使用者可以 cd 進不同目錄，或使用 Worktree 功能）。當這種情況發生時，cached 版本的 env_info_simple 可能過期，導致模型基於錯誤的工作目錄資訊執行操作。

這是一個「快取正確性」問題：為了降低 token 成本而犧牲了準確性。

**更好的做法：**
1. `cwd` 單獨提取成一個獨立 section，設為 `cacheBreak: true`（或動態注入），其餘不變的資訊保持可快取
2. 或 session 中偵測到 cwd 變化時（例如 Worktree 切換），主動清除 env_info_simple 的快取
3. 最輕量的做法：只在 cwd 確實改變時刷新此 section，使用 hash 對比

**預估影響：**
- 主要是正確性問題而非成本問題
- 消除模型在 cwd 變更後執行操作時的潛在混亂
- 對使用 Worktree 功能的用戶：防止錯誤的工作目錄指引

---

### 問題 11：Compaction Prompt 繼承父 Agent 完整工具集（為了維持 cache key），但本質上不需要任何工具

**嚴重程度：** 🟡 中

**現狀：**
壓縮 Fork Agent 繼承父 Agent 的完整工具集（來源：`phase-10-cost-quota/07-compaction-as-cost-saving.md`）：

```
cache-sharing fork path：Compact 請求繼承父 agent 的完整工具集和系統提示，
確保與主 thread 共享同一份 prompt cache，避免為 compact 操作單獨付出 cache creation 費用。
```

這是有意識的設計決策。目的是讓 compact 操作共享主 thread 的 prompt cache（省去重新建立 cache 的成本）。

但代價是 Sonnet 4.6 的 adaptive thinking 有 2.79% 的機率會嘗試工具呼叫，需要用 `NO_TOOLS_PREAMBLE` + `NO_TOOLS_TRAILER` 的雙重強調才能抑制（從 2.79% 降到接近 0.01%）。

**問題：**
2.79% 的工具呼叫嘗試率意味著每 36 次壓縮就有一次失敗，需要回退到備援路徑（另一次 API 呼叫）。這個設計用「cache 節省」換來了「偶發的雙倍壓縮成本」加上「需要複雜的 preamble/trailer 雙重強調」。

本質上，這是用工程複雜度（雙重提醒）來解決一個可以通過架構調整（不繼承工具集）來徹底消除的問題。

**更好的做法：**
1. 評估「不繼承工具集，直接用純文字系統提示做壓縮」的 cache miss 成本，與「繼承工具集導致 2.79% 失敗 + 備援呼叫」的成本對比
2. 如果 cache 節省確實划算，維持現有設計；如果不划算，改為獨立的無工具壓縮 Agent
3. 或使用 structured output API 來強制返回純文字，而非依賴提示強調

**預估影響：**
- 消除每 36 次壓縮中的 1 次備援呼叫
- 減少 compaction prompt 的複雜度（移除 NO_TOOLS_PREAMBLE 和 NO_TOOLS_TRAILER）
- 對高頻壓縮場景（如 1M context 的長任務）：可節省 3–5% 的備援呼叫成本

---

### 問題 12：Skills 列表作為 system-reminder 每輪注入，token 量隨 skill 數量增長

**嚴重程度：** 🟡 中

**現狀：**
可用 skills 列表每輪作為 system-reminder 注入（來源：`system-reminder-injection/report-zh.md`）：

```
invoked skills | 每輪注入可用 skills 列表 | ~2000+ tokens（隨 skill 數量增長）
```

一個 GitHub issue（#27721）報告「Skills 被 system prompt 重複註冊，context 用量翻倍」。

**問題：**
Skills 列表是 session 穩定的資訊（除非使用者安裝/卸載 skill，否則不會變）。把它作為每輪 system-reminder 注入，是在動態部分重複傳遞本可以靜態快取的內容。

特別是當 skill 數量多時（例如安裝了 10+ 個 skill 的進階用戶），每輪注入的 skills 列表可能超過 2000 tokens，且無法被快取（system-reminder 不是系統提示的一部分，不參與 prompt cache 的靜態前綴）。

**更好的做法：**
1. Skills 列表應放在系統提示的靜態部分（dynamic boundary 之前），而非每輪 system-reminder
2. 或只在 skills 列表發生變化時才重新注入
3. 或只在模型真正需要時（例如首次對話、或使用者明確詢問可用指令）才注入完整列表

**預估影響：**
- 對有 5+ skills 的用戶：每輪節省 1000–3000 tokens 的 system-reminder 注入
- 將 skills 移至靜態部分後可被 prompt cache 覆蓋

---

### 問題 13：session 啟動時同時計算多個動態 section，但 env_info_simple 被計算兩次

**嚴重程度：** 🟢 低

**現狀：**
`getSystemPrompt` 的組裝邏輯（來源：`phase-01-system-prompt/02-prompt-assembly-logic.md`）：

```typescript
// 1. 並行計算初始資訊
const [skillToolCommands, outputStyleConfig, envInfo] = await Promise.all([
  getSkillToolCommands(cwd),
  getOutputStyleConfig(),
  computeSimpleEnvInfo(model, additionalWorkingDirectories),  // 第一次
])

// 2. 動態 section 定義中又定義了一次
systemPromptSection('env_info_simple', () =>
  computeSimpleEnvInfo(model, additionalWorkingDirectories),  // 第二次
),
```

`computeSimpleEnvInfo` 在 session 初始化時被呼叫兩次（一次為了 `outputStyleConfig` 的 intro section，一次在動態 sections 中）。由於 section cache 機制，第二次會從快取取，但第一次的結果（`envInfo`）實際上只用於確定 intro section 的措辭，而非直接使用其內容。

**問題：**
這是一個輕微的重複計算問題，透過快取機制大部分已被緩解。但代碼設計上仍存在不清晰之處（並行計算的 envInfo 與動態 sections 的 env_info_simple 之間的關係不明確）。

**更好的做法：**
第一次計算的 `envInfo` 結果應直接傳入動態 sections 的工廠函式（或直接使用已計算的結果），而非再次定義計算函式。

**預估影響：**
- 主要是代碼可讀性問題
- 輕微的 CPU 計算優化（消除一次 uname 系統呼叫等）

---

### 問題 14：Bash 安全檢查器有兩個版本（同步/非同步），且標記為 DEPRECATED 卻仍在使用

**嚴重程度：** 🟢 低

**現狀：**
Bash 安全過濾 API（來源：`phase-06-security-permissions/02-bash-security-rules.md`）：

```typescript
`bashCommandIsSafe_DEPRECATED(command)` — 同步版（當 tree-sitter 不可用時）
`bashCommandIsSafeAsync_DEPRECATED(command, onDivergence?)` — 非同步版（優先用 tree-sitter）
```

兩個對外 API 都標記為 `_DEPRECATED`，但仍是主要的安全檢查入口點。

**問題：**
標記為 `DEPRECATED` 但仍在廣泛使用，表示重構計劃被中途放棄，或標記本身是誤導性的。這導致後續開發者不確定是否應該使用這些 API，也不清楚計劃中的替代方案是什麼。

**更好的做法：**
1. 如果有計劃中的新 API，完成替換並移除舊版本
2. 如果沒有，移除 `_DEPRECATED` 標記，避免誤導
3. 在代碼中清楚說明使用場景（何時用同步版，何時用非同步版）

**預估影響：**
- 主要是代碼維護性問題，無直接的 token 或體驗影響

---

## 問題優先矩陣

| 問題 | 嚴重程度 | 修復難度 | 預估節省 |
|------|---------|---------|---------|
| 1. Auto Mode 分類器使用 Opus | 🔴 高 | 低（單行修改） | 分類器成本 -83% |
| 2. file modification 注入 Bug | 🔴 高 | 中（修復狀態管理） | -5-15% context window/session |
| 3. session 啟動探測請求 | 🔴 高 | 低（移除或替換） | -1 次啟動 API 呼叫 |
| 4. Compaction 後 cache 全滅 | 🔴 高 | 高（架構調整） | 壓縮成本 -30-50% |
| 5. 日期字串破壞靜態 cache | 🔴 高 | 低（移動注入位置） | -每日 1 次全量 cache 重建 |
| 6. Memory 提取每 turn 執行 | 🟡 中 | 低（調整閾值） | -50% 記憶提取 API 呼叫 |
| 7. AgentSummary 30s 定時器 | 🟡 中 | 低（調整觸發條件） | 短 task 節省 1 次 API 呼叫 |
| 8. 分類器 transcript 含完整檔案內容 | 🟡 中 | 低（修改序列化） | Write 工具分類器成本 -90% |
| 9. MCP 指令每輪不快取 | 🟡 中 | 低（改 feature flag 預設值） | -500-2000 tokens/輪 |
| 10. env_info_simple 含可變 cwd | 🟡 中 | 低（分離 cwd section） | 主要是正確性改善 |
| 11. Compaction 繼承完整工具集 | 🟡 中 | 中（評估替代方案） | 每 36 次壓縮 -1 次備援呼叫 |
| 12. Skills 列表每輪注入 | 🟡 中 | 中（移動注入位置） | -1000-3000 tokens/輪 |
| 13. env_info_simple 重複計算 | 🟢 低 | 低（代碼清理） | 輕微 CPU 優化 |
| 14. DEPRECATED 標記誤導 | 🟢 低 | 低（移除標記） | 維護性改善 |

---

## 整體分析結論

### 最關鍵的系統性問題

**額度消耗的主要驅動力（前 3 大）：**

1. **分類器成本（問題 1 + 8）**：Auto Mode 是功能性設計，但用 Opus 等級模型做簡單分類，且 transcript 包含完整檔案內容，導致每次工具操作都有不必要的高成本。這兩個問題合計，對 Max 用戶的分類器成本可降低 90%+。

2. **system-reminder 注入失控（問題 2 + 12）**：file modification 注入存在根本性 Bug，且整個 system-reminder 系統在沒有透明度的情況下持續擴張（15+ 種類型）。使用者無法關閉，代碼中也沒有明確的 token 預算控制。

3. **Cache 穩定性問題（問題 4 + 5）**：Compaction 和日期注入各自導致定期的完整 cache 失效，而且兩者都是設計時可以避免的。

**架構合理性最值得檢討的地方：**

Compaction 作為「節省成本的機制」，其本身的成本結構（觸發時付全額、失效後重建、可能連鎖反應）設計得相當不透明。使用者看不到何時觸發、觸發成本多少、是否發生連鎖反應。一個本應「省錢」的功能，在某些情況下實際上在「燒錢」，且完全不透明。

Auto Mode 的分類器設計邏輯上是合理的（讓另一個模型判斷是否放行），但成本控制機制明顯不成熟：繼承主模型（問題 1）、transcript 無上限增長（問題 8）、沒有快取優化的 two-stage 選項（問題 1）。這幾個問題合在一起，使 Auto Mode 成為隱性的高費用消耗源。
