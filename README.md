# pr-codex-pipeline

Kick off a task, walk away, come back for manual testing.

`pr-codex-pipeline` drives the full implement-review cycle for one task: a Codex session plans and implements, the tool pushes and opens the PR, waits for CI, fixes red CI, then loops [`pr-codex-review`](https://github.com/yungweng/pr-codex-review) and fix rounds until the review reports zero Blockers and Critical findings. macOS notifications ping you at the points that need a human: plan approval, open product questions, and the final result.

## Requirements

```text
gh
git
jq
codex
pr-codex-review  (>= 1.2.0, needed for findings.json)
direnv           (optional, skip with --no-direnv)
```

The implementation Codex sessions inherit your `~/.codex/config.toml`. They must be allowed to run commands (tests, linters) and, for CI fix rounds, to call `gh`. A restrictive sandbox without network will make CI fix rounds fail.

## Install

```bash
ln -sf "$PWD/bin/pr-codex-pipeline" ~/.local/bin/pr-codex-pipeline
```

## Usage

From inside the target repo checkout:

```bash
pr-codex-pipeline 1798                                  # GitHub issue number
pr-codex-pipeline https://github.com/o/r/issues/42      # issue URL
pr-codex-pipeline "Add CSV export to the report page"   # free-text task
```

With model and effort control:

```bash
pr-codex-pipeline 1798 --model gpt-5.5 --effort high --review-effort high --reviewers 6
```

## What It Does

1. **Plan.** A Codex session explores the repo and writes a plan. You approve it in the terminal (Enter), give feedback (typed text triggers a revision), or abort (`q`). Skip this gate with `--no-plan-gate`.
2. **Implement.** The same session implements the approved plan, runs the repo's checks, and commits. It never pushes; the pipeline owns git push and the PR.
3. **PR.** The pipeline pushes the branch and opens the PR with a Codex-written title and body, including an Assumptions section and `Closes #<issue>` when the task came from an issue.
4. **CI.** `gh pr checks --watch` waits for CI. Red checks go back to the session for a fix round (up to `--max-ci-fixes` attempts).
5. **Review loop.** `pr-codex-review` reviews the PR and posts its comment. If `findings.json` reports Blockers or Critical items, the full review comment goes back to the session, which fixes, commits, pushes, and waits for CI again. Suggestions and Questions are addressed once per round but do not keep the loop alive. After `--max-iter` rounds without convergence the pipeline stops and tells you.
6. **Done.** Notification with the result: ready for manual testing, or stopped with a reason and exit code.

## Open Questions Gate

The implementation session is instructed to make routine technical decisions itself and record them in the PR's Assumptions section. For product decisions it cannot responsibly make (data semantics, user-visible policy, migration of existing data), it ends its message with an `OPEN QUESTIONS:` block. The pipeline then notifies you, prints the questions, and blocks until you type answers in the terminal; the answers go back into the same session and the run continues.

## Options

```text
--branch NAME
Branch to create. Default: the name the plan suggests, validated with
git check-ref-format; falls back to pipeline/<run-id>.

--base BRANCH
PR base branch. Default: the repository's default branch.

--model MODEL / --effort LEVEL
Model and reasoning effort for the implementation Codex session.
Effort is one of: minimal, low, medium, high, xhigh.

--reviewers N / --review-model MODEL / --review-effort LEVEL
Passed through to pr-codex-review for every review round.

--max-iter N
Maximum review->fix rounds. Default: 3.

--max-ci-fixes N
Maximum CI fix attempts per green-CI phase. Default: 3.

--draft
Open the PR as a draft.

--no-plan-gate
Skip interactive plan approval and go straight to implementation.

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
0  success, PR is green and review-clean
2  aborted at a gate (plan, questions, .envrc change)
3  CI still red after --max-ci-fixes attempts
4  review not converged after --max-iter rounds
5  a fix round produced no changes although findings remain
```

## How Sessions Are Resumed

All phases share one Codex session so context carries through plan, implementation, CI fixes, and review fixes. The session id is recovered from `~/.codex/sessions/` by matching the run's unique worktree path against the rollout file's recorded `cwd`, so concurrent Codex sessions elsewhere do not interfere. Phases continue via `codex exec resume <session-id>`.

## Safety Stops

- **Do not push to the PR branch manually while a run is active.** `pr-codex-review` refuses to post when the PR head moves during its review, and the pipeline treats that as a hard failure.
- If a Codex step changes any `.envrc`, the pipeline stops and asks before running `direnv allow` on the changed file.
- If a fix round ends with findings still open but no new commits, the pipeline stops (exit 5) instead of looping on a stuck state.
- A branch that already exists on `origin` aborts the run; pick another name with `--branch`.

## Run Directory

Each run writes logs and Codex messages under:

```text
~/.cache/pr-codex-pipeline/<repo>-<timestamp>/
  worktree/    dedicated git worktree (removed on success)
  logs/        codex and CI logs per phase
  messages/    each phase's final Codex message
```

Run directories older than 7 days are removed at the start of the next run.
