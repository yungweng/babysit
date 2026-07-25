# CLAUDE.md

Guidance for coding agents working in this repository.

## What this is

A single Bash script (`bin/babysit`) that automates the iterate-until-clean part of a PR: wait for CI, let a resumable `codex exec` session fix red checks, run `pr-codex-review`, feed the findings back as fix rounds (Codex checks each finding for real-vs-intended, fixes, commits, pushes, and babysits CI until green), post a fix-log comment to the PR after each round, and stop when `findings.json` reports zero Blockers and Critical findings. Issue selection, planning, and the first implementation are deliberately NOT part of this tool; the user does those. The run is fully autonomous by default: open questions bounce back to Codex to decide itself, a dispute is accepted only after one forced adversarial re-check, and changed `.envrc` files are allowed automatically. With `--interactive`, these gates block in the terminal and fire macOS notifications instead; without a terminal on stdin they then abort (exit 2).

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
- The dispute handling never weakens the no-commit stop: exit 5 still fires whenever a fix round ends without commits and without a live `DISPUTED FINDINGS:` marker. A dispute is never accepted on first sight: in autonomous mode Codex must uphold it through one forced adversarial re-check, in `--interactive` mode a human keystroke accepts it. Do not remove the re-check to make runs finish faster.
- The Codex session id is recovered by matching the worktree path against rollout files under `~/.codex/sessions/`; the worktree path is unique per run, which is what makes this safe. Keep it that way.
- The pipeline works on a detached worktree of the pushed PR head. Codex pushes during fix steps via `git push origin HEAD:refs/heads/<branch>` (the exact command is part of its standing rules because a bare `git push` fails on a detached HEAD); the pipeline re-pushes as a safety net and waits until GitHub reports the pushed sha as the PR head before reading checks. It must never touch the user's checkout.
- Codex fix sessions run with `--dangerously-bypass-approvals-and-sandbox` by default (`--sandboxed` opts out): they push and watch CI unattended, which a sandboxed `codex exec` would silently skip. This applies only to the fix sessions; `pr-codex-review` manages its own read-only sandbox.
- The fix-log comment after each review round is posted by the pipeline via `gh pr comment` from the `PR COMMENT:` section of the round's Codex message (fallback: the round's commit list). Codex itself must never create or edit PR comments, and no comment may mention AI or automation.

## Checks before committing

```bash
bash -n bin/babysit
shellcheck bin/babysit
/bin/bash bin/babysit --help
```
