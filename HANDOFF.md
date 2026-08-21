# Handoff

Everything an incoming agent needs to continue this work. Written at the end of the
authoring session, 2026-08-21.

## Contents
- The mission and its acceptance gates
- What was built
- Upstream ground truth (verified, with sources)
- Verified negative findings that shaped the design
- Design decisions and why they are the way they are
- Verification actually performed
- Known gaps and caveats
- Environment facts
- Suggested next steps

## The mission and its acceptance gates

Author a new personal Claude skill, `hugegraph-helm-ktest`, plus a zip of it, from live
research into community skills — without copying any existing HugeGraph or Kubernetes
skill. Five gates had to be true to finish:

1. Live research happened, and `references/sources.md` cites fetched sources with
   keep / drop / adapt decisions.
2. The skill exists at `~/.claude/skills/hugegraph-helm-ktest/` and behaves as specified.
3. A zip exists and unpacks to that skill.
4. Any older HugeGraph Kubernetes test skill under `~/.claude/skills/` is retired and was
   not used as a draft.
5. Three independent reviews are done with no unresolved high-severity finding.

All five were met. Gate 4 was vacuous: `~/.claude/skills/` contained only `goal-prompt` and
`impeccable`, so there was nothing to retire. Two unrelated HugeGraph skill folders exist
elsewhere on the authoring machine under a different agent's tree; they were deliberately
**never opened**, and no text from them is present here.

## What was built

`SKILL.md` is a navigation layer (373-line body, limit is 500) that defines an 8-phase
workflow and defers all depth to seven one-level reference files:

| Phase | What it does |
|---|---|
| P0 | Session dir outside the repo, `env.sh`, tool + capacity probe, original context saved |
| P1 | Locate `helm/hugegraph`, confirm `Chart.yaml` name, read the **chart contract** |
| P2 | Static checks — lint, render, kubeconform, helm-unittest, ct |
| P3a | **Prove** cluster ownership (three-legged evidence gate) or stop and ask |
| P3b | Acquire a cluster via the kind CLI if none is reachable |
| P4 | Install 3 PD + 3 Store + 3 Server + Hubble, auth on |
| P5 | Smoke gate — C1's oracle, and nothing stronger than `PASS-smoke` |
| P6 | Claim-driven scenarios C1–C9 |
| P7 | Findings report with verdicts and blame |
| P8 | Guarded teardown |

| Reference file | Purpose |
|---|---|
| `scenarios.md` | Claims C1–C9: fault, landing evidence, oracle, negative control, ambiguity, blame |
| `verdicts-and-blame.md` | 12 verdict states, severity order, blame classes, green-but-broken checks |
| `topology-and-endpoints.md` | Ports, health, PD/Store/Server/Hubble APIs, auth, credential-safe recipes |
| `kind-cluster.md` | Ownership proof, kind config, image loading, guarded teardown |
| `chart-static-checks.md` | The static layers and the render-review table |
| `report-template.md` | Session log, pre-fault declaration, findings report |
| `sources.md` | Research provenance — every source with keep / adapt / drop |

The mandated scenario is **C4: PD identity survives a pod IP change.** An in-place kill
that preserves the pod IP is explicitly forbidden as its fault, because it never exercises
the claim.

## Upstream ground truth (verified, with sources)

All fetched live from `apache/hugegraph@master` during authoring.

**Ports** — from `docker/docker-compose-3pd-3store-3server.yml`:

| Component | Ports | Health |
|---|---|---|
| PD | 8686 gRPC, 8620 REST, 8610 raft | `GET :8620/v1/health` |
| Store | 8500 gRPC, 8520 REST, 8510 raft | `GET :8520/v1/health` |
| Server | 8080 | `GET :8080/versions` |
| Hubble | 8088 | HTTP 200 on `/` |

These are the **upstream reference shape, not a contract.** The skill treats them as
starting points to confirm against the chart's own Services, and every runnable command
uses `$PD_REST_PORT` / `$STORE_REST_PORT` / `$SVR_PORT` / `$HUBBLE_PORT` from `env.sh`.

**PD REST** — from `hugegraph-pd/hg-pd-service/.../rest/MemberAPI.java` and `StoreAPI.java`:

- `GET /v1/members` returns `{state, pdList[], pdLeader, numOfService, numOfNormalService,
  stateCountMap}`; each `pdList` entry carries `raftUrl`, `grpcUrl`, `restUrl`, `state`,
  `dataPath`, `role`, `replicateState`.
- Also `GET /v1/stores`, `/v1/storesAndStats`, `/v1/shardGroups`, `/v1/shardLeaders`,
  `/v1/health`.
