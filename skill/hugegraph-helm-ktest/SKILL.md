---
name: hugegraph-helm-ktest
description: Tests the Apache HugeGraph Helm chart (helm/hugegraph) against a real Kubernetes API — static chart checks first, then a 3 PD + 3 Store + 3 Server + Hubble install with auth enabled, then claim-driven distributed-systems scenarios that require fault-landing evidence, a named oracle, a verdict, and SUT/harness/checker/environment blame. Creates a local cluster with the kind CLI when no Kubernetes cluster is reachable. Use when asked to test, validate, verify, smoke-test, or CI-check the HugeGraph Helm chart or a HugeGraph deployment on Kubernetes or Kind, when asked "does the cluster work" or whether a HugeGraph cluster actually works, or when working on HugeGraph PD, Store, Server, or Hubble behaviour, chart auth, chart lint, or chart CI.
---

# Testing the HugeGraph Helm chart on Kubernetes

## What this skill is for

Turning "does the HugeGraph chart work?" into evidence: static chart checks, a real
install on a real kube API, and fault scenarios that each name a claim, prove the fault
landed, run a declared oracle, and end in a verdict with a blame class.

**A green install is not a passing test suite.** The smoke gate is C1's four-arm oracle,
defined once in `references/scenarios.md` § C1. It is reported as `PASS-smoke` and never as
a suite pass. See `references/verdicts-and-blame.md`.

## Hard rules

1. **Use the CLI only:** `kind`, `helm`, `kubectl`, `curl`, `docker`. Never use a
   Kubernetes or Kind MCP server for any step in this workflow, even if one is connected.
2. **Never invent chart flags.** Every `--set` path and every values key must be read out
   of the chart's own `values.yaml` / `helm show values` first. If a needed key does not
   exist, say so and stop — do not guess a plausible path.
3. **Do not modify the chart or HugeGraph source.** Test values files, manifests, and
   scripts live in the session directory, never in the chart, and are never committed.
4. **Prove cluster ownership before anything mutating** (P3a). A context *named* `kind-…`
   is not proof. No mutating command runs until P3a passes or the user confirms in words.
5. **Pin the context on every command.** Every `kubectl` carries `--context "$KTEST_CTX"`
   and every `helm` carries `--kube-context "$KTEST_CTX"`. A command in the session log
   without it is a harness defect.
6. **Delete only what this session created**, verified by the session label — never by
   name alone.
7. **No oracle improvised at run time.** Declare the oracle before injecting the fault.
8. **A fault with no landing evidence is `INCONCLUSIVE-fault-not-proven`, never a PASS.**

## Workflow

Copy this checklist into your response and tick items off:

```
- [ ] P0   Session dir, env.sh, tool + capacity probe, original context saved
- [ ] P1   Locate the chart and confirm Chart.yaml name: hugegraph
- [ ] P2   Static chart checks (no cluster needed)
- [ ] P3a  Cluster ownership PROVEN (kind evidence) or user confirmation quoted
- [ ] P3b  Cluster acquired; ownership recorded as created-by-session / pre-existing
- [ ] P4   Install default topology: 3 PD + 3 Store + 3 Server + Hubble, auth on
- [ ] P5   Smoke gate  -> PASS-smoke only
- [ ] P6   Claim-driven scenarios
- [ ] P7   Findings report with verdicts and blame
- [ ] P8   Guarded teardown (PIDs, probe pods, release, ns, cluster, context restore)
```

### P0 — Session directory, env.sh, probes

The session directory lives **outside the repository under test**, because it will hold
credential-bearing values files and rendered Secrets:

```bash
umask 077
KTEST_TS=$(date -u +%Y%m%d-%H%M%S)
KTEST_DIR=$HOME/.cache/hugegraph-ktest/$KTEST_TS
mkdir -p "$KTEST_DIR" && chmod 700 "$KTEST_DIR"
mkdir -p "$KTEST_DIR"/logs "$KTEST_DIR"/artifacts "$KTEST_DIR"/findings
echo "SESSION ENV: $KTEST_DIR/artifacts/env.sh"      # <- paste this literal path everywhere
```

