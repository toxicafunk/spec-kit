# Speckit Dot-to-Hyphen Prompt Reference Fixes

The frontmatter `name:` fields in `.forge/commands/` are already correct (hyphens).
The `.specify/` extension/hook/workflow system uses dots internally and does **not** need changing.

The problem is in **prompt bodies and templates**: the LLM is instructed using `/speckit.foo`
dot notation, so it will tell users to type commands that silently fail in zsh mode.
All `/speckit.foo` references need to become `/speckit-foo`.

---

## Command files — `.forge/commands/`

| File | References to fix |
|---|---|
| `speckit.analyze.md` | `/speckit.tasks`, `/speckit.implement`, `/speckit.specify`, `/speckit.plan`, `/speckit.analyze` |
| `speckit.clarify.md` | `/speckit.specify`, `/speckit.plan`, `/speckit.clarify` |
| `speckit.specify.md` | `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.clarify` |
| `speckit.checklist.md` | `/speckit.checklist` |
| `speckit.implement.md` | `/speckit.tasks` |
| `speckit.git.feature.md` | `/speckit.specify` |

## Template files — `.specify/templates/`

| File | References to fix |
|---|---|
| `checklist-template.md` | `/speckit.checklist` |
| `plan-template.md` | `/speckit.plan`, `/speckit.tasks` |
| `tasks-template.md` | `/speckit.tasks` |

---

## What does NOT need changing

- `.forge/commands/speckit*.md` frontmatter `name:` fields — already use hyphens
- `.specify/extension.yml` `provides.commands[].name` and `hooks[].command` — speckit's internal hook dispatch, not forgecode command names
- `.specify/extensions.yml` hook entries — same
- `.specify/workflows/speckit/workflow.yml` `steps[].command` — speckit's workflow runner
- `.specify/extensions/.registry` `registered_commands` — speckit's extension registry
- All README and documentation under `.specify/extensions/` — speckit docs, not LLM prompts
