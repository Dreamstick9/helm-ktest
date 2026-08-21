# Research provenance for `hugegraph-helm-ktest`

## Contents
- How to read this file
- A. Distributed-systems testing method (design + execute; claims, faults, oracles, verdicts, blame)
- B. Kubernetes testing / ops skills
- C. Kind and local-cluster acquisition
- D. Helm chart testing tools
- E. Community indexes and overlapping DevOps skills
- F. HugeGraph ground truth (fetched from apache/hugegraph@master)
- G. Skill-authoring rules
- H. Verified negative findings
- I. Review history
- J. What was deliberately NOT used

## How to read this file

Every row was fetched live while this skill was authored (fetch date 2026-08-21).
Decisions are one of:

- **KEEP** — the fact is used directly and is load-bearing.
- **ADAPT** — the *idea* is reused, restated in this skill's own words and re-grounded
  in HugeGraph specifics. No text was copied.
- **DROP** — read, judged out of scope, not used.

Nothing in this skill is a copy of another skill's prose. Where a source supplied a
concept (for example "a fault must be proven to have landed"), the concept is
re-expressed here against HugeGraph's own PD/Store/Server surfaces.

---

## A. Distributed-systems testing method (design + execute; claims, faults, oracles, verdicts, blame)

| Source | What it is for | Decision |
|---|---|---|
| https://github.com/shenli/distributed-system-testing | AI-agent skill pair for designing and executing distributed-systems tests. Repo README documents a 10-state verdict set, an SUT/harness/checker/environment blame split, and a rule that a cleanly-exiting chaos script is not evidence a claim survived. | **ADAPT** — the discipline (claim → hypothesis → fault → landing evidence → oracle → verdict → blame) is the backbone of `references/scenarios.md` and `references/verdicts-and-blame.md`. Verdict names and blame classes are reused as vocabulary; every scenario, oracle and evidence signal is written fresh against HugeGraph. |
| https://raw.githubusercontent.com/shenli/distributed-system-testing/main/skills/executing-distributed-system-tests/SKILL.md | Execution-side workflow: discover the existing toolbox before inventing one, probe the environment, per-scenario session directory, capture fault-landing evidence, apply the plan's oracle (never improvise one), run green-but-broken checks before any PASS, classify failures. | **ADAPT** — the "discover before inventing", "no improvised oracle", "capture landing evidence" and "PASS needs a red-flag pass" rules are carried over as this skill's phase discipline. The 7-phase HugeGraph workflow, the smoke-vs-suite gate and the report shape are this skill's own. |
| https://raw.githubusercontent.com/shenli/distributed-system-testing/main/skills/designing-distributed-system-tests/SKILL.md | Planning side: numbered claims (C1, C2 …), hypotheses that threaten named claims, smoke/hardening/release budget tiers, and a per-scenario model block naming model, history, checker, nemesis + landing evidence, ambiguous-outcome handling and reduction plan. | **ADAPT** — numbered claims, budget tiers and the six-field scenario block are reused as structure. All nine HugeGraph claims and their oracles are authored here. |
| https://arxiv.org/pdf/2201.07521 (ThorFI, network fault injection as a service) | Confirms that network-level fault injection needs an independent signal that the injected fault actually affected the target, separate from the workload's own error rate. | **KEEP** as justification for the "fault landing evidence is mandatory" rule; not cited in SKILL.md to save context. |

## B. Kubernetes testing / ops skills

