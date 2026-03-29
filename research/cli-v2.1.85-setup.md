# CLI v2.1.85 逆向環境建置

## 安裝

```bash
mkdir /tmp/cc-research && cd /tmp/cc-research
npm init -y
npm install @anthropic-ai/claude-code@2.1.85
```

安裝完成後，目標檔案在：
```
/tmp/cc-research/node_modules/@anthropic-ai/claude-code/cli.js
```

這是一個 ~12.9MB 的 minified JavaScript 檔案，包含完整的 Claude Code CLI 邏輯。

## 基本搜尋

函式名稱被壓縮（如 `XU`, `Z57`, `vm8`），但字串常數保留明文。用字串常數作為定位錨點：

```bash
# 搜尋快取相關
grep -n "cache_control" cli.js | head -20
grep -n "tengu_sysprompt" cli.js | head -10
grep -n "__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__" cli.js

# 搜尋工具相關
grep -n "defer_loading" cli.js | head -20
grep -n "tool_reference" cli.js | head -10

# 搜尋 beta headers
grep -n "prompt-caching-scope\|context-management\|advanced-tool-use" cli.js

# 搜尋內嵌文件
grep -n "Render order is" cli.js
grep -n "Don't change tools" cli.js

# 搜尋 compact 相關
grep -n "querySource.*compact\|compact.*querySource" cli.js
grep -n "preCompactTokenCount" cli.js

# 搜尋 streaming 相關
grep -n "CLAUDE_ENABLE_STREAM_WATCHDOG" cli.js
grep -n "content_block_delta" cli.js

# 搜尋 memory/session 相關
grep -n "memoryUsage\|heapUsed" cli.js | head -10
grep -n "maxMessages" cli.js

# 搜尋日期注入
grep -n "currentDate" cli.js
grep -n "date_change" cli.js

# 搜尋 auto-mode classifier
grep -n "auto_mode" cli.js | head -10
```

## 已知的關鍵函式對照表（v2.1.85）

| 功能 | minified 名稱 | 定位方式 |
|------|--------------|---------|
| cache_control 工廠 | `XU()` | `grep "type:\"ephemeral\""` |
| 系統提示策略 | `Z57()` | `grep "skipGlobalCacheForSystemPrompt"` |
| 系統提示格式化 | `yVY()` | `grep "cacheScope"` |
| 工具 schema 建構 | `vm8()` | `grep "defer_loading"` near `input_schema` |
| 訊息斷點處理 | `kVY()` | `grep "skipCacheWrite"` |
| 主查詢函式 | `vuK()` | `grep "skipGlobalCacheForSystemPrompt"` in large function |
| 延遲工具檢查 | `I0()` | `grep "isMcp.*true"` near `shouldDefer` |
| 發現掃描器 | `wU()` | `grep "tool_reference"` near `Set` |
| Tool search 模式 | `Ir6()` | `grep "tst-auto\|tst.*standard"` |
| 日期產生器 | `iQ6()` | `grep "getFullYear.*getMonth.*getDate"` |
| userContext builder | `WA()` | `grep "currentDate.*Today"` |
| context 注入 | `bu8()` | `grep "system-reminder.*context"` |
| date_change 偵測 | `i4Y()` | `grep "date_change.*newDate"` |
| compact executor | `CEq()` | `grep "preCompactTokenCount"` |
| auto-mode classifier | `Ny8()` | `grep "querySource.*auto_mode"` |
| 動態邊界標記 | `$A6` | `grep "SYSTEM_PROMPT_DYNAMIC_BOUNDARY"` |
| memoized 日期 | `uP8` | `grep "z1(iQ6)\|z1(GD6)"` |

## 注意事項

- cli.js 是單行檔案，`grep -n` 的行號基本都是同一行（line 7660 之類的巨大行）
- 用 `grep -b` 取得 byte offset 更精確
- 不同版本的 minified 名稱會不同，但字串常數和 telemetry event 名稱通常穩定
- `z1()` 是 memoize/once 包裝器，出現在很多地方
- `y(()=>{})` 是 esbuild 的 module lazy init（479 個），不是 app-level lazy loading
- `g8("tengu_xxx", false)` 是 feature flag 讀取（GrowthBook）
- `d("tengu_xxx", {...})` 是 telemetry event 發送
