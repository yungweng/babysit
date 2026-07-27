# babysit

> [!IMPORTANT]
> **Deprecated.** This tool is now part of
> [quorum](https://github.com/yungweng/quorum), which merges
> `pr-codex-review`, `babysit` and `prbot` into a single binary.
>
> ```bash
> brew install yungweng/tap/quorum
> ```
>
> `babysit 1811` becomes `quorum babysit 1811`.
>
> Behaviour is unchanged. What carried over and what deliberately did not is
> recorded in [PARITY.md](https://github.com/yungweng/quorum/blob/main/PARITY.md).
> This repository is archived and kept for reference.

You implement, `babysit` iterates: review, fix, CI, repeat, until the PR is clean. Then you get a notification and do the manual test.

`babysit` automates the loop after the first implementation of a PR exists. It runs [`pr-codex-review`](https://github.com/yungweng/pr-codex-review) and feeds the review findings into a resumable Codex session, which checks each finding (real issue or intended?), fixes the real ones, commits, and pushes. Watching CI is the pipeline's job, not Codex's, and the next review runs in the background while the pipeline waits for the checks. After each fix round the pipeline posts a comment to the PR logging what was fixed. The loop ends when a review reports zero Blockers and Critical findings and CI is green. Deliberately out of scope: picking issues, planning, and the first implementation. That part stays with you.

## Requirements

```text
gh
git
jq
codex
pr-codex-review  (>= 1.2.0, needed for findings.json)
direnv           (optional, skip with --no-direnv)
```

The Codex fix sessions run with `--dangerously-bypass-approvals-and-sandbox` by default: they must run tests, use `gh`, and `git push`, all unattended, and a sandboxed or approval-gated `codex exec` would silently skip those commands. Be aware what that means: an unattended agent with full file and network access on your machine, for up to `--fix-timeout` per step. Pass `--sandboxed` to use your `~/.codex/config.toml` defaults instead; then your config must allow commands, network, and push, or fix rounds will fail. The review side is unaffected: `pr-codex-review` manages its own (read-only) sandbox.

## Install

```bash
brew install yungweng/tap/babysit
```

Or from a checkout:

```bash
ln -sf "$PWD/bin/babysit" ~/.local/bin/babysit
```

## Usage

From inside the repo checkout, with the PR's implementation committed and pushed:

```bash
babysit                 # PR of the current branch
babysit 1811            # explicit PR number
babysit https://github.com/owner/repo/pull/1811
```

Extra positional text becomes additional context for the Codex fix session:

```bash
babysit 1811 --effort high "Focus on the time-tracking module"
```

## What It Does

1. **CI.** `gh pr checks --watch` waits for CI. Red checks go to a Codex fix session together with the failing-check details; it fixes, commits, pushes, and babysits CI until green (up to `--max-ci-fixes` attempts, each verified by the pipeline).
2. **Review.** `pr-codex-review` reviews the PR and posts its comment, then writes `findings.json`.
3. **Decision.** Zero Blockers and Critical: done, notification, ready for manual testing. Otherwise the full review comment goes back into the Codex session, which checks each finding for whether it is a real issue or intended behavior, fixes the real ones, commits, pushes, and babysits CI until green. The pipeline verifies the push and CI state, posts a fix-log comment to the PR, and returns to step 2. After `--max-iter` rounds without convergence it stops and tells you.

Only Blockers and Critical keep the loop alive. Suggestions and Questions are handed to each fix round once (implement or consciously reject), so the loop cannot chase moving targets forever.

All fix rounds share one Codex session, so context carries from CI fixes through every review round. The session id is recovered from `~/.codex/sessions/` by matching the run's unique worktree path, so concurrent Codex sessions elsewhere do not interfere.

## Fix-Log Comments

Every fix round ends with a comment on the PR describing what was fixed and how, and which findings were left unchanged because they are intended. The Codex session writes the text (in the language of the PR description); the pipeline posts it via `gh pr comment`, so it appears under your account. If the session fails to produce the text, the pipeline posts the round's commit list instead.

## Quiet Output

Fix and review steps show a spinner status line with the step name and elapsed time, followed by a one-line completion mark (review rounds add the finding counts); the full Codex and pr-codex-review output always goes to the run directory's logs. Pass `--verbose` to stream everything to the terminal instead.

## Gates: Autonomous by Default

The run is fully autonomous by default: start it and walk away. The three situations that used to need a human are decided automatically:

- **Product decisions.** Codex is instructed to decide everything itself, picking the most conservative reasonable option and recording notable decisions in the fix-log comment. If it still ends a step with an `OPEN QUESTIONS:` block, the questions are bounced back for it to decide itself; after three bounces the run gives up (exit 2) rather than looping.
- **Disputed findings.** Review findings can be wrong. If a fix round ends with no commits but a `DISPUTED FINDINGS:` block, the dispute is never accepted as-is: Codex must first survive one forced adversarial re-check (actively trying to reproduce each finding). If it fixes something after all, the loop continues normally; if it upholds the dispute, the rebuttals are accepted, shown in the final summary, and the run finishes successfully. The review comment on the PR still lists those findings, so check the summary before merging.
- **Changed `.envrc` files.** The diff is printed, a notification fires, and `direnv allow` runs automatically. With the sandbox bypassed Codex can execute anything anyway, so this gate adds no protection in the default mode.

With `--interactive`, the pipeline instead notifies you and blocks in the terminal at each gate: type answers to open questions, and at a dispute type `a` to accept, a reply to send back into the session, or `q` to abort. Interactive gates need a terminal on stdin; without one the run aborts with exit 2 instead of hanging.

Either way, a fix round that ends with findings still open, no commits, and no live dispute stops the run (exit 5).

## Options

```text
--model MODEL / --effort LEVEL
Model and reasoning effort for the Codex fix sessions.
Effort is one of: minimal, low, medium, high, xhigh.

--reviewers N / --review-model MODEL / --review-effort LEVEL
Passed through to pr-codex-review for every review round.

--max-iter N
Maximum review->fix rounds. Default: 12.

--max-ci-fixes N
Maximum CI fix attempts per green-CI phase. Default: 3.

--fix-timeout DURATION
Kill a Codex fix step that runs longer than this. A fix step
includes waiting for CI, so keep this above your CI runtime.
Seconds or values like 30m, 2h; 0 disables. Default: 2h.

--sandboxed
Run the Codex fix sessions with your codex sandbox/approval
defaults instead of --dangerously-bypass-approvals-and-sandbox.

--interactive
Stop at gates (open questions, disputes, .envrc changes) and
ask in the terminal instead of deciding autonomously.

--verbose
Stream the full Codex and review output to the terminal
instead of the quiet status line.

--no-notify
Disable terminal notifications (the terminal bell stays).

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
   disputed by Codex and accepted (by you, or automatically
   after the forced re-check in autonomous mode)
2  aborted at a gate: user abort or missing terminal with
   --interactive, or Codex keeps asking questions although
   autonomous mode told it to decide itself
3  CI still red after --max-ci-fixes attempts
4  review not converged after --max-iter rounds
5  a fix round produced no changes although findings remain
```

## Safety Stops

- **Do not push to the PR branch manually while a run is active.** `pr-codex-review` refuses to post when the PR head moves during its review, and the pipeline treats that as a hard failure. The Codex session pushes during fix rounds; the pipeline re-pushes as a safety net and waits until GitHub reports the new head before reading checks, so a stale check result cannot pass as green.
- The pipeline refuses to start when your checkout of the PR branch has uncommitted changes, or when the local branch differs from `origin`: it reviews the pushed head and would otherwise silently ignore your local work.
- Cross-repository (fork) PRs are refused at start: the pipeline fetches and pushes `refs/heads/<branch>` on `origin`, which for a fork PR would hit the wrong branch.
- If a Codex step changes a `.envrc` that existed when the run started, or creates a new non-ignored `.envrc`, the diff is shown before `direnv allow` runs (automatically by default, after your approval with `--interactive`). Generated `.envrc` files that appear later inside Git-ignored caches such as `.devbox/` do not trigger this.
- If a fix round ends with findings still open but no new commits, the pipeline stops (exit 5) instead of looping on a stuck state; the only exception is an explicit `DISPUTED FINDINGS:` block, which goes through the dispute handling (see Gates).
- Codex fix steps are killed after `--fix-timeout` (default 2h), so a hung session cannot stall the run silently.
- Interactive gates abort with exit 2 when stdin is not a terminal instead of hanging or spinning.

## Run Directory

Each run writes logs and Codex messages under:

```text
~/.cache/babysit/<repo>-pr-<number>-<timestamp>/
  worktree/    dedicated git worktree on the PR head (removed on success)
  logs/        codex and CI logs per phase
  messages/    each fix round's final Codex message
```

Run directories older than 7 days are removed at the start of the next run.