| Source | What it is for | Decision |
|---|---|---|
| https://github.com/kubetail-org/kstack | 12-skill Kubernetes pack. Read-only by default; mutating actions need explicit confirmation; honours the local kubeconfig context and RBAC; the two destructive skills carry `disable-model-invocation: true`. | **ADAPT** — the "read-only by default, name the context you are about to touch, confirm before mutating" posture becomes this skill's context-safety gate. `disable-model-invocation` is deliberately **not** set here (this skill must stay model-invocable), so that specific mechanism is **DROP**. |
| https://github.com/foxj77/claude-code-skills | Kubernetes/GitOps/Helm skill collection, including `helm-chart-testing` (test pod annotations, categories, debugging) and `helm-chart-review`. Uses a flat `SKILL.md` + `references/*.md` layout. | **ADAPT** — the `SKILL.md` + one-level `references/` layout and the "quick reference table then detail" shape. Its helm-test content is generic; HugeGraph-specific test design here is original. |
| https://github.com/LukasNiessen/kubernetes-skill | Kubernetes skill whose stated purpose is grounding an agent in real cluster facts to stop it inventing resource fields and flags. | **KEEP as a principle** — reinforces this skill's hard rule that every `--set` path must be read out of the chart's own `values.yaml`, never guessed. Content not used. |
| https://metalbear.com/blog/claude-code-skills-for-kubernetes/ | Survey of agent skills that close the feedback loop against a live cluster. | **DROP** — vendor-specific (mirrord) and not needed for a Kind-based loop. |

## C. Kind and local-cluster acquisition

| Source | What it is for | Decision |
|---|---|---|
| https://kind.sigs.k8s.io/docs/user/quick-start/ | Exact CLI: `kind create cluster --config`, `kind get clusters`, `kind delete cluster --name`, `kind load docker-image … --name`, `kind load image-archive`, kubeconfig at `${HOME}/.kube/config`, context name is `kind-<cluster-name>`. | **KEEP** — verbatim command surface in `references/kind-cluster.md`. The `kind-` context prefix is what the safety gate keys on. |
| https://kind.sigs.k8s.io/docs/user/configuration/ | Cluster config schema: `apiVersion: kind.x-k8s.io/v1alpha4`, `name`, `nodes[].role` (`control-plane`/`worker`), `image` pinned by `@sha256:` digest, `extraMounts`, `extraPortMappings`, `kubeadmConfigPatches`. | **KEEP** — the 1-control-plane + 3-worker config in `references/kind-cluster.md` uses these fields only. |

## D. Helm chart testing tools

| Source | What it is for | Decision |
|---|---|---|
| https://helm.sh/docs/topics/chart_tests/ | `helm.sh/hook: test` is the current annotation (`test-success` kept for backwards compatibility); `helm test <RELEASE>` runs test pods and reads container exit codes; documented caveat that testing immediately after install shows transient failures. | **KEEP** — drives the rule that `helm test` runs only after a readiness gate, and that a chart with no test hook yields NOT-RUN rather than a silent pass. |
| https://github.com/helm/chart-testing | `ct lint`, `ct install`, `ct lint-and-install`, `ct list-changed`; `--charts`, `--chart-dirs`, `--target-branch`, `--build-id`; requires Git ≥ 2.17 because it diffs against a target branch. | **ADAPT** — `ct` is offered as optional, and explicitly **skipped** when the chart is not in a git worktree with a target branch, instead of being run blind. |
| https://github.com/helm-unittest/helm-unittest | Helm plugin; `helm plugin install https://github.com/helm-unittest/helm-unittest.git`; suites live in `tests/*_test.yaml`; schema `suite`/`templates`/`tests[].it`/`set`/`asserts` (`equal`, `matchRegex`, `isKind`, `failedTemplate`); run with `helm unittest <chart>`. | **KEEP** — used as an optional static layer; the skill only runs it if suites already exist, and never authors suites into the chart. |
| https://github.com/yannh/kubeconform | `helm template … \| kubeconform -strict -summary -kubernetes-version X.Y.Z -ignore-missing-schemas`; repeatable `-schema-location` for CRDs. | **KEEP** — exact static-validation pipe in `references/chart-static-checks.md`. |
| https://alexandre-vazquez.com/helm-chart-testing-best-practices/ | Layered model: lint → unit → schema/policy → install → in-cluster test. | **ADAPT** — the layering justifies running static checks before touching a kube API. Ordering is reused; wording is not. |
| https://oneuptime.com/blog/post/2026-01-17-helm-chart-testing-ct-helm-test/view | Practitioner walkthrough of `ct` plus `helm test` in CI. | **DROP** — no fact needed beyond the upstream docs already cited. |