- **Mutating endpoints on the same port**: `POST /v1/members/change` (carries an in-repo
  comment saying it has *no authentication check*), `POST /v1/store/{storeId}`,
  `POST /v1/store/log`. The skill names these so an agent knows what it must never call.

**Why C4 exists:** PD member identity is **address-shaped** (`raftUrl` / `grpcUrl`). A
chart that wires raft peers to pod IPs breaks on the first reschedule; one that wires them
to StatefulSet DNS names survives. That is a real risk for a Kubernetes chart, not a
synthetic scenario.

**Auth** — from `docker/README.md` and `hugegraph-server/hugegraph-dist/docker/docker-entrypoint.sh`:

- The entrypoint maps `PASSWORD` to the property `auth.admin_pa` in
  `rest-server.properties` and runs `enable-auth.sh` before init-store. Default admin user
  is `admin`.
- `auth.token_secret` (env `HG_SERVER_AUTH_TOKEN_SECRET`) is the JWT secret; ≥ 32 bytes if
  set explicitly.
- Server accepts HTTP Basic and Bearer.

**Vertex API** — from `hugegraph-server/hugegraph-api/.../graph/VertexAPI.java`:

- `@Path("{id}")` under the vertices resource, so `GET .../graph/vertices/{id}` is real.
- `checkAndParseVertexId` runs the id through `JsonUtil.fromJson`, which is **why a string
  id must be JSON-quoted and URL-encoded** as `%22ktest-1%22`.
- On `master` the resource is annotated
  `graphspaces/{graphspace}/graphs/{graph}/graph/vertices`, while the published docs show
  `/graphs/{graph}/…`. **The base path is version-dependent**, which is why the skill
  records `API_BASE` as a discovered contract field rather than hardcoding either shape.

**Hubble** — from `docker/hugegraph-hubble.properties`: listens on 8088; keys `pd.peers`
(PD gRPC), `pd.server` (PD REST), `server.direct_url`, `operations.store.allowed_targets`.

## Verified negative findings that shaped the design

These were checked live and are the reason the skill is built the way it is:

1. **There is no `helm/` directory in `apache/hugegraph@master`.** The full recursive tree
   was fetched: it has `docker/`, `hugegraph-pd/`, `hugegraph-store/`, `hugegraph-server/`,
   `hugegraph-commons/`, `hugegraph-struct/`, `hugegraph-cluster-test/`, `install-dist/` —
   no chart.
2. **No `hugegraph` chart is published on ArtifactHub** (`ts_query_web=hugegraph` returns
   an empty package list).
3. **skills.sh lists no HugeGraph, Helm-chart-testing, or Kubernetes-chart-testing skill**
   in its ranked index. Nothing to fork; this was authored fresh.

Consequence: there is **no values contract to lean on.** The skill must discover the chart,
halt if absent, and read every key, port, selector, and path out of the chart in front of
it. That is why "never invent a flag" is hard rule 2 and is restated in every long
reference file.

## Design decisions and why

**The smoke gate can never be a suite pass.** Pods Ready, `helm test`, one CRUD round-trip,
and Hubble are C1's four arms, capped at `PASS-smoke`. Three separate loopholes that would
have let a smoke run be written up as a pass were closed in review: `PASS-smoke` used to
accept generic fault evidence; session `DONE` used to need no `PASS-hardening`; and unrun
scenarios could simply be omitted. `DONE` now requires at least one `PASS-hardening`, which
requires a landed fault.

**Cluster ownership is proven, not named.** A context called `kind-anything` is not
evidence — `kubectl config rename-context` can point it at production. The gate requires a
loopback API server, a `kind://` providerID on *every* node (via `custom-columns`, because
`jsonpath` over `items[*]` silently hides nodes lacking the field), and a Docker-label match
tying the API server port to a cluster `kind get clusters` lists.

**Nothing is deleted by name.** Cluster, namespace, and release names carry the session
timestamp; teardown verifies a `hugegraph-ktest/session` label before deleting anything, and
both sides of that comparison are checked for emptiness because `"" = ""` would otherwise
authorise the delete.

**No node-level mutation.** C4 originally escalated by cordoning a node to force
relocation. Two independent reviewers converged on why that is wrong: Kind's default
StorageClass is node-bound local-path, so a cordoned-node reschedule leaves the PD pod
`Pending` rather than relocating it — measuring the escalation, not the claim — and an
aborted run leaves the node unschedulable. C4 now escalates by repeating the plain delete,
which Kind's host-local IPAM rotates.

**Every negative control tests the checker, not the chart.** A control must exercise the
matcher against data known to contain the failure. C4's control injects the old IP into a
copy of the real post-fault membership artifact and requires a hit; C9's diffs against a
deliberately mutated manifest copy. An earlier C4 control was *inverted* — it could only be
satisfied when the chart was broken, so a correctly DNS-wired chart would have been blamed
on the checker.

