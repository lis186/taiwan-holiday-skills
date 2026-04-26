---
name: taiwan-holiday
description: Query Taiwan national holidays, working-day counts, and compensatory days via the taiwan-holiday-cli npm package (invoked over npx, no install needed). Use this skill whenever the user asks anything that combines Taiwan with dates — whether it is "is X a holiday", "how many working days in May", "when is the next long weekend", "next Chinese New Year", planning leave, or any schedule question that depends on the 中華民國政府行政機關辦公日曆表. Also use it for any date-math question the user anchors to Taiwan (e.g. "我下個月工作幾天", "10/10 是什麼日子", "接下來連假"), even when the user does not say the word "holiday" explicitly. Prefer this skill over guessing from memory — the CLI has authoritative data for 2017–2026 and knows about makeup workdays that models frequently get wrong.
allowed-tools: Bash
---

# taiwan-holiday

A thin wrapper around the `taiwan-holiday-cli` npm package. You call it with `npx`, parse its output, and answer the user in natural Chinese/English.

## When to reach for this skill

Any question where the answer depends on the Taiwan public-holiday calendar. Common shapes:

- Single-day check: "今天是假日嗎?", "10/10 要上班嗎?", "is 2026-02-17 a holiday"
- Upcoming holidays: "下一個連假什麼時候", "接下來 5 個假日"
- Range / month: "五月有幾天假", "2026 Q2 放哪些假"
- Working-day math: "這個月工作幾天", "我請假到 5/15，中間幾個工作天"
- Annual planning: "2026 全年國定假日", "下次春節哪幾天"
- Compensatory / makeup days: "補班日", "補假" — important because model memory is often wrong here; always defer to the CLI.

Do *not* use this skill for non-Taiwan holidays, religious observances outside the 中華民國政府行政機關辦公日曆表, or years outside 2017–2026 (check with `years` if unsure).

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

Returns totals plus a `holidayTypes` map (e.g. `"春節":3, "國慶日":1, "補假":12`). Good for "這一年放了哪些類別的假" questions.

### Metadata

```bash
npx --yes taiwan-holiday-cli years        # supported year range (plain text, no -f needed)
```

Use this once if the user asks about a year you're unsure the CLI covers. Currently 2017–2026.

## How to answer the user

1. **Pick one command** that answers the question most directly. Don't chain three calls when one suffices (e.g. don't call `list` then filter — call `range` or `month`).
2. **Run it with `-f json`** unless the question is a single-day yes/no you'll quote verbatim.
3. **Parse and summarize in the user's language.** The CLI outputs Chinese day-of-week (`一二三四五六日`) and Chinese holiday names — keep those in Chinese even when the surrounding reply is English; they are proper nouns.
4. **Lead with the answer.** "是, 10/10 是國慶日 (週六)" — then any supporting detail. Don't dump raw JSON at the user unless they asked for raw data.
5. **Flag makeup workdays** when relevant. If the user is planning leave and `makeupWorkdays > 0`, mention it: "注意五月有 1 個補班日 (5/23 週六要上班)".

## Error handling

- `error: missing required argument 'month'` → you passed `2026-05` instead of `2026 5`. Fix and retry.
- `sh: holiday: command not found` → you're running from a directory whose `package.json` lacks the bin. Re-run with `(cd /tmp && npx …)`.
- `年份超出支援範圍` / year-out-of-range messages → run `npx --yes taiwan-holiday-cli years` to confirm the window, then tell the user the CLI can't help for that year (don't invent dates from memory).
- Network / download timeout on first run → the package is ~fast but npx download can be slow on cold cache; retry once, then surface the error to the user.

## Treat the CLI as authoritative — even when it conflicts with what you "know"

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