## E. Community indexes and overlapping DevOps skills

| Source | What it is for | Decision |
|---|---|---|
| https://skills.sh/ | Open agent-skill index; browsable by topic including Testing; ranked by installs. Searched for HugeGraph, Helm-chart-testing and Kubernetes-testing skills. | **KEEP as a negative result** — no HugeGraph skill and no Helm/Kubernetes chart-testing skill in the ranked listing; testing entries there are application-level (`test-driven-development`, `webapp-testing`). Confirms this skill is not duplicating an existing published one. |
| https://github.com/BagelHole/DevOps-Security-Agent-Skills | 80+ DevOps/security skills across Kubernetes, Terraform, cloud, container hardening, incident response, with scripts and playbooks. | **DROP for content, KEEP for scope check** — broad-knowledge base, no chart-under-test workflow; overlaps only in generic kubectl usage. |
| https://github.com/chaterm/terminal-skills | Terminal/Kubernetes/DevOps skill collection. | **DROP** — command-recall oriented, no test method. |
| https://jeffallan.github.io/claude-skills/skills/infrastructure/kubernetes-specialist/ | Kubernetes specialist skill listing. | **DROP** — advisory/manifest-authoring, not chart validation. |
| https://mcp.directory/skills/k8s-helm | Index entry for a `k8s-helm` skill about progressive delivery (canary, blue-green, rollouts). | **DROP** — release-strategy scope, not chart testing. Recorded so the overlap check is complete. |

## F. HugeGraph ground truth (fetched from apache/hugegraph@master)

