# pr-codex-pipeline

You implement, the pipeline iterates: review, fix, CI, repeat, until the PR is clean. Then you get a notification and do the manual test.

`pr-codex-pipeline` automates the loop after the first implementation of a PR exists. It waits for CI and lets a Codex session fix red checks, runs [`pr-codex-review`](https://github.com/yungweng/pr-codex-review), and feeds the review findings back into the same Codex session for a fix round. The loop ends when a review reports zero Blockers and Critical findings and CI is green. Deliberately out of scope: picking issues, planning, and the first implementation. That part stays with you.

## Requirements

```text
gh
git
jq
codex
pr-codex-review  (>= 1.2.0, needed for findings.json)
direnv           (optional, skip with --no-direnv)
```

The Codex fix sessions inherit your `~/.codex/config.toml`. They must be allowed to run commands (tests, linters) and, for CI fix rounds, to call `gh`. A restrictive sandbox without network will make CI fix rounds fail.

## Install

```bash
ln -sf "$PWD/bin/pr-codex-pipeline" ~/.local/bin/pr-codex-pipeline
```

## Usage

From inside the repo checkout, with the PR's implementation committed and pushed:

```bash
pr-codex-pipeline                 # PR of the current branch
pr-codex-pipeline 1811            # explicit PR number
pr-codex-pipeline https://github.com/owner/repo/pull/1811
```

Extra positional text becomes additional context for the Codex fix session:

```bash
pr-codex-pipeline 1811 --effort high "Focus on the time-tracking module"
```

## What It Does

1. **CI.** `gh pr checks --watch` waits for CI. Red checks go to a Codex fix session together with the failing-check details; it fixes, commits, the pipeline pushes and waits again (up to `--max-ci-fixes` attempts).
2. **Review.** `pr-codex-review` reviews the PR and posts its comment, then writes `findings.json`.
3. **Decision.** Zero Blockers and Critical: done, notification, ready for manual testing. Otherwise the full review comment goes back into the Codex session, which fixes, commits; the pipeline pushes and returns to step 1. After `--max-iter` rounds without convergence it stops and tells you.

Only Blockers and Critical keep the loop alive. Suggestions and Questions are handed to each fix round once (implement or consciously reject), so the loop cannot chase moving targets forever.

All fix rounds share one Codex session, so context carries from CI fixes through every review round. The session id is recovered from `~/.codex/sessions/` by matching the run's unique worktree path, so concurrent Codex sessions elsewhere do not interfere.

## Open Questions Gate

The fix session makes routine technical decisions itself. For product decisions it cannot responsibly make (data semantics, user-visible policy, migration of existing data), it ends its message with an `OPEN QUESTIONS:` block. The pipeline notifies you, prints the questions, and blocks until you type answers in the terminal; the answers go back into the session and the run continues. Since everything streams to your terminal, you can also just watch or intervene at the next gate.

Gates need an interactive terminal. If a gate is hit while stdin is not a terminal (redirected input, background run), the pipeline aborts with exit 2 instead of hanging.

## Dispute Gate

Review findings can be wrong. If a fix round ends with no commits but the session ends its message with a `DISPUTED FINDINGS:` block, Codex is claiming the remaining Blocker/Critical findings are false positives and no code change is warranted. The pipeline notifies you, prints the rebuttals, and asks: type `a` to accept them and finish the run successfully (you signed off on the remaining findings; note the review comment on the PR still lists them), type a reply to send it back into the session (Codex then fixes, or disputes again), or `q` to abort. Without a dispute, a fix round with no commits still stops the run (exit 5).

## Options

```text
--model MODEL / --effort LEVEL
Model and reasoning effort for the Codex fix sessions.
Effort is one of: minimal, low, medium, high, xhigh.

--reviewers N / --review-model MODEL / --review-effort LEVEL
Passed through to pr-codex-review for every review round.

--max-iter N
Maximum review->fix rounds. Default: 3.

--max-ci-fixes N
Maximum CI fix attempts per green-CI phase. Default: 3.

--fix-timeout DURATION
Kill a Codex fix step that runs longer than this.
Seconds or values like 30m, 1h; 0 disables. Default: 1h.

--no-notify
Disable macOS notifications (the terminal bell stays).

--no-direnv
Skip direnv allow/exec.

--keep-worktree
Keep the temporary worktree after a successful run.
Failed runs always keep it for inspection.

--version / -h, --help
```

## Exit Codes

```text
0  success: CI green and review clean, or remaining findings
   disputed by Codex and accepted by you
2  aborted at a gate (open questions, .envrc change, dispute,
   or a gate was hit without an interactive terminal)
3  CI still red after --max-ci-fixes attempts
4  review not converged after --max-iter rounds
5  a fix round produced no changes although findings remain
```

## Safety Stops

- **Do not push to the PR branch manually while a run is active.** `pr-codex-review` refuses to post when the PR head moves during its review, and the pipeline treats that as a hard failure. After each of its own pushes the pipeline waits until GitHub reports the new head before reading checks, so a stale check result cannot pass as green.
- The pipeline refuses to start when your checkout of the PR branch has uncommitted changes, or when the local branch differs from `origin`: it reviews the pushed head and would otherwise silently ignore your local work.
- Cross-repository (fork) PRs are refused at start: the pipeline fetches and pushes `refs/heads/<branch>` on `origin`, which for a fork PR would hit the wrong branch.
- If a Codex step changes any `.envrc`, the pipeline stops and asks before running `direnv allow` on the changed file.
- If a fix round ends with findings still open but no new commits, the pipeline stops (exit 5) instead of looping on a stuck state; the only exception is an explicit `DISPUTED FINDINGS:` block, which hands the decision to you (see Dispute Gate).
- Codex fix steps are killed after `--fix-timeout` (default 1h), so a hung session cannot stall the run silently.
- Gates abort with exit 2 when stdin is not a terminal instead of hanging or spinning.

## Run Directory

Each run writes logs and Codex messages under:

```text
~/.cache/pr-codex-pipeline/<repo>-pr-<number>-<timestamp>/
  worktree/    dedicated git worktree on the PR head (removed on success)
  logs/        codex and CI logs per phase
  messages/    each fix round's final Codex message
```

Run directories older than 7 days are removed at the start of the next run.
