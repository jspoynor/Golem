# Build Prompt: `golem` — a Claude Code plugin for large-scope task orchestration

Paste this into Claude Code in an empty directory. It is a specification, not a
sketch — the design decisions below were resolved deliberately and each one has a
reason. Where you disagree, **say so before building**, don't silently improve it.

---

## 0. Before you write anything

Some fields in this spec need verification against current Claude Code
documentation. Check these first and report what you find:

1. **`effort` field on subagent definitions** — this spec assumes agents can pin a
   model *and* an effort level (`low`/`medium`/`high`/`xhigh`/`max`). This appeared
   in third-party documentation, not the official subagents page. If it does not
   exist, degrade to model-only tiers and tell me.
2. **`maxTurns` on subagent definitions** — used as a safety belt on implementers.
3. **`isolation: worktree`** — not used in v1, but confirm it exists so the v2
   parallelism path is real.
4. **Model aliases** — documented aliases are `sonnet`, `opus`, `haiku`. If a
   Fable alias or full model ID is available, report it. Do not guess.
5. **Skill namespacing** — plugin skills are namespaced `/plugin-name:skill-name`.
   Confirm the final invocation names.

Then read `https://code.claude.com/docs/en/plugins` and
`https://code.claude.com/docs/en/sub-agents` before implementing.

---

## 1. What Golem is

A **plugin**, not a repo template. The orchestration logic travels with the plugin
so improvements propagate; only per-run state lives in the working repo.

Golem is **inert by default**. Installing it changes nothing about a normal Claude
Code session. It activates only on explicit skill invocation.

**Do not set the `agent` key in the plugin's `settings.json`.** It would activate a
custom agent as the main thread and silently change default Claude Code behavior.
This is explicitly forbidden.

### Structure

```
golem/
├── .claude-plugin/
│   └── plugin.json          name, description, version (explicit, not SHA-derived)
├── skills/
│   ├── init/SKILL.md
│   ├── spec/SKILL.md
│   ├── plan/SKILL.md
│   ├── run/SKILL.md
│   ├── resume/SKILL.md
│   └── status/SKILL.md
├── agents/
│   ├── planner.md
│   ├── implementer-light.md
│   ├── implementer-standard.md
│   └── reviewer.md
├── hooks/
│   └── hooks.json
└── README.md
```

Directories go at the plugin **root**, never inside `.claude-plugin/`.

---

## 2. Repo state: the `.golem/` contract

```
.golem/
├── config.json          COMMITTED
├── spec.md              COMMITTED — the source of truth
├── tasks.json           GITIGNORED — derived build artifact
└── runs/                GITIGNORED
    └── <run-id>/
        ├── progress.log
        ├── T-014.md          full subagent writeups
        └── halt.md           written on halt
```

Ship a `.gitignore` fragment that `/golem:init` appends.

**`spec.md` describes the desired end state, not remaining work.** This matters for
recovery (§8): the planner reads the repo, sees what already exists, and plans only
the gap. The user never surgically edits completed work out of the spec.

**`tasks.json` is never hand-edited.** It is regenerated from `spec.md` by
`/golem:plan`. This is why there is no spec/DAG drift and no DAG validator.

### `config.json`

```json
{
  "verify": "npm test && npm run typecheck",
  "maxRetries": 2,
  "maxParallel": 1,
  "tiers": {
    "implementer-light":    { "model": "sonnet", "effort": "medium" },
    "implementer-standard": { "model": "sonnet", "effort": "high" },
    "reviewer":             { "model": "opus",   "effort": "high" },
    "planner":              { "model": "opus",   "effort": "high" }
  }
}
```

Invocation flags override config values per run (e.g. `--max-retries 4`).
There is **no budget ceiling** — runs take as long as they take. The run is bounded
structurally: finite DAG, capped retries, halt-on-failure, serial execution.

---

## 3. Skills

| Skill | Does |
|---|---|
| `init` | Establishes `.golem/`, writes `config.json`, appends `.gitignore`. **Interactively establishes `verify`** — never auto-detect it. |
| `spec` | Runs the interview, writes `spec.md`, **stops**. |
| `plan` | `spec.md` + repo → `tasks.json`. Also the re-plan path after a halt. |
| `run` | Executes the frozen DAG. |
| `resume` | Reconstructs from `progress.log` + `tasks.json` after a compaction. |
| `status` | Reads current run state, reports, exits. |

**`verify` must be established by asking the user.** A wrong guess produces a
command that exits 0 without testing anything, which is worse than no gate because
it looks like one.