| Source | Facts used | Decision |
|---|---|---|
| https://raw.githubusercontent.com/apache/hugegraph/master/docker/docker-compose-3pd-3store-3server.yml | The reference 3 PD + 3 Store + 3 Server topology. PD: gRPC 8686, REST 8620, raft 8610, env `HG_PD_GRPC_HOST`, `HG_PD_GRPC_PORT`, `HG_PD_REST_PORT`, `HG_PD_RAFT_ADDRESS`, `HG_PD_RAFT_PEERS_LIST`, `HG_PD_INITIAL_STORE_LIST`, `HG_PD_INITIAL_STORE_COUNT`, `HG_PD_DATA_PATH`. Store: gRPC 8500, raft 8510, REST 8520, env `HG_STORE_PD_ADDRESS`, `HG_STORE_GRPC_HOST`, `HG_STORE_RAFT_ADDRESS`, `HG_STORE_DATA_PATH`. Server: 8080, env `HG_SERVER_PD_PEERS`, `HG_SERVER_BACKEND: hstore`, `STORE_REST`. Health probes: PD `GET :8620/v1/health`, Store `GET :8520/v1/health`, Server `GET :8080/versions`. | **KEEP** — this is the topology and endpoint truth the oracles read. Note it is the *compose* reference, so the skill treats it as expected shape and still reads the chart's own values for key names. |
| https://raw.githubusercontent.com/apache/hugegraph/master/docker/README.md | Auth is enabled by setting `PASSWORD`, which the entrypoint maps to `auth.admin_pa` before init-store runs; default admin user is `admin`; `HG_SERVER_AUTH_TOKEN_SECRET` is the JWT secret and must be ≥ 32 bytes when set explicitly; `HG_SERVER_INIT_STORE_ENABLED=false` in distributed deployments. | **KEEP** — grounds the auth-on default and the auth-enforcement scenario. |
| https://raw.githubusercontent.com/apache/hugegraph/master/docker/hugegraph-hubble.properties | Hubble listens on 8088; keys `pd.peers` (gRPC), `pd.server` (PD REST), `server.direct_url`, `operations.store.allowed_targets`. | **KEEP** — grounds the Hubble wiring check and the Hubble login smoke step. |
| https://raw.githubusercontent.com/apache/hugegraph/master/hugegraph-pd/hg-pd-service/src/main/java/org/apache/hugegraph/pd/rest/MemberAPI.java | `GET /v1/members` on the PD REST port returns `{state, pdList[], pdLeader, numOfService, numOfNormalService, stateCountMap}`; each member carries `raftUrl`, `grpcUrl`, `restUrl`, `state`, `dataPath`, `role`, `replicateState`. | **KEEP** — this is the oracle for PD membership, leadership and, critically, whether member identity is an address that changes with the pod IP. |
| Same file plus StoreAPI.java, mutating endpoints | `POST /v1/members/change` (MemberAPI), `POST /v1/store/{storeId}` and `POST /v1/store/log` (StoreAPI) are the write endpoints on the PD management port. | **KEEP** — this is the provenance for the "GET only" boundary. The skill names them so an agent knows what it must not call; it never calls them. |
| Same file, `POST /v1/members/change` | Carries an in-repo comment stating the endpoint has no authentication check, so any caller with network access to the management port can change the PD peer list. | **KEEP** — grounds the management-surface boundary scenario. The skill only *probes reachability*; it never calls the mutating endpoint. |
| https://raw.githubusercontent.com/apache/hugegraph/master/hugegraph-pd/hg-pd-service/src/main/java/org/apache/hugegraph/pd/rest/StoreAPI.java | PD REST also exposes `GET /v1/stores`, `GET /v1/shardGroups`, `GET /v1/shardLeaders`, `GET /v1/storesAndStats`, `GET /v1/health`. | **KEEP** — store-registration and shard-leader oracles. |
| https://raw.githubusercontent.com/apache/hugegraph/master/hugegraph-server/hugegraph-dist/docker/docker-entrypoint.sh | Fetched to verify the auth property name during review round 1. Confirms the entrypoint sets the property `auth.admin_pa` from `PASSWORD`, sets `auth.token_secret` from `HG_SERVER_AUTH_TOKEN_SECRET`, and runs `enable-auth.sh` before init-store. | **KEEP** — resolves a reviewer challenge that `auth.admin_pa` looked truncated; the property name is real and is now cited rather than assumed. |
| https://hugegraph.apache.org/docs/quickstart/hugegraph-server/ | Server REST shape: `/graphs`, `/graphs/{graph}/schema/propertykeys`, `/graphs/{graph}/schema/vertexlabels`, `/graphs/{graph}/graph/vertices`, `/graphs/{graph}/gremlin`; auth accepts Basic and Bearer; Swagger UI at `/swagger-ui/index.html`. | **KEEP** — the CRUD smoke path and the auth negative control. |
| https://raw.githubusercontent.com/apache/hugegraph/master/hugegraph-server/hugegraph-api/src/main/java/org/apache/hugegraph/api/graph/VertexAPI.java | Fetched during review round 3 to ground the CRUD read path. Establishes `@Path("{id}")` under the vertices resource (so `GET …/graph/vertices/{id}` is real), and that `checkAndParseVertexId` runs the id through `JsonUtil.fromJson` — which is why a string id must be JSON-quoted and URL-encoded as `%22ktest-1%22`. Also shows the resource annotated `graphspaces/{graphspace}/graphs/{graph}/graph/vertices` on `master`, while the published docs show `/graphs/{graph}/…`. | **KEEP, and it changed the design** — the base path is version-dependent, so the skill records `API_BASE` as a discovered chart-contract field instead of hardcoding either shape. |
| https://hugegraph.apache.org/docs/quickstart/hugegraph-hubble/ | Hubble is the toolchain dashboard component. | **KEEP (weak)** — only used to confirm Hubble's role; port comes from the properties file above. |

## G. Skill-authoring rules

