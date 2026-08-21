# Verdicts, blame classes, and the green-but-broken checks

## Contents
- Verdict states
- Severity order and which cap wins
- The smoke ceiling
- Fault-landing evidence
- Blame classes
- Green-but-broken checks (run before any PASS)
- Session verdict
- Downgrade rules

## Verdict states

One of these per scenario, plus two for the static layer. Never invent a state, and never
write a bare "passed".

| Verdict | Meaning |
|---|---|
| `PASS-static` | A static check (lint, render, schema, unit test) ran clean. Touches no cluster and is evidence for no claim. |
| `FAIL-static` | A static check failed: template error, schema violation, failing unit test. Deterministic, so no re-run requirement. |
| `PASS-smoke` | Met the smoke budget: the thing came up and answered once, with the declared oracle and its negative control both executed. |
| `PASS-hardening` | Met the hardening budget: the declared fault landed **with the declared evidence**, the declared oracle ran, the negative control ran, and the green-but-broken checks are clean. Requires a landed fault, so faultless scenarios (C1, C5, C8) can never reach it. |
| `FAIL-reproducible` | The oracle failed and the failure reproduces on re-run. Report n/N. |
| `FAIL-nondeterministic` | The oracle failed but the failure does not reproduce. Still a FAIL; report n/N. |
| `INCONCLUSIVE-fault-not-proven` | The fault's landing evidence was absent, ambiguous, generic, or contradicted. |
| `INCONCLUSIVE-oracle-too-weak` | The oracle cannot distinguish pass from fail, or has no negative control. |
| `INCONCLUSIVE-env` | The environment blocked the run: no cluster, image pull failure, no storageClass, insufficient Docker memory or disk. |
| `PARTIAL-surface` | Only some arms or surfaces ran. |
| `PARTIAL-model` | The scenario exercised the claim only partially — C7 judged on cluster-level reads alone is the worked example. |
| `NOT-RUN` | The scenario never started, or a required chart key does not exist. Give the one-line reason. |

`INCONCLUSIVE` is a first-class result, not a soft failure and not a near-pass. Every
`INCONCLUSIVE` and every `NOT-RUN` carries a single-line reason.

## Severity order and which cap wins

From weakest to strongest:

```
NOT-RUN  <  INCONCLUSIVE-*  <  PARTIAL-*  <  PASS-smoke  <  PASS-hardening
```

`FAIL-*` sits outside this order: a FAIL is never capped or upgraded, and it always
decides the session verdict.

**When two caps apply, the lower one wins** — with one stated exception: an arm the
scenario **declares non-blocking** is exempt from the `NOT-RUN` cap. C1's `helm test` and
Hubble arms are the only such arms in this skill. A `NOT-RUN` there is recorded in the row,
does not cap C1 below `PASS-smoke`, and does not block session `DONE`. Without that
exemption, any chart shipping no `helm.sh/hook: test` resource — the common case, and the
case the non-blocking arm exists to tolerate — could never reach `DONE`.

For every other arm the lower cap wins as normal.

## The smoke ceiling

C1's oracle **is** the smoke gate. There is one definition — `references/scenarios.md`
§ C1 — which names all four arms and marks each blocking or non-blocking. This file does
not restate them; it only sets the ceiling.

Only the blocking arms gate the rest of the suite. All four together are `PASS-smoke` for
C1 and nothing more.

They may **never** be reported as:

- "the chart passes", or "the test suite passed";
- evidence for any claim in C2–C9;
- a substitute for a scenario that was not run.

## Fault-landing evidence

For every scenario that injects a fault, the evidence is declared before the fault runs
and must be **specific to that fault**:

| Fault | Acceptable landing evidence | Not acceptable |
|---|---|---|
| Delete PD leader (C2) | a 2 s poll artifact containing a sample with `pdLeader` null or changed; new pod uid | "the delete command exited 0"; final leader identity alone |
| Delete a Store (C3) | timestamped window where PD stops listing the Store as healthy; shard leaders move | "the pod disappeared briefly" |
| Reschedule a PD pod (C4) | non-empty IPv4 before and after, differing, pod Ready | pod uid alone; an empty `podIP` from a `Pending` pod |
| Rolling Server restart (C6) | all Server pod uids changed; new revision | "the rollout command returned" |
| Upgrade (C9) | `helm history` transition plus both revision manifests captured | revision number incremented |

A command exiting cleanly is never landing evidence. Neither is the workload's own error
rate — that is the thing being measured, and using it as evidence makes the test circular.

**Generic evidence does not earn a PASS of any class.** If the declared signal was not
captured, the verdict is `INCONCLUSIVE-fault-not-proven`, however thorough the oracle was.

