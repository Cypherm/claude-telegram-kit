# claude-telegram-kit — Implementation Plan

## Repo 定位

OpenClaw → Claude Code 遷移套件。給所有因 Anthropic ban subscription 而失去 OpenClaw 的用戶一個 90% 功能覆蓋的替代方案。

核心公式：**bridge.js（通道） + workspace template（規範） + migrate.sh（搬移）**

---

## 架構

```
claude-telegram-kit/
├── bridge.js                       ← Telegram ↔ claude -p，含 heartbeat
├── package.json                    ← name, bin, engines
├── .env.example                    ← 所有環境變數範例
├── workspace-template/             ← OpenClaw 受災戶的起點
│   ├── CLAUDE.md                   ← 合併 SOUL+IDENTITY+AGENTS+wiki 記憶規則
│   ├── HEARTBEAT.md                ← 主動行為定義（可自訂範例）
│   ├── wiki/                       ← Karpathy wiki pattern
│   │   ├── index.md                ← 導航目錄
│   │   └── log.md                  ← 操作紀錄
│   └── memory/                     ← raw daily logs（空，結構佔位）
│       └── .gitkeep
├── migrate.sh                      ← OpenClaw workspace → 新格式自動轉換
└── README.md                       ← 含 OpenClaw 遷移指南 + Karpathy wiki 說明
```

---

## 實作項目

### 1. bridge.js（基於本地 ~/ccc/bridge.mjs 改寫）

**保留原有功能：**
- Telegram polling + serial message queue
- Session continuity（--session-id / --resume）
- Auto-compaction every N turns + session-summary.md
- Photo download + cleanup
- Voice transcription（whisper-cli）
- Sticker → emoji
- Reply quote context
- Markdown → HTML rendering + fallback
- Emoji reactions（[react: 💎]）
- Long message splitting（4096 char）
- Error retry with fresh session
- 409 conflict backoff

**新增：Heartbeat**
- 環境變數 HEARTBEAT_INTERVAL（分鐘，default 0 = 停用）
- 環境變數 HEARTBEAT_CHAT_ID（發送 proactive 訊息的 chat）
- 每 N 分鐘：讀 HEARTBEAT.md → 送進 claude -p → 如果回覆 ≠ "HEARTBEAT_OK" → 發到 Telegram
- Heartbeat 跑在 message queue 之外（不阻塞用戶訊息），但共用同一個 session
- 如果 claude 正在處理用戶訊息，heartbeat 排隊等，不並行（避免 session lock）
- Console log: [heartbeat] triggered / [heartbeat] OK / [heartbeat] sent response

**通用化改動（從 hardcoded → configurable）：**
- BOT_TOKEN: env var 或 --token CLI arg
- ALLOWED_IDS: env var 或 --allow CLI arg（空 = allow all + warn）
- WORK_DIR: env var 或 --dir CLI arg（default cwd）
- COMPACT_INTERVAL: env var 或 --compact（default 100）
- CLAUDE_TIMEOUT: env var（default 300000）
- POLL_INTERVAL: env var（default 2000）
- COMPACT_PROMPT: env var（default 中文摘要 prompt）
- WHISPER_MODEL: env var（default ggml-base.bin）
- --dangerously-skip-permissions: 保留，headless 必需
- --add-dir: 可選 env var CLAUDE_ADD_DIRS（逗號分隔）
- --help / -h: 顯示用法

### 2. workspace-template/CLAUDE.md

合併以下 OpenClaw 檔案為單一 CLAUDE.md：
- IDENTITY.md → "## 你是誰" section
- SOUL.md → "## 你的靈魂" section
- USER.md → "## 你的人類" section
- AGENTS.md 行為規則 → "## 行為規則" section（安全、外部 vs 內部、群組規則）

加上新的記憶系統指令：
- "## 記憶系統" section — Karpathy wiki pattern
  - wiki/index.md 是導航，每次 session 開始讀
  - wiki/ 下按主題建頁面（entity pages, concept pages, decision logs）
  - wiki/log.md 是 append-only 時間序列
  - 新對話素材 → ingest（更新 index + 相關頁面）
  - 定期 lint（via heartbeat）：找矛盾、過期、孤立頁面
  - memory/YYYY-MM-DD.md 是 raw daily log（optional）
- "## Claude Code Auto-Memory" section
  - auto-memory 只用於 feedback 類型（行為糾正）
  - declarative knowledge 寫 wiki/，不寫 auto-memory
  - 避免兩個系統重疊
- "## Heartbeat" section
  - 參考 HEARTBEAT.md 做主動行為
  - HEARTBEAT_OK = 沒事，其他 = 發給用戶

模板中用 `{{placeholder}}` 標記需要用戶自訂的部分。

### 3. workspace-template/HEARTBEAT.md

提供範例結構（用戶自訂）：
```markdown
# Heartbeat — 主動巡邏清單

## 每次 heartbeat 都檢查
- [ ] 讀 wiki/index.md 確認記憶完整性
- [ ] 如果有未讀的 daily log → 做 ingest 更新 wiki

## 觸發式
- 如果用戶 >8h 沒說話 → 簡短 check-in
- 23:00-08:00 → HEARTBEAT_OK（靜默時段）

如果沒事做，回覆 HEARTBEAT_OK
```

