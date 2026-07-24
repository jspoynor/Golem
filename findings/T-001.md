# FINDINGS.md

M0 capability verification for `golem`. Each task appends its own section. Nothing
here is overwritten.

---

## Subagent fields

**Task:** T-001 · Verify subagent frontmatter fields
**Status:** complete. All five items resolved; none left uncertain.
**Method:** official docs only. Per the session constraint, T-001 wrote no code and ran
no plugin probes — every claim below is **confirmed (doc)** and cited. Local Claude Code
is `2.1.219`, which is at or above every `min-version` note quoted below.

**Sources:**

- https://code.claude.com/docs/en/sub-agents (frontmatter table, "Choose a model",
  "Available tools", "Choose the subagent scope")
- https://code.claude.com/docs/en/plugins (plugin structure, `settings.json` keys)

### 1 · The five items T-001 asked about

| Field | Verdict | Doc source |
|---|---|---|
| `effort` | **confirmed — exists** | sub-agents, frontmatter table |
| `maxTurns` | **confirmed — exists** | sub-agents, frontmatter table |
| `isolation: worktree` | **confirmed — exists** | sub-agents, frontmatter table |
| `model` | **confirmed — exists**, alias set is wider than SPEC assumed | sub-agents, "Choose a model" |
| omitted `model` | **confirmed — resolves to `inherit`** | sub-agents, "Choose a model" |

#### 1.1 `effort` — confirmed, exactly as §3.1 assumes

Verbatim: *"Effort level when this subagent is active. Overrides the session effort
level. Default: inherits from session. Options: `low`, `medium`, `high`, `xhigh`,
`max`; available levels depend on the model."*

The five-value set in T-001's brief is correct. **SPEC §3.1's `tiers` assumption — that
model and effort can both be pinned per agent — holds. No degrade needed.**

One qualifier worth carrying into T-013: *"available levels depend on the model."* The
docs do not publish the per-model matrix, so a tier pinning `xhigh`/`max` is not
guaranteed portable. §3.1 only uses `medium` and `high`, so this does not bite today.

#### 1.2 `maxTurns` — confirmed

Verbatim: *"Maximum number of agentic turns before the subagent stops."*

Exists and is usable as the implementer safety belt T-001 intended. Note the documented
behavior is that the subagent **stops** — the docs describe no distinct failure signal
on exhaustion, so a `maxTurns` cutoff and a clean finish are not obviously
distinguishable to the orchestrator. Relevant to §7.5, since a truncated subagent still
returns *something*. Not resolved here; flagged for T-015.

#### 1.3 `isolation: worktree` — confirmed, so the §7.7 path is real

Verbatim: *"Set to `worktree` to run the subagent in a temporary git worktree, giving it
an isolated copy of the repository branched by default from your default branch rather
than the parent session's `HEAD`. The worktree is automatically cleaned up if the
subagent makes no changes."*

Unused in v1 per §7.7 and §11. It exists, so the deferred parallelism experiment is not
built on a fiction. Recorded and left alone.

One detail that matters if that experiment ever happens: the worktree branches from the
**default branch, not the parent session's `HEAD`.** Under §7.8 (one commit per accepted
node) a parallel node would therefore start from a tree missing every commit the current
run has already made. Not a v1 problem; do not let it be forgotten.

#### 1.4 `model` — confirmed, with a wider alias set than SPEC assumed

Verbatim: *"Model to use: `sonnet`, `opus`, `haiku`, `fable`, a full model ID (for
example, `claude-opus-5`), or `inherit`. Defaults to `inherit`."*

"Choose a model" expands it:

- **Aliases:** `sonnet`, `opus`, `haiku`, **and `fable`**
- **Full model IDs:** e.g. `claude-opus-5`, `claude-sonnet-5` — *"Accepts the same
  values as the `--model` flag"*
- **`inherit`:** use the main conversation's model

T-001 asked specifically whether a Fable alias exists. **It does: `fable`, documented in
the same alias list as the other three.** Full model IDs are also accepted, so §3.1's
`config.tiers` may pin either form. The `"sonnet"` / `"opus"` values already in §3.1 are
valid as written.

#### 1.5 Omitted `model` — confirmed, sources no longer disagree

Verbatim: *"**Omitted**: defaults to `inherit` and uses the same model as the main
conversation."*

That is unambiguous on the official page. Golem pins explicitly anyway (§3.3), so this
is recorded rather than acted on.

### 2 · Resolution order — a real hazard to §3.3

Not requested by T-001, but it is the direct consequence of asking what `model` means,
and it contradicts a plain reading of §3.3. Documented order:

1. `CLAUDE_CODE_SUBAGENT_MODEL` environment variable, when set to an alias or model ID
2. The per-invocation `model` parameter
3. The subagent definition's `model` frontmatter
4. The main conversation's model

**Frontmatter is third.** §3.3 says *"Every agent pins its model explicitly. None
inherit."* As written that overstates what frontmatter can guarantee: a user with
`CLAUDE_CODE_SUBAGENT_MODEL` exported silently runs every Golem tier on one model, and
the reviewer quietly stops being the independent strong-model check §8.2 depends on.

Two further documented ways a pin fails to hold:

- *"Claude Code checks the environment variable, per-invocation parameter, and
  frontmatter values against your organization's `availableModels` allowlist. It skips a
  value that resolves to an excluded model and runs the subagent on the inherited model
  instead."* An org allowlist can therefore demote a pinned `opus` reviewer to the
  session model **with no error.**
- As of v2.1.196, `CLAUDE_CODE_SUBAGENT_MODEL=inherit` is treated as unset (earlier
  versions forced inheritance). Behavior here is version-dependent.

Golem cannot prevent any of this from a plugin. The honest options are to soften §3.3 to
"pins are declared, and the environment may override them," or to have `/golem:run` read
back the effective model and say so. **This is a T-003 reconciliation decision; not
amending SPEC.md here.**

### 3 · Adjacent constraints found while confirming the above

These came out of the same two pages and bear on later tasks. Recorded so T-003 sees
them once rather than each task rediscovering them.

#### 3.1 Plugin subagents silently ignore three frontmatter fields

Verbatim: *"For security reasons, plugin subagents don't support the `hooks`,
`mcpServers`, or `permissionMode` frontmatter fields. These fields are ignored when
loading agents from a plugin."*

Golem is a plugin (§1.2), so **all three are unavailable to every agent in §10.**

The load-bearing one is `permissionMode`. §6.7 and §10 require the planner to be *"read
plus write to `.golem/` only."* A `tools` allowlist restricts **which tools**, not
**which paths** — nothing in the frontmatter table scopes `Write` to a directory. So the
§10 planner restriction is **not fully expressible in plugin frontmatter as currently
specified.** Prompt instruction plus omitting `Edit` gets close; it is not the guarantee
§10's own "prompts are suggestions; allowlists are not" argues for. **Raise in T-003,
resolve in T-010.**

#### 3.2 Subagents run in the background by default, with a reduced tool set

Two filters narrow subagent tools. The first strips `Agent`, `AskUserQuestion`,
`EndConversation`, `EnterPlanMode`, `ExitPlanMode`, `ScheduleWakeup`, `TaskOutput`,
`WaitForMcpServers`, and `Workflow` from **every** subagent, *"even when listed in the
`tools` field."*

The second applies to background subagents — and *"as of v2.1.198 it runs subagents in
the background by default."* A background subagent keeps every MCP tool but only these
built-ins: `Read`, `Grep`, `Glob`, `Bash`, `PowerShell`, `Edit`, `Write`,
`NotebookEdit`, `WebFetch`, `WebSearch`, `TodoWrite`, `Skill`, `ToolSearch`,
`EnterWorktree`, `ExitWorktree`, `Monitor`, `TaskStop`, `SendMessage`, `Artifact`.

Consequences for Golem:

- Every tool §10 actually needs (read, edit, bash, search) survives both filters. **No
  §10 allowlist is broken by this.**
- `AskUserQuestion` is stripped from all subagents. This independently confirms §10's
  *"No interviewer agent — subagents can't talk to the user."* That non-goal is correct
  for a documented reason, not just a stylistic one.
- The same definition *"can resolve to different tools in the foreground and the
  background,"* and a `background: true` field exists. T-013 should not assume a tools
  list behaves identically in both.

#### 3.3 The `settings.json` `agent` key is real, and §1.3 is right to forbid it

The plugins page: plugin `settings.json` supports only `agent` and
`subagentStatusLine`, and *"Setting `agent` activates one of the plugin's custom agents
as the main thread, applying its system prompt, tool restrictions, and model."*

§1.3's prohibition targets a real key with exactly the effect §1.3 describes. Confirmed,
not hypothetical.

#### 3.4 Agent naming and hook matchers

- Frontmatter `name` is *"a unique identifier using lowercase letters and hyphens…
  the filename doesn't have to match."* The §1.4 filenames (`planner.md`,
  `implementer-light.md`, …) are all valid names; set `name` explicitly anyway.
- *"Hooks receive this value as `agent_type`."*
- Plugin `agents/` is scanned recursively, and subfolders join the scoped identifier —
  `agents/review/security.md` in plugin `my-plugin` registers as
  `my-plugin:review:security`. §1.4 keeps agents flat, so Golem's are
  `golem:planner`, `golem:implementer-light`, `golem:implementer-standard`,
  `golem:reviewer`. **This agrees with T-002 §2.3's matcher-scoping finding.**
- Explicit @-mention (§7.6) is `@agent-golem:implementer-light`.

### 4 · Acceptance check

| T-001 acceptance requirement | Status |
|---|---|
| `effort` named confirmed/absent/uncertain, with doc URL | ✅ §1.1 — confirmed |
| `maxTurns` | ✅ §1.2 — confirmed |
| `isolation: worktree` | ✅ §1.3 — confirmed |
| `model` accepted values, no guessing | ✅ §1.4 — confirmed; `fable` alias and full model IDs both documented |
| What an omitted `model` resolves to | ✅ §1.5 — confirmed `inherit` |
| Doc URL for each | ✅ header + per-section citations |
| Section is `## Subagent fields` in `FINDINGS.md`, committed | ✅ this section; commit pending user approval |

**Net effect on SPEC.md:** §3.1's model-plus-effort tier assumption is **verified — no
degrade required.** Three items are raised for T-003: the model-pin override hazard
(§2), the planner path-restriction gap (§3.1), and the `maxTurns` stop-vs-fail
ambiguity (§1.2). T-001 amends nothing itself.
