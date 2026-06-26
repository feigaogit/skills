# Repository Instructions

## What This Repo Is

- This is an OpenCode/XCode custom skills workspace, not a traditional application codebase.
- Skills live under `.opencode/skills/<skill-name>/SKILL.md`.
- Use `.opencode/skills/README.md` as the skills catalog before editing or adding skills.
- `README.md` is the repository overview; executable workflow rules live inside each skill's `SKILL.md`.

## Files That Matter

- `.opencode/skills/README.md`: catalog of all skills, categories, trigger phrases, and outcomes.
- `.opencode/skills/dingding-report-writer/SKILL.md`: DingTalk daily/weekly draft, confirmation, config, and send workflow.
- `.opencode/skills/frontend-design/SKILL.md`: frontend UI/design workflow and preview behavior.
- `.opencode/skills/k0s-installer/SKILL.md`: k0s installation, repair, DNS/CoreDNS, airgap, token, and API URL workflow.
- `.xcode/dingding-report-config.yaml`: local DingTalk runtime config; `.xcode/` is gitignored, so do not commit or copy private values.
- `.xcode/scheduled-tasks.yaml`: local XCode schedule config; runtime state only.
- `.opencode/package.json`: declares the OpenCode plugin dependency; there are no project scripts.

## Skill Maintenance Rules

- When adding a new skill, create `.opencode/skills/<skill-name>/SKILL.md` with frontmatter containing `name` and `description`.
- Keep each skill focused on one domain. Do not merge unrelated workflows into one skill.
- Write `description` as the trigger surface: include when to use the skill, common user phrases, and expected capability.
- After adding or renaming a skill, update both `.opencode/skills/README.md` and root `README.md`.
- Do not write user-variable config, tokens, passwords, server credentials, or private IDs into skill directories.

## Existing Skill Rules

- For daily or weekly DingTalk reports, use `dingding-report-writer`.
- For frontend, UI, layout, styling, page, component, dashboard, or visual design work, use `frontend-design`.
- For k0s installation, uninstallation, reinstall, Kubernetes single-node deployment, airgap, CoreDNS, conntrack, kube-router, API URL, or admin token work, use `k0s-installer`.

## DingTalk Workflow Rules

- Always generate and show a checklist-style draft first, then wait for explicit natural-language confirmation before sending.
- Do not use the `question` component for send confirmation; ask in plain text instead.
- Never call `dingding_worklog_create_report` before confirmation, and always pass `toChat: false` when sending.
- Read recipients, template names, and `ddFrom` from `.xcode/dingding-report-config.yaml`; never write user-variable config into the skill directory.
- Resolve recipient names or phone numbers with DingTalk contact tools; never guess or invent `userId` values.
- Compute "today" and "this week" using `Asia/Shanghai`.
- Weekly reports default to summarizing this week's already sent daily reports; only fall back to workspace sessions if the user explicitly allows it.

## Commands And Verification

- There are no repo-level build, lint, typecheck, test, CI, pre-commit, dev-server, migration, or codegen commands configured.
- Do not invent `npm test`, `npm run build`, or similar checks.
- If you change Markdown/YAML/JSON, verify by rereading the changed file and checking the related source file it references.
- `.opencode/package-lock.json` is an npm lockfile for the OpenCode plugin dependency; avoid reinstalling dependencies unless the plugin dependency changes.

## State And Privacy

- `.omo/` and `.xcode/` are ignored runtime/local state. Treat contents as environment-specific and potentially sensitive.
- If documenting local config, describe keys and behavior, not concrete `userId`, channel account, tokens, passwords, or other private values.
