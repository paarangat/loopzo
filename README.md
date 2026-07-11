# loopzo

`loopzo` is a Claude Code plugin for turning GitHub issues into tightly scoped pull
requests. Its first skill, `issue-loop`, keeps planning, implementation, scope judgment,
and auditing in separate model sessions, persists state in files, caps review rounds, and
pauses at two human gates. It works with any repository, but deliberately relies on a
specific multi-CLI toolchain.

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
and `claude-sonnet-5`, `claude-opus-4-8`, and `claude-haiku-4-5-20251001` through Claude
Code. Model names change over time; replace them in `skills/issue-loop/SKILL.md` with
models available to your account, using the lower-cost fallbacks documented in the table.

The optional `ponytail:ponytail-review` skill adds a dedicated simplification pass. If it
is not installed, `issue-loop` applies the same checks itself.

## Model routing

| Stage | Default route | Effort |
|---|---|---|
| Scope and plan | Current Claude Code session | Session default |
| Plan review | Codex `gpt-5.6-sol` | Medium |
| Verify-only review | Codex `gpt-5.6-terra` | Low |
| Scope judge | Claude `claude-sonnet-5` | Low |
| Implementation | Codex `gpt-5.6-sol` | Medium |
| Fresh code audit | Claude `claude-sonnet-5` | Medium |
| Rare escalation | Claude `claude-opus-4-8` | High |

The loop is bounded to two plan-review rounds, one implementation-fix round, and one
arbiter call. Unless `--auto` is set, it stops for approval twice:

1. Gate 1 presents the reviewed final plan before implementation.
2. Gate 2 presents the audit verdict and diff summary before commit and PR creation.

## License

MIT © 2026 Paarangat. See [LICENSE](LICENSE).
