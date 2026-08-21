# Review log

Three rounds of independent review. Each round ran three read-only reviewers in parallel —
**test method**, **scope and authoring boundaries**, **safety** — none of which was the
implementer. Roughly 170 findings total, of which about 60 were high-severity. All
high-severity findings were resolved.

## Contents
- How to read this
- Round 1
- Round 2
- Round 3
- Findings that were rejected
- What remains unreviewed

## How to read this

Only high-severity findings are listed individually — those are defects that would make a
following agent produce a wrong verdict, damage something the user did not intend to touch,
or violate a stated requirement. Medium and low findings were also applied but are not
enumerated here; the current files reflect them.

## Round 1

65 findings (18 high). The skill at this point installed, ran scenarios, and reported
verdicts, but had three classes of structural problem.

**Safety**

1. The cluster-ownership gate keyed on a context *name* starting with `kind-`. A renamed
   production context passes that trivially and would receive a ten-pod install, pod
   deletions, and `kubectl delete namespace`. → Replaced with a three-legged evidence gate.
2. That gate was prose, not a checklist item, so it left no artifact and could be skipped.
   → Became its own phase, P3a, with a recorded outcome.
3. A fixed cluster name meant a *second* run would reuse and then delete the first run's
   cluster. → Timestamped cluster / namespace / release names.
4. `helm install --create-namespace` silently adopts an existing namespace; teardown then
   cascade-deleted it. → Explicit pre-existence check, session label, label-verified delete.
5. Cordon had no recovery path, and the snippet's 10-minute `kubectl wait` sat between
   cordon and uncordon — the shell tool would kill it mid-way, leaving a node unschedulable.
   → Cordon removed entirely (see round 1, method).
6. C5's extension instructed a *write* probe against PD 8620, contradicting C8's read-only
   boundary — the only documented write there is `POST /v1/members/change`, which rewrites
   the raft peer list. → Extension reduced to `GET`, prohibition restated at that site.
7. `kind create cluster` rewrites the user's `current-context`, and no command pinned a
   context. → Original context captured and restored; every command pins explicitly.

**Method**

8. `PASS-smoke` explicitly accepted "a fault with only generic evidence", contradicting the
   skill's own rule. → Generic evidence now yields `INCONCLUSIVE-fault-not-proven`.
9. Session `DONE` was reachable with zero `PASS-hardening` and zero fault scenarios. →
   `DONE` now requires a `PASS-hardening` and a verdict row for every scenario in the suite.
10. The negative-control check was unsatisfiable for 8 of 9 scenarios, so `PASS-hardening`
    was formally unreachable. → Every scenario now declares a real control.
11. C4's oracle *passed* the failure it exists to detect (identity changing from IP A to
    IP B). → That branch is now a FAIL with SUT blame.
12. C4's landing evidence was satisfied by an empty `podIP` from a `Pending` pod.
13. C4's cordon escalation could not work with node-bound local-path PVs. → Escalation is
    now a repeated plain delete; Kind's IPAM rotates addresses.
14. C8 had no oracle, only an instruction to narrate. → Pass/fail condition defined.
15. C3's CRUD could run after the Store had already recovered. → Must run inside the
    timestamped absence window.
16. The smoke gate was self-blocking: "all four must pass" versus "`helm test` is
    `NOT-RUN`". → Arms marked blocking / non-blocking.
17. The Hubble arm blocked the suite but no request for it existed anywhere.

**Boundaries**

18. `chart-static-checks.md` asserted upstream probe ports as a chart-correctness criterion
    with pre-assigned SUT blame — the exact "never assume" failure the skill forbids.

## Round 2

83 findings (23 high). The round-1 fixes were largely sound; most of this round was defects
*introduced* by those fixes, plus a deep pass on executability.

**Safety**

1. **The `env.sh` bootstrap was circular.** Every block ran
   `source "$KTEST_DIR/artifacts/env.sh"` — but `$KTEST_DIR` is defined only *inside* that
   file. In a fresh shell it expands to `/artifacts/env.sh`, fails, and the block continues
   with every variable empty.
