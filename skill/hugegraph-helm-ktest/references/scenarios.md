# Claim-driven scenario catalogue

## Contents
- How to use this catalogue
- The command preamble
- Scenario block format
- The membership oracle (used by C1, C2, C4, C9)
- Resolving a pod safely
- C1 — Cluster forms and serves
- C2 — PD leader failover
- C3 — Store minority loss
- C4 — PD identity survives a pod IP change
- C5 — Auth is enforced
- C6 — Durability across a rolling Server restart
- C7 — Store data survives pod restart
- C8 — PD management surface exposure
- C9 — Re-apply / upgrade idempotency
- Faults that are not allowed to stand in for these

## How to use this catalogue

Default suite: **C1–C5**. Add C6–C9 for a release check, a chart-CI gate, or an explicit
"test it properly". **Every scenario in the selected suite gets a verdict row**, even the
ones never started — those are `NOT-RUN` with a reason. Silence is not a pass.

Rules that apply to every scenario here:

- **Declare before you run.** Write `$KTEST_DIR/findings/<ID>.md` with all eight fields
  *before* the fault command executes. The fault log must carry a later timestamp than
  that file. An oracle written after the result is not an oracle.
- **Values keys, ports, and selectors come from the P1 mapping.** If a scenario needs a
  chart key that does not exist, the verdict is `NOT-RUN` with that reason. Never invent a
  `--set` path, a port, or a label.
- **CLI only.** Every command runs through `kubectl` / `helm` / `curl`. Do not substitute a
  Kubernetes or Kind MCP tool; every action must appear in the session log as a shell
  command with its context pinned.
- **`GET` only against PD and Store management ports.** Never send a write to one.
- **Re-run policy.** On any oracle failure, re-run the scenario once — twice for the
  timing-sensitive C2 and C3 — and report n/N. A failure that reproduces is
  `FAIL-reproducible`; one that does not is `FAIL-nondeterministic`. Both are FAILs.
- **Artifacts are attempt-scoped.** Set `ATTEMPT=1` in the first run of a scenario and
  increment it on every re-run, in `env.sh` so it survives between blocks. C4's IP
  escalation uses a **separate** counter `TRY`, so retrying the delete never overwrites the
  attempt's evidence. Guard every artifact redirect so a re-run cannot silently clobber the
  evidence the n/N report cites:

  ```bash
  ART="$KTEST_DIR/artifacts/<name>.attempt$ATTEMPT.<ext>"
  [ -e "$ART" ] && { echo "REFUSING to overwrite $ART"; exit 1; }
  ```

- **Workload runs inside the cluster.** `$SVR` is a Service DNS name that resolves only in
  the cluster, and `$CFG` on the host is a host path — mixing them makes every request fail
  with curl exit 6 while the CSV still fills with rows, which several oracles would then
  read as vacuously satisfied. Create one long-lived load pod at P5 and run every workload
  loop through it:

  ```bash
  # envFrom exposes the chart's admin-password Secret; <PW_ENV>/<secret-name> come from P1
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" run "ktest-load-$KTEST_TS" \
    --restart=Never --image=curlimages/curl:<pinned> \
    --labels="hugegraph-ktest/session=$KTEST_TS" \
    --overrides='{"spec":{"containers":[{"name":"c","image":"curlimages/curl:<pinned>",
      "command":["sleep","7200"],"envFrom":[{"secretRef":{"name":"<secret-name>"}}]}]}}'
  LOADPOD="ktest-load-$KTEST_TS"
  ```

  **Every oracle that scores a workload must first require ≥ 10 rows with a 2xx status in
  the pre-fault window.** Fewer means the loop never reached the Server: the verdict is
  `INCONCLUSIVE-fault-not-proven` with blame `harness`, and no SUT blame rule may be
  applied.

Baseline before each scenario, saved as attempt-scoped artifacts. Capture pod **uids and
IPs** explicitly — three scenarios cite a uid change as their landing evidence, and
`-o wide` does not print uids:

```bash
kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pods \
  -l "app.kubernetes.io/instance=$KTEST_REL" \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.uid}{"\t"}{.status.podIP}{"\n"}{end}' \
  > "$KTEST_DIR/artifacts/pods-uids.<ID>.attempt$ATTEMPT.tsv"
```

Also capture: pods with `-o wide`,
`/v1/members` **from all three PD pods**, `/v1/stores`, `/v1/shardGroups` (record the
replica count), the **healthy `state` string** PD reports for a member on a known-good
cluster, and one successful authenticated read.

## The command preamble

Shell state does not survive between commands, and an unset variable expanding to `""` is
dangerous: `kubectl --context ""` and `helm --kube-context ""` are treated by both tools as
"flag not given" and silently fall back to the user's `current-context`. Every command
block in this file therefore opens with:

```bash
set -u -o pipefail; umask 077
. "$HOME/.cache/hugegraph-ktest/<KTEST_TS>/artifacts/env.sh"   # literal path from P0, never a variable
: "${KTEST_DIR:?}" "${KTEST_CTX:?context not pinned}" "${KTEST_NS:?}" "${KTEST_TS:?}" "${KTEST_REL:?}"
[ "$KTEST_DIR" = "$HOME/.cache/hugegraph-ktest/$KTEST_TS" ] || { echo "wrong session env"; exit 1; }
kubectl config get-contexts "$KTEST_CTX" >/dev/null || exit 1   # the pinned context must still exist
```

