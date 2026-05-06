# Forge Dot-to-Hyphen Prompt Reference Fixes

## Objective

Fix the Forge integration so that slash-command references in the bodies of generated `.forge/commands/` files, `.specify/templates/` files, and extension command files use hyphen notation (`/speckit-foo`) instead of dot notation (`/speckit.foo`). This prevents the LLM from telling users to type commands that silently fail in ZSH.

### Root Causes

Three independent code paths each omit the hyphen separator for Forge:

1. **Command files** — `ForgeIntegration.setup()` calls `process_template()` without passing `invoke_separator`, so `__SPECKIT_COMMAND_*__` tokens resolve to `/speckit.foo` instead of `/speckit-foo`.

2. **Shared templates** — `install_shared_infra` derives the separator via `effective_invoke_separator()`. Because `ForgeIntegration` lacks `invoke_separator = "-"`, it returns `"."`, so `plan-template.md`, `tasks-template.md`, and `checklist-template.md` also get dot-notation references.

3. **Extension command** — `extensions/git/commands/speckit.git.feature.md` contains a hard-coded literal `/speckit.specify` string. The `agents.py` `register_commands` pipeline never applies `resolve_command_refs`, so even if the source used a token, it would be left unresolved.

---

## Implementation Plan

### Phase 1 — Fix `ForgeIntegration` class (`src/specify_cli/integrations/forge/__init__.py`)

- [ ] Task 1. Add `invoke_separator = "-"` as a class-level attribute on `ForgeIntegration` (below `context_file`). This single attribute fixes the shared-template path: `effective_invoke_separator()` now returns `"-"` for Forge, so `install_shared_infra` will pass the correct separator to `resolve_command_refs` when installing `.specify/templates/` files.

- [ ] Task 2. Add `"invoke_separator": "-"` as a key inside `ForgeIntegration.registrar_config`. This makes the separator available to `agents.py`'s `CommandRegistrar` through `AGENT_CONFIGS["forge"]`, enabling the extension-command fix in Phase 3.

- [ ] Task 3. In `ForgeIntegration.setup()`, pass `invoke_separator=self.invoke_separator` as a keyword argument to the `self.process_template(...)` call (line ~133). This aligns Forge's command-file generation with the existing `SkillsIntegration.setup()` pattern, which already passes `invoke_separator=self.invoke_separator` to the same method.

### Phase 2 — Fix git extension template (`extensions/git/commands/speckit.git.feature.md`)

- [ ] Task 4. On line 7, replace the literal string `/speckit.specify` with the token `__SPECKIT_COMMAND_SPECIFY__`. This converts the hard-coded reference into a resolvable placeholder that will be emitted correctly for every agent.

### Phase 3 — Apply `resolve_command_refs` in the extension registration pipeline (`src/specify_cli/agents.py`)

- [ ] Task 5. In `CommandRegistrar.register_commands()`, after all argument-placeholder conversions on `body` (around line 509 / 513), add a `re.sub` call that resolves `__SPECKIT_COMMAND_*__` tokens using the agent's `invoke_separator`. The separator should be read from `agent_config.get("invoke_separator", ".")` so that the default `"."` is preserved for every other agent. The regex and replacement logic should mirror `IntegrationBase.resolve_command_refs` exactly:
  ```
  separator = agent_config.get("invoke_separator", ".")
  body = re.sub(
      r"__SPECKIT_COMMAND_([A-Z][A-Z0-9_]*)__",
      lambda m: "/speckit" + separator + m.group(1).lower().replace("_", separator),
      body,
  )
  ```
  Apply this transformation to the `body` variable in all three format branches (`markdown`, `toml`, `yaml`) before the `render_*` call, or apply it once after the shared argument-placeholder conversion block (lines 491–493) before the format branches diverge.

### Phase 4 — Update tests (`tests/integrations/test_integration_forge.py`)

- [ ] Task 6. In `TestForgeIntegration.test_templates_are_processed`, add an assertion that no generated command file contains the pattern `/speckit.` (dot-notation command reference) in its body. This catches any future regression where `process_template` loses the hyphen separator. A targeted check like: for each `.md` file under `.forge/commands/`, assert that the body does not contain `/speckit.` followed by a lowercase command name.

