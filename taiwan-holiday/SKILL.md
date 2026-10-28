---
name: taiwan-holiday
description: Query Taiwan national holidays, working-day counts, and compensatory days via the taiwan-holiday-cli npm package (invoked over npx, no install needed). Use this skill whenever the user asks anything that combines Taiwan with dates — whether it is "is X a holiday", "how many working days in May", "when is the next long weekend", "next Chinese New Year", planning leave, or any schedule question that depends on the 中華民國政府行政機關辦公日曆表. Also use it for any date-math question the user anchors to Taiwan (e.g. "我下個月工作幾天", "10/10 是什麼日子", "接下來連假"), even when the user does not say the word "holiday" explicitly. Prefer this skill over guessing from memory — the CLI has authoritative data from 2017 onward (the upper bound follows the upstream data source; run `years` to check) and knows about makeup workdays that models frequently get wrong. Also use it for typhoon/natural-disaster day-off questions (颱風假、停班停課、「明天要上班/上課嗎」when the user mentions a typhoon, heavy rain, or other disaster) — the CLI's `typhoon` subcommand reads DGPA's live page and archive since 2001; ordinary holiday questions without a disaster context should not use `typhoon`.
allowed-tools: Bash
---

# taiwan-holiday

A thin wrapper around the `taiwan-holiday-cli` npm package. You call it with `npx`, parse its output, and answer the user in natural Chinese/English.

Requires `taiwan-holiday-cli` >= 3.0.0 and Node.js 22+ (the `typhoon` subcommand arrived in 2.1.0; 3.x is the current generation shared with this skill; holiday year support is probed from the upstream data source at runtime; `npx` fetches the latest release, which satisfies this automatically).

## When to reach for this skill

Any question where the answer depends on the Taiwan public-holiday calendar. Common shapes:

- Single-day check: "今天是假日嗎?", "10/10 要上班嗎?", "is 2026-02-17 a holiday"
- Upcoming holidays: "下一個連假什麼時候", "接下來 5 個假日"
- Range / month: "五月有幾天假", "2026 Q2 放哪些假"
- Working-day math: "這個月工作幾天", "我請假到 5/15，中間幾個工作天"
- Annual planning: "2026 全年國定假日", "下次春節哪幾天"
- Compensatory / makeup days: "補班日", "補假" — important because model memory is often wrong here; always defer to the CLI.
- Typhoon / natural-disaster days off: "明天颱風會放假嗎", "台北市今天停班停課嗎", "2024/10/31 有颱風假嗎" — use the `typhoon` subcommand (see [Typhoon days off](#typhoon-days-off-颱風假)).

**Routing rule for「明天要上班嗎」:** if the user (or the recent conversation) mentions a typhoon, heavy rain, earthquake, or 停班停課 → `typhoon tomorrow`. With no disaster context → `check tomorrow` (holiday calendar). If a typhoon is in the news but the user only asked about the calendar, answer with `check` and don't volunteer a typhoon lookup unless asked.

Do *not* use this skill for non-Taiwan holidays or religious observances outside the 中華民國政府行政機關辦公日曆表. For years that might be out of range (before 2017, or future years the upstream may not have published yet), still run the CLI — it distinguishes "not yet published" from "outside the queryable window", and you should relay its answer instead of guessing (see Error handling).

## Invocation — always use `npx` like this

```bash
(cd /tmp && npx --yes taiwan-holiday-cli <subcommand> [args] -f <format>)
```

Two important invocation rules:

1. **Change directory first.** `npx` inspects the current working directory's `package.json` to resolve bins. If you run it inside the `taiwan-holiday-cli` repo (or any project whose `node_modules` doesn't have the `holiday` bin), you will get `sh: holiday: command not found`. Running from `/tmp` (or any neutral directory) avoids this. Using a subshell `(cd /tmp && …)` keeps your working directory unchanged.
2. **`--yes`** suppresses the interactive "install this package?" prompt on first run. First run is slow (~5–15s download); subsequent runs hit the npx cache and are fast.

## Choosing an output format

Always pass `-f` explicitly so you control parsing. Two formats matter:

- `-f json` — **Default choice.** Structured, easy to parse, stable schema. Use this for anything you plan to reformat, aggregate, or summarize (lists, ranges, stats, workdays, next N holidays).
- `-f simple` — Human-readable one-liner. Use this only for single-date checks (`today`, `tomorrow`, a specific date) where you plan to quote the CLI's answer verbatim and don't need to restructure it.

Never use `-f table` from this skill — box-drawing characters waste tokens and you can always rebuild a table from JSON if the user wants one.

## Subcommand cheat-sheet

### Single-date check

```bash
# JSON (when you'll reformat or combine with other info)
npx --yes taiwan-holiday-cli check 2026-10-10 -f json
npx --yes taiwan-holiday-cli today -f json
npx --yes taiwan-holiday-cli tomorrow -f json
npx --yes taiwan-holiday-cli "next monday" -f json
```

JSON shape:
```json
{"date":"2026-10-10","normalizedDate":"20261010","week":"六","isHoliday":true,"description":"國慶日"}
```

Key fields: `isHoliday` (bool), `description` (empty string means weekday; `"週末"` means weekend with no special name; otherwise the holiday name).

### Upcoming holidays

```bash
npx --yes taiwan-holiday-cli next -f json          # next 1
npx --yes taiwan-holiday-cli next 5 -f json        # next 5
npx --yes taiwan-holiday-cli next 5 --skip-weekends -f json   # only named holidays, no plain weekends
```

JSON shape: `{"found": true, "count": N, "holidays": [{"date","week","description"}, …]}`.

`--skip-weekends` is usually what the user wants when they say "接下來的連假 / 國定假日" — without it the output is mostly Saturdays and Sundays.

### Range / month / full year

```bash
npx --yes taiwan-holiday-cli range 2026-05-01 2026-05-31 -f json
npx --yes taiwan-holiday-cli month 2026 10 -f json   # YEAR and MONTH as TWO args, not "2026-10"
npx --yes taiwan-holiday-cli list 2026 -f json       # full year, every day
```

`list` returns all 365/366 days (holiday + non-holiday). `month` returns only the holidays/weekends in that month. Prefer `month` unless the user wants every day.

**Trap:** `month` and `workdays` and `stats <year> <month>` take year and month as **two separate arguments**. `2026-10` or `2026/10` will fail with `error: missing required argument 'month'`. Pass `2026 10`.

### Working-day math

```bash
npx --yes taiwan-holiday-cli workdays 2026 5 -f json              # workdays in May 2026
npx --yes taiwan-holiday-cli between 2026-05-01 2026-05-31 -f json  # workdays between two dates (inclusive)
```

Output of `workdays`:
```json
{"year":2026,"month":5,"totalDays":31,"workdays":20,"holidays":11,"makeupWorkdays":0}
```

Output of `between`:
```json
{"startDate":"2026-05-01","endDate":"2026-05-31","totalDays":31,"workdays":20,"holidays":11,"makeupWorkdays":0}
```

`makeupWorkdays` (補班日) is the field the model most often gets wrong from memory — Taiwan shifts workdays around long weekends, so a Saturday can be a workday. Always trust this field over your own arithmetic.

### Statistics

```bash
npx --yes taiwan-holiday-cli stats 2026 -f json       # full year breakdown
npx --yes taiwan-holiday-cli stats 2026 10 -f json    # single month
```

Returns totals plus a `holidayTypes` map (e.g. `"春節":3, "國慶日":1, "補假":6`). Good for "這一年放了哪些類別的假" questions.

### Metadata

```bash
npx --yes taiwan-holiday-cli years        # supported year range (plain text, no -f needed)
```

Use this once if the user asks about a year you're unsure the CLI covers. The range starts at 2017; the upper bound follows the upstream data source and extends automatically when a new year is published — no CLI update needed.

## Typhoon days off (颱風假)

```bash
(cd /tmp && npx --yes taiwan-holiday-cli typhoon -f json)                       # today (Taiwan time), DGPA live page
(cd /tmp && npx --yes taiwan-holiday-cli typhoon tomorrow -c 臺北市 -f json)     # tomorrow, live page
(cd /tmp && npx --yes taiwan-holiday-cli typhoon 2024-10-31 -c 臺北市 -f json)   # any other date: archive (from 2001-07-30)
```

- `[date]` is optional (default: today in Taiwan time); same formats as `check` (`tomorrow`, `yesterday`, `2024-10-31`, …).
- `-c <縣市>` filters one city. Pass the full official name (`臺北市`, not `台北`); 台／臺 are treated as the same. Names are matched literally and keep their historical form (e.g. 臺北縣 before 改制) — they are not mapped to today's administrative divisions.
- Today and tomorrow come from the DGPA live page (`source.kind: "live"`); every other date comes from the archive (`source.kind: "archive"`).
- `-f simple` prints the verdict on line 1 with a status tag (e.g. `〔已定案〕`). That is fine for quoting a single date + single city verbatim; otherwise use `-f json`.

### Reading the JSON — use the top level, not the rows

Real output (`typhoon 2024-10-31 -c 臺北市`, trimmed):

```json
{"date":"2024-10-31","city":"臺北市",
 "work":{"status":"stopped","period":"allDay","from":""},
 "school":{"status":"stopped","period":"allDay","from":""},
 "basis":"own","reason":null,
 "source":{"kind":"archive","url":"https://www.dgpa.gov.tw/information?uid=374&pid=12171","announcementDate":"20241031","updatedAt":"2024-10-31T23:38:13","fetchedAt":null},
 "cities":[{"city":"臺北市","coverage":"city","subAreas":{"stopped":0,"total":0},"rows":[{"scope":"","forDate":"20241031","work":"stop","school":"stop","period":"allDay","from":"","raw":"今天停止上班、停止上課。"}], "work":{…}, "school":{…}}],
 "notes":[]}
```

The headline answer is **`work.status` and `school.status`** (judged separately — 上班 and 上課 can differ):

| `status` | Meaning | How to say it |
|---|---|---|
| `stopped` | The whole city stops. Timing in `period` (`allDay`/`morning`/`afternoon`/`evening`/`from`); when `period` is `from`, `from` holds `HH:MM`. | 「停止上班（全天）」, 「下午停止上課」, 「18:00 起停止上班」 |
| `conditional` | Only some sub-areas, some school levels, or some time slots stop. | Say it is partial, then list the affected scopes from `cities[].rows`. Never collapse it into 「停班」 or 「照常」, and don't claim 「其餘地區照常」 — the announcement only lists affected places; say 「公告只列出以上地點」. |
| `normal` | An announcement says 照常; **or** the live page has no announcement for that city yet (`reason: "live"`, may still change); **or** >21 days past with no announcement (`basis: "no_announcement"`, `reason: null`). | Distinguish 「公告照常」 from 「目前沒有公告」 by `basis`. |
| `unknown` | Cannot tell — record unparseable, archive not yet caught up, tomorrow not announced yet. | 「目前無法確認」. **Never** turn this into 照常. |

`cities[].rows` use a *different* vocabulary (`work`: `stop`/`normal`/`null`; `school`: `stop`/`normal`/`partial`/`null`) and carry the original sentence in `raw`. Use rows only to explain a `conditional` result or to quote the source — never to form the headline yourself. If a row's `work`/`school` is `null`, the sentence couldn't be parsed: quote `raw`. `cities` can be empty (e.g. no announcement at all).

**`city: null` (no `-c`) is a nationwide summary.** The DGPA page only lists cities with something to report, so the CLI can never conclude 「全臺停班」: any stop anywhere yields `conditional`. If the user didn't name a city, either ask which city or list the cities in `cities[]` that stop — don't say 「全臺停班停課」.

### Is it final? — `basis` and `reason`

- `basis`: `own` (the day's own announcement), `forecast` (the previous day's 「明天…」 announcement — can be overturned by a same-day announcement), `no_announcement`.
- `reason`: why the answer might still change. `live` (live page; can be updated later the same day — e.g. 下午停班 later becomes 全天), `pending` (within 21 days; the archive may not have caught up yet), `source_not_ready` (this year's archive file doesn't exist yet), `null` (settled).
- `notes[]` are human-readable caveats written for you — relay them, don't drop them. One exception: for `basis: "forecast"` with `reason: null` (archive, settled), the note 「可能已被當日公告推翻」 does not apply — the CLI already prefers the day's own announcement, so getting `forecast` from the archive means there was none. Don't repeat that note in this case.
- `source.updatedAt` is Taiwan local time without offset; `source.fetchedAt` is UTC ISO 8601. Don't compare them directly.
- Exit code is 0 for `unknown` too. Branch on `status`, not on the exit code.

### Answer template

Lead with the verdict, then always say how settled it is, then point to the official source:

> **臺北市 2024-10-31：停止上班、停止上課（全天）。** 這是當日公告，已定案。來源：行政院人事行政總處公告（附 `source.url`）。

If you passed `-c` and `cities` is empty, resolve that first (see the `cities: []` item in Common mistakes). Otherwise, pick the matching row for settledness:

| `basis` / `reason` | Say |
|---|---|
| `own` / `null` | 已定案（當日公告）。 |
| `forecast` / `null` | 依前一天公告的「明天」預告；當天沒有另行公告，所以預告就是最終結果。（Don't add 「可能被推翻」.） |
| any / `live` | 這是即時資料（頁面更新時間 `updatedAt`），當天仍可能更新，請以各縣市政府最新公告為準。 |
| any / `pending` | 歸檔可能尚未收錄（事件後 1～19 天才收錄），目前無法確認。 |
| `no_announcement` / `null` | 該日沒有停班停課公告（照常）。 |

For any live result, end with 「請以各縣市政府公告為準」. Worked examples from real output:

- `typhoon -f json` (today, no typhoon): `normal` / `no_announcement` / `live`, `cities: []`, note 「即時頁顯示『無停班停課訊息』」 → 「目前人事行政總處沒有停班停課訊息（即時資料，當天仍可能更新）」. Not 「全臺照常，確定」.
- `typhoon tomorrow -c 臺北市` (evening before, nothing announced): `unknown` / `no_announcement` / `live`, note 「明天的狀況通常在前一天晚上公告，請稍後再查」 → 「明天臺北市目前還沒有公告，晚點再查」. Not 「明天照常上班」.
- `typhoon yesterday`: `unknown` / `pending` → 「歸檔還沒收錄，目前無法確認」.
- `typhoon 2022-10-17 -c 桃園市`: `conditional` / `forecast` / `null`, `coverage: "subAreas"`, rows: 復興區三光里、高義里、華陵里 照常上班、停止上課; 三光國小、光華國小 停止上班、停止上課 → 「桃園市只有復興區部分地點停止：三光里、高義里、華陵里停止上課（照常上班）；三光國小、光華國小停止上班、停止上課。依前一天公告的預告，當天沒有另行公告。」 Only name what the rows list; don't add claims about unlisted areas.

### Common mistakes

- Saying 照常 when `status` is `unknown`. `unknown` means "can't tell", especially tomorrow before the evening announcement and for the last ~3 weeks in the archive.
- Presenting `basis: "forecast"` or `reason: "live"` as final.
- Answering 「全臺停班」 when no city was given — the nationwide summary is never `stopped`.
- Reducing `conditional` to a yes/no instead of listing the affected areas / school levels / time slots.
- Misreading `normal` with `cities: []` after `-c`. When the announcement exists but has no record for that name, the CLI returns `normal` with `cities: []` and the note 「該日有停班停課公告，但沒有『<name>』的記錄」. The note is identical for two different situations, so check the name yourself:
  - **Name wrong** — e.g. `-c 台北` (missing 市) on 2024-10-31 returns `normal`, even though 臺北市 stopped that day. Retry with the full official name (臺北市、新北市、桃園市…) before answering.
  - **Name correct, just not listed** — e.g. `-c 臺北市` on 2022-10-17. Say 「該日公告沒有列出臺北市（照常）」, not 「當日公告照常，已定案」.
- Mixing up 上班 and 上課 — report `work` and `school` separately when they differ.
- Using `typhoon` for an ordinary 「明天要上班嗎」 with no disaster context (use `check tomorrow`), or answering a typhoon question from the holiday calendar (`check` knows nothing about typhoons).

## How to answer the user

1. **Pick one command** that answers the question most directly. Don't chain three calls when one suffices (e.g. don't call `list` then filter — call `range` or `month`).
2. **Run it with `-f json`** unless the question is a single-day yes/no you'll quote verbatim.
3. **Parse and summarize in the user's language.** The CLI outputs Chinese day-of-week (`一二三四五六日`) and Chinese holiday names — keep those in Chinese even when the surrounding reply is English; they are proper nouns.
4. **Lead with the answer.** "是, 10/10 是國慶日 (週六)" — then any supporting detail. Don't dump raw JSON at the user unless they asked for raw data.
5. **Flag makeup workdays** when relevant. If the user is planning leave and `makeupWorkdays > 0`, mention it: "注意五月有 1 個補班日 (5/23 週六要上班)".