`<KTEST_TS>` is the literal timestamp P0 printed — paste it, do not introduce a shared
shortcut such as a fixed `current` symlink, which a second concurrent run would repoint at
its own session.

The blocks below are shown without the preamble to stay readable. **Prepend it to every
one.** A block that runs without it is a harness defect, and any verdict that depended on
it is `INCONCLUSIVE-env` until re-run.

## Scenario block format

Eight fields, all mandatory. They are the same eight the pre-fault declaration template in
`references/report-template.md` asks for:

1. **Claim** — the guarantee this can falsify.
2. **Budget tier and its numeric budget** — smoke or hardening, fixed before the run.
3. **Fault** — exactly what is injected, with the command.
4. **Landing evidence** — the observable that proves the fault hit. Missing or ambiguous ⇒
   `INCONCLUSIVE-fault-not-proven`. A command exiting 0 is never landing evidence.
5. **Oracle** — the check, its inputs, and the pass condition.
6. **Negative control** — a demonstration that the *checker* detects the failing case. A
   good control exercises the matcher against data known to contain the failure; it must
   not require the SUT to be broken. An oracle with no working control is
   `INCONCLUSIVE-oracle-too-weak`.
7. **Ambiguity** — how timeouts, retries, and unknown-outcome writes are treated.
8. **Blame rule** — how to split SUT / harness / checker / environment.

Scenarios without a fault (C5, C8) declare fields 2–4 as `n/a` with a reason, and are
capped at `PASS-smoke` — `PASS-hardening` requires a landed fault.

## The membership oracle (used by C1, C2, C4, C9)

Always query **all three** PD pods, one valid JSON object per line, each tagged with the
vantage that produced it:

```bash
for p in $(kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pods \
             -l "app.kubernetes.io/instance=$KTEST_REL,<pd-selector>" -o name); do
  B=$(kubectl --context "$KTEST_CTX" -n "$KTEST_NS" exec "$p" -- \
        curl -s --max-time 5 "http://localhost:${PD_REST_PORT}/v1/members"); RC=$?
  printf '{"vantage":"%s","t":"%s","curl_rc":%d,"members":%s}\n' \
    "$p" "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$RC" "${B:-null}"
done > "$KTEST_DIR/artifacts/<tag>.attempt${ATTEMPT}.jsonl"

# validate before any oracle reads it - a malformed fault-window sample is a checker defect
python3 -c 'import json,sys
bad=[i for i,l in enumerate(open(sys.argv[1]),1) if l.strip() and not _try(l)]' 2>/dev/null || \
python3 - "$KTEST_DIR/artifacts/<tag>.attempt${ATTEMPT}.jsonl" <<'EOF'
import json,sys
bad=[]
for i,l in enumerate(open(sys.argv[1]),1):
    if not l.strip(): continue
    try: json.loads(l)
    except Exception: bad.append(i)
print("unparseable lines:", bad or "none")
EOF
```

Two things this guards. `curl` emits no trailing newline, so without the wrapper the three
documents run together and the file is neither JSON nor JSONL. And `curl` prints **nothing**
when it is refused, times out, or execs into a terminating pod — exactly the samples the
fault window produces — so the body is captured first and substituted as literal `null`
rather than pasted in raw, which would emit `"members":}`. An empty `members` must never be
read as "the old IP is absent": validate the file, and treat unparseable lines inside the
fault window as `checker`.

The JSON envelope key that wraps `pdList` / `pdLeader` is not assumed by this skill — read
one response during the baseline, record the path in the chart-contract table, and use it.

Healthy means **all** of:

- `numOfNormalService == 3`. `numOfService` is the size of the member list and does not
  shrink when peers go unhealthy, so it is a size cross-check only — confirm how both
  fields behave on the known-good baseline before relying on either.
- every `pdList[].state` equals **the healthy value recorded in the P4 baseline**. Do not
  guess the enum string; a guessed value that never matches silently fails every run.
- all three vantages agree on the same member set and the same `pdLeader.grpcUrl`. One
  vantage cannot see split-brain, and `pdLeader` is scalar so "exactly one leader" is
  trivially true in any single response.

`PD_REST_PORT`, `STORE_REST_PORT`, `SVR_PORT` and `HUBBLE_PORT` come from `env.sh`, filled
in P1 from the ports this chart's own Services declare. The upstream reference shape is
8620 / 8520 / 8080 / 8088 — a starting point to confirm, never a value to hardcode.

## Resolving a pod safely

Never type a pod name into a delete. Resolve it from the release's own labels, in the
session namespace, and assert the namespace carries this session's label first:

```bash
[ "$(kubectl --context "$KTEST_CTX" get ns "$KTEST_NS" \
     -o jsonpath='{.metadata.labels.hugegraph-ktest/session}')" = "$KTEST_TS" ] \
  || { echo "REFUSING: namespace is not this session's"; exit 1; }
POD=$(kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pods \
       -l "app.kubernetes.io/instance=$KTEST_REL,<component-selector>" \
       -o jsonpath='{.items[0].metadata.name}')      # bare name, no pod/ prefix
```

