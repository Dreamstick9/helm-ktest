# Static chart checks (no cluster required)

## Contents
- Order and rationale
- helm lint
- helm template
- kubeconform
- helm-unittest
- chart-testing (ct)
- Reading the render for HugeGraph-specific defects
- Recording results

Every command here runs through the `helm` / `kubeconform` / `git` CLI. Do not substitute
an MCP tool. This workflow **reads** the repository and writes only into `$KTEST_DIR` —
never run `git add`, `commit`, `push`, `checkout -b`, or `stash`, and never open a PR.

## Order and rationale

Static checks run before any cluster is touched, because a template error, a bad API
version, or a Store on `emptyDir` is cheaper to find in a render than in a 15-minute
install. Layers, weakest to strongest: lint → render → schema validation → unit tests → CI
harness.

A missing optional tool is `NOT-RUN` with a reason. Never substitute a different check
silently, and **never install a tool without asking the user first**.

Render with the **session release name** (`$KTEST_REL`), not a placeholder: Helm
interpolates the release name into resource names, and the later phases mine
`render-3x3.yaml` for Service names, ports and image tags that must match the objects
actually installed.

Every values key, port, and label used here comes from the P1 mapping, read out of
`helm show values` and the render. Never guess one.

`values-3x3-auth.yaml` is written at the end of P1, immediately after the values mapping,
because every check below that exercises the real topology depends on it. It is
credential-bearing — see `references/topology-and-endpoints.md` § Authentication.

## helm lint

```bash
helm lint helm/hugegraph --strict
helm lint helm/hugegraph --strict -f "$KTEST_DIR/artifacts/values-3x3-auth.yaml"
```

Lint both the defaults and the topology under test — a values file can break templates
that the defaults never reach. `--strict` turns warnings into failures; if it fails only on
chart-metadata warnings (icon, maintainers), record that separately from a template error,
because the two carry different severity.

## helm template

```bash
# <preamble>
helm template "$KTEST_REL" helm/hugegraph > "$KTEST_DIR/artifacts/render-default.yaml"
helm template "$KTEST_REL" helm/hugegraph -f "$KTEST_DIR/artifacts/values-3x3-auth.yaml" \
  > "$KTEST_DIR/artifacts/render-3x3.yaml"
chmod 600 "$KTEST_DIR/artifacts/render-"*.yaml   # the render embeds the Secret
```

Do **not** add `--debug` to the values-bearing render: it prints the fully computed values,
password included, into the transcript. If a template error needs debugging, reproduce it
with a values file that has the password key removed.

Rendering is a check, not a formality. Confirm the render contains what the values asked
for:

```bash
grep -cE '^kind: StatefulSet' "$KTEST_DIR/artifacts/render-3x3.yaml"
grep -nE '^\s+replicas:' "$KTEST_DIR/artifacts/render-3x3.yaml"
grep -nE 'emptyDir|volumeClaimTemplates|storageClassName' "$KTEST_DIR/artifacts/render-3x3.yaml"
```

## kubeconform

```bash
kubeconform -strict -summary -ignore-missing-schemas \
  -kubernetes-version "$(kubectl version --client -o json | sed -n 's/.*"gitVersion":"v\([0-9.]*\).*/\1/p')" \
  "$KTEST_DIR/artifacts/render-3x3.yaml"
```

Validate against the version you will actually install onto — validating against a
different version than the Kind node image is a checker-class mistake.

If the chart ships CRDs and you want them validated, a remote schema catalogue can be
added. Note that this makes a "static" check perform outbound network calls against a
third-party repository at `main`, which may be unacceptable in an air-gapped or audited
environment; the default above (`-ignore-missing-schemas`) stays local.

```bash
kubeconform -strict -summary \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  "$KTEST_DIR/artifacts/render-3x3.yaml"
```

## helm-unittest

Only if the chart already ships suites:

```bash
ls helm/hugegraph/tests/*_test.yaml 2>/dev/null && helm unittest helm/hugegraph
```

If the plugin is missing, the default outcome is **`NOT-RUN`**, not an install. Installing
it mutates the user's helm installation by executing an install hook from a remote
repository, so it happens only on an explicit request:

```bash
# REQUIRES EXPLICIT USER CONSENT. Do not run this merely because the tool was missing —
# report NOT-RUN instead. Pin the version so the executed code is reproducible.
helm plugin install https://github.com/helm-unittest/helm-unittest.git --version <pinned>
```

**Do not author new `tests/*_test.yaml` into the chart.** That is a change to the chart,
which this workflow does not make. If unit coverage is missing, report it as a gap and, if
asked, write example suites into `$KTEST_DIR` instead.

## chart-testing (ct)

`ct` diffs changed charts against a target branch and therefore needs a git worktree:

```bash
git rev-parse --is-inside-work-tree >/dev/null 2>&1 && \
  ct lint --charts helm/hugegraph \
    --target-branch "$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|origin/||')"
```

Run it from the repository root, not from the session directory, or it inspects the wrong
worktree. If the chart is not in a git worktree, or no target branch resolves, `ct` is
`NOT-RUN` — not a failure of the chart. Do **not** run `ct install`; installation is done
explicitly in P4 so the topology, namespace, and values are the ones under test.

## Reading the render for HugeGraph-specific defects

These are findable statically, before any fault injection. The upstream reference shapes
are given for orientation — **check each against what this chart's own Services and values
declare**, and never report a defect merely because the chart differs from the compose
reference.

| Check | Why it matters | Where |
|---|---|---|
| PD and Store are StatefulSets with `volumeClaimTemplates` | Raft state on `emptyDir` loses data on restart (C7) | `kind:`, `volumeClaimTemplates` |
| Headless Services exist for PD and Store | Stable DNS is what makes member identity survive rescheduling (C4) | `clusterIP: None` |
| PD peer list uses DNS names, not pod IPs or `localhost` | Direct input to C4 | `HG_PD_RAFT_PEERS_LIST` or the chart's equivalent |
| Store's PD address points at the PD Service | Store cannot register otherwise | `HG_STORE_PD_ADDRESS` equivalent |
| Server's PD peers list all three PDs | Single-PD wiring silently removes the HA property | `HG_SERVER_PD_PEERS` equivalent |
| Hubble points at Service DNS for PD gRPC, PD REST, Server REST, Store REST | A Hubble on `localhost` renders fine and works never | Hubble ConfigMap |
| Auth secret is a Secret, not an inline env value | Credential hygiene | `kind: Secret`, `valueFrom.secretKeyRef` |
| Token secret is not regenerated per render | Causes C9 to break auth on upgrade | `randAlphaNum` without a `lookup` guard |
| Probes exist, and their ports and paths match the ports **this chart's own Services declare** | A probe pointing at a port nothing serves fails readiness forever. Upstream reference shape: PD 8620 `/v1/health`, Store 8520 `/v1/health`, Server 8080 `/versions` — confirm, do not assume | `readinessProbe`, `livenessProbe`, `Service.spec.ports` |
| Resource requests are set | Three StatefulSets with no requests will schedule and then thrash on a Kind node | `resources:` |
| **Cluster-scoped resources are listed** | ClusterRole, ClusterRoleBinding, CRD, StorageClass, PriorityClass and similar live **outside** the namespace, so the session's namespace guard does not protect them and `helm uninstall` deletes them. On a cluster this session did not create, stop and ask before installing. | `grep -E '^kind: (ClusterRole\|ClusterRoleBinding\|CustomResourceDefinition\|StorageClass\|PriorityClass)' "$KTEST_DIR/artifacts/render-3x3.yaml"` |
| Credential is set via `secretKeyRef`, not inline | An inline `PASSWORD` env value is printed by `kubectl describe pod` on the failure path | `valueFrom.secretKeyRef` |

A **mismatch between a probe and this chart's own declared Service port** is a finding with
blame `SUT`. A port that merely differs from the upstream compose reference is **not** a
finding — record it as the chart's contract and use it for the rest of the run.

## Recording results

One row per check in the findings report: tool, command, exit status, and verdict —
`PASS-static` for a clean layer, `FAIL-static` for a template or schema error or a failing
unit test, `NOT-RUN` with a reason for a missing tool or an inapplicable harness. Static
checks exercise no fault and belong to no scenario: they never produce `PASS-smoke`,
`PASS-hardening`, or any scenario verdict, and they are evidence for no claim.
