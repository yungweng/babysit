# CLAUDE.md

Guidance for coding agents working in this repository.

## What this is

A single Bash script (`bin/babysit`) that automates the iterate-until-clean part of a PR: wait for CI, let a resumable `codex exec` session fix red checks, run `pr-codex-review`, feed the findings back as fix rounds, and stop when `findings.json` reports zero Blockers and Critical findings. Issue selection, planning, and the first implementation are deliberately NOT part of this tool; the user does those. Gates (OPEN QUESTIONS from Codex, DISPUTED FINDINGS, `.envrc` changes) block in the terminal and fire macOS notifications; without an interactive terminal they abort (exit 2).

## Layout

```text
bin/babysit               the entire tool
README.md                 user-facing docs, keep in sync with --help
.github/workflows/ci.yml  CI: bash -n, shellcheck, --help, --version
```

## Constraints

- The script must stay compatible with the macOS system Bash 3.2. No `mapfile`, no associative arrays, no `${var,,}`.
- Runtime dependencies are `gh`, `git`, `jq`, `codex`, `pr-codex-review` (>= 1.2.0 for `findings.json`), and optionally `direnv`. Do not add new ones casually.
- The `--help` text, the option parsing, and the README option list describe the same flags. When you touch one, update all three.
- The loop condition is Blockers + Critical only. Suggestions and Questions are handed to the fix round once but must never keep the loop alive; that would prevent convergence.
- Safety stops are deliberate: refusing dirty/diverged local branches and fork PRs at start, gating changed `.envrc` files, aborting when a fix round produces no commits, and treating a failed `pr-codex-review` run as fatal. Do not weaken them to make a run pass.
- The dispute gate never weakens the no-commit stop: exit 5 still fires whenever a fix round ends without commits and without a live `DISPUTED FINDINGS:` marker, and only a human keystroke at the gate can accept a dispute.
- The Codex session id is recovered by matching the worktree path against rollout files under `~/.codex/sessions/`; the worktree path is unique per run, which is what makes this safe. Keep it that way.
- The pipeline works on a detached worktree of the pushed PR head and pushes via `HEAD:refs/heads/<branch>`. It must never touch the user's checkout.

## Checks before committing

```bash
bash -n bin/babysit
shellcheck bin/babysit
/bin/bash bin/babysit --help
```
