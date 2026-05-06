# Should ForgeIntegration Use SkillsIntegration?

**Short answer: No.**

Forge and skills-based agents (Claude, Codex, Kimi, …) share only one thing in common —
`invoke_separator = "-"` (hyphen notation for slash-command references). Every other structural
detail is incompatible. The right fix is to propagate `invoke_separator` from the integration
class attribute into `AGENT_CONFIGS` at build time, not to change Forge's base class.

---

## Output format comparison

### Forge — `.forge/commands/speckit.plan.md`

```markdown
---
name: speckit-plan
description: Execute the implementation planning workflow using the plan template to generate design artifacts.
---

## User Input

{{parameters}}

(body continues …)
```

### Claude skill — `.claude/skills/speckit-plan/SKILL.md`

```markdown
---
name: "speckit-plan"
description: "Execute the implementation planning workflow using the plan template to generate design artifacts."
argument-hint: "Optional guidance for the planning phase"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/plan.md"
user-invocable: true
disable-model-invocation: false
---

## User Input

$ARGUMENTS

(body continues …)
```

---

## Structural differences

| Concern | Forge (`MarkdownIntegration`) | Claude / Codex (`SkillsIntegration`) |
|--|--|--|
| **File layout** | Flat file: `.forge/commands/speckit.plan.md` | Directory-per-skill: `.claude/skills/speckit-plan/SKILL.md` |
| **File naming** | `speckit.<name>.md` — named after the command | Always `SKILL.md` — the directory carries the name |
| **Argument placeholder** | `{{parameters}}` | `$ARGUMENTS` |
| **Frontmatter origin** | Passed through from the source template; specific keys stripped or injected | Rebuilt entirely from scratch; template frontmatter is discarded |
| **Frontmatter fields** | `name` + `description` only | `name`, `description`, `argument-hint`, `compatibility`, `metadata`, `user-invocable`, `disable-model-invocation` |
| **Specification** | Forge-proprietary `.md` command format | [agentskills.io](https://agentskills.io/specification) skill spec |
| **Forge-specific transforms** | Strips `handoffs` key; injects `name` in hyphenated format; replaces `$ARGUMENTS` → `{{parameters}}` | None of these apply |
| **invoke_separator** | `"-"` (class attribute + explicit `registrar_config` entry) | `"-"` (inherited from `SkillsIntegration` class attribute only) |

---

## Why switching Forge to SkillsIntegration would break things

`SkillsIntegration.setup()` would:

1. **Change the output directory structure** — generate `speckit-plan/SKILL.md` directories instead
   of the flat `.forge/commands/speckit.plan.md` files that Forge expects to read.
2. **Use `$ARGUMENTS`** instead of `{{parameters}}` — Forge does not recognise `$ARGUMENTS` and
   would surface it as literal text in the UI.
3. **Rebuild frontmatter from scratch** — inject `compatibility`, `metadata`, `user-invocable`, and
   `author` fields that Forge either ignores or may reject.
4. **Lose Forge-specific transforms** — the `handoffs` key stripping and the `inject_name`
   hyphenation logic (via `format_forge_command_name`) would disappear.

---

## The actual fix (PR #2462, commit `cd6f677`)

The only real problem was that `_build_agent_configs()` in `agents.py` copied each integration's
`registrar_config` verbatim. Skills agents declare `invoke_separator = "-"` as a class attribute
on `SkillsIntegration`, but they don't repeat it in `registrar_config`. So when
`register_commands()` resolved `__SPECKIT_COMMAND_*__` tokens, it fell back to `"."` for all
skills agents.

The fix propagates `invoke_separator` from the integration class into the config dict at build
time, when `registrar_config` doesn't already declare it:

```python
# agents.py — _build_agent_configs()
if "invoke_separator" not in config:
    config["invoke_separator"] = integration.invoke_separator
```

This makes the integration class the single source of truth for the separator across both the
template-processing path (`MarkdownIntegration.setup()` / `ForgeIntegration.setup()`) and the
extension-registration path (`CommandRegistrar.register_commands()`), without forcing incompatible
agents into a shared base class.

---

## When SkillsIntegration IS the right base class

Use `SkillsIntegration` when the agent follows the agentskills.io spec:

- Commands are installed as `speckit-<name>/SKILL.md` directories.
- The agent reads `SKILL.md` files from those directories at invocation time.
- `$ARGUMENTS` is the correct argument placeholder.
- Rich frontmatter (`compatibility`, `metadata`, etc.) is expected or at least harmless.

Current integrations that correctly extend `SkillsIntegration`: `claude`, `codex`, `kimi`, `agy`.

Forge does **not** meet any of these criteria.
