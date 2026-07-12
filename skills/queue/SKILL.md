---
name: queue
description: "Process one loopzo-ready GitHub issue through issue-loop, with label-based claiming, priority ordering, success/failure transitions, and safe recovery. Built as the body of /loop: /loop /loopzo:queue drains the backlog one issue per tick. Trigger: /loopzo:queue"
---

# /loopzo:queue — process one queued issue

This skill is the body of a loop, not a loop itself. Each invocation processes exactly
one issue and returns:

```text
/loopzo:queue            # process the next ready issue once, or report empty
/loop /loopzo:queue      # drain until the queue is empty
/loop 30m /loopzo:queue  # check for new work every 30 minutes
```

Because no human is present between ticks, run `issue-loop --auto`. For human gates, run
`/loopzo:issue-loop <n>` directly instead.

## Queue contract

The lifecycle is:

```text
loopzo-ready -> loopzo-running -> loopzo-pr-open
                         `-----> loopzo-blocked
```

- `loopzo-ready`: eligible to be selected.
- `loopzo-running`: claimed; never select automatically.
- `loopzo-pr-open`: the pipeline opened a PR; the issue closes when that PR merges.
- `loopzo-blocked`: the pipeline stopped or failed and needs a human.
- `loopzo-priority-high` / `loopzo-priority-low`: optional ordering signals; no priority
  label means normal.

Never add `loopzo-done`. Older versions used it to mean "claimed," so treat any issue
that still has it as unavailable until a human runs `/loopzo:enqueue <n> --requeue`.

## Steps

1. Resolve the current repository as `OWNER/REPO`, then ensure the queue labels exist.
This setup is idempotent:

   ```bash
   gh label create loopzo-ready         --color 0E8A16 --description "queued for /loopzo:queue" --force -R OWNER/REPO
   gh label create loopzo-running       --color FBCA04 --description "claimed by /loopzo:queue" --force -R OWNER/REPO
   gh label create loopzo-pr-open       --color 5319E7 --description "Loopzo opened a pull request" --force -R OWNER/REPO
   gh label create loopzo-blocked       --color B60205 --description "Loopzo needs human intervention" --force -R OWNER/REPO
   gh label create loopzo-priority-high --color D93F0B --description "run before normal-priority Loopzo issues" --force -R OWNER/REPO
   gh label create loopzo-priority-low  --color C5DEF5 --description "run after normal-priority Loopzo issues" --force -R OWNER/REPO
   ```

2. Pick one open `loopzo-ready` issue. Fetch up to 1,000 candidates with
`number,createdAt,labels`; exclude any carrying `loopzo-running`, `loopzo-pr-open`,
`loopzo-blocked`, or legacy `loopzo-done`. Sort by:
   1. `loopzo-priority-high`
   2. normal (no priority label)
   3. `loopzo-priority-low`
   4. `createdAt`, then issue number

   A suitable command shape is:

   ```bash
   gh issue list --state open --label loopzo-ready --limit 1000 \
     --json number,createdAt,labels \
     --jq '
       map(select(
         (.labels | map(.name)) as $names |
         (["loopzo-running", "loopzo-pr-open", "loopzo-blocked", "loopzo-done"] |
           map(select(. as $state | $names | index($state))) | length) == 0
       ))
       | sort_by([
           (if (.labels | map(.name) | index("loopzo-priority-high")) then 0
            elif (.labels | map(.name) | index("loopzo-priority-low")) then 2
            else 1 end),
           .createdAt,
           .number
         ])
       | .[0].number // empty'
   ```

3. If no issue is returned, report `queue empty` and stop this invocation. For a
self-paced `/loop /loopzo:queue` drain, do not request another wakeup. Under a fixed
interval such as `/loop 30m /loopzo:queue`, return normally and let the configured
schedule check again later.

4. Prepare an isolated worktree before claiming. This prevents later queue ticks from
stacking new issue commits onto an earlier issue's PR branch.

   ```bash
   ROOT=$(git rev-parse --show-toplevel)
   DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
   BRANCH="loopzo/issue-<n>"
   WORKTREE="$ROOT/.claude-runs/queue-worktrees/issue-<n>"
   RUN="$WORKTREE/.claude-runs/issue-<n>"
   git fetch origin "$DEFAULT_BRANCH"
   ```

   Ensure `.claude-runs/` is ignored through the path returned by
`git rev-parse --git-path info/exclude`. Then:
   - If `WORKTREE` is already a registered worktree for `BRANCH`, reuse it. This preserves
     state for an explicit `enqueue --requeue` after a crash or hard stop.
   - If neither the worktree nor local branch exists, create it from the latest remote
     default branch:

     ```bash
     mkdir -p "$(dirname "$WORKTREE")"
     git worktree add -b "$BRANCH" "$WORKTREE" "origin/$DEFAULT_BRANCH"
     ```

   - If the branch exists without that registered worktree, or the path/branch points
     somewhere unexpected, stop without claiming. Never delete, reset, or overwrite it.

5. Claim before work with one transition:

   ```bash
   gh issue edit <n> --remove-label loopzo-ready --add-label loopzo-running -R OWNER/REPO
   ```

   Immediately re-read the issue. Continue only if it has `loopzo-running` and no other
lifecycle label. This is duplicate prevention, not a distributed lock; support one queue
consumer per repository.

6. Run the pipeline inside the isolated worktree:
   - If `$RUN/state.json` exists, invoke `/loopzo:issue-loop resume "$RUN"`.
   - Otherwise invoke `/loopzo:issue-loop <n> "$WORKTREE" --auto`.

   Wait for it to open a PR or hard-stop. Never run issue-loop against the queue
controller's working tree.

7. Transition the result:
   - PR opened: require a non-empty `$RUN/09-pr-url.txt`, remove `loopzo-running`, add
     `loopzo-pr-open`, and report that URL. Remove the now-clean worktree with
     `git -C "$ROOT" worktree remove "$WORKTREE"`; if cleanup fails, keep the successful
     state and report the cleanup warning. Never force removal.
   - Error, round cap, or other hard stop: remove `loopzo-running`, add
     `loopzo-blocked`, post one concise issue comment with the stop reason and recovery
     command, then report the stop.

   Recovery is explicit:

   ```text
   /loopzo:enqueue <n> --requeue
   ```

8. Return after the one result so `/loop` can fire the next tick.

## Crash and concurrency behavior

- A process crash after claiming can leave `loopzo-running` and its worktree. This is
  intentionally safer than re-picking the issue and opening a duplicate PR. Inspect or
  fix the retained worktree, then recover with `enqueue --requeue`; the next tick resumes
  its run state.
- A hard stop retains the issue worktree and reports its path for human inspection.
- Do not run multiple queue consumers for the same repository. GitHub label edits are not
  an atomic compare-and-set lock.
- Never retry `loopzo-blocked`, `loopzo-running`, `loopzo-pr-open`, or legacy
  `loopzo-done` automatically.