### 4. workspace-template/wiki/

**index.md** — 初始模板：
```markdown
# Wiki Index
<!-- LLM 維護，人類閱讀。每次 session 開始讀這個檔案。 -->

## Pages
（尚無頁面。對話累積後，在此建立主題頁面的連結。）

## Recent
（最近 5 次操作，從 log.md 摘要。）
```

**log.md** — 初始模板：
```markdown
# Wiki Log
<!-- Append-only. 格式: ## [YYYY-MM-DD] operation | details -->
```

### 5. migrate.sh

自動化 OpenClaw → claude-telegram-kit 的轉換：

```bash
#!/bin/bash
# Usage: ./migrate.sh /path/to/openclaw/workspace /path/to/new/workdir

SRC=$1
DST=$2

# 1. 複製 workspace-template 結構
cp -r workspace-template/* "$DST/"

# 2. 合併人格檔 → CLAUDE.md（保留 template 的結構，填入內容）
# 讀取 IDENTITY.md, SOUL.md, USER.md, AGENTS.md
# 插入 CLAUDE.md 對應 section

# 3. MEMORY.md → wiki/ 拆分
# 解析 P0/P1/P2 段落
# P0 各段 → wiki/core-*.md（永久知識）
# P1 各項 → wiki/project-*.md（帶日期）
# P2 各項 → wiki/note-*.md（帶日期）
# 生成 wiki/index.md 連結

# 4. Skills 搬移
# 找到所有 skills/*/SKILL.md
# 複製到 ~/.claude/skills/*/skill.md（改大小寫）

# 5. HEARTBEAT.md → 直接複製
cp "$SRC/HEARTBEAT.md" "$DST/HEARTBEAT.md" 2>/dev/null

# 6. Daily logs → memory/
cp "$SRC/memory/"*.md "$DST/memory/" 2>/dev/null

# 7. 報告結果
echo "Migration complete. Review $DST/CLAUDE.md and customize."
```

### 6. README.md

結構：
1. **What is this** — one paragraph
2. **Quick Start** — 3 步驟（clone, configure, run）
3. **Coming from OpenClaw?** — migration section
   - 概念對照表（OpenClaw feature → 這裡怎麼做）
   - migrate.sh 用法
   - 手動調整清單
4. **How It Works** — bridge 架構圖 + session lifecycle
5. **Heartbeat** — 設定方式 + 範例
6. **Wiki Memory System** — Karpathy pattern 簡介 + 使用方式
7. **Configuration** — 所有 env vars 表格
8. **Running as a Service** — pm2 / tmux

### 7. package.json

```json
{
  "name": "claude-telegram-kit",
  "version": "0.1.0",
  "description": "OpenClaw → Claude Code migration kit. Telegram bridge + wiki memory + heartbeat.",
  "type": "module",
  "bin": { "claude-tg": "./bridge.js" },
  "engines": { "node": ">=18.0.0" },
  "scripts": { "start": "node bridge.js" },
  "license": "MIT"
}
```

### 8. .env.example

```bash
# Required
TELEGRAM_BOT_TOKEN=your-bot-token-from-botfather

# Optional
ALLOWED_TELEGRAM_IDS=123456,789012    # comma-separated, empty = allow all
WORK_DIR=./workspace                   # where CLAUDE.md and wiki/ live
COMPACT_INTERVAL=100                   # turns before session rotation
CLAUDE_TIMEOUT=300000                  # ms (5 min)
POLL_INTERVAL=2000                     # ms
HEARTBEAT_INTERVAL=30                  # minutes, 0 = disabled
HEARTBEAT_CHAT_ID=123456              # Telegram chat to send proactive messages
WHISPER_MODEL=ggml-base.bin           # whisper-cli model name or full path
CLAUDE_ADD_DIRS=                      # extra dirs for claude --add-dir (comma-separated)
```

---

## 執行順序

1. ✅ 建立 repo（已完成）
2. [ ] bridge.js — 基於本地 bridge.mjs + GitHub 版合併，加 heartbeat
3. [ ] workspace-template/ — CLAUDE.md + HEARTBEAT.md + wiki/
4. [ ] migrate.sh — 自動轉換腳本
5. [ ] README.md — 含遷移指南
6. [ ] package.json + .env.example
7. [ ] 測試：本地跑起來驗證 heartbeat
8. [ ] push to GitHub

---

## 設計決策紀錄

- **為什麼新 repo 不 fork 原本的？** 定位不同。原本是純 bridge，新的是遷移套件。
- **為什麼 bridge.js 單檔？** OpenClaw 受災戶要的是「clone + 設定 + 跑」，不是裝一堆 dependency。零依賴是核心賣點。
- **為什麼 wiki 不用 auto-memory？** 兩個系統各司其職：wiki = declarative knowledge（你知道什麼），auto-memory = behavioral feedback（你被糾正過什麼）。不重疊。
- **為什麼 heartbeat 不用 Claude Code cron？** cron 的輸出不經過 Telegram。Heartbeat 的核心需求是「主動在 Telegram 跟用戶說話」，必須在 bridge 層實作。
- **為什麼 CLAUDE.md 不拆成多個檔案？** Claude Code 的 CLAUDE.md 是單一進入點。拆開反而增加載入不確定性。用 section 分隔比多檔案更可靠。
