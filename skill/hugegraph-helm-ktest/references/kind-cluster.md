# Acquiring a local cluster with the kind CLI

## Contents
- Hard rule: CLI only
- When to create a cluster
- Proving cluster ownership (the P3a gate)
- Context pinning and restoring the user's context
- Sizing and capacity for 3 PD + 3 Store + 3 Server + Hubble
- Cluster config
- Create
- Getting HugeGraph images onto the nodes
- Storage
- No node-level mutation
- Teardown (guarded)
- Gotchas that look like HugeGraph bugs

## Hard rule: CLI only

Use the `kind` binary. Do not use a Kind or Kubernetes MCP server for creating,
inspecting, or deleting clusters, even when one is connected. Every cluster action must
appear in the session log as a shell command.

## When to create a cluster

Only after `kubectl cluster-info` shows no reachable cluster, or after the user has
explicitly asked for a fresh Kind cluster.

## Proving cluster ownership (the P3a gate)

**A context named `kind-…` is not evidence.** Context names are arbitrary local labels:
`kubectl config rename-context` can point `kind-staging` at an EKS cluster, and a shared
kubeconfig may name a real cluster `kind-shared-dev`. Gate on facts:

First branch on whether a cluster exists at all: if `kubectl cluster-info` reports none,
this gate is `n/a — no pre-existing cluster`. Go to § Create; a cluster this session creates
is owned by construction, and the gate exists only to judge clusters it did not create.

If a cluster does answer:

```bash
CTX=$(kubectl config current-context 2>/dev/null); echo "context: $CTX"
kubectl --context "$CTX" config view --minify -o jsonpath='{.clusters[0].cluster.server}'; echo
# custom-columns shows <none> for a node with no providerID; jsonpath over items[*] hides it
kubectl --context "$CTX" get nodes --no-headers \
  -o custom-columns=NAME:.metadata.name,PID:.spec.providerID
kind get clusters
docker ps --filter "label=io.x-k8s.kind.cluster" \
  --format '{{.Label "io.x-k8s.kind.cluster"}} {{.Names}} {{.Ports}}'
```

Treat the cluster as owned and disposable **only if all three hold**:

1. the API server URL host is `127.0.0.1` or `localhost`;
2. **every** node row shows a `kind://` providerID and none shows `<none>` — a mixed
   cluster must fail this leg;
3. the API server port matches a container labelled `io.x-k8s.kind.cluster` for a cluster
   `kind get clusters` lists. Deriving the cluster from the `kind-<name>` context
   convention would not be independent evidence — that convention is exactly what leg 3
   exists to stop trusting.

Any mismatch ⇒ **stop and ask.** Quote the context name, the API server address, and the
namespace you intend to create; wait for the user to confirm in words. Never treat a name
as approval.

Even on a proven Kind cluster this session did not create, `KTEST_CLUSTER_OWNED` stays
`no`: install and delete the release and namespace, and leave the cluster itself alone.

## Context pinning and restoring the user's context

`kind create cluster` **rewrites the user's `current-context`** in `~/.kube/config`, and
`kind delete cluster` leaves it unset. P0 captured `KTEST_ORIG_CTX` for that reason.

The gate is evaluated once; the mutations run over the following hour. In between, the
user in another terminal, a parallel agent session, or another `kind create` can change
`current-context` underneath you. So pin the context and use it explicitly on every
command — and note that an **empty** `KTEST_CTX` is worse than no pinning at all, because
`kubectl --context ""` and `helm --kube-context ""` are treated as "flag not given" and fall
straight back to `current-context`. That is why the preamble refuses to run without it.

The recipe below is for the **pre-existing-cluster** branch: pin the context the gate
actually proved. On that branch `$KTEST_CLUSTER` names a cluster that does not exist yet, so
deriving the context from it would point every later command at nothing. The
created-by-session branch pins `kind-$KTEST_CLUSTER` in § Create.

