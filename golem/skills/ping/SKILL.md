---
name: ping
description: Load check for the golem plugin. Prints PONG and the plugin version, then stops.
disable-model-invocation: true
---

# ping

A load check, not a feature. It exists so that `claude plugin validate ./golem` and
`claude --plugin-dir ./golem` have something to prove they loaded (SPEC.md §1.4,
TASKS.md T-004).

Do exactly this and nothing else:

1. Output the single line `PONG golem v0.1.0`.
2. Stop. Do not read files, do not offer follow-up work, do not start an interview.
