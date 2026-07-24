# TASKS.md — building `golem`

Build order for the plugin specified in `SPEC.md`. Work one task at a time, in a
fresh session per milestone. Paste `SPEC.md` in full each session, then say which
task you're implementing.

**The order is a constraint, not a suggestion.** M1 exists so every later milestone
has a working `--plugin-dir` load as its verify command. Building the hook system
before the plugin loads means debugging two unknowns at once.

Within a milestone, decompose freely. Across milestones, don't reorder.

**Verify command for this build**, established at M1:

```
claude plugin validate ./golem && claude --plugin-dir ./golem
```

Then `/reload-plugins` and exercise the skill under test.

---

## M0 — Capability verification

No code. Establishes facts the rest of the build depends on.

**All M0 findings are written to `FINDINGS.md` at repo root and committed.** Session
output is not durable — a later session cannot read a report that only existed in a
terminal. Each task appends its own section; none overwrite the file.

### T-001 · Verify subagent frontmatter fields

**Deps:** none

Read `https://code.claude.com/docs/en/sub-agents` and
`https://code.claude.com/docs/en/plugins`. Determine which of these exist and how
they behave:

- `effort` (`low`/`medium`/`high`/`xhigh`/`max`) — `SPEC.md` §3.1 assumes model and
  effort can both be pinned. This came from third-party documentation, not the
  official page.
- `maxTurns` — intended as a safety belt on implementers.
- `isolation: worktree` — unused in v1; confirm it exists so the §7.7 parallelism
  path is real.
- `model` — confirm accepted values. Documented aliases are `sonnet`, `opus`,
  `haiku`. If a Fable alias or full model ID is available, note it. **Do not guess.**
- Confirm what an **omitted** `model` resolves to. Sources disagree. Golem pins
  explicitly either way (§3.3), but the answer should be recorded.

**Contract:** writes `## Subagent fields` to `FINDINGS.md`. T-003 reads it.

**Acceptance:** `FINDINGS.md` contains a `## Subagent fields` section naming each
field as confirmed, absent, or uncertain, with the doc URL for each. The file is
committed.

### T-002 · Verify skill namespacing and hook events

**Deps:** none — independent of T-001; both feed T-003

Confirm the final invocation names for the six skills in §4 (expect
`/golem:run` form). Confirm which hook event fires on subagent completion — this is
what §8.1 depends on, and it is the single most load-bearing mechanism in the design.

**Contract:** writes `## Skills and hooks` to `FINDINGS.md`. T-003 reads it, and T-014
depends on the hook event name recorded here.

**Acceptance:** `FINDINGS.md` contains a `## Skills and hooks` section with the exact
invocation strings and the exact hook event name plus its input schema. The file is
committed.

### T-003 · Reconcile SPEC.md with findings

**Deps:** T-001, T-002 — reads `FINDINGS.md`, not session history

Amend `SPEC.md` in place where T-001/T-002 contradict it. If `effort` does not exist,
degrade §3.1 tiers to model-only and say so. If the subagent-completion hook event
doesn't exist or can't run a command, **stop and raise it** — §8.1 is not
substitutable with an instruction.

**Acceptance:** `SPEC.md` contains no unverified assumptions. Every change is listed
in the session output.

---

## M1 — Plugin skeleton

Establishes the verify command. Nothing after this is trustworthy without it.

### T-004 · Manifest and loadable plugin

**Deps:** T-003

Create `golem/.claude-plugin/plugin.json` per §1.4 with an explicit `version`.
Create one trivial skill to prove loading works.

**Acceptance:** `claude plugin validate` passes. `claude --plugin-dir ./golem` loads
without error. The trivial skill runs and `/help` lists it under the namespace.

### T-005 · Deliberately break it

**Deps:** T-004

Introduce a manifest error, confirm `claude plugin validate` fails, then revert.

**Acceptance:** the verify command has been observed failing. A gate never seen to
fail is trusted on faith.

---

## M2 — Repo state and `/golem:init`

### T-006 · `.golem/` scaffolding

**Deps:** T-004