```bash
KTEST_CTX="$CTX"   # the context P3a proved, not a name derived from KTEST_CLUSTER
python3 - "$KTEST_CTX" <<'EOF'
import re,sys,os,shlex,tempfile
p=os.environ["KTEST_DIR"]+"/artifacts/env.sh"
# shlex.quote: a context name is an arbitrary kubeconfig-controlled string. Unquoted, one
# containing a space truncates the assignment (silently retargeting every later command),
# and one containing ; or $(...) would execute in every block that sources env.sh.
t=re.sub(r"^export KTEST_CTX=.*$","export KTEST_CTX="+shlex.quote(sys.argv[1]),
         open(p).read(), flags=re.M)
d=os.path.dirname(p)
fd,tmp=tempfile.mkstemp(dir=d); os.write(fd,t.encode()); os.close(fd)
os.chmod(tmp,0o600); os.replace(tmp,p)          # atomic: a killed rewrite cannot truncate
EOF

# verify the write took, exactly as the created-by-session branch does
. "$KTEST_DIR/artifacts/env.sh"
[ "$KTEST_CTX" = "$CTX" ] || { echo "PIN FAILED"; exit 1; }
```

(Rewriting through Python rather than `sed -i.bak` keeps mode 600 and leaves no `.bak`
copy behind.) From then on: `kubectl --context "$KTEST_CTX" …` and `helm --kube-context "$KTEST_CTX" …`,
without exception. A command in the session log missing its context flag is a harness
defect, and any verdict that depended on it is `INCONCLUSIVE-env` until re-run.

Keep the shell's working directory at the repository root for the whole run, so
`helm/hugegraph` always resolves. Session paths are absolute via `$KTEST_DIR`; never use a
bare `artifacts/…` path.

## Sizing and capacity for 3 PD + 3 Store + 3 Server + Hubble

Ten pods across three raft groups, landing on the three workers — kind's control-plane
carries a `NoSchedule` taint by default and runs none of them. One-node Kind clusters routinely schedule this and
then die on memory. Use one control-plane plus three workers, and check capacity before
creating:

```bash
docker info --format '{{.NCPU}} cpus / {{.MemTotal}} bytes'
docker system df
df -h ~
```

Below roughly 8 GB of memory available to Docker, expect OOM-killed Store or Server pods.
Below roughly 40 GB free disk, expect eviction partway through: the chart's images are
stored **once per Kind node**, plus PVC data for six StatefulSet pods. Either shortfall is
`INCONCLUSIVE-env` — say so rather than reporting a HugeGraph defect. Do not silently
shrink the topology to fit: report the shortfall, and only reduce the node count if the
user asks, noting it in the report.

## Cluster config

**Generate** the config so the name is substituted, never hand-edit a placeholder — a
cluster literally named `hugegraph-ktest-<KTEST_TS>` would leave `kind delete cluster
--name "$KTEST_CLUSTER"` matching nothing and orphan four containers:

```bash
# <preamble, without the KTEST_CTX guard - the context does not exist yet>
cat > "$KTEST_DIR/artifacts/kind-cluster.yaml" <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: $KTEST_CLUSTER
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
EOF
```

The cluster name carries the session timestamp so this run can never adopt or delete a
cluster left by an earlier run.

Pin the node image when the Kubernetes version matters (for example when kubeconform was
run against a specific version) by adding `image: kindest/node:vX.Y.Z@sha256:<digest>` to
each node, using the digest from that kind release's notes. Otherwise record the version
you actually got:

```bash
kubectl --context "$KTEST_CTX" version -o json | sed -n 's/.*"gitVersion":"\(v[0-9.]*\)".*/\1/p'
```

Add `extraPortMappings` on the control-plane node only if you need host access without
`kubectl port-forward`; otherwise leave them out and port-forward, which keeps the test
path identical to the in-cluster path.

## Create

