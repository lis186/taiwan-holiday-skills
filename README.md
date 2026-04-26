# taiwan-holiday-skills

Claude Agent Skills for working with Taiwan public holidays. Compatible with [Claude Code](https://claude.com/claude-code), [claude.ai](https://claude.ai), and any agent loader that supports the [Anthropic Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) format (e.g. [OpenSkills](https://github.com/numman-ali/openskills)).

## Skills in this repo

### [`taiwan-holiday/`](./taiwan-holiday/SKILL.md)

Wraps the [`taiwan-holiday-cli`](https://github.com/lis186/taiwan-holiday-cli) npm package. Lets Claude answer questions like "下個月有幾個工作天?", "10/10 是禮拜幾?", "下一個連假什麼時候?" with authoritative, structured data — without the model second-guessing itself or telling the user to "verify with the official announcement".

Trigger phrases (auto-detected by the description): any Taiwan + date question, working-day math, leave planning, compensatory days (補假/補班日), upcoming holidays.

## Install

### Option A — OpenSkills (recommended for cross-tool agents)

```bash
npx openskills install lis186/taiwan-holiday-skills
npx openskills sync
```

This drops the `taiwan-holiday/` folder into your project's `.claude/skills/` (or `.agent/skills/` if you used `--universal`). Add `--global` to install into `~/.claude/skills/` instead.

### Option B — Claude Code native (manual)

```bash
# project-level
git clone https://github.com/lis186/taiwan-holiday-skills.git /tmp/ths
mkdir -p .claude/skills
cp -r /tmp/ths/taiwan-holiday .claude/skills/

# global (any project picks it up)
mkdir -p ~/.claude/skills
cp -r /tmp/ths/taiwan-holiday ~/.claude/skills/
```

### Option C — git submodule (always track upstream)

```bash
git submodule add https://github.com/lis186/taiwan-holiday-skills.git .claude/skills/_taiwan-holiday-skills
ln -s _taiwan-holiday-skills/taiwan-holiday .claude/skills/taiwan-holiday
```

## Verifying the install

After installing, open Claude Code in any project and ask:

```
下個月有幾個工作天?
```

Claude should run `npx --yes taiwan-holiday-cli workdays <year> <month> -f json` in the background and answer with the exact number, no hedging.

## Requirements

- Node.js 20+ (for `npx`)
- Network access on first run (`npx` downloads `taiwan-holiday-cli` ≈ 5–15s; subsequent runs hit the npx cache and are <3s)

If you want to skip the first-run download, install once globally:

```bash
npm install -g taiwan-holiday-cli
```

The skill keeps invoking via `npx`; Node will short-circuit to the global install automatically.

## Supported years

2017–2026. The underlying data source is [TaiwanCalendar](https://github.com/ruyut/TaiwanCalendar) (thanks [@ruyut](https://github.com/ruyut)). Data correctness is the responsibility of the upstream project — file issues there if you spot a bad date.

## Limitations

- Only ROC government calendar data — no lunar calendar, solar terms, or non-Taiwan holidays.
- Doesn't do leave planning math, calendar sync, or notifications. The skill answers "what is this date?" — composing that into trip plans or alerts is up to you (or a separate skill on top).
- Compensatory days (補假/補班日) come straight from the data source. If the cabinet adjusts the official 行政機關辦公日曆表 mid-year, you'll get the new value once `taiwan-holiday-cli` ships an updated package.

## Evals

`taiwan-holiday/evals/evals.json` contains 3 realistic test prompts the skill is benchmarked against. Pass rate at the time of release: with-skill 100% / without-skill 83%, on these test cases. The largest gap is on the 2026/10 working-day count (補假 from 10/25 光復節 falling on a Sunday) — exactly the kind of compensatory rule that model memory routinely misses.

## Contributing

Pull requests welcome — particularly for additional skills built on top of `taiwan-holiday-cli` (e.g. leave planning, ICS export). Please match the writing style of the existing `taiwan-holiday/SKILL.md`: imperative voice, explicit reasoning, examples in JSON when relevant.

## License

MIT — see [LICENSE](./LICENSE).

## Related

- [`taiwan-holiday-cli`](https://github.com/lis186/taiwan-holiday-cli) — the underlying CLI this skill wraps
- [TaiwanCalendar](https://github.com/ruyut/TaiwanCalendar) — the data source
- [Anthropic Agent Skills docs](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
- [OpenSkills](https://github.com/numman-ali/openskills) — universal skill loader
