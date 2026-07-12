---
name: queue
description: "Drain a backlog of GitHub issues through issue-loop, one issue per invocation. Built to be the body of a /loop: `/loop /loopzo:queue` runs the pipeline on the next ready issue each tick and stops when the queue is empty. Trigger: /loopzo:queue"
---

# /loopzo:queue — one issue off the queue, per invocation

This skill is the **body of a loop**, not a loop itself. Each invocation processes
exactly ONE issue and returns. Repetition is the caller's job:

```
/loopzo:queue            # process the next ready issue once, or report empty
/loop /loopzo:queue      # drain: re-fire until the queue is empty (self-paced)
/loop 30m /loopzo:queue  # standing drainer on a 30-minute cadence
```

Because there is no human present between ticks, this path runs `issue-loop --auto`,
which skips both human gates and goes straight to commit + PR. If you want the gates,
run `/loopzo:issue-loop <n>` by hand instead — gates and unattended looping are mutually
exclusive by design.

## Queue signal

The queue is GitHub labels, so it is stateless and survives across ticks and machines:

- `loopzo-ready` — eligible to be worked.
- `loopzo-done` — already claimed/worked; skip.

One-time setup per repo (`--add-label` errors on labels that don't exist yet):

```
gh label create loopzo-ready --color 0E8A16 --description "queued for /loopzo:queue"
gh label create loopzo-done  --color 5319E7 --description "claimed by /loopzo:queue"
```

To swap the signal (milestone, project column, assignee), change only the `gh issue
list` query in step 1. Nothing else depends on it.

## Steps

1. **Pick the next issue.** Lowest number first:

   ```
   gh issue list --state open --label loopzo-ready --json number,labels \
     --jq 'map(select((.labels|map(.name)|index("loopzo-done"))|not)) | sort_by(.number) | .[0].number // empty'
   ```

2. **Empty queue → stop.** If step 1 prints nothing (empty output), report "queue empty — N issues
   drained this run" and STOP. Under `/loop`, do NOT schedule another wakeup; the drain
   is complete.

3. **Claim it BEFORE working.** Add the done label first, so a crash mid-run cannot make
   the next tick re-pick the same issue and open a duplicate PR:

   ```
   gh issue edit <n> --add-label loopzo-done
   ```

   ponytail: the failure mode is a stuck issue you re-`--remove-label` by hand, not a
   duplicate PR. That's the cheaper direction to fail in.

4. **Run the pipeline unattended.** Invoke `/loopzo:issue-loop <n> --auto` and let it run
   to completion (PR opened) or to a hard stop (round cap / error).

5. **Report and yield.** One line: issue number, PR link or stop reason. Return so the
   caller can fire the next tick.

## Boundaries

- One issue per invocation. Never loop internally — that is what `/loop` is for, and
  internal looping would defeat its cadence and stop controls.
- Never reopen or re-run a `loopzo-done` issue. Re-work is a human decision: remove the
  label to requeue.