```bash
# <preamble, without the KTEST_CTX guard>
# Redirect rather than pipe, and do NOT reach for the bash PIPESTATUS array to recover a
# piped exit status: it does not exist in zsh (this harness runs zsh), where under `set -u`
# referencing it is a fatal "parameter not set" - aborting this block on the SUCCESS path
# and leaving a live cluster with ownership unrecorded and current-context hijacked by kind.
# zsh spells it `pipestatus` and indexes from 1; avoid the whole problem with a redirect.
kind create cluster --config "$KTEST_DIR/artifacts/kind-cluster.yaml" --wait 300s \
  > "$KTEST_DIR/logs/kind-create.log" 2>&1
RC=$?

# Whatever happened, if the cluster now exists it is ours and must be recorded, so that
# even a partial create can be torn down. Restore the user's context on every path.
if kind get clusters | grep -qx "$KTEST_CLUSTER"; then
  OWNED=yes
else
  OWNED=no
fi
[ -n "${KTEST_ORIG_CTX:-}" ] && kubectl config use-context "$KTEST_ORIG_CTX" >/dev/null 2>&1
[ "$RC" -eq 0 ] && [ "$OWNED" = yes ] || {
  echo "kind create failed (rc=$RC, cluster present=$OWNED); see logs/kind-create.log"
  echo "  a cluster recorded as present is still deleted by P8"; }

python3 - <<'EOF'
import re,os
p=os.environ["KTEST_DIR"]+"/artifacts/env.sh"; t=open(p).read()
t=re.sub(r"^export KTEST_CLUSTER_OWNED=.*$","export KTEST_CLUSTER_OWNED='"+os.environ["OWNED"]+"'",t,flags=re.M)
t=re.sub(r"^export KTEST_CTX=.*$","export KTEST_CTX='kind-"+os.environ["KTEST_CLUSTER"]+"'",t,flags=re.M)
open(p,"w").write(t); os.chmod(p,0o600)
EOF

# verify the pins actually took - a silent no-op here breaks every later guard
. "$KTEST_DIR/artifacts/env.sh"
[ "$KTEST_CTX" = "kind-$KTEST_CLUSTER" ] && [ "$KTEST_CLUSTER_OWNED" = yes ] \
  || { echo "PIN FAILED"; exit 1; }
echo "pinned: ctx=$KTEST_CTX owned=$KTEST_CLUSTER_OWNED" >> "$KTEST_DIR/session-log.md"

kubectl --context "$KTEST_CTX" get nodes -o wide
```

Set `KTEST_CLUSTER_OWNED=yes` **in the same block that created the cluster**, and verify
it: `sed`-style in-place edits exit 0 even when the pattern matches nothing, so an
unverified pin can leave the agent believing a context is pinned when it is not. Teardown
reads that flag; if it is not set, the cluster is never deleted.

Restoring `KTEST_ORIG_CTX` inside this block, on **every** exit path, means a run that is
abandoned — or one whose teardown is deferred after a failure — never leaves the user's
default kubectl target pointing at a throwaway cluster. `kind create` rewrites
`current-context` as a side effect, so this is the only place that can undo it promptly.

## Getting HugeGraph images onto the nodes

Kind nodes do not share the host's Docker image cache. Take the exact repository and tag
from the chart — `helm show values` plus the rendered manifest — **before** running
anything here. Do not assume image names.

```bash
grep -nE 'repository|tag|image' "$KTEST_DIR/artifacts/chart-values.yaml"
grep -nE '^\s+image:' "$KTEST_DIR/artifacts/render-3x3.yaml" | sort -u
```

Then, with the values that produced:

```bash
docker pull <repo>:<tag>            # once per image the render actually references
kind load docker-image <repo>:<tag> --name "$KTEST_CLUSTER"
```

Or from an archive: `kind load image-archive images.tar --name "$KTEST_CLUSTER"` — cheaper
on disk when space is tight.

If the chart's `pullPolicy` is `Always`, a loaded image is still re-pulled and the load
was pointless; check the rendered manifest. `ErrImagePull` / `ImagePullBackOff` after
loading means the manifest's tag differs from the tag you loaded — a harness or
environment finding, not a chart defect, unless the chart's default tag genuinely does
not exist.

Before pulling, record whether the host **already had** each image; teardown must not
delete an image the user had before this run:

```bash
docker image inspect "<repo>:<tag>" >/dev/null 2>&1 \
  || echo "<repo>:<tag>" >> "$KTEST_DIR/artifacts/loaded-images.txt"
```

Only images written to that file are removed at teardown, by name and tag. When
`KTEST_CLUSTER_OWNED=yes`, host-side removal is optional anyway — the images inside the
Kind nodes disappear with the cluster. Never run `docker system prune` or `docker image prune -a`: they delete the
user's unrelated images.

## Storage

Kind ships a local-path provisioner, conventionally named `standard`. Confirm the actual
name rather than assuming it:

```bash
kubectl --context "$KTEST_CTX" get storageclass
```

If the chart's persistence values name a StorageClass that does not exist, PVCs stay
`Pending` and every StatefulSet hangs. Set the chart's storageClass value to `standard` in
the test values file — a values choice for the test environment, not a chart change. If
the chart offers no storageClass key, report the gap rather than inventing one.

## No node-level mutation

This workflow performs **no node-level mutation**: do not `cordon`, `drain`, `uncordon`,
taint, or label nodes.

It is not free of cluster-scoped *objects*, and pretending otherwise would be misleading.
It creates and deletes a Namespace, and the chart itself may ship ClusterRoles,
ClusterRoleBindings, CRDs or StorageClasses — those live outside the namespace guard, and
`helm uninstall` removes them. That is why the P2 render review lists every non-namespaced
kind, and why installing onto a cluster this session did not create requires asking first.

C4 previously reached for a cordon to force a PD pod onto a different node. That does not
work here and is not needed: Kind's default `standard` StorageClass is node-bound
local-path, so with the original node cordoned the recreated PD pod hits a volume
node-affinity conflict and sits `Pending` for the whole timeout while the cluster runs 2/3
PD — measuring the escalation, not the claim. Kind's host-local IPAM rotates pod addresses
on its own, so C4 escalates by repeating the plain pod delete instead. If the IP still has
not changed after three attempts, the verdict is `INCONCLUSIVE-fault-not-proven` with blame
`environment`.

If some other scenario ever appears to need a node-level action, stop and ask the user
rather than improvising one — a cordon left behind by an aborted run makes a node
unschedulable indefinitely.

## Teardown (guarded)

Runs only after the report is written, and only against session-labelled objects. Never
emit a delete command with a literal name.