**The gate between `spec` and `run` is hard.** `/golem:spec` never flows into
execution. The user reads `spec.md` and `tasks.json` and approves by invoking the
next command.

---

## 4. The interview (`/golem:spec`)

An unbounded, adaptive grilling session. Blunt, not deferential. It ends when the
interviewer is satisfied, not at a question count.

**Phase 0 — reconnaissance, zero questions.** Read the repo. Report findings: stack,
conventions, test setup, relevant existing code. Facts discoverable in the codebase
are looked up, never asked. This also surfaces a badly-misread project before the
misreading contaminates thirty questions.

**Then:** one question at a time. Each question states the interviewer's recommended
answer and its reasoning. Never ask multiple questions in one turn.

**Only ask questions whose answer would change the plan.** If the user's answer
wouldn't alter the default, don't ask — record the default under `## Assumptions`
in `spec.md` for review at the gate.

**Anti-rubber-stamp rule (non-configurable):**

- After **one** consecutive unmodified agreement, the interviewer **must** present
  the strongest case *against its own recommendation* before accepting.
- Bare agreement ("sounds good", "I agree") does not advance. The user must
  articulate the tradeoff — what they're accepting and what they're giving up.

Do **not** implement a rule requiring the user to change something. It manufactures
fabricated objections that then enter `spec.md` as real requirements. The goal is
engagement, not disagreement.

**Must extract:** what's explicitly out of scope; milestone-level acceptance
criteria; the `verify` command; and "which part of this is most likely to be wrong?"
— asked of the user and answered independently by the interviewer.

---

## 5. Planning (`/golem:plan`)

The `planner` agent reads `spec.md` and the repo, emits `tasks.json`.

**Node schema:** id, description, dependencies, acceptance criteria (concrete —
"`npi.test.ts` passes"), `complexity` (`light` | `standard`), files likely touched.

**Decomposition rules:**

- **Greenfield: the first milestone is always scaffold-first** — project skeleton,
  toolchain, one passing smoke test. Without it the verification gate has nothing to
  run during the exact phase where errors compound hardest.
- **Test-first for feature nodes:** `tests-for-X` precedes `implement-X`. This
  roughly doubles DAG size. That is the accepted cost of a real gate.

The planner **cannot edit source files** — tool allowlist is read + write to
`.golem/` only.

---

## 6. Execution (`/golem:run`)

The **main thread is the product owner.** It holds the loop, makes routing
decisions, applies creative judgment at the plan level, and never gets into the
weeds.

**The DAG is frozen for the duration of the run.** No mid-run replanning.

### Context hygiene (load-bearing — this is why the orchestrator survives)

- Never reads source diffs.
- Never reads `runs/*.md` unless a `detail_ref` warrants it.
- Reads `tasks.json` **once per planning cycle**, edits status in place, and does
  **not** re-read to confirm its own write. Re-reading state every turn is the
  failure mode that makes disk-backed state *more* expensive than in-context.
- Appends one line to `progress.log` after each node: node id, verdict, next ready
  node. This makes compaction a hiccup rather than a run-ender.

### Loop

1. Select next ready node.
2. Dispatch to `implementer-light` or `implementer-standard` per node `complexity`.
   **@-mention the agent explicitly** — do not rely on auto-delegation.
3. Verification gate fires (§7).
4. On pass: `reviewer` adjudicates the diff against acceptance criteria.
5. On accept: commit, mark complete, continue.
6. On failure: §8.

**Serial only. `maxParallel` stays at 1.** The key exists for a later experiment;
v1 does not implement concurrency.

### Subagent return shape

Every subagent returns exactly this. Nothing else enters the orchestrator's context.

```
verdict       pass | fail | spec-problem
summary       2–3 lines — what changed
plan_impact   see below, or null
detail_ref    .golem/runs/<id>/T-014.md
```

`detail_ref` is the escape hatch: the orchestrator reads the full writeup **on
demand** when it decides it needs to. Cheap by default, expensive by choice.

**`plan_impact`** — a fact that makes a downstream node's premise false:

```
affected    [T-022, T-031]     node IDs — required
claim       one sentence — what is no longer true
evidence    file:line, or the command output that proves it
```

**Hard filter: if it cannot name an affected node id, it is not `plan_impact`.**
It's an observation and belongs in the detail file. This prevents the field becoming
a token printer.

**A non-null `plan_impact` halts the run.** The alternative is knowingly executing a
plan built on a false premise.

### Commits

**One commit per gate-passing, reviewer-accepted node.** Message includes the node
id and its acceptance criteria. This is what makes a halt state readable, bisectable,
and revertable per-task. History noise is fixable with a squash-on-merge; missing
granularity at halt time is not fixable at all.