2. **An empty `KTEST_CTX` is worse than none.** `kubectl --context ""` and
   `helm --kube-context ""` are treated as "flag not given" and fall back to the user's
   `current-context` — the precise failure the pinning rule exists to prevent. → Mandatory
   preamble with `set -u` and `:?` guards.
3. `KTEST_CTX` was pinned as `kind-$KTEST_CLUSTER` even on the pre-existing-cluster branch,
   where that cluster does not exist. → The two branches pin different values.
4. The in-place `env.sh` edits exited 0 when the pattern matched nothing, so ownership could
   be recorded when it was not. → Verified after writing.
5. The `env.sh` heredoc was unquoted and broke on a home directory containing a space.
6. The teardown namespace guard passed when both sides were empty (`"" = ""`).
7. `helm uninstall` ran unguarded, before the label check.
8. Foreground `--wait --timeout 15m` exceeds the shell tool's timeout, leaving releases
   stuck in `pending-install`. → Detached install plus polling.
9. `| tee` discards exit status, so a failed `kind create` still recorded ownership.
10. Pod deletes carried literal names with no namespace or label guard.
11. The password had no defined origin or cross-block persistence; the in-pod `printf`
    interpolated it into a *format string*; the "mode 600" claim was never enforced by any
    command.
12. `kubectl describe pod` on the failure path can print an inline password.

**Method**

13. C4's negative control was **inverted** — it could only be satisfied when the chart was
    broken, so a correctly DNS-wired chart would be blamed on the checker.
14. C5 claimed `PASS-hardening` with no fault, and defined the bar as "results recorded" —
    a Server returning 200 to an unauthenticated write also records results.
15. Green-but-broken checks 2 and 7 had no `n/a` carve-out, so no faultless scenario could
    reach any PASS.
16. A `NOT-RUN` on a non-blocking arm still capped C1 at `PARTIAL-surface`, making session
    `DONE` unreachable for any chart without a `helm.sh/hook: test` resource.
17. C7's control was not a control, and its oracle could not distinguish persistence from
    re-replication.
18. C9's control was circular — diffing a file against itself.
19. C2's control was degenerate over ≤ 1 pre-fault sample.
20. **C2's poll artifact was malformed.** `curl` emits no trailing newline, so samples ran
    together and the `.jsonl` file was neither JSON nor JSONL.
21. C3's fault was too weak to land — a StatefulSet recreates the pod faster than PD's
    heartbeat detection. → 3→2 scale.
22. The membership oracle asserted an ungrounded "healthy" state string and read
    `numOfService` where `numOfNormalService` was meant.

**Boundaries**

23. The session directory sat inside the repository under test, so the credential-bearing
    values file and rendered Secret could be swept into a `git add -A`.

## Round 3

23 findings, all high. This was the final permitted round.

**Safety**

1. **The `current` symlink introduced in round 2 was a machine-global mutable pointer.** A
   second concurrent run repoints it; the first run's next block then loads the second run's
   `KTEST_NS` / `KTEST_TS`, passes the teardown label gate against the *other* session's
   namespace, and deletes it. → Symlink removed; P0 prints an absolute path the agent pastes
   literally, and the preamble asserts the loaded env belongs to this session.
2. **`${PIPESTATUS[0]}` is a bash-ism and this harness runs zsh**, where under `set -u` it
   is a fatal "parameter not set" — aborting the cluster-create block *on the success path*,
   leaving a live cluster with ownership unrecorded and `current-context` hijacked. →
   Redirect instead of pipe; ownership recorded from `kind get clusters`; context restored
   on every exit path.
3. The teardown block still carried the round-1 circular bootstrap, so the single most
   dangerous block in the skill would not run at all.
4. The pre-existing-cluster branch wrote `KTEST_CTX` unquoted and unverified — a context
   name containing a space truncates the assignment; one containing `;` or `$(…)` executes
   in every block that sources `env.sh`. → `shlex.quote` plus atomic replace plus
   verification.
5. The `curl.cfg` recipe still used `$HG_ADMIN_PASSWORD`, a variable rule 1 forbids and
   nothing defines.