Shell state does not survive between commands, so every later block re-reads `env.sh`.
**The path to `env.sh` must not itself depend on a variable** — `source
"$KTEST_DIR/artifacts/env.sh"` in a fresh shell expands to `source "/artifacts/env.sh"`,
fails, and leaves the block running with every `KTEST_*` empty. That matters because
`kubectl --context ""` and `helm --kube-context ""` are treated as "flag not given" and
silently fall back to the user's `current-context`.

So take the absolute path printed above and **paste that literal string** into every later
block. Do not introduce a shared shortcut such as a fixed `current` symlink: a second run
of this skill on the same machine would repoint it, and the first run's next block would
then load the second run's `KTEST_NS` / `KTEST_TS` / `KTEST_CLUSTER` — passing the teardown
label gate against the *other* session's namespace and deleting it.

Quote every value: an unquoted heredoc on a home directory containing a space writes a line
that truncates the path and executes the remainder.

```bash
cat > "$KTEST_DIR/artifacts/env.sh" <<EOF
export KTEST_TS='$KTEST_TS'
export KTEST_DIR='$KTEST_DIR'
export KTEST_NS='hugegraph-ktest-$KTEST_TS'
export KTEST_REL='hgtest-$KTEST_TS'
export KTEST_CLUSTER='hugegraph-ktest-$KTEST_TS'
export KTEST_CTX=''            # set only after P3a/P3b
export PD_REST_PORT='' STORE_REST_PORT='' SVR_PORT='' HUBBLE_PORT=''   # filled in P1
export KTEST_CLUSTER_OWNED='no'   # yes only if this session created the cluster
export KTEST_ORIG_CTX='$(kubectl config current-context 2>/dev/null)'
EOF
chmod 600 "$KTEST_DIR/artifacts/env.sh"
bash -n "$KTEST_DIR/artifacts/env.sh" && ( . "$KTEST_DIR/artifacts/env.sh"; echo "$KTEST_DIR" )
```

`env.sh` never holds the admin password — that lives in its own mode-600 file, see
`references/topology-and-endpoints.md` § Authentication and credential handling.

**The preamble.** Open every later command block with exactly this, and treat a block that
runs without it as a harness defect:

```bash
set -u -o pipefail; umask 077
. "$HOME/.cache/hugegraph-ktest/<KTEST_TS>/artifacts/env.sh"   # literal timestamp from P0
: "${KTEST_DIR:?}" "${KTEST_CTX:?context not pinned}" "${KTEST_NS:?}" "${KTEST_TS:?}" "${KTEST_REL:?}"
[ "$KTEST_DIR" = "$HOME/.cache/hugegraph-ktest/$KTEST_TS" ] || { echo "wrong session env"; exit 1; }
kubectl config get-contexts "$KTEST_CTX" >/dev/null || exit 1
```

(Before P3b pins the context, drop the `KTEST_CTX` guard from the preamble — nothing
mutating runs until then.) Timestamped names mean this session can never adopt, reuse, or
delete a namespace, release, or cluster left behind by an earlier run.

Probe and record: `kind version`, `helm version --short`, `kubectl version --client`,
`docker version --format '{{.Server.Version}}'`, plus capacity —
`docker info --format '{{.NCPU}} cpus / {{.MemTotal}} bytes'`, `docker system df`,
`df -h ~`. Below ~8 GB of Docker memory or ~40 GB free disk, stop with
`INCONCLUSIVE-env`; ten JVMs across four Kind nodes will not fit.

Optional static tools: `kubeconform -v`, `helm unittest --help`, `ct version`. A missing
optional tool downgrades that check to `NOT-RUN` with a one-line reason — never silently
skip it, never substitute a different check, and never install a tool without asking.

### P1 — Locate the chart

```bash
find . -type f -name Chart.yaml -path '*helm*' -not -path '*/charts/*' | head
grep -E '^(name|version|appVersion|type):' helm/hugegraph/Chart.yaml
```

Expect `helm/hugegraph/Chart.yaml` declaring `name: hugegraph`. If no chart in the repo
declares that name, **stop and report `BLOCKED — chart not found`** with the paths
searched. Do not scaffold a chart, and do not test a different chart.

Then dump the values contract you are allowed to use:

```bash
helm show values helm/hugegraph > "$KTEST_DIR/artifacts/chart-values.yaml"
grep -nE '^[a-zA-Z].*:|replicas|enabled|auth|password|image|storageClass' \
  "$KTEST_DIR/artifacts/chart-values.yaml"
```

Build a written mapping from the required topology to the chart's **actual** keys and
record it in the session log. Every key name used anywhere later in this workflow comes
from that mapping. Missing key ⇒ report it; do not invent one, and mark any scenario that
needed it `NOT-RUN`.

Then read the **chart contract** out of `render-3x3.yaml` and record every field below in
`env.sh` and in the session log. The reference files use these as placeholders; each one an
agent invents instead of reads is exactly the failure hard rule 2 exists to prevent:

| Contract field | Used by | Read from |
|---|---|---|
| `PD_REST_PORT`, `STORE_REST_PORT`, `SVR_PORT`, `HUBBLE_PORT` | every probe | `Service.spec.ports` |
| `SVR` — Server Service DNS name | CRUD, C2, C6 | `kind: Service` for the Server |
| PD / Store / Server / Hubble **pod selector labels** | every `get pods -l …` | workload `spec.selector.matchLabels` |
| PD / Store **StatefulSet names**, Server **workload name** | C3, C6, C7 | `metadata.name` |
| PD / Store / Server **Service names** | C8, port-forward | `kind: Service` |
| Store **data path** (the mounted volume path) | C7 | `volumeMounts.mountPath` |
| `PW_ENV` — env var name exposing the admin password | every authenticated call | `env[].valueFrom.secretKeyRef` |
| Hubble **properties file path** | Hubble arm | Hubble ConfigMap mount |
| **API base path** — see below | CRUD, C1, C2, C6 | probe, see below |

The Server's REST base path is version-dependent: some HugeGraph versions serve
`/graphs/{graph}/…` and others nest it under `/graphspaces/{graphspace}/graphs/{graph}/…`.
Determine which by probing (`GET /graphs`, then the Swagger UI at
`/swagger-ui/index.html`), record it as `API_BASE`, and use it everywhere. Do not assume
either shape.

Any contract field you cannot read is reported as a gap, and every scenario that needs it
is `NOT-RUN` with that reason.

Then write `$KTEST_DIR/artifacts/values-3x3-auth.yaml` **here**, using only those keys —
replica counts for pd/store/server, the Hubble enable key, the auth enable key, the admin
password key, and the storageClass key. P2's strongest checks render with this file, so it
has to exist before P2. It is credential-bearing: mode 600, inside `$KTEST_DIR`, never
committed. Credential rules are in `references/topology-and-endpoints.md`
§ Authentication and credential handling.

### P2 — Static chart checks

Run every layer in `references/chart-static-checks.md` — lint, render, kubeconform,
helm-unittest, ct — before touching any cluster. Verdicts here are `PASS-static` /
`FAIL-static` / `NOT-RUN`; static checks are evidence for no claim.

Two outputs the later phases depend on: `$KTEST_DIR/artifacts/render-default.yaml` and
`$KTEST_DIR/artifacts/render-3x3.yaml`.

Read the render, do not just check that it rendered. The render-review table in that file
is the checklist; the row that matters most is **how PD peers and Store's PD address are
wired — DNS names or pod IPs** — because it is the static half of C4.

### P3a — Prove cluster ownership (gate)

**First, branch on whether a cluster exists at all:**

```bash
kubectl cluster-info >/dev/null 2>&1 || echo "no reachable cluster"
```

*No reachable cluster* ⇒ this gate is `n/a — no pre-existing cluster`. Go straight to P3b
and create one; a cluster this session creates is owned by construction, and the evidence
gate exists only to judge clusters it did not create. Record that outcome and do not stop
to ask.

*A cluster answers* ⇒ run the evidence gate below. A context *name* proves nothing; names
are arbitrary local labels and can be renamed onto a production cluster.

Run the three-legged evidence check in `references/kind-cluster.md` § Proving cluster
ownership (the P3a gate): a loopback API server, a `kind://` providerID on **every** node,
and a Docker-label match tying the API server port to a cluster `kind get clusters` lists.