**Workload runs inside the cluster.** `$SVR` is a Service DNS name and the host `curl.cfg`
is a host path; mixing them makes every request fail with curl exit 6 while the CSV still
fills with rows — and C2's oracle ("every 2xx write is readable afterwards") is *vacuously
true* over a CSV with zero 2xx rows. Loops now run through a long-lived load pod, and every
workload oracle first requires ≥ 10 pre-fault rows with a 2xx status.

**Credentials never reach argv.** The password is generated once into a mode-600 file and
read as `$(cat …)`; curl takes it via `-K` config files, built inside the pod from the
chart's Secret for in-cluster calls. `helm template --debug` is banned on the values-bearing
render, and `kubectl describe pod` is redacted on the failure path when the chart sets the
credential inline.

**The shell is zsh.** `${PIPESTATUS[0]}` is a bash-ism; in zsh under `set -u` it is a fatal
"parameter not set" — which in an earlier draft aborted the cluster-create block *on the
success path*, orphaning a cluster with ownership unrecorded and the user's `current-context`
hijacked by kind. All snippets now parse under both bash and zsh.

## Verification actually performed

- All 55 fenced `bash` snippets parse under **both** `bash -n` and `zsh -n`.
- No bash-only constructs remain (`PIPESTATUS`, `[[`, process substitution).
- Every `$VAR` used in a snippet is defined in `env.sh` or documented as a chart-contract
  field.
- Every `kubectl` / `helm` cluster command carries an explicit context flag.
- All 7 reference files are reachable directly from `SKILL.md`; no orphans, nothing two
  levels deep.
- Every Contents entry is an exact substring of a real heading, in all 7 files.
- Frontmatter: folder name equals YAML `name`; description is third person, 748 chars,
  contains all 8 required trigger terms; `disable-model-invocation` unset; only `name` and
  `description` keys.
- The zip round-trips byte-identically to the skill directory.

## Known gaps and caveats

Read these before trusting the skill in anger.

1. **It has never been executed end to end.** No `helm/hugegraph` chart is published
   anywhere reachable, so there was nothing to run it against. Its correctness rests on
   source-grounded facts and reasoning, not on a green run. **The single highest-value next
   step is a real execution against a real chart.**
2. **The final round of fixes was not independently re-reviewed.** The mission capped
   review at 3 rounds and all three were used. Round-3 high-severity findings were fixed
   and then verified *mechanically* by the implementer (the checks listed above), not by a
   fourth reviewer. Treat round-3 changes as the least-scrutinised part of the skill —
   principally the C2/C3 in-pod workload loops, the C7 filesystem oracle, and the C8
   NetworkPolicy precondition.
3. **`scenarios.md` is 712 lines.** Only `SKILL.md` carries the 500-line rule and the file
   has a full table of contents, but it is large. Splitting the C6–C9 extended suite into
   its own one-level reference is the natural next refactor.
4. **Placeholders are deliberate.** `<pd-selector>`, `<store-sts>`, `<data-path>`,
   `<PW_ENV>` and friends are chart-contract fields that P1 must read from the render. They
   are *not* omissions to be guessed — an agent that invents one has violated the skill's
   central rule.
5. **C8 cannot fully separate SUT from environment on Kind.** kindnet does not enforce
   NetworkPolicy, so a correct policy and no policy look identical from a probe. The
   scenario handles this by grounding the finding in the render, but on an enforcing CNI it
   would be a stronger test.
6. **No eval suite exists.** Anthropic's authoring guidance recommends building evaluations
   before extensive documentation; that was not done here.

## Environment facts

Authoring machine, for reference — the skill itself assumes none of this beyond the CLIs:

- macOS (Darwin 25.3.0), shell `/bin/zsh` 5.9
- `kind` v0.32.0, `helm` v4.2.3, `kubectl` client v1.34-era, `docker` present
- `~/.claude/skills/` contained only `goal-prompt` and `impeccable`
- The skill writes all session artifacts to `~/.cache/hugegraph-ktest/<timestamp>/`,
  deliberately **outside** any repository, because they hold the values file and the
  rendered Secret

## Suggested next steps

1. **Run it.** Point it at a real `helm/hugegraph` chart on a Kind cluster and work the
   full C1–C5 suite. Expect the chart-contract discovery in P1 to need the most iteration.
2. Feed what breaks back into the skill — particularly any placeholder that turned out to
   be unreadable from the render, and any oracle that could not be evaluated from the
   artifact it produced.
3. Consider a fourth review round focused on the round-3 changes listed in caveat 2.
4. Split `scenarios.md` if it proves unwieldy in practice.
5. Build the three evaluations the authoring guidance asks for.