- [ ] Task 7. Add a new test method `test_command_refs_use_hyphen_notation` in `TestForgeIntegration` that sets up Forge in a `tmp_path`, reads every generated `.forge/commands/speckit.*.md` file, and asserts that none of them contain the regex pattern `/speckit\.[a-z]` (dot-separated command references), while at least one file contains a `/speckit-` reference (confirming hyphen notation is present).

- [ ] Task 8. Add a new test method `test_git_extension_command_uses_hyphen_notation` in `TestForgeCommandRegistrar` that registers the real `speckit.git.feature.md` extension command for Forge and asserts the output file body contains `/speckit-specify` (not `/speckit.specify`).

---

## Verification Criteria

- `ForgeIntegration.invoke_separator` equals `"-"`.
- `ForgeIntegration.registrar_config["invoke_separator"]` equals `"-"`.
- All files generated by `forge.setup()` in a temp project contain zero occurrences of the pattern `/speckit.[a-z]` in their body text.
- `.specify/templates/plan-template.md`, `tasks-template.md`, and `checklist-template.md` installed under a Forge project contain `/speckit-plan`, `/speckit-tasks`, `/speckit-checklist` rather than `/speckit.plan`, `/speckit.tasks`, `/speckit.checklist`.
- The git extension's `speckit.git.feature.md` source file no longer contains the literal string `/speckit.specify`; it uses `__SPECKIT_COMMAND_SPECIFY__` instead.
- After `register_commands("forge", ...)` processes the git extension command, the output file body contains `/speckit-specify` not `/speckit.specify`.
- All existing Forge tests continue to pass.
- The `re.sub` call in `agents.py` does not alter output for agents that don't set `invoke_separator` in their `registrar_config` (defaults to `"."`).

---

## Potential Risks and Mitigations

1. **Circular import when adding `re.sub` in `agents.py`**
   Mitigation: The fix is a plain `re.sub` inline call using `re` (already imported at the top of `agents.py`). No new imports are required; `IntegrationBase` is not imported.

2. **`registrar_config` key collision — `invoke_separator` already used elsewhere**
   Mitigation: No existing integration has `invoke_separator` in its `registrar_config`. The key is read with `.get("invoke_separator", ".")`, so omitting it for all other agents is safe.

3. **Extension commands for non-Forge agents inadvertently get hyphen refs**
   Mitigation: The `agents.py` change uses `agent_config.get("invoke_separator", ".")` as default. Since no other agent sets this key in `registrar_config`, every other agent continues to produce dot-notation references as before.

4. **`__SPECKIT_COMMAND_SPECIFY__` token unresolved in other extension pipelines**
   Mitigation: The token is already understood by `IntegrationBase.resolve_command_refs` for all agents. The change in Task 4 moves from a literal that was always wrong for Forge to a token that is correctly resolved for every agent by the Phase 3 change.

5. **Existing tests that assert on specific output file content**
   Mitigation: Task 6 and 7 add new assertions; no existing assertions are removed. The only behavioral change is `/speckit.` → `/speckit-` inside Forge-generated command bodies, which no current test asserts on positively.

---

## Alternative Approaches

1. **Post-process approach in `ForgeIntegration._apply_forge_transformations()`**: Add a body-level `str.replace("/speckit.", "/speckit-")` inside the existing Forge-specific transformation step. Trade-off: simpler but fragile — it would also replace `/speckit.` in non-command-reference contexts (e.g., file paths or YAML keys that happen to contain that prefix). Using the token-based `resolve_command_refs` approach (Phase 3) is safer and composable.

2. **Skip `agents.py` changes; handle extension commands in `ForgeIntegration.setup()` only**: Forge could override an extension-command post-processing hook. Trade-off: requires a larger refactor of the integration interface and still doesn't fix the `speckit.git.feature.md` literal string without the token change in Task 4. The Phase 3 approach is simpler and affects only one centralized location.