Every `<…>` placeholder in this file — `<pd-selector>`, `<component-selector>`,
`<store-sts>`, `<pd-service>`, `<server-svc>`, `<store-pod>`, `<hubble-pod>`,
`<server workload>`, `<data-path>`, `<PW_ENV>`, `<hubble-properties-path>` — is a **chart
contract field recorded in P1**, not something to invent. If P1 could not read it, the
scenario is `NOT-RUN` with that reason.

`$POD` is always a **bare name**: `kubectl wait --for=condition=Ready "pod/$POD"` adds the
prefix itself, and passing `pod/name` here produces `pod/pod/name`. The selector labels
come from the P1 mapping, read out of the render — not guessed.

---

## C1 — Cluster forms and serves

- **Claim:** With 3 PD, 3 Store, 3 Server and auth on, the chart brings up a cluster that
  registers all Stores and serves an authenticated write and read.
- **Budget tier:** smoke. **Fault:** none. **Landing evidence:** n/a.
- **Oracle — four arms. This is the single definition of the smoke gate; P5 runs exactly
  this and does not restate it.**
  1. **Readiness and membership — blocking.** Every PD/Store/Server/Hubble pod Ready, the
     membership oracle healthy, and `/v1/stores` listing 3 registered Stores. Record the
     healthy `state` string here — the rest of the suite compares against it.
  2. **`helm test` — non-blocking.**
     `helm --kube-context "$KTEST_CTX" test "$KTEST_REL" -n "$KTEST_NS" --logs --timeout 8m`.
     A chart with no `helm.sh/hook: test` resource yields `NOT-RUN`; that neither blocks
     C2–C9 nor caps C1.
  3. **CRUD round-trip — blocking.** The round-trip in
     `references/topology-and-endpoints.md` returns the **written value**, asserted by
     comparison, not merely a 2xx.
  4. **Hubble — non-blocking.** `NOT-RUN` is acceptable. See that file's § Hubble.
- **Negative control:** read back an id that was never written and assert the checker
  rejects it. Record the status the Server actually returns for a missing id during the
  baseline and use that observed value as the control — do not assume 404. The point is to
  show the comparison distinguishes "found with the right value" from "not found".
- **Ambiguity:** a Server restart during initial PD election is not a failure; re-check
  once PD reports a leader. A Server still crash-looping after PD has a leader and all
  Stores are registered **is** a failure.
- **Blame rule:** pods Ready but PD short of 3 healthy members ⇒ SUT (peer wiring). Pods
  never scheduled ⇒ environment (Kind capacity, image pull, storageClass).
- **Verdict ceiling:** `PASS-smoke`. A `NOT-RUN` on arm 2 or 4 is recorded in the row but
  does not cap C1 and does not block session `DONE`.

## C2 — PD leader failover

- **Claim:** Losing the PD leader elects a new leader and the cluster keeps serving writes
  within the tier budget.
- **Budget tier — declare before the fault:** smoke ≤ 120 s to a new leader, hardening
  ≤ 60 s. Choosing the tier after seeing the number invalidates the run.
- **Fault:** resolve the leader, start both loops, accumulate a quiet window, then delete
  **the leader** — not an arbitrary PD pod.

  ```bash
  A=$ATTEMPT

  # 0. RESOLVE THE LEADER. Deleting a follower does not exercise this claim, yet the
  #    follower's departure still produces a transient null pdLeader that would satisfy
  #    the landing evidence - a PASS on a claim never tested.
  SURVIVOR=$(kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pods \
    -l "app.kubernetes.io/instance=$KTEST_REL,<pd-selector>" \
    -o jsonpath='{.items[0].metadata.name}')
  # extract pdLeader.grpcUrl using the envelope path recorded in the chart contract,
  # then map its host to a pod by name or podIP from pods-uids.*.tsv
  LEADER_HOST=$(...)                       # host part of pdLeader.grpcUrl
  PD_LEADER_POD=$(grep -F "$LEADER_HOST" "$KTEST_DIR/artifacts/pods-uids.C2.attempt$A.tsv" \
                    | cut -f1)
  [ -n "$PD_LEADER_POD" ] && [ "$PD_LEADER_POD" != "$SURVIVOR" ] \
    || { echo "could not resolve a leader distinct from the poll vantage"; exit 1; }
  echo "leader=$PD_LEADER_POD survivor=$SURVIVOR" > "$KTEST_DIR/artifacts/c2-leader.attempt$A.txt"

  # 1. membership poll (uses the validated wrapper from the membership-oracle section)
  nohup sh -c 'for i in $(seq 1 300); do … ; sleep 2; done' \
    > "$KTEST_DIR/artifacts/c2-poll.attempt$A.jsonl" 2>&1 &
  printf '%s\tc2-poll %s\n' "$!" "$KTEST_TS" >> "$KTEST_DIR/logs/bg.pids"; disown 2>/dev/null || true

  # 2. write loop - runs INSIDE the load pod, so $SVR resolves and the credential
  #    comes from the chart Secret rather than a host path
  nohup kubectl --context "$KTEST_CTX" -n "$KTEST_NS" exec "$LOADPOD" -- sh -c '
      umask 077; printf "user = \"admin:%s\"\n" "$<PW_ENV>" > /tmp/c
      i=0; while [ $i -lt 300 ]; do i=$((i+1))
        ID="c2-'"$A"'-$i"
        CODE=$(curl -s -o /dev/null -w "%{http_code}" -K /tmp/c \
          -H "Content-Type: application/json" -X POST "$0/graph/vertices" \
          -d "{\"label\":\"$1\",\"id\":\"$ID\",\"properties\":{\"$2\":\"$ID\"}}")
        printf "%s,%s,%s\n" "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$ID" "$CODE"
        sleep 1
      done' "$API_BASE/graphs/$G" "$L" "$K" \
    > "$KTEST_DIR/artifacts/c2-writes.attempt$A.csv" 2>&1 &
  printf '%s\tc2-writes %s\n' "$!" "$KTEST_TS" >> "$KTEST_DIR/logs/bg.pids"; disown 2>/dev/null || true

  sleep 40    # quiet window: >= 15 poll samples and >= 10 rows with a 2xx status

  # 3. confirm the resolved leader is STILL leader in the latest sample, then delete it
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" delete pod "$PD_LEADER_POD"
  ```

  Both loops are bounded by `seq`/counter, `nohup`+`disown` so they survive the block
  returning, and recorded with a **tab** separator plus the session timestamp so teardown
  can identify them. Stop them at the end of C2 by killing the recorded PIDs and confirming
  the artifacts stopped growing — do not leave them for P8.

