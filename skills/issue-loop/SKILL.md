---
name: issue-loop
description: "Loop-engineered issue pipeline for any repo: Claude plans, Codex reviews + implements, a context-poor judge enforces scope, fresh-session Claude audits. Bounded rounds, file-based state, two human gates. Trigger: /loopzo:issue-loop"
---

# /loopzo:issue-loop — issue → scoped PR, Claude + Codex + judge

You (the interactive session) are the ORCHESTRATOR. You write the scope contract and plan
yourself (you already have repo context — do not spawn a planner). Every other role is a
fresh, headless, single-purpose invocation. State lives in files, never in chat.

## Usage

```
/loopzo:issue-loop <issue# or URL> [repo-path]  # start; repo-path defaults to cwd
/loopzo:issue-loop resume [run-dir]             # continue from state.json (after a human gate)
/loopzo:issue-loop <issue#> --auto               # skip human gates (unattended; use with care)
```

## Model & effort routing (token economy is a design goal)

| Step | Tool + model | Effort | Why |
|---|---|---|---|
| Scope + plan (S1–S2) | Orchestrator (this session) | session default | Already has context; a fresh planner would re-read the repo — pure waste |
| Plan review R1 (S3) | `codex exec` `gpt-5.6-sol` | `medium` | Highest-leverage external call; don't cheap out. Drop to `terra`+`high` if cost bites |
| Plan review R2 verify-only (S3') | `codex exec` `gpt-5.6-terra` | `low` | Checklist work: "were accepted findings addressed?" |
| Judge (S4, S8b) | `claude -p $BARE` `claude-fable-5` | `low` | Tiny context (scope + items), batched in ONE call. Swap to `claude-haiku-4-5-20251001` for high volume |
| Implementation (S6) | `codex exec` `gpt-5.6-sol` | `medium` | Use `gpt-5.6-terra` when final plan has ≤3 steps and no migrations/schema changes |
| Fix round (S8) | `codex exec resume --last` | inherits | Keeps implementer context; no re-read |
| Code audit (S7) | `claude -p $BARE` `claude-opus-4-8` | `medium` | Mostly mechanical diff-vs-manifest + criterion check |
| Ponytail pass (S7.5) | orchestrator via `/ponytail-review` | session default | Diff is already in context from S7; no extra call |
| Escalation arbiter (rare) | `claude -p $BARE` `claude-opus-4-8` | `high` | Only on deadlock or second audit failure |

`gpt-5.6-luna` is deliberately unused — nothing in this loop is latency-bound.

Token rules (always): `$BARE` (see Constants) on every headless claude call — `--bare`
skips hooks/plugins, but on some installs it also skips credential loading and the call
dies with "Not logged in"; the probe in Constants degrades it to empty, and judge/audit
hygiene then rests on the prompt's "do not use tools" plus `--allowed-tools ""`;
prompts reference run-dir file paths for codex (it reads them itself — run dir is inside
the repo) instead of embedding file bodies; judge gets NO repo access and NO history;
one batched judge call per gate, never per-finding calls; codex uses `-o` (last message
only), never `--json` event streams.

## Constants & state

```
RUN=<repo>/.claude-runs/issue-<n>          # inside the repo so codex can read it
# --bare skips hooks/plugins (token economy) but on some installs it also skips
# credential loading ("Not logged in"). Probe once per run; empty = plain claude -p.
BARE=--bare
claude -p --bare --model claude-haiku-4-5-20251001 'ok' >/dev/null 2>&1 || BARE=
```

`$RUN/state.json`: `{"step": "S3", "review_round": 1, "fix_round": 0, "auto": false}`.
Update it after every step. On `resume`, read it and continue from `step`.

Authority table (cite it, don't re-litigate):
- **Judge** is final on scope. Vetoed hunks get reverted, not debated.
- **Orchestrator** is final on technical approach within scope; may contest ONE ruling on
  technical (never scope) grounds → Opus arbiter, whose ruling is final.
- **Implementer** owns nothing: every off-plan change goes in `deviations[]`;
  an undeclared deviation is an automatic audit fail regardless of merit.

Hard caps: review rounds ≤ 2, fix rounds ≤ 1, arbiter calls ≤ 1. Cap hit → stop, report
the disagreement in ≤5 lines, wait for the human. Never iterate on non-blocking
"improvements" — those are comments by definition.

## S0 — Setup

1. Resolve the issue: `gh issue view <n> --json number,title,body,url,comments` (add
   `--repo` if given a URL). Read body AND all comments — scope often shifts in-thread.
2. Create `$RUN`, then idempotently add `.claude-runs/` to the repository's exclude file.
   Resolve the path through Git because `<repo>/.git` is a file inside linked worktrees:

   ```bash
   mkdir -p "$RUN"
   EXCLUDE=$(git -C <repo> rev-parse --git-path info/exclude)
   mkdir -p "$(dirname "$EXCLUDE")"
   grep -qxF '.claude-runs/' "$EXCLUDE" 2>/dev/null || printf '%s\n' '.claude-runs/' >> "$EXCLUDE"
   ```
3. Resolve and copy the bundled schemas into the run directory once:

```bash
# Plugin installs substitute CLAUDE_PLUGIN_ROOT below. For a standalone skill, substitute
# the absolute directory containing the SKILL.md you loaded (the Read result gives its path).
PLUGIN_ROOT="${CLAUDE_PLUGIN_ROOT}"
if [ -n "$PLUGIN_ROOT" ]; then
  SKILL_DIR="$PLUGIN_ROOT/skills/issue-loop"
else
  SKILL_DIR="<absolute path of the directory containing the SKILL.md you loaded>"
fi

if [ ! -d "$SKILL_DIR/schemas" ]; then
  echo "ERROR: schemas not found under SKILL_DIR=$SKILL_DIR" >&2
  exit 1
fi

mkdir -p "$RUN/schemas"
cp "$SKILL_DIR"/schemas/*.json "$RUN/schemas/"
```

4. Save the verbatim issue to `$RUN/00-issue.json`. Init `state.json`.

## S1 — Scope contract (FROZEN after this step)

Write `$RUN/01-scope.md` yourself:

```markdown
# Scope contract — issue #<n>
## Objective (one sentence)
## Acceptance criteria   (numbered AC1..ACn — testable, from the issue only)
## Non-goals             (explicit exclusions, incl. tempting adjacent work)
## Operator impact       (who experiences this change and what they see — the END user,
                          not the first user)
```

Nothing may be added to this file later. Every downstream artifact traces to an AC number.
If the issue is too vague to write testable ACs, STOP and ask the user — that's cheaper
than looping on a bad contract.

## S2 — Plan

Write `$RUN/02-plan.md`: ordered steps, each with file/symbol and the `AC#` it serves.
A step that serves no AC doesn't go in the plan. Mark net-new code explicitly.

## S3 — Codex plan review (round `review_round`)

```bash
cd <repo> && codex exec --sandbox read-only -C <repo> \
  -m gpt-5.6-sol -c model_reasoning_effort="medium" \
  --output-schema $RUN/schemas/plan-review.json \
  -o $RUN/03-plan-review-r${ROUND}.json - <<'EOF'
You are an adversarial plan reviewer. Read .claude-runs/issue-<n>/01-scope.md and
02-plan.md (round 2: also 05-final-plan.md and 04-rulings.json), plus any repo files the
plan names. Find: logic bugs, missing edge cases, unstated assumptions, sequencing
problems, inconsistencies between plan and actual code, and over-engineering — steps that
don't need to exist, abstractions with one consumer, anything a stdlib/existing helper or
a smaller change already covers (propose the simpler approach as the finding). For each finding, honestly
classify scope_claim against the acceptance criteria: "in", "out" or —
only if an AC literally cannot be satisfied without it — "out-but-material".
Round 2 is VERIFY-ONLY: check accepted findings were addressed; new findings only if blocking.
EOF
```

Round 2 swaps in `-m gpt-5.6-terra -c model_reasoning_effort="low"`.
Zero blocking findings in R1 → skip S4/S5, go to the S5 gate with `02-plan.md` as final.

## S4 — Judge rules on ALL findings (one batched call)

The judge sees ONLY the scope contract and the findings JSON. No plan rationale, no repo,
no history — that hygiene is what makes it objective.

```bash
claude -p $BARE --model claude-fable-5 --effort low --allowed-tools "" \
  --json-schema "$(cat $RUN/schemas/rulings.json)" --output-format json <<EOF \
  | jq -r '.result' > $RUN/04-rulings.json
You are a scope judge. Do not use tools. For EACH finding, apply exactly this test:
"Does the proposed change trace to a numbered acceptance criterion, or is it strictly
required to make one true?" Yes → accept (cite the AC#). No, but genuinely valuable →
comment-only. No → reject. "Would be better/cleaner/safer" NEVER qualifies as material.

--- SCOPE CONTRACT ---
$(cat $RUN/01-scope.md)
--- FINDINGS ---
$(cat $RUN/03-plan-review-r${ROUND}.json)
EOF
```

## S5 — Revise → final plan → HUMAN GATE 1

1. Apply **accepted** findings only → `$RUN/05-final-plan.md`. Adopted changes keep their
   finding-id. `comment-only` items append to `$RUN/comments.md`.
2. If you contest a ruling (technical grounds only, once): send scope + finding + ruling +
   your objection to the Opus arbiter (`claude -p $BARE --model claude-opus-4-8 --effort high`). Final.
3. If `review_round == 1` and any finding was blocking: increment round, repeat S3' verify-only.
4. **GATE 1** (unless `--auto`): show the user final plan + rulings summary. STOP.
   Resume continues at S6.

## S6 — Codex implements

Branch first if on the default branch. Then:

```bash
cd <repo> && codex exec --sandbox workspace-write -C <repo> \
  -m <sol-or-terra per routing table> \
  --output-schema $RUN/schemas/manifest.json -o $RUN/06-manifest.json - <<'EOF'
Implement .claude-runs/issue-<n>/05-final-plan.md EXACTLY. No improvements, no drive-by
fixes, no refactors the plan doesn't name. Any change not literally in the plan MUST be
declared in deviations[] with the reason. Run the tests the plan names. Do not commit.
EOF
```

## S7 — Fresh-session audit + leftover scope check

```bash
git -C <repo> diff > $RUN/diff.patch
claude -p $BARE --model claude-opus-4-8 --effort medium \
  --allowed-tools "Read Grep Glob" \
  --json-schema "$(cat $RUN/schemas/audit.json)" --output-format json <<EOF \
  | jq -r '.result' > $RUN/07-audit.json
You are auditing an implementation you did not write. Do not fix anything.
Check: (1) each AC in the scope contract is satisfied — cite file:line evidence;
(2) every diff hunk traces to a final-plan step (which traces to an AC) — untraceable
hunks are findings; (3) manifest deviations[] vs actual diff — undeclared deviations are
automatic failures; (4) operator impact: does the change behave as the scope contract's
operator-impact section describes?

--- SCOPE --- $(cat $RUN/01-scope.md)
--- FINAL PLAN --- $(cat $RUN/05-final-plan.md)
--- MANIFEST --- $(cat $RUN/06-manifest.json)
--- DIFF --- $(cat $RUN/diff.patch)
EOF
```

**S8b:** if `untraceable_hunks` is non-empty, batch ONLY those hunks + scope contract to
the judge (same call shape as S4). Judge says out → revert those hunks
(`git checkout -p` / manual), don't debate. Judge says in → note it and proceed.

## S7.5 — Ponytail pass (over-engineering / cleanliness)

Invoke the `ponytail:ponytail-review` skill (Skill tool) on the working diff. If the
plugin isn't installed, do the check yourself against these lenses: reinvented
stdlib/existing helpers, unneeded dependencies, speculative abstractions (interface with
one implementation, config for a constant), dead flexibility, duplicated logic, naming and
file organization that doesn't match the surrounding code.

Sort findings into exactly two buckets — no third option:
- **Behavior-preserving deletions/simplifications** → append to `$RUN/07-audit.json`'s
  `fix_list` (prefix `ponytail:`). These shrink the diff, so they never add scope and need
  no judge ruling.
- **Anything that changes behavior or touches code outside the diff** → `comments.md`.
  Out of scope by definition, even when it's a good idea.

Ponytail findings ride the S8 fix round — they never get a round of their own.

## S8 — Fix round (≤1)

- `verdict: pass` and empty `fix_list` → S9.
- `fix_round == 0` and there is work (`verdict: fix` and/or `ponytail:` items):
  `codex exec resume --last` with the merged `fix_list`
  (run it before any other codex use, so `--last` is still the implementer session).
  Increment `fix_round`, re-run S7 (audit only — no second ponytail pass).
- `verdict: fix` again, or `escalate` → STOP. Report audit verdict + fix list in ≤5 lines.

## S9 — HUMAN GATE 2 → PR

1. **GATE 2** (unless `--auto`): show audit verdict + diff stat. STOP until approved.
2. Commit (repo conventions), push, and write a scope-only plain-prose PR body to
   `$RUN/09-pr-body.md`: objective, AC checklist with evidence, and "Closes #<n>".
   Create the PR and persist success for callers:

   ```bash
   PR_URL=$(gh pr create --title "<scope-only title>" --body-file "$RUN/09-pr-body.md") || exit 1
   test -n "$PR_URL" || exit 1
   printf '%s\n' "$PR_URL" > "$RUN/09-pr-url.txt"
   ```
3. Post each `comments.md` idea as a PR comment (`gh pr comment`) prefixed
   `Out of scope for this PR:`. Never fold them into the diff.
4. Final report: PR URL, rounds used, rulings summary, anything escalated.

## Failure guards (enforce, don't re-derive)

| Failure | Guard |
|---|---|
| Infinite review loop | round caps; R2 verify-only; blocking-only bar in R2 |
| Scope creep re-entry | frozen 01-scope.md; AC→plan-step→finding-id→hunk traceability; judge sees no history |
| Codex grades own code | it never does — code audit is fresh-session Claude |
| Planner audits own plan | auditor is a fresh `-p` session with artifacts only, never this session |
| Silent deviation | mandatory deviations[]; manifest≠diff = auto-fail |
| Deadlock | authority table; one arbiter call; then human |
| Context loss | run dir is the bus; state.json enables cold resume |