If any leg fails — including a context called `kind-anything` that is not actually a Kind
cluster — **stop and ask.** Quote the context name, the API server address, and the
namespace you intend to create, and wait for the user to confirm in words. Never infer
approval from a name. Record the outcome in the session log as
`cluster ownership: created-by-this-session | pre-existing-kind-proven | user-confirmed`.

### P3b — Acquire a cluster

If no cluster answers, create one with the **kind CLI** (never a Kind MCP tool). Config,
image loading, context pinning, ownership recording, and teardown: `references/kind-cluster.md`.

`kind create cluster` rewrites the user's `current-context`; that file restores
`KTEST_ORIG_CTX` immediately after creation, so an abandoned run never leaves the user's
default kubectl target pointing at a throwaway cluster.

Pin `KTEST_CTX` in `env.sh` — and **verify the write took** — before any mutating command.

### P4 — Install the default topology

Default under test: **3 PD, 3 Store, 3 Server, Hubble enabled, authentication on.**

Confirm the namespace and release are genuinely new before creating them — the
timestamped names make collision unlikely, not impossible:

```bash
# <preamble>
# Distinguish "NotFound" from "RBAC denied / API unreachable" - only NotFound may proceed.
OUT=$(kubectl --context "$KTEST_CTX" get ns "$KTEST_NS" 2>&1); RC=$?
if [ $RC -eq 0 ]; then echo "STOP: namespace exists"; exit 1; fi
echo "$OUT" | grep -q 'NotFound' || { echo "STOP: cannot determine namespace state: $OUT"; exit 1; }

helm --kube-context "$KTEST_CTX" status "$KTEST_REL" -n "$KTEST_NS" >/dev/null 2>&1 \
  && { echo "STOP: release exists"; exit 1; }

# Create the namespace and its ownership label ATOMICALLY. Two commands can leave the
# namespace unlabelled, and teardown then refuses to delete it - 10 pods stranded.
kubectl --context "$KTEST_CTX" apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: $KTEST_NS
  labels:
    hugegraph-ktest/session: "$KTEST_TS"
EOF
```

That label is what P8 checks before deleting anything. Install with the values file
written in P1 — do not author it again here, or P2's renders no longer describe what you
installed.

The shell tool that runs these commands has its own timeout (commonly 10 minutes), which is
shorter than a full HugeGraph bring-up. A foreground `--timeout 15m` gets killed mid-flight
and leaves the release in `pending-install`, where later `helm upgrade` and `helm uninstall`
refuse to act. Run it **detached and poll**:

```bash
# <preamble>
nohup helm --kube-context "$KTEST_CTX" install "$KTEST_REL" helm/hugegraph -n "$KTEST_NS" \
  -f "$KTEST_DIR/artifacts/values-3x3-auth.yaml" --wait --timeout 15m \
  > "$KTEST_DIR/logs/install.log" 2>&1 &
printf '%s\thelm install\n' "$!" >> "$KTEST_DIR/logs/bg.pids"
```

Then poll in separate blocks: `helm --kube-context "$KTEST_CTX" status "$KTEST_REL" -n
"$KTEST_NS"` plus `kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pods`. Do not proceed
to P5 until the release reports `deployed`; a release stuck in `pending-install` is a
finding, and recovering it needs `helm rollback` or an uninstall of that failed release.

Never pipe these into `tee` without `set -o pipefail` — `tee` returns 0, so a failed install
would read as success.

If `--wait` times out, that is a finding, not a retry loop: capture
`kubectl --context "$KTEST_CTX" get pods,sts,svc,pvc -n "$KTEST_NS" -o wide`, describe the
stuck pod, take its logs, then classify per `references/verdicts-and-blame.md`. If the P2
render review found the admin credential set as an inline env value rather than via
`secretKeyRef`, redact `describe` output before saving it — see
`references/topology-and-endpoints.md` § Authentication and credential handling.

Bring-up order matters: PD must form a quorum before Store registers, and Store must
register before Server can serve. A Server crash-looping while PD is still electing is
expected during the first minutes; distinguish it from a real failure by reading PD's
`/v1/members` rather than by waiting longer.