- **Landing evidence:** the poll artifact parses as JSONL, contains **≥ 10 pre-fault
  samples**, and contains at least one post-fault sample whose `pdLeader` is null or differs
  from the resolved pre-fault leader; the post-fault leader identity differs from the
  **deleted pod's**; and the deleted pod's uid differs from the one in
  `pods-uids.C2.attempt$A.tsv`. Re-electing the same node afterwards is fine — the
  evidence is the observed gap, not the final leader. Without the poll artifact there is no
  landing evidence: `INCONCLUSIVE-fault-not-proven`.
- **Oracle:** the pre-fault window contains **≥ 10 rows with a 2xx status** (fewer ⇒
  `INCONCLUSIVE-fault-not-proven`, blame harness — the loop never reached the Server, and an
  all-non-2xx CSV would make the clauses below vacuously true); a leader is present again
  within the declared budget; every write whose CSV row shows a 2xx is readable afterwards;
  no 2xx write is missing. After the budget window, run the **three-vantage** membership
  oracle once and save it — that artifact, not the single-vantage poll, is what green-but-
  broken check 7 is scored on.
- **Negative control:** run the leader-change detector over **only the ≥ 10 pre-fault
  samples** — it must report no change. That window is why the `sleep 40` exists: over one
  or two samples a change detector cannot fire, so the control would pass even for a
  completely broken detector.
- **Ambiguity:** writes whose CSV row shows a timeout or 5xx are *unknown-outcome*. Present
  or absent, neither is a failure. Only 2xx writes are held to the durability condition.
- **Blame rule:** acknowledged write lost ⇒ SUT. Write loop died on its own (CSV stops
  early) ⇒ harness. CSV missing status codes ⇒ checker.

## C3 — Store minority loss

- **Claim:** With 3 Stores and a replica count ≥ 3, losing one leaves a majority and the
  cluster continues to serve reads and writes.
- **Precondition:** record the replica count from `/v1/shardGroups` in the baseline. **If
  replication is 1, this claim does not apply** — losing a Store legitimately loses its
  partitions. Record `NOT-RUN — replica count 1, claim not applicable`.
- **Budget tier:** smoke — the round-trip succeeds at all during the window; hardening —
  it succeeds throughout the window with no 5xx.
- **Fault:** a plain pod delete is usually **too weak to land**: the StatefulSet recreates
  the pod in seconds while PD's detection is heartbeat-based, so the Store may never leave
  the healthy set. Hold it down long enough to be detected:

  Run the full ordered recipe — both scored artifacts must already be streaming before the
  scale-down, and the scale-back must be guaranteed:

  ```bash
  A=$ATTEMPT
  # 1. absence poll (landing evidence) and CRUD loop (oracle), both in-cluster, both
  #    started BEFORE the fault. Same construction as C2's loops.
  nohup sh -c '... /v1/stores every 2s ...' > "$KTEST_DIR/artifacts/c3-stores.attempt$A.jsonl" 2>&1 &
  printf '%s\tc3-poll %s\n' "$!" "$KTEST_TS" >> "$KTEST_DIR/logs/bg.pids"; disown 2>/dev/null || true
  nohup kubectl --context "$KTEST_CTX" -n "$KTEST_NS" exec "$LOADPOD" -- sh -c '...' \
    > "$KTEST_DIR/artifacts/c3-crud.attempt$A.csv" 2>&1 &
  printf '%s\tc3-crud %s\n' "$!" "$KTEST_TS" >> "$KTEST_DIR/logs/bg.pids"; disown 2>/dev/null || true

  sleep 40                       # quiet window: >= 10 rows with a 2xx status

  # 2. minority loss, then a BOUNDED wait for PD to notice
  trap 'kubectl --context "$KTEST_CTX" -n "$KTEST_NS" scale sts <store-sts> --replicas=3' EXIT
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" scale sts <store-sts> --replicas=2
  SEEN=no
  for i in $(seq 1 90); do        # 3 minutes, never an unbounded foreground wait
    <PD reports only 2 healthy Stores> && { SEEN=yes; sleep 20; break; }
    sleep 2
  done

  # 3. restore unconditionally, then block until 3 healthy before any later scenario
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" scale sts <store-sts> --replicas=3
  trap - EXIT
  ```

  This is 3→2, a genuine minority loss. It is not the banned "scale to 0", which removes
  quorum entirely.

  The bound and the `trap` matter: the shell tool kills a block at around ten minutes, and
  an unbounded wait between the two `scale` commands would leave the StatefulSet at 2
  replicas. C4 and C5 would then run against a silently degraded cluster and their verdicts
  would be unattributable. If the bound expires with `SEEN=no`, scale back and record
  `INCONCLUSIVE-fault-not-proven`.

  **Before any later scenario starts**, confirm `/v1/stores` shows 3 healthy again.
