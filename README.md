# hugegraph-helm-ktest

A personal Claude Agent Skill that teaches an agent how to **test the Apache HugeGraph
Helm chart against a real Kubernetes API** — static chart checks first, then a
3 PD + 3 Store + 3 Server + Hubble install with auth on, then claim-driven fault
scenarios that each require fault-landing evidence, a named oracle, a verdict, and a
SUT / harness / checker / environment blame class.

It creates a local cluster with the **kind CLI** when no cluster is reachable, and never
uses a Kubernetes or Kind MCP server.

## Repo layout

| Path | What it is |
|---|---|
| `skill/hugegraph-helm-ktest/` | The skill. `SKILL.md` + 7 one-level `references/`. This is the source of truth. |
| `dist/hugegraph-helm-ktest.zip` | The same skill, zipped. Unpacks to `hugegraph-helm-ktest/`. |
| `research/` | Ground-truth artifacts fetched from upstream during authoring. |
| `HANDOFF.md` | **Read this first.** Full context: mission, design decisions and why, upstream facts, known gaps, next steps. |
| `REVIEW-LOG.md` | Three rounds of independent review — every high-severity finding and how it was resolved. |

## Install it

```bash
cp -R skill/hugegraph-helm-ktest ~/.claude/skills/
```

Or from the zip:

```bash
unzip dist/hugegraph-helm-ktest.zip -d ~/.claude/skills/
```

The folder name must stay `hugegraph-helm-ktest` — for a personal skill, Claude Code takes
the invocation name from the directory name, and it matches the YAML `name`.

## Status

All five acceptance gates met. The skill has been reviewed three times by independent
read-only reviewers across test method, scope boundaries, and safety, with every
high-severity finding resolved. See `REVIEW-LOG.md`.

**It has never been executed end to end against a real chart** — see the "Known gaps"
section of `HANDOFF.md` for why, and what to do about it.
