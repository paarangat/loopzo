# loopzo

`loopzo` is a Claude Code plugin for turning GitHub issues into tightly scoped pull
requests. It ships three skills:

- **`issue-loop`** — the pipeline. One issue in, one scoped PR out. Keeps planning,
  implementation, scope judgment, and auditing in separate model sessions, persists state
  in files, caps review rounds, and pauses at two human gates.
- **`enqueue`** — the triage gate. Validates one issue for unattended work, assigns an
  optional priority, and tags it `loopzo-ready`.
- **`queue`** — the unattended drainer. Pulls the next labeled issue and runs `issue-loop`
  on it with the gates skipped. Designed to be the body of Claude Code's `/loop` command,
  so one `/loop /loopzo:queue` works through an entire backlog.

It works with any repository, but deliberately relies on a specific multi-CLI toolchain.

## Install

Run these commands inside Claude Code:

```text
/plugin marketplace add paarangat/loopzo
/plugin install loopzo@loopzo
```

Then start a run:

```text
/loopzo:issue-loop <issue# or URL> [repo-path]
```

Resume after a human gate or opt into unattended execution:

```text
/loopzo:issue-loop resume [run-dir]
/loopzo:issue-loop <issue# or URL> --auto
```

Plugin skills are namespaced, so `/loopzo:issue-loop` is the supported invocation. The
plugin does not provide a short `/issue-loop` alias; that command resolves only when a
separate standalone copy of the skill is installed.

## Prerequisites

Install and authenticate the required CLIs before starting a run:

- `codex` — OpenAI Codex CLI
- `claude` — Claude Code CLI
- `gh` — GitHub CLI
- `jq`

The default routing assumes access to `gpt-5.6-sol` and `gpt-5.6-terra` through Codex,
and `claude-fable-5`, `claude-opus-4-8`, and `claude-haiku-4-5-20251001` through Claude
Code. Model names change over time; replace them in `skills/issue-loop/SKILL.md` with
models available to your account, using the lower-cost fallbacks documented in the table.

The optional `ponytail:ponytail-review` skill adds a dedicated simplification pass. If it
is not installed, `issue-loop` applies the same checks itself.

## How the loop works

Each box below is a separate, single-purpose model session; state passes between them
through files in the run directory, never through chat.

```mermaid
flowchart TD
    S0[S0 Setup: resolve issue, copy schemas, init state.json]
    S1[S1 Scope contract — FROZEN]
    S2[S2 Plan: steps traced to AC#]
    S3["S3 Codex plan review<br/>(round 1: gpt-5.6-sol · round 2: terra verify-only)"]
    B1{Blocking findings?}
    S4[S4 Scope judge — Claude fable-5, scope-only]
    S5[S5 Revise → final plan]
    G1{{HUMAN GATE 1: final plan + rulings}}
    S6[S6 Codex implements plan EXACTLY, logs deviations]
    S7[S7 Fresh Claude audit — Opus, AC + traceability]
    S75[S7.5 Ponytail simplification pass]
    S8b[S8b Judge untraceable hunks → revert if out]
    V{Audit verdict?}
    S8[S8 Fix round: codex resume --last]
    G2{{HUMAN GATE 2: audit verdict + diff stat}}
    S9[S9 Commit, push, open PR + out-of-scope comments]

    S0 --> S1 --> S2 --> S3 --> B1
    B1 -- no --> G1
    B1 -- yes --> S4 --> S5 --> G1
    S5 -. "round ≤ 2, verify-only" .-> S3
    G1 --> S6 --> S7 --> S75 --> S8b --> V
    V -- pass --> G2
    V -- "fix (fix_round ≤ 1)" --> S8 --> S7
    V -- "fix again / escalate" --> STOP[Stop, report, wait for human]
    G2 --> S9
```

Round caps (2 reviews, 1 fix, 1 arbiter) bound every cycle; hitting a cap stops the run
and hands back to the human rather than looping.

## Queueing and draining a backlog

`issue-loop` runs one issue and stops twice for a human. `queue` is for when you have a
pile of issues and nobody watching: each invocation dequeues ONE ready issue and runs
`/loopzo:issue-loop <n> --auto` on it — gates skipped, because a loop tick has no human
present. Repetition comes from wrapping it in `/loop`, not from the skill itself.

The queue is GitHub labels, so it needs no state files and survives across machines:

| Label | Meaning |
|---|---|
| `loopzo-ready` | Enqueued: eligible to be worked |
| `loopzo-running` | Claimed by the queue |
| `loopzo-pr-open` | Pipeline succeeded and opened a PR |
| `loopzo-blocked` | Pipeline stopped and needs a human |
| `loopzo-priority-high` | Run before normal-priority issues |
| `loopzo-priority-low` | Run after normal-priority issues |

Use `enqueue` to validate and tag one issue. The command creates missing queue labels
idempotently:

```text
/loopzo:enqueue 42
/loopzo:enqueue 43 --priority high
/loopzo:enqueue 44 --priority low
```

Run it:

```text
/loopzo:queue            # process the single next ready issue, then stop
/loop /loopzo:queue      # drain the backlog self-paced; stops when the queue is empty
/loop 30m /loopzo:queue  # standing drainer: check for new ready issues every 30 minutes
```

The lifecycle is:

```text
loopzo-ready → loopzo-running → loopzo-pr-open
                         └────→ loopzo-blocked
```

Behavior to know before looping it:

- **Priority first:** high, then normal, then low; ties use creation time and issue number.
- **Claim-before-work:** `ready` is replaced by `running` before the pipeline starts. A
  crash may leave a stale running issue, but it cannot be selected again automatically.
- **Explicit recovery:** resolve the blocker, then run
  `/loopzo:enqueue <n> --requeue`. Nothing retries blocked or stale work by itself.
- **One consumer per repository:** labels prevent ordinary re-picks but are not a
  distributed lock for several simultaneous queue runners.
- **One isolated worktree per issue:** queue runs each issue from the latest remote default
  branch under `.claude-runs/queue-worktrees/`, so successive PRs never stack. Successful
  worktrees are removed; blocked worktrees remain for inspection and resume.
- **Unattended means unattended:** every drained issue ends in a commit, push, and open
  PR with no approval step. If you want the two human gates, run
  `/loopzo:issue-loop <n>` directly instead — gates and looping are mutually exclusive.
- **Drain or watch:** an empty queue ends the self-paced `/loop /loopzo:queue`; a fixed
  interval such as `/loop 30m /loopzo:queue` keeps checking until you stop the schedule.

Older releases used `loopzo-done` to mean "claimed." Queue skips that legacy label. Review
the issue and use `enqueue --requeue` to migrate it deliberately; this avoids duplicate PRs.

## Model routing

| Stage | Default route | Effort |
|---|---|---|
| Scope and plan | Current Claude Code session | Session default |
| Plan review | Codex `gpt-5.6-sol` | Medium |
| Verify-only review | Codex `gpt-5.6-terra` | Low |
| Scope judge | Claude `claude-fable-5` | Low |
| Implementation | Codex `gpt-5.6-sol` | Medium |
| Fresh code audit | Claude `claude-opus-4-8` | Medium |
| Rare escalation | Claude `claude-opus-4-8` | High |

The loop is bounded to two plan-review rounds, one implementation-fix round, and one
arbiter call. Unless `--auto` is set, it stops for approval twice:

1. Gate 1 presents the reviewed final plan before implementation.
2. Gate 2 presents the audit verdict and diff summary before commit and PR creation.

## License

MIT © 2026 Paarangat. See [LICENSE](LICENSE).