- **Landing evidence:** a timestamped poll (same shape as C2's) showing a window in which
  PD does not list the third Store as healthy, and `/v1/shardLeaders` showing leadership
  off it.
- **Oracle:** the pre-fault window contains **≥ 10 rows with a 2xx status** (fewer ⇒
  `INCONCLUSIVE-fault-not-proven`, blame harness — a loop that never reached the Server
  fails every request, and the SUT blame rule below must not be applied to it); and the
  CRUD loop has requests whose timestamps fall **inside** the absence window, and those
  requests succeed. Intersect the two artifacts after the run. A
  round-trip that only ran after recovery proves nothing: that is
  `INCONCLUSIVE-fault-not-proven`, not a pass. After scale-up, `/v1/stores` shows 3 healthy
  again.
- **Negative control:** run the same absence detector against the pre-fault baseline — it
  must report all 3 Stores healthy.
- **Ambiguity:** elevated latency is not a failure unless a budget was declared.
- **Blame rule:** writes fail while 2 of 3 Stores are up **and** replica count ≥ 3 ⇒ SUT.
  Only the Server pod that was talking to the downed Store fails while others succeed ⇒ SUT
  (client failover); name the pod. No absence window ever observed ⇒
  `INCONCLUSIVE-fault-not-proven`, blame harness if the poll was too slow, environment if
  PD's heartbeat timeout exceeds the outage the chart allows.

## C4 — PD identity survives a pod IP change  (mandatory)

- **Claim:** A PD pod that is rescheduled and comes back with a **different pod IP** rejoins
  the same raft group **under the same identity**, without manual peer-list surgery, and the
  cluster keeps a quorum.
- **Why it matters:** PD member identity is address-shaped (`raftUrl` / `grpcUrl`). A chart
  that wires raft peers to pod IPs breaks on the first reschedule; one that wires them to
  StatefulSet DNS names survives.
- **Budget tier:** hardening — the settled state is judged within 5 minutes of Ready.
- **Fault — must change the IP:**

  ```bash
  # Capture the pre-fault identity ONCE, before attempt 1, and never rewrite it.
  [ -f "$KTEST_DIR/artifacts/c4-before.txt" ] || {
    kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pod "$POD" \
      -o jsonpath='{.status.podIP}{" "}{.spec.nodeName}{" "}{.metadata.uid}' \
      > "$KTEST_DIR/artifacts/c4-before.txt"
    # plus the membership oracle, saved as c4-members-before.jsonl
  }

  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" delete pod "$POD"
  for i in $(seq 1 90); do
    kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pod "$POD" >/dev/null 2>&1 && break
    sleep 2
  done
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" wait --for=condition=Ready "pod/$POD" --timeout=8m

  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pod "$POD" \
    -o jsonpath='{.status.podIP}{" "}{.spec.nodeName}{" "}{.metadata.uid}' \
    > "$KTEST_DIR/artifacts/c4-after.attempt$ATTEMPT.txt"
  ```

  The bounded `for` poll matters: `kubectl delete pod` returns once the object is gone, and
  `kubectl wait` against a not-yet-recreated pod errors immediately instead of waiting. If
  the pod never reappears within 180 s, that is `INCONCLUSIVE-fault-not-proven`.

  **An in-place kill inside the container (`kubectl exec … kill 1`) is not a valid fault for
  this claim** — the pod keeps its IP, so the claim is never exercised.

- **Landing evidence:** `c4-before.txt` and the attempt's `c4-after` file both contain a
  **non-empty IPv4**, the pod is `Ready`, and the two IPs **differ**. An empty `podIP` (a
  `Pending` pod) is not a changed IP — it is a failed fault. If the IP is unchanged, repeat
  the plain delete up to three times; Kind's host-local IPAM rotates addresses, so a repeat
  usually lands. If it still has not changed, record `INCONCLUSIVE-fault-not-proven`, blame
  `environment`, and move on.

  **Do not cordon the node to force relocation.** Kind's default StorageClass is node-bound
  local-path: with the original node cordoned the recreated PD pod hits a volume
  node-affinity conflict, sits `Pending` for the whole timeout, and the cluster runs 2/3 PD
  throughout — measuring the escalation, not the claim.

- **Oracle:** once the pod is Ready, run the membership oracle from **all three** PD pods
  and require:
  1. `numOfNormalService == 3`, every `pdList[].state` equal to the baseline healthy value,
     all vantages agreeing;
  2. exactly one `pdLeader`, identical across vantages;
  3. no member's `raftUrl` or `grpcUrl` contains the **old** pod IP from `c4-before.txt`;
  4. the restarted member's `raftUrl`/`grpcUrl` is **the same string as before the fault** —
     a stable DNS name. If instead it is the **new IP**, the claim "under the same identity"
     is falsified: **FAIL**, blame SUT, even though quorum survived. Re-run once and
     classify reproducible or nondeterministic per the re-run policy — expect reproducible,
     since it is a wiring property. Only the stable-DNS branch can be a PASS;
  5. an authenticated CRUD round-trip succeeds after recovery.
- **Negative control — runs the real matcher over the real input shape.** Take the
  **post-fault membership artifact**, inject the old IP into one member's `raftUrl` in a
  copy, and run the *same* matcher command over the copy:

  ```bash
  OLD_IP=$(cut -d' ' -f1 "$KTEST_DIR/artifacts/c4-before.txt")
  sed "0,/\"raftUrl\":\"[^\"]*\"/s//\"raftUrl\":\"$OLD_IP:0\"/" \
    "$KTEST_DIR/artifacts/c4-members-after.attempt$ATTEMPT.jsonl" \
    > "$KTEST_DIR/artifacts/c4-control.attempt$ATTEMPT.jsonl"
  <the condition-3 matcher> "$KTEST_DIR/artifacts/c4-control.attempt$ATTEMPT.jsonl"  # must HIT
  ```

  It **must** report a hit. Also require the matcher's input to be non-empty and to parse as
  JSONL before condition 3 is evaluated at all. Grepping `c4-before.txt` — a three-token
  text file — would prove only that the IP string is greppable; it would not catch the
  failure this control names, which is the matcher reading the wrong file or the wrong
  field and therefore reporting "absent" for an artifact it never really parsed. And do not
  use "the old IP appears in the pre-fault membership snapshot" as the control: on a
  correctly DNS-wired chart it does not appear there, so a correct chart would be blamed.
- **Ambiguity:** a brief window where the restarted member reports an unhealthy state is
  expected. Re-poll for up to 5 minutes; only the settled state is judged.
- **Blame rule:** stale old IP still in the peer list, quorum lost, or identity changed to a
  new IP ⇒ SUT (chart wires identity to pod IPs). `kubectl` could not read the pod IP ⇒
  harness. Pre-fault snapshot missing, so conditions 3–4 cannot be evaluated ⇒ checker. IP
  never changed after three attempts ⇒ environment.

## C5 — Auth is enforced

- **Claim:** With auth on, the Server rejects unauthenticated and wrongly-authenticated
  requests and accepts correct ones.
- **Budget tier / Fault / Landing evidence:** `n/a` — a boundary probe, no fault injected.
- **Oracle:** three arms against the same read endpoint **and** the same write endpoint —
  six results, each recorded with the status-code recipe in
  `references/topology-and-endpoints.md` (`curl -sf` prints nothing on 401 and is useless
  here). Pass requires all six statuses to **match the declared expected values**: no
  credentials ⇒ 401/403; wrong password ⇒ 401/403; correct admin credentials ⇒ 2xx.
  "Six results recorded" is not the bar — a Server that returns 200 to an unauthenticated
  write also records six results.
- **Negative control:** the no-credential and wrong-password arms are themselves the
  controls; they demonstrate the checker can observe a rejection.
- **Ambiguity:** a 404 where 401 was expected means the path is wrong, not that auth failed
  — fix the path and re-run before judging.
- **Blame rule:** unauthenticated request succeeds ⇒ SUT, high severity. Endpoint
  unreachable ⇒ harness.
- **Verdict ceiling:** `PASS-smoke`, because no fault landed. Any arm not run caps it at
  `PARTIAL-surface`.
- **Extension when time allows:** repeat only the **`GET`** arm against Hubble, the PD REST
  port (`/v1/members`) and the Store REST port (`/v1/health`). Those arms are recorded as
  observations in C5's artifact and are **scored only under C8** — a PD management port
  answering 200 is C8's finding, not a C5 failure. Never send any `POST`, `PUT`, or
  `DELETE` to a PD or Store management port.

## C6 — Durability across a rolling Server restart

- **Claim:** Data acknowledged before a **rolling** restart of the Server pods is readable
  after it. (Rolling: a Server stays alive throughout. This does not test a full
  simultaneous outage.)
- **Budget tier:** hardening — all acknowledged data readable within 2 minutes of rollout
  completion.
- **Fault:** `kubectl --context "$KTEST_CTX" -n "$KTEST_NS" rollout restart <server workload>`,
  then `rollout status --timeout=8m`.
- **Landing evidence:** every Server pod has a new `metadata.uid`, and the rollout reports a
  new revision.
- **Oracle:** a set of vertices written with recorded ids and values before the restart is
  readable, complete, and value-identical afterwards.
- **Negative control:** read back an id never written; the checker must reject it, using the
  missing-id status observed in the baseline.
- **Ambiguity:** writes in flight during the restart are unknown-outcome and excluded.
- **Blame rule:** acknowledged data missing ⇒ SUT. Ids not recorded before the fault ⇒
  checker.

## C7 — Store data survives pod restart

- **Claim:** Store state is persisted to its PVC and survives pod restart — the restarted
  Store returns with **its own data on disk**, not by re-replicating from peers.
- **Budget tier:** hardening — judged on the data directory, not on cluster-level reads.
- **Fault:** delete one Store pod and let the StatefulSet recreate it.
- **Landing evidence:** new pod uid, and the **same PV** bound to the same PVC:

  ```bash
  PVC=$(kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pod "$POD" \
         -o jsonpath='{.spec.volumes[?(@.persistentVolumeClaim)].persistentVolumeClaim.claimName}')
  PV=$(kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pvc "$PVC" -o jsonpath='{.spec.volumeName}')
  kubectl --context "$KTEST_CTX" get pv "$PV" -o jsonpath='{.metadata.uid}'
  ```

  Compare the **PV** uid before and after. The PVC's own uid is invariant unless the PVC is
  deleted, so it is only a sanity cross-check, not evidence.
- **Oracle — filesystem, not cluster reads.** Record a **filename set**, not `ls -la`: a
  live Store churns sizes and mtimes through compaction and WAL rotation, so a byte-level
  diff mismatches on a perfectly persistent Store, and a bare file *count* can match across
  an entirely different set.

  ```bash
  # before the fault
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" exec "$STOREPOD" -- \
    sh -c 'cd <data-path> && find . -type f | sort' \
    > "$KTEST_DIR/artifacts/c7-files-before.attempt$ATTEMPT.txt"

  # arm the post-delete read BEFORE deleting, so it can catch Running-but-not-Ready
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" delete pod "$STOREPOD"
  for i in $(seq 1 600); do
    P=$(kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pod "$STOREPOD" \
          -o jsonpath='{.status.phase}' 2>/dev/null)
    [ "$P" = Running ] && break
    sleep 0.2
  done
  READY_AT_READ=$(kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get pod "$STOREPOD" \
    -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" exec "$STOREPOD" -- \
    sh -c 'cd <data-path> && find . -type f | sort' \
    > "$KTEST_DIR/artifacts/c7-files-after.attempt$ATTEMPT.txt"
  ```

  **Pass condition:** the pre-fault filename set is a **subset** of the post-fault set
  (`comm -23 before after` is empty). Record `READY_AT_READ`; if the pod was already `Ready`
  when the read landed, re-replication may have completed and the verdict is capped at
  `PARTIAL-model` — that is the state this scenario exists to exclude.

  Cluster-level "the data is still readable" is **not** sufficient: with three replicas, a
  Store that came back with a wiped directory re-replicates and every read still succeeds,
  so that check passes identically whether persistence works or not. If only cluster-level
  reads are available, the verdict is capped at `PARTIAL-model`.
- **Negative control — deterministic:** run the same comparator with the post-fault side
  replaced by an **empty** listing; it must report the pre-fault files as missing.

  ```bash
  : > "$KTEST_DIR/artifacts/c7-files-empty.txt"
  comm -23 "$KTEST_DIR/artifacts/c7-files-before.attempt$ATTEMPT.txt" \
           "$KTEST_DIR/artifacts/c7-files-empty.txt"   # must be NON-empty
  ```

  Comparing against another Store pod is not a control: two Stores can legitimately hold the
  same file count, and "these two differ" does not demonstrate the comparator detects a
  *wiped* directory — which is the failure C7 exists to catch.
- **Ambiguity:** partial re-replication after a short outage is expected; judge on whether
  the pre-existing files returned with the pod, not on transfer counters.
- **Blame rule:** data directory empty on return with a stable PV ⇒ SUT. `emptyDir` in the
  rendered manifest ⇒ SUT (chart), knowable from P2 with no fault at all. PVC rebound to a
  different PV ⇒ environment or SUT depending on whether the claim template changed.

## C8 — PD management surface exposure

- **Claim:** PD's management endpoints are not reachable unauthenticated from a workload
  that has no business calling them.
- **Budget tier / Fault / Landing evidence:** `n/a` — probe only.
- **Preconditions — both required before this scenario may produce a FAIL:**
  1. Resolve the Service, so an unresolvable name is never mistaken for a closed port:
     `kubectl --context "$KTEST_CTX" -n "$KTEST_NS" get svc <pd-service>` must succeed.
  2. Establish whether the cluster **enforces NetworkPolicy at all**. Kind's default CNI
     (kindnet) does not. Read the CNI without changing anything:
     `kubectl --context "$KTEST_CTX" -n kube-system get ds`. On a non-enforcing cluster a
     chart shipping a *correct* NetworkPolicy produces exactly the same HTTP 200 as a chart
     shipping none, so reachability alone cannot separate SUT from environment. Record
     whether the render contains `kind: NetworkPolicy`
     (`grep -c 'kind: NetworkPolicy' "$KTEST_DIR/artifacts/render-3x3.yaml"`).
- **Probe** — record both the HTTP code **and** curl's exit status:

  ```bash
  kubectl --context "$KTEST_CTX" -n "$KTEST_NS" run "ktest-probe-$KTEST_TS-c8-$RANDOM" --rm -i \
    --restart=Never --labels="hugegraph-ktest/session=$KTEST_TS" \
    --image=curlimages/curl:<pinned> -- sh -c \
    'curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 "$0"; echo "exit=$?"' \
    "http://<pd-service>:${PD_REST_PORT}/v1/members"
  ```

- **Oracle:** **pass** = curl exit **7** (connection refused) or **28** (timeout), or an
  HTTP 401/403.

  **HTTP 2xx** is judged against precondition 2:
  - cluster **enforces** NetworkPolicy ⇒ `FAIL-reproducible`, blame SUT;
  - cluster does **not** enforce (the default Kind case) ⇒ `INCONCLUSIVE-env`, **unless**
    the render also shows no `kind: NetworkPolicy` *and* a management Service exposed
    beyond `ClusterIP`. In that case the finding is grounded in the render rather than in
    the reachability result, and is reported as `FAIL-reproducible` / SUT citing the
    manifest — never the probe alone.

  The reason 2xx matters at all is that `POST /v1/members/change` on that
  same port has no authentication check upstream, so any pod that can route there can
  rewrite the PD raft peer list. The chart must scope the management port via Service type
  or NetworkPolicy. Where possible, run the probe from a namespace that plausibly has no
  business calling PD, rather than from PD's own namespace. `%{http_code}` of `000` is ambiguous on its own — refused, timed out and
  DNS-failed all report it — which is why the exit status and the Service precondition are
  both required.
- **Negative control:** the same probe against the Server's authenticated REST port must
  return 401/403, proving the probe records status codes rather than always reporting
  success.
- **Ambiguity:** exit 6 (could not resolve host) is `harness`, not a pass — fix the Service
  name and re-run.
- **Hard boundary:** **`GET` only.** The impact above is known from the upstream source and
  must **not** be demonstrated. Send no `POST`, `PUT`, or `DELETE` to a PD or Store
  management port — that port exposes `POST /v1/members/change`, `POST /v1/store/{storeId}`
  and `POST /v1/store/log`, all of which mutate cluster state.
- **Verdict ceiling:** `PASS-smoke`, because no fault landed.

## C9 — Re-apply / upgrade idempotency

- **Claim:** `helm upgrade` with unchanged values is a no-op that leaves a working cluster.
- **Budget tier:** hardening — the manifest diff is empty and no PD/Store pod restarts.
- **Fault:**
  `helm --kube-context "$KTEST_CTX" upgrade "$KTEST_REL" helm/hugegraph -n "$KTEST_NS" -f "$KTEST_DIR/artifacts/values-3x3-auth.yaml" --wait --timeout 8m`
- **Landing evidence:** `helm --kube-context "$KTEST_CTX" history "$KTEST_REL" -n "$KTEST_NS"`
  shows the previous revision superseded and a new revision deployed, **and** both revision
  manifests are captured:

  ```bash
  # N is the deployed revision BEFORE the upgrade - derive it, never assume a number
  N=$(helm --kube-context "$KTEST_CTX" history "$KTEST_REL" -n "$KTEST_NS" -o json \
       | python3 -c 'import json,sys; print(max(r["revision"] for r in json.load(sys.stdin)))')
  for R in $N $((N+1)); do
    helm --kube-context "$KTEST_CTX" get manifest "$KTEST_REL" -n "$KTEST_NS" --revision "$R" \
      > "$KTEST_DIR/artifacts/c9-manifest-r$R.yaml"
  done
  ```

  A revision number alone is close to "the command exited cleanly" — the meaningful artifact
  is the manifest pair.
- **Oracle:**
  `diff "$KTEST_DIR/artifacts/c9-manifest-r$N.yaml" "$KTEST_DIR/artifacts/c9-manifest-r$((N+1)).yaml"`
  is empty; every
  StatefulSet's `.metadata.generation` is unchanged; no pod restarts that a values change
  explains; membership oracle still healthy; CRUD round-trip still succeeds. A chart that
  regenerates a password or token secret on every render breaks auth here — that is the
  finding.
- **Negative control — must produce a non-empty diff:** compare revision N's manifest
  against a deliberately mutated copy:
  ```bash
  sed '5d' "$KTEST_DIR/artifacts/c9-manifest-r$N.yaml" \
    > "$KTEST_DIR/artifacts/c9-mutated.yaml"          # stays inside the mode-700 session dir
  diff "$KTEST_DIR/artifacts/c9-manifest-r$N.yaml" "$KTEST_DIR/artifacts/c9-mutated.yaml"
  ```
  must report a difference. The manifests embed the rendered Secret, so the mutated copy
  never goes to `/tmp`. (Diffing a file against itself proves nothing — an empty result
  is indistinguishable from `diff` being replaced by `true`.)
- **Ambiguity:** a restart caused by a deliberately changed image tag is expected; a restart
  with an identical manifest is not.
- **Blame rule:** unchanged values produce a rolling restart of PD ⇒ SUT
  (non-deterministic template output, e.g. `randAlphaNum` without a `lookup` guard).

## Faults that are not allowed to stand in for these

- An in-place process kill that preserves the pod IP **for C4** — it does not exercise
  identity across a reschedule.
- Cordoning a node to force relocation **for C4** — node-bound local-path PVs make the pod
  unschedulable instead of relocating it, and it leaves a cluster-scoped mutation behind.
- A plain pod delete **for C3** when no absence window is ever observed — use the 3→2 scale
  instead, and say which was used.
- `helm uninstall` then `helm install` as a stand-in for C6/C9 — that is a fresh cluster,
  not a restart or an upgrade.
- Scaling a StatefulSet to **0** and back for C2/C3 — it removes quorum entirely rather than
  testing minority loss.
- Deleting a namespace or a PVC to "reset" mid-suite. Start a clean release instead, and say
  in the report that the suite was restarted.