```bash
# The standard preamble - the literal session path from P0, never "$KTEST_DIR/..." which
# would be circular here and abort this block under set -u. This is the most dangerous
# block in the skill; it must not be the one that fails to run.
set -u -o pipefail; umask 077
. "$HOME/.cache/hugegraph-ktest/<KTEST_TS>/artifacts/env.sh"
: "${KTEST_DIR:?}" "${KTEST_CTX:?context not pinned - refusing to tear down}"
: "${KTEST_NS:?}" ; : "${KTEST_REL:?}" ; : "${KTEST_TS:?}"
[ "$KTEST_DIR" = "$HOME/.cache/hugegraph-ktest/$KTEST_TS" ] \
  || { echo "env.sh does not belong to this session"; exit 1; }

# 0. kill only this session's own processes - verify the PID is still what we recorded,
#    because a PID freed an hour ago may now belong to something of the user's.
for f in portforward.pids bg.pids; do
  [ -f "$KTEST_DIR/logs/$f" ] || continue
  while IFS=$'\t' read -r pid what; do
    # Match on THIS SESSION's timestamp in the recorded label and in the live command line.
    # A bare grep for 'kubectl|helm' would kill the user's own port-forward or helm run
    # after PID reuse, and would miss our subshell loops, whose argv is just the shell.
    case "$what" in *"$KTEST_TS"*) ;; *) echo "SKIP pid $pid ($what): not this session"; continue;; esac
    LIVE=$(ps -p "$pid" -o command= 2>/dev/null)
    case "$LIVE" in
      *"$KTEST_TS"*|*"$KTEST_NS"*|*"$KTEST_DIR"*) kill "$pid" 2>/dev/null ;;
      "") echo "SKIP pid $pid ($what): already gone" ;;
      *)  echo "SKIP pid $pid ($what): PID reused by '$LIVE'" ;;
    esac
  done < "$KTEST_DIR/logs/$f"
done

# 1. OWNERSHIP GATE - both sides must be non-empty, or "" = "" would authorise a delete
LBL=$(kubectl --context "$KTEST_CTX" get ns "$KTEST_NS" \
  -o jsonpath='{.metadata.labels.hugegraph-ktest/session}' 2>/dev/null); RC=$?
if [ $RC -eq 0 ] && [ -n "$LBL" ] && [ -n "$KTEST_TS" ] && [ "$LBL" = "$KTEST_TS" ]; then
  # 2-4: each step independent, so one failure cannot skip the rest
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" delete pod \
    -l "hugegraph-ktest/session=$KTEST_TS" --ignore-not-found || true
  helm --kube-context "$KTEST_CTX" uninstall "$KTEST_REL" -n "$KTEST_NS" || true
  kubectl --context "$KTEST_CTX" delete namespace "$KTEST_NS" --ignore-not-found || true
else
  echo "SKIP: namespace $KTEST_NS carries label '$LBL', not '$KTEST_TS' - deleting nothing"
fi

# 5. cluster - only if this session created it
[ "${KTEST_CLUSTER_OWNED:-no}" = yes ] && { kind delete cluster --name "$KTEST_CLUSTER" || true; }

# 6. images - only refs this session pulled fresh. Never prune.
[ -f "$KTEST_DIR/artifacts/loaded-images.txt" ] && \
  while read -r img; do docker image rm "$img" 2>/dev/null || true; done \
    < "$KTEST_DIR/artifacts/loaded-images.txt"

# 7. context was already restored right after cluster creation; re-assert only if it moved
[ -n "${KTEST_ORIG_CTX:-}" ] && kubectl config get-contexts "$KTEST_ORIG_CTX" >/dev/null 2>&1 \
  && kubectl config use-context "$KTEST_ORIG_CTX"
```

The ownership gate wraps steps 2–4 as one block, so `helm uninstall` can never run against
a namespace this session did not label. Both sides of the label comparison are checked for
emptiness: with `$KTEST_TS` empty the jsonpath side is empty too, and `"" = ""` would
authorise the delete. Each step carries `|| true` so one failure — a release that was never
created, say — cannot abort the block and strand the rest. Step 5 is separate because
cluster ownership is tracked separately, and `set -u` plus the `:?` guards in the preamble
mean an unset variable aborts rather than expanding to `""`.

Never `pkill -f kubectl` or `pkill -f port-forward` — that kills forwards the user is
running against unrelated clusters. Kill recorded PIDs only.

**If a run failed, ask before tearing down.** The cluster is the evidence.

## Gotchas that look like HugeGraph bugs

| Symptom | Actual cause | Blame |
|---|---|---|
| All pods `Pending` | PVCs pending on a non-existent StorageClass | environment |
| Store/Server OOMKilled | Docker memory limit too low for ten JVMs | environment |
| Pods evicted after ~10 min | Docker disk pressure; check `kubectl get events` and `docker system df` | environment |
| `ImagePullBackOff` after `kind load` | tag mismatch or `pullPolicy: Always` | harness |
| Server crash-loops for the first few minutes | PD has not elected a leader yet | not a failure — re-check via PD `/v1/members` |
| PD pod keeps the same IP after delete | Kind's CNI reassigned the same address | environment; retry the plain delete up to 3× (§ C4), then `INCONCLUSIVE-fault-not-proven`. Do **not** force relocation |
| `curl` from the host fails | port-forward died | harness |
| Commands suddenly hit a different cluster | `current-context` changed mid-session, or `KTEST_CTX` was empty so `--context ""` fell back to it | harness — re-run the preamble guards, then re-run the scenario |
| `helm upgrade`/`uninstall` refuse to act | a foreground `--wait` was killed by the shell timeout and the release is stuck `pending-install` | harness — `helm rollback` or uninstall that failed release; run installs detached and poll |
