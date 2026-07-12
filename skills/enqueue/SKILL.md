---
name: enqueue
description: "Validate a GitHub issue for unattended Loopzo processing, assign optional queue priority, and move it to loopzo-ready. Also requeue blocked or stale work explicitly. Use for /loopzo:enqueue before draining issues with /loopzo:queue."
---

# /loopzo:enqueue — validate and tag one issue

Put a specific issue into the Loopzo queue. Do not scan or enqueue every open issue.
`issue-loop` remains queue-agnostic; this skill owns the human/triage decision that an
issue is safe for unattended processing.

## Usage

```text
/loopzo:enqueue <issue# or URL> [repo-path]
/loopzo:enqueue <issue# or URL> [repo-path] --priority high|normal|low
/loopzo:enqueue <issue# or URL> [repo-path] --requeue
```

Priority defaults to `normal`. `--requeue` permits a deliberate transition from
`loopzo-running`, `loopzo-pr-open`, `loopzo-blocked`, or legacy `loopzo-done`; it does not
skip validation.

## Labels

Resolve the issue's `OWNER/REPO` and pass `-R OWNER/REPO` to every `gh` command. Ensure
these labels exist before reading or changing issue state. `--force` makes setup
idempotent.

```bash
gh label create loopzo-ready         --color 0E8A16 --description "queued for /loopzo:queue" --force -R OWNER/REPO
gh label create loopzo-running       --color FBCA04 --description "claimed by /loopzo:queue" --force -R OWNER/REPO
gh label create loopzo-pr-open       --color 5319E7 --description "Loopzo opened a pull request" --force -R OWNER/REPO
gh label create loopzo-blocked       --color B60205 --description "Loopzo needs human intervention" --force -R OWNER/REPO
gh label create loopzo-priority-high --color D93F0B --description "run before normal-priority Loopzo issues" --force -R OWNER/REPO
gh label create loopzo-priority-low  --color C5DEF5 --description "run after normal-priority Loopzo issues" --force -R OWNER/REPO
```

`normal` has no priority label. Never create or add `loopzo-done`; it is a legacy label
whose old meaning was "claimed," not "completed."

## Workflow

1. Read the issue and all comments:

   ```bash
   gh issue view <issue> -R OWNER/REPO \
     --json number,title,body,url,state,labels,comments
   ```

2. If the issue has `loopzo-running`, `loopzo-pr-open`, `loopzo-blocked`, or legacy
`loopzo-done` and the user did not pass `--requeue`, stop without changing labels or
posting a comment. Report the current state and the explicit recovery command.

3. Validate all of the following:
   - The issue is open and has a clear objective.
   - Its body/comments support testable acceptance criteria without inventing scope.
   - It is small enough for one scoped pull request.
   - It has no unresolved dependency, product decision, or required credential/input.

4. If validation fails, make `loopzo-blocked` the only lifecycle label: remove any present
`loopzo-ready`, `loopzo-running`, `loopzo-pr-open`, or legacy `loopzo-done`, then add
`loopzo-blocked`. Post one concise issue comment listing the actionable blockers. Do not
duplicate the comment when the issue was already blocked for the same reasons. Report
that nothing was queued.

5. If validation passes, make the priority labels mutually exclusive, remove any present
`loopzo-blocked` label, and add `loopzo-ready`. With `--requeue`, also remove any present
`loopzo-running`, `loopzo-pr-open`, and legacy `loopzo-done` labels. Use one `gh issue
edit` call for the transition; include only labels actually present in `--remove-label`.

6. Report the issue number, priority, and resulting `loopzo-ready` state. If it was
already ready, treat the operation as idempotent and only update priority if needed.

## Boundaries

- Never run `issue-loop`; enqueue only validates and tags.
- Never infer that every open issue belongs in the queue.
- Never use `--requeue` implicitly. Re-running work can create duplicate pull requests.
- One issue per invocation. Bulk selection needs a separate, explicit triage workflow.
