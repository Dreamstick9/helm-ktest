# Session log, declaration, and findings templates

## Contents
- Session directory layout
- Session log template
- Pre-fault declaration template (written BEFORE the fault)
- Findings report template
- Per-scenario finding template
- Rules for filling these in

## Session directory layout

`$KTEST_DIR` is `~/.cache/hugegraph-ktest/<KTEST_TS>/`, **outside the repository**, mode
700, because it holds the credential-bearing values file and rendered Secrets.

```
$KTEST_DIR/
├── session-log.md
├── logs/          install.log, kind-create.log, portforward.pids, bg.pids, pod logs
├── artifacts/     env.sh, admin.pw, curl.cfg, values-3x3-auth.yaml, render-*.yaml,
│                  kind-cluster.yaml, chart-values.yaml, loaded-images.txt,
│                  c4-before.txt, c4-members-before.jsonl,
│                  c4-after.attempt<N>.txt, c2-poll.attempt<N>.jsonl,
│                  c2-writes.attempt<N>.csv, c9-manifest-r<N>.yaml
└── findings/      report.md, <scenario>.md
```

## Session log template

```markdown
# HugeGraph Helm chart test session — <KTEST_TS>

## Environment
| Item | Value |
|---|---|
| chart path | |
| Chart.yaml name / version / appVersion | |
| kube context (KTEST_CTX) | |
| original context (restored at P8) | |
| cluster ownership | created-by-this-session / pre-existing-kind-proven / user-confirmed |
| P3a evidence: API server address | |
| P3a evidence: node providerIDs | |
| P3a evidence: kind get clusters | |
| Kubernetes version | |
| kind / helm / kubectl / docker versions | |
| Docker cpus / memory / free disk | |
| namespace (labelled hugegraph-ktest/session) | |
| release | |
| images loaded into the cluster | |

## Values mapping (requirement -> this chart's actual key)
| Requirement | Chart values key | Value used | Source |
|---|---|---|---|
| PD replicas = 3 | | | helm show values |
| Store replicas = 3 | | | |
| Server replicas = 3 | | | |
| Hubble enabled | | | |
| Auth enabled | | | |
| Admin password | | (from Secret; not recorded here) | |
| StorageClass | | | |

Keys this chart does not have: <list, or "none">
Scenarios NOT-RUN because of a missing key: <list, or "none">

## Chart contract observed (use these, not the upstream reference)
| Contract field | Value read from the render | Where it was read |
|---|---|---|
| PD / Store / Server / Hubble Service ports | | Service.spec.ports |
| Server Service DNS name (SVR) | | |
| Server REST API base path (API_BASE) | | probe + Swagger UI |
| PD / Store / Server / Hubble pod selector labels | | spec.selector.matchLabels |
| PD / Store StatefulSet names, Server workload name | | metadata.name |
| Store data path | | volumeMounts.mountPath |
| Admin-password env var name (PW_ENV) | | env[].valueFrom.secretKeyRef |
| Hubble properties file path | | Hubble ConfigMap mount |
| PD member healthy `state` string | | baseline /v1/members |
| Status returned for a missing vertex id | | baseline negative control |

Contract fields that could not be read: <list, or "none">

## Timeline
- HH:MM:SS  <event>
```

## Pre-fault declaration template (written BEFORE the fault)

Write `$KTEST_DIR/findings/<ID>.md` with this section **before** running the fault command. The fault
log must carry a later timestamp than this file.

```markdown
# <ID> — <claim in one line>

## Declared before the run    (written at <UTC timestamp>)
- Claim:
- Budget tier and its numeric budget:
- Fault (exact command):
- Landing evidence (the specific signal):
- Oracle, inputs, and pass condition:
- Negative control and the failing result it must produce:
- Ambiguity handling (timeouts, retries, unknown-outcome writes):
- Blame rule:
```

## Findings report template

```markdown
# Findings — HugeGraph Helm chart, <KTEST_TS>

**Session verdict:** DONE | DONE_WITH_CONCERNS | FAIL | INCONCLUSIVE | BLOCKED

## Static checks
| Check | Command | Result | Verdict |
|---|---|---|---|
| helm lint (defaults) | | | |
| helm lint (3x3 auth) | | | |
| helm template | | | |
| kubeconform | | | |
| helm-unittest | | | |
| ct lint | | | |
| render review | | | |

(Verdicts here are PASS-static / FAIL-static / NOT-RUN — see
`references/verdicts-and-blame.md`. A NOT-RUN needs its own reason, e.g. "no suites in
chart" or "not a git worktree" — verify before writing it.)

## Scenarios
Every scenario in the selected suite gets a row, including ones never started.

| ID | Claim | Fault landed? | Oracle result | Negative control ran? | Verdict | Blame |
|---|---|---|---|---|---|---|
| C1 | cluster forms and serves | | | | | |
| C2 | PD leader failover | | | | | |
| C3 | Store minority loss | | | | | |
| C4 | PD identity survives pod IP change | | | | | |
| C5 | auth enforced | | | | | |

## What was NOT tested
- <claim> — <reason>

## Confidence delta
Believe more: <...>
Believe less: <...>
Unchanged: <...>

## Smoke disclosure
C1's oracle is the smoke gate. It is PASS-smoke for C1 only and is evidence for no other
claim. DONE requires at least one PASS-hardening.

## Teardown
| Step | Done | Note |
|---|---|---|
| port-forwards killed (recorded PIDs only) | | |
| background poll loops killed (bg.pids) | | |
| probe pods deleted | | |
| release uninstalled | | |
| namespace deleted (session label verified) | | |
| cluster deleted (only if KTEST_CLUSTER_OWNED=yes) | | |
| loaded images removed (named, no prune) | | |
| original kube context restored | | |
```

## Per-scenario finding template

```markdown
# <ID> — <claim in one line>

**Verdict:** <one of the twelve states>   **Blame:** <SUT | harness | checker | environment | n/a>
**Re-runs:** <n of N>

## Declared before the run    (written at <UTC timestamp>)
<the eight declaration fields, unchanged from the pre-fault file>

## What happened
- Baseline: <artifact path>
- Fault command and output: <log path, with timestamp>
- Landing evidence observed: <the actual values, e.g. podIP 10.244.2.7 -> 10.244.3.4>
- Oracle output: <artifact path, plus the decisive values inline>
- Negative control result: <what failed, as it was supposed to>

## Green-but-broken checks
1..10 with yes/no and a note on every "no".

## Assessment
- What this does and does not establish:
- If FAIL: component, reproducer, and the smallest sequence that shows it.
```

## Rules for filling these in

- Write the **declaration** before the fault and the **result** as each scenario finishes —
  never both at the end.
- Quote the decisive values inline (leader address before and after, pod IP before and
  after, healthy member count). A path to a log file alone is not a result.
- Every verdict that is not a PASS carries a blame class and a one-line reason.
- Never leave a verdict cell as "ok", "green", or "passed", and never ship a template cell
  pre-filled — every cell is verified before it is written.
- Never record a password, token, or Secret value in any of these files.
- If the suite was restarted mid-run, say so and say why.
