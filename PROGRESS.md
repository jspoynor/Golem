# PROGRESS.md

Tracks the most recently **completed** task from `TASKS.md`. The `/next` skill reads
this file to determine which task to implement, and updates it on completion.

Last completed task: **T-004**

Task IDs run T-001 … T-026. When the last completed task is T-026, the build is done.

## Log

- T-001 — complete
- T-002 — complete
- T-003 — complete (plus two addenda: maxTurns reviewer-truncation gap, derived `criteria_count`)
- T-004 — complete (plugin.json v0.1.0 + `skills/ping` load check; validate passes incl. `--strict`, `/golem:ping` runs under `--plugin-dir`). Note: `findings/T-002.md` §1's claim that the un-namespaced form does not exist did not reproduce — bare `/ping` resolved. Re-verify at T-006.
