# Repository Instructions

## What This Repo Is

- This is an OpenCode/XCode workspace for generating DingTalk daily and weekly worklogs, not a traditional application codebase.
- The primary behavior source is `.opencode/skills/dingding-report-writer/SKILL.md`; read it before changing report behavior.
- `README.md` is useful orientation, but executable workflow rules come from the skill file and XCode config.

## Files That Matter

- `.opencode/skills/dingding-report-writer/SKILL.md`: defines the daily/weekly draft, confirmation, config, and send rules.
- `.xcode/dingding-report-config.yaml`: local runtime config for template names, recipients, and `ddFrom`; `.xcode/` is gitignored, so do not commit or copy its private values.
- `.xcode/scheduled-tasks.yaml`: local XCode schedule config; current daily draft task is also gitignored runtime state.
- `.opencode/package.json`: only declares the OpenCode plugin dependency; there are no project scripts.

## Report Workflow Rules

- For daily or weekly reports, use the `dingding-report-writer` skill.
- Always generate and show a checklist-style draft first, then wait for explicit natural-language confirmation before sending.
- Do not use the `question` component for send confirmation; ask in plain text instead.
- Never call `dingding_worklog_create_report` before confirmation, and always pass `toChat: false` when sending.
- Read recipients, template names, and `ddFrom` from `.xcode/dingding-report-config.yaml`; never write user-variable config into the skill directory.
- Resolve recipient names or phone numbers with DingTalk contact tools; never guess or invent `userId` values.
- Compute "today" and "this week" using `Asia/Shanghai`.
- Weekly reports default to summarizing this week's already sent daily reports; only fall back to workspace sessions if the user explicitly allows it.

## Commands And Verification

- There are no repo-level build, lint, typecheck, test, CI, pre-commit, dev-server, migration, or codegen commands configured.
- Do not invent `npm test`, `npm run build`, or similar checks. If you change Markdown/YAML/JSON, verify by rereading the changed file and checking the relevant source file it references.
- `.opencode/package-lock.json` is an npm lockfile for the OpenCode plugin dependency; avoid reinstalling dependencies unless the plugin dependency changes.

## State And Privacy

- `.omo/` and `.xcode/` are ignored runtime/local state. Treat contents as environment-specific and potentially sensitive.
- If documenting local config, describe keys and behavior, not concrete `userId`, channel account, or other private values.
