# CLAUDE.md

Guidance for coding agents working in this repository.

## What this is

A single Bash script (`bin/pr-codex-pipeline`) that automates the implement-review cycle for one task: a resumable `codex exec` session plans and implements, the script pushes and opens a PR, waits on CI with `gh pr checks --watch`, and loops `pr-codex-review` plus fix rounds until `findings.json` reports zero Blockers and Critical findings. Interactive gates (plan approval, OPEN QUESTIONS, `.envrc` changes) block in the terminal and fire macOS notifications.

## Layout

```text
bin/pr-codex-pipeline   the entire tool
README.md               user-facing docs, keep in sync with --help
```

## Constraints

- The script must stay compatible with the macOS system Bash 3.2. No `mapfile`, no associative arrays, no `${var,,}`.
- Runtime dependencies are `gh`, `git`, `jq`, `codex`, `pr-codex-review` (>= 1.2.0 for `findings.json`), and optionally `direnv`. Do not add new ones casually.
- The `--help` text, the option parsing, and the README option list describe the same flags. When you touch one, update all three.
- The loop condition is Blockers + Critical only. Suggestions and Questions are handed to the fix round once but must never keep the loop alive; that would prevent convergence.
- Safety stops are deliberate: gating changed `.envrc` files, aborting when a fix round produces no commits, refusing existing remote branches, and treating a failed `pr-codex-review` run as fatal. Do not weaken them to make a run pass.
- The Codex session id is recovered by matching the worktree path against rollout files under `~/.codex/sessions/`; the worktree path is unique per run, which is what makes this safe. Keep it that way.

## Checks before committing

```bash
bash -n bin/pr-codex-pipeline
shellcheck bin/pr-codex-pipeline
/bin/bash bin/pr-codex-pipeline --help
```