---

## 7. Verification

**The deterministic gate is a hook, not an instruction.** Implement it in
`hooks/hooks.json` on the appropriate subagent-completion event. An instruction
saying "verify before marking complete" is a suggestion to a system strongly
motivated to report success. A hook is a fact.

The gate runs `config.verify` plus any node-specific command in the acceptance
criteria. Exit 0 or the node fails.

**Then `reviewer`, layered on top.** Fresh context, reads the diff and the acceptance
criteria, hunts for the failure. Returns `pass`, `fail`, or `spec-problem`.

**`reviewer` is read-only.** Tool allowlist: read/search/bash, no edit. A reviewer
that can edit will "helpfully" fix what it finds, and its verdict then describes its
own repair rather than the implementer's work.

---

## 8. Failure handling

**Halt on failure. Do not continue independent branches.** A clean halt at task 14 is
recoverable; a swiss-cheese repo at task 44 is a bisect.

### Implementation failures (gate fails, or reviewer returns `fail`)

- Retry up to `maxRetries` (default 2, overridable per-run).
- **Always a fresh subagent.** Never continue the failed one — its dead-end reasoning
  biases it toward the approach it just disproved.
- The fresh agent receives **a high-level summary of the previous attempt's
  approach**, not just the error output. "Tried X, X failed because Y" — so it takes
  a different path rather than reconstructing the same one from the stack trace.
- **Each retry escalates one tier** (`light` → `standard`). Difficulty is discovered
  empirically by the gate rather than predicted by the planner.
- **Escalation is best-effort.** A node already at `standard` retries at `standard`,
  then halts. Do not invent a third tier.
- Retries exhausted → halt.

### Spec failures (reviewer returns `spec-problem`)

The classification is itself a model judgment and can be wrong, so it gets **exactly
one** tiebreak:

- Spawn a **fresh `reviewer`, blind** — same diff, same criteria, **never shown the
  first verdict.** Anchoring is strong; a reviewer told "this is a spec contradiction"
  will find one. The independence is the entire mechanism.
- Both say `spec-problem` → halt with confidence.
- Split → treat as an implementation failure, fall through to the retry path above.

### Halt output

Write `.golem/runs/<id>/halt.md`: trigger, the `plan_impact` or failure detail,
which nodes completed, and the last commit. If it looks like a fundamental spec
problem rather than a local one, say so in prose.

### Recovery (this is the whole loop — there is no reconciliation machinery)

1. Run halts. Clean git history through the last accepted node.
2. User edits `spec.md` — correcting what was wrong, not excising completed work.
3. `/golem:plan` — the planner reconnoiters a repo that now contains more code and
   emits a **fresh** DAG for whatever the spec describes that doesn't exist yet.
4. `/golem:run` — new run id, new DAG.

**A re-plan is just a plan against a repo with more code in it.** The old DAG is
discarded, never merged. Hand the planner the previous run's completed-node manifest
as a hint, but **the repo is authoritative** — the manifest never overrides disk.

---

## 9. Agents

Every agent **pins its model explicitly.** None inherit. Documentation is ambiguous
about what an omitted `model` resolves to, and Golem should never find out.

| Agent | Model/effort | Tools | Why it's separate |
|---|---|---|---|
| `planner` | strong / high | read, write to `.golem/` only | A planner that can edit source will start coding instead of planning. |
| `implementer-light` | cheap / medium | read, edit, bash | Throughput tier. |
| `implementer-standard` | cheap / high | read, edit, bash | Escalation tier. |
| `reviewer` | strong / high | read, search, bash — **no edit** | Read-only is load-bearing (§7). |

Read tiers from `config.tiers` rather than hardcoding them in the agent files, so
they're tunable per repo.

**No debugger agent** — the design halts on failure rather than escalating to
open-ended debugging.
**No interviewer agent** — the interview runs in the main thread; subagents can't
talk to the user.
**No scaffolder agent** — scaffold-first is a milestone in the DAG, built by an
implementer like anything else.

---

## 10. Explicit non-goals for v1

- Parallel execution
- Mid-run replanning
- A debugger/escalation agent
- Budget ceilings
- Auto-detecting the `verify` command
- Anything set via the plugin's `settings.json` `agent` key

---

## 11. Deliverables

1. The plugin, complete and loadable via `claude --plugin-dir ./golem`.
2. A `README.md` covering install, the six skills, the `.golem/` contract, and the
   config schema.
3. **A report of the §0 verification findings** — which fields exist, which don't,
   and what you changed as a result.
4. A short list of anything in this spec you think is wrong, with your reasoning.