## Blame classes

Every non-PASS carries exactly one:

- **SUT** — HugeGraph or the chart is wrong. The only class that is a real defect report.
  Name the component (PD / Store / Server / Hubble / chart template) and the behaviour.
- **harness** — the test rig failed: the write loop died, a port-forward dropped, the probe
  pod was evicted, `kubectl` hit the wrong context. Fix and re-run before claiming
  anything about the SUT.
- **checker** — the oracle is wrong or too weak: it did not record what it needed, it
  cannot tell two outcomes apart, it asserted on a field that does not exist (reading
  `numOfService` where `numOfNormalService` was meant is the worked example). Fix the
  oracle, then re-run.
- **environment** — outside both: Docker out of memory or disk, no storageClass, image
  pull denied, Kind reused an IP, no node capacity.

When two classes plausibly apply, pick the one you can *demonstrate* and say what would
distinguish it from the other. Never report SUT on a hunch.

## Green-but-broken checks (run before any PASS)

Record each in the findings entry. Any "no" blocks `PASS-hardening` and downgrades to
`PARTIAL-*` or the relevant `INCONCLUSIVE-*`.

**`n/a` is not a "no" and does not downgrade.**

- Check 2 (fault landing) is `n/a` for every scenario whose declared fault is `none`:
  **C1, C5, C8**.
- Check 7 (three PD vantages) is `n/a` for every scenario whose oracle does not read PD
  membership from three pods: **C3** (reads `/v1/stores`), **C5**, **C6**, **C7**, and
  **C8** (a single-endpoint probe by construction). **C2** must answer *yes*, not `n/a` —
  it scores check 7 on the three-vantage membership snapshot taken after the budget window,
  not on its single-vantage poll.

Without these carve-outs no faultless scenario could reach any PASS — which would destroy
the smoke gate — and C2 and C8 would be forced to `PARTIAL-*`, making session `DONE`
unreachable whenever they run.

1. Did the workload actually run — non-zero operations recorded, not an empty loop?
2. Did the fault land, with the **declared** evidence captured as an artifact?
3. Did the oracle actually execute, with its output saved — not just "no error"?
4. Was the read-back compared to the written **value**, not only to a 2xx status?
5. Did the scenario's declared **negative control** run and produce the failing result it
   is supposed to produce? A control must exercise the *checker* against data known to
   contain the failure — never require the SUT to be broken. Every scenario in the
   catalogue declares one; if this scenario's block has none, the verdict is
   `INCONCLUSIVE-oracle-too-weak`.
6. Were unknown-outcome operations (timeouts, 5xx) separated from acknowledged ones?
7. Did the check run against the whole cluster — all three PD vantages — rather than a
   single pod that happened to be healthy?
8. Could this scenario have passed even if the feature were removed? If yes, the oracle is
   too weak.
9. Was the baseline captured before the fault, and the declaration file written before the
   fault command ran?
10. Are all artifacts referenced by the verdict actually on disk in the session directory?

## Session verdict

- Any `FAIL-*` ⇒ session is **FAIL**.
- **DONE** requires at least one `PASS-hardening`, plus an explicit verdict row for every
  scenario in the selected suite, and no `INCONCLUSIVE-*`, `PARTIAL-*` or `NOT-RUN` among
  them.
- At least one `PASS-*` but no `PASS-hardening` ⇒ **DONE_WITH_CONCERNS** — no fault was
  proven to land, so nothing beyond the smoke gate was established.
- At least one `PASS-hardening` but some `INCONCLUSIVE-*`, `PARTIAL-*`, or a `NOT-RUN` on a
  blocking arm ⇒ **DONE_WITH_CONCERNS**.
- Only `INCONCLUSIVE-*` / `PARTIAL-*` ⇒ **INCONCLUSIVE**.
- Nothing ran ⇒ **BLOCKED**.

A run consisting only of faultless scenarios — however many pass — can never be **DONE**,
because `PASS-hardening` requires a landed fault. That is the rule that stops a smoke run
being reported as a suite pass.

## Downgrade rules

- Any arm `NOT-RUN` or `PARTIAL-*` caps the scenario at `PARTIAL-surface`, **except** an
  arm the scenario declares non-blocking (see § Severity order and which cap wins).
- C1, C5 and C8 are capped at `PASS-smoke`: none of them injects a fault, and
  `PASS-hardening` requires a landed fault.
- A fault scenario whose landing evidence is generic rather than the declared signal is
  `INCONCLUSIVE-fault-not-proven` — it is not downgraded to a PASS class.
- Never aggregate upward: three `PASS-smoke` results do not make a `PASS-hardening`, and
  static checks contribute no scenario verdict at all.