| Source | What it is for | Decision |
|---|---|---|
| https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices | `name` ≤ 64 chars, lowercase/digits/hyphens, no reserved words; `description` ≤ 1024 chars, third person, must state **what** and **when**; keep SKILL.md body under 500 lines; progressive disclosure with references **one level deep** from SKILL.md; table of contents in reference files over 100 lines; forward-slash paths; give one default rather than a menu of options; avoid time-sensitive statements. | **KEEP** — every rule is applied. SKILL.md is a navigation layer; all depth sits in one-level `references/`. |
| https://code.claude.com/docs/en/skills | Personal skills live at `~/.claude/skills/<skill-name>/SKILL.md`; for personal skills the **directory name** is the invocation name and frontmatter `name` is the display label; allowed frontmatter keys include `name`, `description`, `allowed-tools`, `disallowed-tools`, `disable-model-invocation`, `license`, `compatibility`, `metadata`; `disable-model-invocation: true` stops Claude loading the skill on its own; the `synced` folder name is reserved. | **KEEP** — folder name equals the YAML `name` (`hugegraph-helm-ktest`), and `disable-model-invocation` is intentionally omitted so the skill stays model-invocable. Uploads/packaging accept the `name`/`description`/`license`/`compatibility`/`metadata`/`allowed-tools` subset, which is why the frontmatter here stays inside that subset. |

## H. Verified negative findings

These were checked live and changed the design:

1. **There is no `helm/` directory in `apache/hugegraph@master`.** The full recursive tree was fetched and contains `docker/`, `hugegraph-pd/`, `hugegraph-store/`, `hugegraph-server/`, `hugegraph-commons/`, `hugegraph-struct/`, `hugegraph-cluster-test/`, `install-dist/` — no chart.
   *Consequence:* the skill must **discover** `helm/hugegraph` in whatever repo it is pointed at and **halt** if it is absent. It must never assume chart contents or invent `--set` paths.
2. **No `hugegraph` chart is published on ArtifactHub** (`/api/v1/packages/search?ts_query_web=hugegraph` returns an empty package list).
   *Consequence:* no upstream values contract to lean on; values keys must be read from the chart in front of the agent.
3. **PD member identity is address-shaped** (`raftUrl`/`grpcUrl` in `GET /v1/members`), and the compose reference wires raft peers as `host:port` lists.
   *Consequence:* "does PD identity survive a pod IP change" is a genuine risk for a Kubernetes chart, not a synthetic scenario — it is the difference between wiring peers to StatefulSet DNS names and wiring them to pod IPs.
4. **skills.sh lists no HugeGraph, Helm-chart-testing or Kubernetes-chart-testing skill** in its ranked index.
   *Consequence:* nothing to fork; this skill is authored fresh.

## I. Review history

This skill was reviewed by three independent read-only reviewers before release —
test-method rigour, scope/authoring boundaries, and safety. Round 1 returned 65 findings
(18 high-severity). The design changes that came out of it, recorded here because they are
not obvious from the finished text:

- The cluster-ownership gate was rewritten from a `kind-` **name** check to a three-part
  **evidence** check (loopback API server, `kind://` providerIDs, `kind get clusters`),
  because a context name can be renamed onto a production cluster.
- Cluster, namespace, and release names are timestamped, and teardown verifies a session
  label, so a run can never adopt or delete objects left by an earlier run.
- C4's cordon-based escalation was **removed**. Two reviewers converged on it: Kind's
  node-bound local-path PVs make a cordoned-node reschedule hang as `Pending` rather than
  relocate, and an aborted run would leave the node unschedulable. C4 now escalates by
  repeating the plain delete, which Kind's host-local IPAM rotates.
- `PASS-smoke` no longer accepts generic fault evidence, `DONE` now requires a
  `PASS-hardening`, and every scenario carries a declared negative control — three
  loopholes by which a smoke run could have been written up as a suite pass.
- The static layer got its own verdicts (`PASS-static` / `FAIL-static`) instead of
  borrowing scenario states it did not fit.

## J. What was deliberately NOT used

- `~/.claude/skills/` contained only `goal-prompt` and `impeccable` when this skill was authored — no prior HugeGraph or Kubernetes test skill existed there to retire or to copy.
- Two unrelated HugeGraph skill folders exist elsewhere on this machine, outside `~/.claude/skills/`, under a separate agent's tree. They are **out of scope**, were **not opened**, and no text from them is present here. If a HugeGraph Kubernetes test skill is ever added under `~/.claude/skills/`, retire it by moving it aside rather than merging it into this one.