## Error handling

- `error: missing required argument 'month'` → you passed `2026-05` instead of `2026 5`. Fix and retry.
- `sh: holiday: command not found` → you're running from a directory whose `package.json` lacks the bin. Re-run with `(cd /tmp && npx …)`.
- `上游尚未發布 <year> 年資料` → the upstream source hasn't published that year yet (通常在前一年年中公布). Tell the user the data isn't available yet — don't invent dates from memory.
- `年份 <year> 超出可查詢範圍` → outside the queryable window entirely (before 2017 or far future). The error message itself states the window — relay it and tell the user the CLI can't help for that year. (Note this window is wider than the published-data range that `years` reports.)
- `typhoon`: `年份 <year> 超出可查詢範圍 (2001-…)` → the date is before the typhoon archive starts (2001-07-30). Tell the user there is no record that early.
- Network / download timeout on first run → the package is ~fast but npx download can be slow on cold cache; retry once, then surface the error to the user.

## Treat the CLI as authoritative — even when it conflicts with what you "know"

This section is about the holiday calendar (`check`/`month`/`workdays`/…). Typhoon results have their own posture: relay how settled they are and defer to the 縣市政府 announcement (see above).

中華民國政府行政機關辦公日曆表 has changed materially in the last few years, and a lot of the holiday folklore your training data carries is now wrong. Two specific examples to internalize:

- **9/28 孔子誕辰紀念日 / 教師節** — since the 2025 amendment to《紀念日及節日實施條例》, this is a **full national holiday for everyone**, including 勞工 and 軍公教. Compensatory leave applies if it lands on a weekend.
- **5/1 勞動節** — current law treats this as a national holiday in the calendar; the CLI returns `isHoliday=true` accordingly.

When the CLI says `isHoliday=true`, that *is* the answer. Do **not** add unprompted warnings like "this only applies to 勞工" or "教師才放假" — those caveats reflect a pre-2025 mental model that is no longer correct, and they erode the user's trust in your reply. If the user *specifically* asks who gets a particular day off, defer to the 行政院人事行政總處 announcement; otherwise, take the CLI at face value.

Symmetrically, when the CLI says `isHoliday=false` (e.g. 2026-10-12), don't manufacture a 補假 because "国慶日 fell on Saturday so Monday should be off" — the current 中華民國政府行政機關辦公日曆表 in the CLI's data already encodes where 補假 actually goes (10/9 in this case, not 10/12).

## What *not* to do

- Don't answer from memory. You will get makeup workdays, lunar-calendar-shifted holidays (春節, 端午, 中秋), and year-specific compensatory days wrong. The CLI is authoritative.
- Don't install the package globally or add it to any project's dependencies — `npx` is enough, and the user hasn't asked you to modify their environment.
- Don't call the CLI multiple times for the same data in one turn. Cache mentally within the turn.
- Don't pass `--no-cache` unless the user explicitly asks for fresh data; the local cache is fine for nearly every question.
- Don't second-guess `isHoliday=true` results. If you find yourself wanting to add "but only for X group", stop — see the section above.