Implement `/golem:init` per §2.1 and §4.1: create the directory tree, write
`config.json` with the §3.1 schema, append ignore rules to `.gitignore`.

**Contract:** every later milestone reads `config.json`. Field names are fixed by
§3.1 and must not drift.

**Acceptance:** running in an empty repo produces the §2.1 layout. `config.json`
validates against the §3.1 schema. `.gitignore` covers `tasks.jsonl`, `tasks/`,
`archive/`, `runs/` and not `spec.md` or `config.json`.

### T-007 · Interactive `verify` establishment

**Deps:** T-006

`/golem:init` asks the user for the verify command. It may inspect the repo and
*suggest*, but never silently auto-detects (§4.1).

**Acceptance:** init cannot complete without an explicitly confirmed `verify` value.
Run it in a repo with no test setup and confirm it still asks rather than inventing
something.

---

## M3 — The interview

Longest prose section. Its own session.

### T-008 · `/golem:spec` interview

**Deps:** T-006

Implement §5 in full: reconnaissance phase with zero questions (§5.2), one question
per turn with stated recommendations (§5.3), the anti-rubber-stamp rule (§5.4), the
required extractions (§5.5).

**Out of scope:** any rule requiring the user to change something. §5.4 explains why —
do not re-add it.

**Acceptance:** run it on a small real task. Confirm the recon phase asks nothing.
Confirm that agreeing twice in a row triggers the counter-argument. Confirm bare "I
agree" does not advance.

### T-009 · `spec.md` output format

**Deps:** T-008

Define and emit the `spec.md` structure: stable numbered anchors (§5.6), milestone
acceptance criteria, out-of-scope section, `## Assumptions`.

**Contract:** §6.4 node briefs cite these anchors. The numbering scheme is consumed
by the planner and must be stable across regenerations of the same spec.

**Acceptance:** a generated `spec.md` has numbered sections, and every question the
interviewer declined to ask appears under `## Assumptions`.

---

## M4 — Planning

### T-010 · `planner` agent

**Deps:** T-003, T-009

Create `agents/planner.md` per §10: tier from `config.tiers`, tools limited to read
plus write to `.golem/` only.

**Acceptance:** the planner cannot edit a source file. Verify by asking it to.

### T-011 · `/golem:plan` — index and briefs

**Deps:** T-010

Emit `tasks.jsonl` (§6.2, routing fields only) and `tasks/T-NNN.md` briefs (§6.3).
Archive the previous plan first (§6.1).

**Contract:** `tasks.jsonl` is parsed by the orchestrator in M6. Field names are fixed
by §6.2. **No acceptance criteria in the index** — §7.2.

**Acceptance:** every index line parses as JSON and has a matching brief file. Every
brief has all §6.3 sections. Re-running archives the prior plan rather than deleting
it. No brief contains the full spec text — only `spec:` pointers.

### T-012 · Decomposition rules

**Deps:** T-011

Enforce §6.6: scaffold-first as the first milestone on greenfield, test-first ordering
for feature nodes. Populate `## Contract` per §6.5.

**Acceptance:** planning a greenfield project produces a scaffold milestone first.
Every `implement-X` node has a `tests-for-X` node in its `deps`. Nodes with
`dependents` have a non-empty `## Contract`.

---

## M5 — Agents and the verification gate

§8 is the accuracy floor. It gets its own milestone and is not bundled with the loop.

### T-013 · Implementer and reviewer agents

**Deps:** T-003

Create `implementer-light.md`, `implementer-standard.md`, `reviewer.md` per §10.
Tiers read from `config.tiers` (§3.3).

**Contract:** all three return the §7.5 shape. The orchestrator in M6 parses it.

**Acceptance:** the reviewer cannot edit a file — verify by asking it to. Each agent
pins a model explicitly; none inherit.

### T-014 · The verification hook

**Deps:** T-002, T-013

Implement `hooks/hooks.json` on the subagent-completion event from T-002. Runs
`config.verify` plus any node-level override. Non-zero exit fails the node.

**This must be a hook, not an instruction** (§8.1). If T-002 found no suitable event,
stop and raise it rather than substituting a prompt.