6. A credential-bearing manifest was copied to a fixed, predictable `/tmp` path.
7. The teardown PID guard was a substring match on any process, and C2 recorded PIDs with a
   space while the reader split on tab.

**Method**

8. **C2's write loop mixed the host-side curl config with the in-cluster Service DNS name**,
   so every request failed with curl exit 6 — and C2's oracle ("every 2xx write is readable
   afterwards") is *vacuously true* over a CSV with zero 2xx rows. C2 would have reported
   `PASS-hardening` on a completely dead workload. The same dead loop under C3 converts a
   harness failure into a session FAIL blamed on the chart.
9. `$ATTEMPT` and `$N` were never defined; under `set -u` every attempt-scoped block aborts.
10. C2's fault target was undefined and nothing verified it was the leader — deleting a
    follower still produces a transient null `pdLeader` that satisfies the landing evidence.
11. The check-7 `n/a` carve-out omitted C2, C3, and C8, which are single-vantage by
    construction, making `DONE` unreachable whenever C2 runs.
12. C8 reports a chart defect unconditionally on Kind, because kindnet does not enforce
    NetworkPolicy — a correct policy and no policy look identical. → Enforcement precondition
    plus render-grounded finding.
13. C7's pass condition was not computable from the artifact the recipe produced (`ls -la`
    churns on a live Store; a file *count* can match a different set), and the
    Running-but-not-Ready window had no recipe.
14. C4's control exercised `grep` against a plain-text file rather than the oracle's actual
    input shape.
15. The `printf` JSON wrapper emitted `"members":}` whenever curl returned nothing — exactly
    the samples the fault window produces.
16. Background loops used a bare `&` and might not survive the block; teardown could not
    identify them.
17. C3's recipe omitted both scored artifacts and had an unbounded wait with no guaranteed
    scale-back, risking leaving the StatefulSet at 2 replicas for later scenarios.
18. uid-based landing evidence had no capture step — `-o wide` does not print uids.

**Boundaries**

19. C9's oracle and control used bare relative paths that resolve inside the chart repo.
20. The pre-fault declaration was instructed to be written into the repository under test.
21. Eleven placeholders (`<pd-selector>`, `<store-sts>`, `<data-path>`, `<PW_ENV>` …) were
    attributed to "the P1 mapping", which never collected them — so an agent would invent
    them. → P1 now reads a full chart-contract table.
22. The vertex read-by-id path and its `%22` quoting had no provenance.
23. The smoke-gate definition still drifted between `SKILL.md` and `scenarios.md § C1`.

## Findings that were rejected

Two reviewer claims were checked against upstream source and found incorrect. Both were
resolved by adding the missing provenance rather than by changing correct content.

- **"`auth.admin_pa` looks truncated and is not a plausible property name."** It is real:
  `docker-entrypoint.sh` sets exactly that property from `PASSWORD` before init-store.
- **"`POST /v1/store/{storeId}` and `POST /v1/store/log` are invented endpoints."** Both are
  real `@PostMapping` annotations in `StoreAPI.java`. They are named in the skill precisely
  so an agent knows what it must not call.

A third claim — that the vertex read-by-id path was ungrounded — was *substantially*
correct and better than the reviewer knew: the path exists, but upstream `master` nests it
under `graphspaces/{graphspace}/`, so the base path is version-dependent and is now a
discovered contract field.

## What remains unreviewed

Round 3 was the last permitted round, so its fixes were verified **mechanically by the
implementer** rather than by a fourth reviewer:

- all 55 snippets parse under both `bash -n` and `zsh -n`
- no bash-only constructs remain
- every `$VAR` is defined or is a documented contract field
- every cluster command pins a context
- all Contents entries resolve; no orphaned or two-level references
- the zip round-trips byte-identically

The least-scrutinised parts of the skill are therefore the round-3 changes: the C2/C3
in-pod workload loops, the C7 filesystem oracle, and the C8 NetworkPolicy precondition. A
fourth review focused there would be the highest-value next check.