Ports, health endpoints, API paths, and credential-safe request recipes:
`references/topology-and-endpoints.md`.

### P5 — Smoke gate

The smoke gate **is C1's oracle** — one definition, in `references/scenarios.md` § C1,
which declares all four arms and marks each blocking or non-blocking. Run it; do not
restate it here.

Only the blocking arms (readiness/membership and the CRUD round-trip) gate the rest of the
suite; a `NOT-RUN` on a non-blocking arm does not stop C2–C9 and does not cap C1. All four
arms together are `PASS-smoke` for C1 and nothing stronger.

### P6 — Claim-driven scenarios

Each scenario declares, before it runs: the claim it can falsify, the fault, the
**landing evidence** that proves the fault hit, the oracle, and the blame rule.
The catalogue — claims C1–C9 with per-scenario blocks — is
`references/scenarios.md`. Run C1–C5 as the default suite.

**C4 is mandatory**, and its fault must produce an observed pod-IP change — an in-place
kill that keeps the IP does not exercise the claim. See `references/scenarios.md` § C4 for
the fault, the evidence rule, and the four oracle conditions.

Every scenario declares its oracle *and its negative control* in
`$KTEST_DIR/findings/<ID>.md` before the fault runs. All probes against PD and Store management ports are **`GET` only**.

### P7 — Findings report

Write `$KTEST_DIR/findings/report.md` from `references/report-template.md`. Every scenario row
carries verdict, landing evidence, oracle output, and blame class when not a PASS.
State the session verdict, and state plainly which claims were not exercised.

### P8 — Guarded teardown

Only after the report is written, and only against session-labelled objects. The full
guarded sequence — background PIDs, probe pods, release, namespace, cluster, loaded
images, context restore — is in `references/kind-cluster.md` § Teardown. Never emit a
`kind delete cluster` or `kubectl delete namespace` line with a literal name.

If a run failed, ask before tearing down: the cluster is the evidence.

## Reference files

- `references/chart-static-checks.md` — lint, template, kubeconform, helm-unittest, ct.
- `references/kind-cluster.md` — ownership proof, kind config, image load, guarded teardown.
- `references/topology-and-endpoints.md` — ports, health, PD/Store/Server/Hubble APIs, auth, credential-safe CRUD.
- `references/scenarios.md` — claims C1–C9, faults, landing evidence, oracles, blame.
- `references/verdicts-and-blame.md` — verdict states, blame classes, green-but-broken checks.
- `references/report-template.md` — session log and findings report templates.
- `references/sources.md` — research provenance and keep/adapt/drop decisions.

## Never do this

- Never report a suite pass on the strength of the smoke gate.
- Never edit `helm/hugegraph` or HugeGraph source to make a test go green.
- Never run a mutating command before P3a has passed, or without an explicit
  `--context` / `--kube-context`.
- Never `kubectl delete namespace`, `helm uninstall`, or `kind delete cluster` against
  anything without first verifying the `hugegraph-ktest/session` label or
  `KTEST_CLUSTER_OWNED=yes`. Never against a literal name.
- Never send any `POST`, `PUT`, or `DELETE` to a PD or Store management port. In
  particular never call `POST /v1/members/change` — it rewrites the PD raft peer list.
  C8 probes reachability with `GET` only.
- Never run a git write command — `add`, `commit`, `push`, `checkout -b`, `stash` — and
  never open a PR. This workflow reads the repo and writes only into `$KTEST_DIR`, which
  holds the values file and the rendered Secret.
- Never print a decoded Secret value, and never put a password in a command line.
- Never `pkill -f kubectl` or `docker system prune` — you will destroy the user's
  unrelated work. Kill only recorded PIDs; remove only images you loaded.
- Never install onto a cluster this session did not create without first listing the
  chart's **cluster-scoped** resources (ClusterRole, ClusterRoleBinding, CRD, StorageClass,
  and similar) from the render and asking. Those live outside the namespace guard, so
  `helm uninstall` would delete them even if they collide with the user's own.
- Never cordon, drain, taint, or otherwise mutate a node. This workflow makes no
  cluster-scoped change; see `references/kind-cluster.md` § No node-level mutation.
- Never use a Kubernetes or Kind MCP tool for any step here.