**Acceptance:** a subagent that produces failing code is marked failed **without the
orchestrator being asked to judge**. Verify by deliberately breaking the code.

### T-015 · Structured return shape

**Deps:** T-013

Enforce §7.5 across all subagents, including the `plan_impact` filter — no affected
node id, not a plan impact.

**Acceptance:** a subagent returning prose instead of the structured shape is
detectable. An observation with no node id lands in the detail file, not in
`plan_impact`.

---

## M6 — The orchestration loop

### T-016 · `/golem:run`

**Deps:** T-011, T-015

Implement §7.6. Load `tasks.jsonl` once at start (§7.4), hold the DAG in context,
dispatch by `complexity`, @-mention agents explicitly.

**Acceptance:** a 5-node plan executes start to finish. The orchestrator reads
`tasks.jsonl` exactly once — verify by inspecting the session.

### T-017 · Context hygiene

**Deps:** T-016

Enforce §7.4: no diff reads, no `runs/*.md` reads without a `detail_ref` decision, no
re-reading to confirm its own writes, `progress.log` appended per node.

**Acceptance:** across a 10-node run, orchestrator context growth is roughly linear in
node count at a small constant — not in lines of code produced.

### T-018 · Commit per node

**Deps:** T-016

Implement §7.8.

**Acceptance:** a 5-node run produces 5 commits, each naming its node id, each
independently revertable.

---

## M7 — Failure, recovery, resume

### T-019 · Retry ladder

**Deps:** T-016

Implement §9.2: fresh subagent every retry, prior-approach summary passed in, one-tier
escalation, best-effort ceiling at `standard`, `maxRetries` from config with flag
override.

**Acceptance:** a deliberately failing node retries with a *new* subagent that receives
the prior approach. A `light` node escalates to `standard`. A `standard` node does not
escalate and halts after its retries.

### T-020 · Spec-failure tiebreak

**Deps:** T-019

Implement §9.3: one blind second reviewer, never shown the first verdict. Agreement
halts; a split falls through to T-019.

**Acceptance:** the second reviewer's prompt provably does not contain the first
verdict.

### T-021 · Halt and `halt.md`

**Deps:** T-019

Implement §9.1 and §9.4. Halt on retry exhaustion, on double `spec-problem`, and on
non-null `plan_impact`. Do not continue other branches.

**Acceptance:** a mid-DAG failure stops the run with untouched downstream nodes and a
`halt.md` naming the trigger, completed nodes, and last commit.

### T-022 · `/golem:resume` and `/golem:status`

**Deps:** T-017, T-021

Reconstruct from `progress.log` + `tasks.jsonl` without re-reading run writeups.

**Acceptance:** kill a run mid-DAG, resume it, confirm it continues at the right node
and does not redo completed work.

### T-023 · Recovery loop

**Deps:** T-021, T-011

Verify §9.4 end to end: halt, edit `spec.md`, `/golem:plan`, `/golem:run`. Confirm the
planner treats the repo as authoritative and does not re-plan existing work.

**Acceptance:** after a halt and re-plan, the new DAG contains no node duplicating
already-committed work.

---

## M8 — Documentation and smoke test

### T-024 · README

**Deps:** T-022

Install, the six skills, the `.golem/` contract, the config schema, and the §11
non-goals with their reasons.

**Acceptance:** someone who hasn't read `SPEC.md` can install and run it.

### T-025 · End-to-end smoke test

**Deps:** T-023, T-024

Point v1 at a small real greenfield project — a CLI tool with tests, 10–15 nodes —
in a scratch repo. Run the whole cycle.

Then **break `verify` on purpose mid-run** and confirm it halts.

**Acceptance:** the project builds and its tests pass. The deliberate break produces a
clean halt with a readable `halt.md`. A gate never observed failing is a gate trusted
on faith.

### T-026 · Design objections

**Deps:** T-025

Write up anything in `SPEC.md` that turned out wrong, awkward, or unnecessary, with
reasoning. A builder that implements a flawed spec faithfully is worse than one that
argues.

**Acceptance:** a written list, or an explicit statement that nothing surfaced.
