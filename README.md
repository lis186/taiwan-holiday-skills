# taiwan-holiday-skills

處理台灣國定假日與颱風假的 Claude Agent Skills 套件。相容於 [Claude Code](https://claude.com/claude-code)、[claude.ai](https://claude.ai)，以及任何支援 [Anthropic Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) 格式的 agent loader（例如 [OpenSkills](https://github.com/numman-ali/openskills)）。

## 本 repo 包含的 Skills

### [`taiwan-holiday/`](./taiwan-holiday/SKILL.md)

封裝 [`taiwan-holiday-cli`](https://github.com/lis186/taiwan-holiday-cli) 這個 npm 套件。讓 Claude 可以直接回答像「下個月有幾個工作天?」、「10/10 是禮拜幾?」、「下一個連假什麼時候?」這類問題，並提供權威、結構化的資料 —— 不會再讓模型自己猜，也不會丟一句「請以官方公告為準」就交差。颱風假也一樣先給明確結論、標明是否定案；只有在即時資料還可能變動時，才附上「以縣市政府公告為準」提醒。

觸發語句（由 description 自動偵測）：任何「台灣 + 日期」的問題、工作天計算、請假規劃、補假/補班日、即將到來的假期，以及颱風假／停班停課（需 `taiwan-holiday-cli` >= 2.1.0 的 `typhoon` 指令）。

## 安裝方式

### 方案 A —— OpenSkills（跨工具 agent 推薦使用）

```bash
npx openskills install lis186/taiwan-holiday-skills
npx openskills sync
```

會把 `taiwan-holiday/` 資料夾放進你專案的 `.claude/skills/`（如果用了 `--universal` 則放進 `.agent/skills/`）。加上 `--global` 則改安裝到 `~/.claude/skills/`。

### 方案 B —— Claude Code 原生安裝（手動）

```bash
# 專案層級
git clone https://github.com/lis186/taiwan-holiday-skills.git /tmp/ths
mkdir -p .claude/skills
cp -r /tmp/ths/taiwan-holiday .claude/skills/

# 全域（所有專案都可使用）
mkdir -p ~/.claude/skills
cp -r /tmp/ths/taiwan-holiday ~/.claude/skills/
```

### 方案 C —— git submodule（隨時追蹤上游版本）

```bash
git submodule add https://github.com/lis186/taiwan-holiday-skills.git .claude/skills/_taiwan-holiday-skills
ln -s _taiwan-holiday-skills/taiwan-holiday .claude/skills/taiwan-holiday
```

## 其他 Agent CLI 整合（已驗證）

### Gemini CLI

原生支援 Anthropic Agent Skills 格式，直接 link 即可：

```bash
git clone https://github.com/lis186/taiwan-holiday-skills.git /tmp/ths
gemini skills link /tmp/ths/taiwan-holiday
```

驗證：

```bash
gemini -y -p "下個月台灣有幾個工作天?"
```

### Codex CLI

Codex 沒有原生 skills 子指令，要透過 [OpenSkills](https://github.com/numman-ali/openskills) 註冊到專案的 `AGENTS.md`（codex 會自動讀）：

```bash
cd your-project
npx openskills install lis186/taiwan-holiday-skills
npx openskills sync   # 會把 skill 列入 AGENTS.md
```

驗證（注意：`--full-auto` 沙箱會擋住網路。全域安裝 CLI 只省下套件下載，查詢時仍要連網抓假期資料與颱風假資料（人事行政總處、GitHub），所以要改用允許網路的模式）：

```bash
npm install -g taiwan-holiday-cli   # 省下 npx 下載；查詢資料仍需網路
codex exec --full-auto "下個月台灣有幾個工作天?"
```

## 驗證安裝（Claude Code）

安裝完成後，在任一個專案打開 Claude Code，輸入：

```
下個月有幾個工作天?
```

Claude 應該會在背景執行 `npx --yes taiwan-holiday-cli workdays <year> <month> -f json`，然後直接給你正確數字，不會含糊其詞。

颱風假可以這樣驗證：

```
2024/10/31 台北有放颱風假嗎?
```

Claude 應該會執行 `npx --yes taiwan-holiday-cli typhoon 2024-10-31 -c 臺北市 -f json`，回答臺北市當天全天停止上班、停止上課，而且是已定案的當日公告。

## 系統需求

- Node.js 20+（執行 `npx` 用）
- `taiwan-holiday-cli` >= 2.1.0（`npx` 會自動抓最新版）。本 repo 以 git tag 標版，與 [`taiwan-holiday-cli`](https://github.com/lis186/taiwan-holiday-cli)/[`taiwan-holiday-mcp`](https://github.com/lis186/taiwan-holiday-mcp) 共享 MAJOR 版號 —— 目前為 2.x 世代
- 第一次執行時需要網路（`npx` 會下載 `taiwan-holiday-cli`，約 5–15 秒；之後會用 npx cache，<3 秒）

如果不想等第一次下載，可以先全域安裝一次：

```bash
npm install -g taiwan-holiday-cli
```

Skill 還是會用 `npx` 呼叫，但 Node 會自動跳到全域安裝的版本。

## 支援年份

2017 年起，上限由上游資料決定 —— 上游發布新年度後自動支援，不需更新 CLI 或本 skill。執行 `npx --yes taiwan-holiday-cli years` 可查目前範圍。底層資料來自 [TaiwanCalendar](https://github.com/ruyut/TaiwanCalendar)（感謝 [@ruyut](https://github.com/ruyut)）。資料正確性由上游專案負責 —— 如果發現日期錯誤，請去那邊回報 issue。

颱風假涵蓋 2001-07-30 起：今天、明天讀[行政院人事行政總處即時頁](https://www.dgpa.gov.tw/typh/daily/nds.html)，其他日期讀 [TaiwanTyphoonDays](https://github.com/lis186/TaiwanTyphoonDays) 歸檔（整理自人事行政總處公告）。

## 限制

- 假期只依中華民國政府行政機關辦公日曆表 —— 不含農曆、節氣或非台灣的假日。
- 不做請假規劃、行事曆同步、通知提醒。Skill 只回答「這天是什麼?」 —— 要把這個資訊組合成行程規劃或提醒，得自己接（或另外做一個 skill 蓋上去）。
- 補假/補班日完全依照資料來源。如果行政院年中調整官方「行政機關辦公日曆表」，要等上游 TaiwanCalendar 更新資料後才會反映（CLI 端另有 24 小時快取）。
- 颱風假：今天、明天的即時資料當天仍可能更新；昨天到 21 天前的日期，歸檔可能還沒收錄，查不到紀錄時會回「無法確認」而不是「照常」。一律以各縣市政府公告為準。

## Evals

`taiwan-holiday/evals/evals.json` 包含 6 個實際的測試 prompt（4 個假期、2 個颱風假），作為這個 skill 的 benchmark。初版發布時以當時的 3 個案例量測的通過率：有 skill 100%、無 skill 83%。差異最大的是 2026/10 的工作天數（10/25 光復節遇到禮拜天的補假）—— 這正是模型記憶最容易漏掉的補假規則。

## 貢獻

歡迎 PR —— 特別是基於 `taiwan-holiday-cli` 延伸的其他 skill（例如請假規劃、ICS 匯出）。寫作風格請對齊現有的 `taiwan-holiday/SKILL.md`：祈使句、明確推理、需要時用 JSON 舉例。

## 授權

MIT —— 詳見 [LICENSE](./LICENSE)。

## 相關連結

- [`taiwan-holiday-cli`](https://github.com/lis186/taiwan-holiday-cli) —— 本 skill 封裝的底層 CLI
- [`taiwan-holiday-mcp`](https://github.com/lis186/taiwan-holiday-mcp) —— 同一資料來源的 MCP server（Claude Desktop / Cursor 適用）
- [TaiwanCalendar](https://github.com/ruyut/TaiwanCalendar) —— 假期資料來源
- [TaiwanTyphoonDays](https://github.com/lis186/TaiwanTyphoonDays) —— 颱風假歸檔資料來源
- [Anthropic Agent Skills 文件](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
- [OpenSkills](https://github.com/numman-ali/openskills) —— 通用 skill loader
