# AGENTS

<skills_system priority="1">

## Available Skills

<!-- SKILLS_TABLE_START -->
<usage>
When users ask you to perform tasks, check if any of the available skills below can help complete the task more effectively. Skills provide specialized capabilities and domain knowledge.

How to use skills:
- Invoke: Bash("openskills read <skill-name>")
- The skill content will load with detailed instructions on how to complete the task
- Base directory provided in output for resolving bundled resources (references/, scripts/, assets/)

Usage notes:
- Only use skills listed in <available_skills> below
- Do not invoke a skill that is already loaded in your context
- Each skill invocation is stateless
</usage>

<available_skills>

<skill>
<name>taiwan-holiday</name>
<description>Query Taiwan national holidays, working-day counts, and compensatory days via the taiwan-holiday-cli npm package (invoked over npx, no install needed). Use this skill whenever the user asks anything that combines Taiwan with dates — whether it is "is X a holiday", "how many working days in May", "when is the next long weekend", "next Chinese New Year", planning leave, or any schedule question that depends on the 中華民國政府行政機關辦公日曆表. Also use it for any date-math question the user anchors to Taiwan (e.g. "我下個月工作幾天", "10/10 是什麼日子", "接下來連假"), even when the user does not say the word "holiday" explicitly. Prefer this skill over guessing from memory — the CLI has authoritative data from 2017 onward (the upper bound follows the upstream data source; run `years` to check) and knows about makeup workdays that models frequently get wrong.</description>
<location>project</location>
</skill>

</available_skills>
<!-- SKILLS_TABLE_END -->

</skills_system>
