# The measured setup

Seven subagent definitions and three policy blocks. This is the configuration evaluated in the root [README](../README.md), kept fixed as the exact setup those results describe.

Model tiering is one layer of a larger stack. The others, tool-output filtering and session hygiene, are not represented here because they were not part of the measured change. They are in [claude-code-token-optimization](https://github.com/makinggainz/claude-code-token-optimization), which is the maintained configuration. This directory is a snapshot.

The design principle is model tiering: work whose failure mode is cheap and immediately visible runs on inexpensive models, and work whose failure mode is expensive stays on capable ones. Reasoning is never moved down a tier to save cost.

## Install

```bash
cp setup/agents/*.md ~/.claude/agents/
```

Then append the contents of [`CLAUDE.md.example`](CLAUDE.md.example) to `~/.claude/CLAUDE.md`.

Optionally, cap delegation depth in `~/.claude/settings.json`:

```json
{ "env": { "CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH": "2" } }
```

Each agent is a single file. Delete a file to remove that agent. No other state is involved.

## The agents

| Agent | Model | Purpose |
|---|---|---|
| `deep-analyst` | fable | Escalation only. Problems that already resisted a serious attempt. |
| `architect` | opus | Design decisions and difficult debugging. Read-only, returns a plan. |
| `implementer` | sonnet | Executes an already-decided specification. |
| `test-runner` | sonnet | Runs verbose commands, reports only failures. Never fixes. |
| `scout` | haiku | Read-only code and file discovery. |
| `web-researcher` | haiku | Documentation and error lookup, returns a distilled answer. |
| `second-opinion` | haiku | Relays a verdict from a non-Claude model via a local CLI. |

Two design notes that the adoption data supports:

`scout` exists because the built-in Explore agent inherits the main conversation model. Unguided searching therefore runs at the main model's rate. A Haiku-pinned alternative with an explicit description displaced it: Explore dispatches fell from 43 to 8 after installation.

The set is deliberately small. Large agent libraries create overlapping descriptions, and the routing model selects by matching a task against those descriptions. Seven differentiated agents were selected in 70% of dispatches, where the prior state was 2%. Curation appears to matter more than coverage.

## The policy blocks

`CLAUDE.md.example` contains three blocks, loaded once per session at a combined cost of roughly 200 tokens.

**Delegation policy.** Constrains when to delegate. Small work is done inline, because a delegation costs a fresh context, a re-exploration, and a report that must then be read. That overhead exceeds the saving on short tasks. Decisions are never delegated.

**Reuse ladder.** Checks whether code needs to exist, whether the codebase already provides it, and whether a dependency covers it, before new code is written. Correctness, boundary validation, security, and accessibility are explicitly excluded from reduction.

**Report calibration.** Matches write-up length to task significance. Failures and caveats are excluded from trimming.

## Verifying it on your own machine

Record a baseline before installing:

```bash
python3 tools/usage_report.py --days 30
```

After a period of normal use, compare:

```bash
python3 tools/usage_report.py --since YYYY-MM-DD
python3 tools/decompose.py YYYY-MM-DD
python3 tools/friction.py YYYY-MM-DD
```

Three values indicate whether the configuration is being used at all: the share of dispatches going to these agents, the Sonnet and Haiku share of tokens, and the Explore dispatch count. If the first two do not rise, the configuration is installed but inactive, and the correct response is to remove it.
