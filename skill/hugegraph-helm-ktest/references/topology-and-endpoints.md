# HugeGraph topology, ports, endpoints, auth

## Contents
- Component roles and bring-up order
- Port and health map
- PD REST API (the membership oracle)
- Store REST API
- Server REST API and the CRUD smoke path
- Recording HTTP status codes
- Authentication and credential handling
- Hubble
- Reaching endpoints from outside the cluster
- Reading the chart instead of assuming

Every command in this file runs through the `kubectl` / `curl` CLI. Do not substitute a
Kubernetes or Kind MCP tool; every action must appear in the session log as a shell
command carrying `--context "$KTEST_CTX"`.

## Component roles and bring-up order

| Component | Role | Kubernetes shape to expect |
|---|---|---|
| PD (Placement Driver) | Raft-replicated metadata and membership: registers Stores, owns shard groups and shard leaders, elects a PD leader | StatefulSet + headless Service + PVC |
| Store | Raft-replicated data nodes; register themselves with PD | StatefulSet + headless Service + PVC |
| Server | Graph engine and REST/Gremlin API; talks to PD for placement and Store for data | Deployment or StatefulSet + Service |
| Hubble | Web dashboard; talks to PD REST, PD gRPC, and Server REST | Deployment + Service |

Order: **PD quorum → Store registration → Server serves → Hubble usable.** Server pods
restarting while PD is still electing is normal early behaviour. Diagnose with PD
`/v1/members` and `/v1/stores`, not by waiting longer.

## Port and health map

These ports come from the upstream 3 PD + 3 Store + 3 Server reference deployment. They
are the **expected shape**, not a contract: confirm every one against the ports this
chart's own Services and probes declare, and use the chart's values where they differ.

In P1, read the real ports out of the render and put them in `env.sh` as `PD_REST_PORT`,
`STORE_REST_PORT`, `SVR_PORT` and `HUBBLE_PORT`. Every command in this skill uses those
variables, so a chart that serves on different ports needs no edits anywhere:

```bash
grep -A6 -E '^kind: Service' "$KTEST_DIR/artifacts/render-3x3.yaml" | grep -nE 'name:|port:'
```

| Component | Port | Purpose | Health probe |
|---|---|---|---|
| PD | 8686 | gRPC (client + Store + Hubble `pd.peers`) | — |
| PD | 8620 | REST / management | `GET /v1/health` |
| PD | 8610 | raft peer traffic | — |
| Store | 8500 | gRPC | — |
| Store | 8520 | REST | `GET /v1/health` |
| Store | 8510 | raft peer traffic | — |
| Server | 8080 | REST + Gremlin | `GET /versions` |
| Hubble | 8088 | web UI / API | HTTP 200 on `/` |

Upstream environment variable names, useful when reading a rendered manifest to see how
the chart wires identity:

- PD: `HG_PD_GRPC_HOST`, `HG_PD_GRPC_PORT`, `HG_PD_REST_PORT`, `HG_PD_RAFT_ADDRESS`,
  `HG_PD_RAFT_PEERS_LIST`, `HG_PD_INITIAL_STORE_LIST`, `HG_PD_INITIAL_STORE_COUNT`,
  `HG_PD_DATA_PATH`
- Store: `HG_STORE_PD_ADDRESS`, `HG_STORE_GRPC_HOST`, `HG_STORE_GRPC_PORT`,
  `HG_STORE_REST_PORT`, `HG_STORE_RAFT_ADDRESS`, `HG_STORE_DATA_PATH`
- Server: `HG_SERVER_PD_PEERS`, `HG_SERVER_BACKEND` (`hstore` for distributed),
  `HG_SERVER_INIT_STORE_ENABLED` (`false` in distributed deployments),
  `HG_SERVER_AUTH_TOKEN_SECRET`, `PASSWORD`

The chart may name these differently or template them from values. **Read the rendered
manifest** to learn what it actually sets; do not assume these names are present.

## PD REST API (the membership oracle)

Against a PD pod on its REST port:

| Request | Returns | Used for |
|---|---|---|
| `GET /v1/health` | plain text health | readiness |
| `GET /v1/members` | `{state, pdList[], pdLeader, numOfService, numOfNormalService, stateCountMap}` | PD membership, leadership, member identity |
| `GET /v1/stores` | registered Stores | Store registration |
| `GET /v1/storesAndStats` | Stores with stats | Store health and partition detail |
| `GET /v1/shardGroups` | shard groups (and replica count) | data placement |
| `GET /v1/shardLeaders` | shard leaders | leadership distribution |

Each entry in `pdList` carries `raftUrl`, `grpcUrl`, `restUrl`, `state`, `dataPath`,
`role`, `replicateState`.

Two rules when reading this:

- Assert on **`numOfNormalService`** — the healthy count derived from PD's state counts.
  `numOfService` is the size of the member list, so it does not shrink when peers go
  unhealthy; treat it as a size cross-check only, and confirm both fields against a known
  3-PD baseline before relying on either.
- Query **all three** PD pods and require agreement. A single vantage cannot see
  split-brain, and `pdLeader` is a scalar so "exactly one leader" is trivially true in any
  one response.

**PD member identity is address-shaped** — that is why C4 exists: if `raftUrl`/`grpcUrl`
hold pod IPs, identity does not survive a reschedule; if they hold StatefulSet DNS names,
it does.

`POST /v1/members/change` mutates the PD peer list and, in the upstream source, carries no
authentication check. **Never call it**, and never send any other write to a management
port. C8 probes reachability with `GET` only.

## Store REST API

Store REST port: `GET /v1/health`. Additional status endpoints exist on the store node
controllers; read the rendered manifest and the pod's own logs rather than guessing paths.

## Server REST API and the CRUD smoke path

Base: the Server Service on its REST port, plus the **API base path recorded in P1 as
`API_BASE`**. The base path is version-dependent — some HugeGraph versions serve
`/graphs/{graph}/…`, while the current upstream `VertexAPI` is annotated
`graphspaces/{graphspace}/graphs/{graph}/graph/vertices`. Probe for it and record it; the
table below shows paths relative to `API_BASE`.

**Discover the graph name — do not assume it:**

```bash
G=$(curl -s -K "$CFG" "http://$SVR:${SVR_PORT}/graphs" | sed -n 's/.*"graphs":\["\([^"]*\)".*/\1/p')
echo "graph under test: $G"
```

| Step | Request |
|---|---|
| version | `GET /versions` |
| list graphs | `GET /graphs` |
| create property key | `POST /graphs/{graph}/schema/propertykeys` |
| create vertex label | `POST /graphs/{graph}/schema/vertexlabels` |
| create vertex | `POST /graphs/{graph}/graph/vertices` |
| read vertex | `GET /graphs/{graph}/graph/vertices/{id}` — the id is **parsed as JSON**, so a string id must carry literal double quotes, URL-encoded: `%22ktest-1%22` |
| gremlin | `POST /graphs/{graph}/gremlin` |
| API browser | `GET /swagger-ui/index.html` |

There are **two** recipes, and mixing them is the most likely way to leak the password.
`$CFG` must exist on the side that actually runs `curl`.

**Recipe A — in-cluster (preferred).** A probe pod that takes the password from the
chart's own Secret via `envFrom`, and builds the config file inside the pod. The credential
never crosses the host command line:

```bash
kubectl --context "$KTEST_CTX" -n "$KTEST_NS" exec "$PROBE" -- sh -c \
  'umask 077; printf "user = \"admin:%s\"\n" "$<PW_ENV>" > /tmp/c; CFG=/tmp/c; …' 
```

`<PW_ENV>` is the env var name the chart's Secret exposes, from the P1 mapping. Note
`printf '…%s…' "$VAR"` — never `printf "…$VAR…"`: a `%` or `\` in the password would be
interpreted as a format directive, and a `"` would break the config file.

**Recipe B — host side, via port-forward.** Uses `$KTEST_DIR/artifacts/curl.cfg`. **A host
path is invisible inside a pod**, so never pass it to `kubectl exec`; the repair an agent
reaches for is `-u admin:<password>`, which puts the credential in `argv`, in the
transcript, and in `ps`.

The block below is written for whichever side you chose; `$SVR` is the Server Service DNS
name from the P1 mapping, and `$CFG` is that side's config file. If the Server image has no
`curl`, use a `curlimages/curl` probe pod — that is harness, not a HugeGraph failure.

Schema element names carry `$KTEST_TS` because the re-run policy runs this a second time
against the same graph: re-creating an existing property key returns 400, and `curl -sf`
would turn that harness artefact into a failure of a **blocking** smoke arm.

```bash
H='Content-Type: application/json'; K="ktest_name_$KTEST_TS"; L="ktest_node_$KTEST_TS"
curl -sf -K "$CFG" "http://$SVR:${SVR_PORT}/versions"
curl -sf -K "$CFG" -H "$H" -X POST "http://$SVR:${SVR_PORT}/graphs/$G/schema/propertykeys" \
  -d "{\"name\":\"$K\",\"data_type\":\"TEXT\",\"cardinality\":\"SINGLE\"}"
curl -sf -K "$CFG" -H "$H" -X POST "http://$SVR:${SVR_PORT}/graphs/$G/schema/vertexlabels" \
  -d "{\"name\":\"$L\",\"id_strategy\":\"CUSTOMIZE_STRING\",\"properties\":[\"$K\"]}"
# If the Server rejects a schema payload, read the exact shape it wants from
# GET /swagger-ui/index.html rather than guessing another field name.
curl -sf -K "$CFG" -H "$H" -X POST "http://$SVR:${SVR_PORT}/graphs/$G/graph/vertices" \
  -d "{\"label\":\"$L\",\"id\":\"ktest-1\",\"properties\":{\"$K\":\"smoke\"}}"
curl -sf -K "$CFG" "http://$SVR:${SVR_PORT}/graphs/$G/graph/vertices/%22ktest-1%22"
# negative control: an id that was never written must NOT return the written vertex.
# Record this status during the baseline and use the observed value as the control.
curl -s -o /dev/null -w '%{http_code}\n' -K "$CFG" \
  "http://$SVR:${SVR_PORT}/graphs/$G/graph/vertices/%22ktest-never-written%22"
```

Assert the value; do not eyeball the response. The comparison **is** the check:

```bash
curl -sf -K "$CFG" "http://$SVR:${SVR_PORT}/graphs/$G/graph/vertices/%22ktest-1%22" \
  | grep -q '"smoke"' || { echo "CRUD read-back value mismatch"; exit 1; }
```

A 2xx on the write alone is not a round-trip. The `ktest_` prefix keeps test artifacts identifiable and
collision-free.

Throwaway probe pods carry the session label and `--rm` so they never linger:

```bash
kubectl --context "$KTEST_CTX" -n "$KTEST_NS" run "ktest-probe-$KTEST_TS-crud-$RANDOM" --rm -i \
  --restart=Never --labels="hugegraph-ktest/session=$KTEST_TS" \
  --image=curlimages/curl:<pinned> -- curl -s ...
```

Give each probe pod a purpose suffix **and** a discriminator (`-crud-$RANDOM`, `-c8-$RANDOM`).
Two probes sharing a name inside one session collide with `AlreadyExists` — and `--rm` does
not delete the pod if the client is interrupted, so the name can stay taken. The
label-based teardown sweeps whatever is left.

## Recording HTTP status codes

`curl -sf` exits non-zero and prints nothing on 401/403, so it cannot be used for C5 or
C8, whose oracles are defined on status codes. Use:

```bash
curl -s -o /dev/null -w '%{http_code}\n' "http://$SVR:${SVR_PORT}/graphs/$G/graph/vertices"
```

Record one status per arm, per endpoint, into an artifact.

## Authentication and credential handling

Auth is on by default in this workflow.

- Setting the admin password puts the Server in authenticated mode. Upstream, the
  container entrypoint takes `PASSWORD`, writes it to the verified property
  `auth.admin_pa` in `rest-server.properties`, and runs `enable-auth.sh` before
  init-store.
- Default admin user is `admin`.
- `auth.token_secret` (env `HG_SERVER_AUTH_TOKEN_SECRET`) is the JWT secret for REST and
  embedded Gremlin auth; if set explicitly it must be at least 32 bytes.
- The Server accepts HTTP **Basic** and **Bearer**.

Find the chart's own auth keys with `helm show values` before writing the values file.

**Credential handling rules:**

1. **Generate it once, into a file — never into a variable the agent retypes.** Shell state
   does not survive between blocks, so an agent that keeps `$HG_ADMIN_PASSWORD` "in mind"
   will re-export it as a literal at the top of the next block, putting the plaintext into
   `argv` and the transcript. Instead:

   ```bash
   umask 077
   LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom | head -c 32 > "$KTEST_DIR/artifacts/admin.pw"
   ```

   Restrict it to `[A-Za-z0-9]` so no character can break a curl config file. Read it only
   as `$(cat "$KTEST_DIR/artifacts/admin.pw")`, inside the same block that consumes it.
   **`env.sh` must never contain it.**
2. `values-3x3-auth.yaml` and every `render-*.yaml` are **credential-bearing** — the
   values file holds the password and the render holds the Secret, base64 is not
   protection. Enforce the mode rather than assuming it: run `umask 077` at the top of
   every block that writes into `$KTEST_DIR`, and `chmod 600` the values and render files
   explicitly after creating them. `$KTEST_DIR` itself is mode 700 and lives outside the
   repository.
   Never `git add` them, and never run `helm template … --debug` with the values file:
   `--debug` prints the computed values, password included, into the transcript.
3. Never run a command whose output is the decoded Secret value. If you must confirm a
   Secret matches, compare hashes. On the install-failure path, `kubectl describe pod`
   prints inline env values — if the P2 render review found the credential set inline
   rather than via `secretKeyRef`, redact it:
   `kubectl describe pod … | sed -E 's/(PASSWORD|auth[._]admin[._]pa)[:=].*/\1: [REDACTED]/I'`.
4. Never put the password in `argv`. Write a curl config file once, at mode 600, and pass
   it with `-K`:

   ```bash
   umask 077
   printf 'user = "admin:%s"\n' "$(cat "$KTEST_DIR/artifacts/admin.pw")" \
     > "$KTEST_DIR/artifacts/curl.cfg"
   chmod 600 "$KTEST_DIR/artifacts/curl.cfg"
   CFG="$KTEST_DIR/artifacts/curl.cfg"
   ```

   Read the password from its file, never from a variable such as `HG_ADMIN_PASSWORD` — an
   agent that keeps it "in mind" re-exports it as a literal in the next block, which puts
   the plaintext into the transcript. For the same reason, write it into
   `values-3x3-auth.yaml` with a shell `printf` from `admin.pw`, never with
   `helm --set <key>=<password>`: `--set` places the expanded password in helm's `argv`,
   visible in `ps` to every user on the machine.

   For probes that must run **inside** a pod, prefer a probe pod whose container takes the
   password from the chart's own Secret via `envFrom`, and build the config inside the pod
   so the credential never crosses the host command line:

   ```bash
   # <PW_ENV> is the env var name the chart's Secret exposes, from the P1 mapping.
   # %s formatting, and the URL passed as a positional arg, not interpolated on the host.
   kubectl --context "$KTEST_CTX" -n "$KTEST_NS" exec <probe-pod> -- sh -c \
     'umask 077; printf "user = \"admin:%s\"\n" "$<PW_ENV>" > /tmp/c; curl -sf -K /tmp/c "$1"' \
     _ "http://$SVR:${SVR_PORT}/graphs"
   ```

**Negative control for the auth claim:** the same request with no credentials, and with a
wrong password, must be rejected. A test that only sends valid credentials proves nothing
about enforcement. See C5.

## Hubble

Hubble reads, upstream: `pd.peers` (PD gRPC), `pd.server` (PD REST), `server.direct_url`
(Server REST), `operations.store.allowed_targets` (Store REST). Confirm the chart wires
all four to in-cluster Service DNS names — a Hubble pointed at a pod IP or at `localhost`
is a real chart defect even while the UI loads.

**The Hubble smoke arm is non-blocking.** This skill does not document Hubble's own
authentication API, and inventing one is worse than not running the check. Do this instead:

1. Confirm Hubble answers on its port: status 200 on `/`.
2. From **inside the Hubble pod**, confirm Hubble's configured Server URL accepts an
   authenticated request with the admin credential — that is the dependency the dashboard
   actually needs. Read the URL from the mounted properties file rather than assuming an
   env var exists, and build the credential file in-pod as in Recipe A:

   ```bash
   kubectl --context "$KTEST_CTX" -n "$KTEST_NS" exec <hubble-pod> -- sh -c '
     U=$(grep -m1 "^server.direct_url=" <hubble-properties-path> | cut -d= -f2-)
     umask 077; printf "user = \"admin:%s\"\n" "$<PW_ENV>" > /tmp/c
     curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 -K /tmp/c "$U/graphs"'
   ```

   If the image has no `curl`, or neither the properties file nor a credential is reachable
   from that pod, record `NOT-RUN — no HTTP client in the Hubble image` (harness, not a
   HugeGraph failure).

3. Otherwise record `NOT-RUN — no documented Hubble auth API` and continue. Do **not** call
   a rendered login page a login, and do not block the suite on this arm.

## Reaching endpoints from outside the cluster

Prefer running checks *inside* the cluster so you exercise the same Service DNS the
components use. When you need host access, record the PID so teardown can kill exactly
that process:

```bash
kubectl --context "$KTEST_CTX" -n "$KTEST_NS" port-forward svc/<server-svc> 18080:${SVR_PORT} &
printf '%s\tkubectl port-forward %s\n' "$!" "<server-svc>" >> "$KTEST_DIR/logs/portforward.pids"
```

Record the command alongside the PID. Teardown may run an hour later, by which time a
port-forward that died early has freed its PID for reuse by an unrelated process — so
teardown verifies `ps -p <pid> -o command=` still matches before killing.

Never `pkill -f kubectl` or `pkill -f port-forward` — that kills forwards the user is
running against unrelated clusters. A port-forward failure is `harness` blame, never a
HugeGraph failure.

## Reading the chart instead of assuming

Before P4, extract from `helm show values helm/hugegraph`:

- replica keys for PD, Store, Server
- the Hubble enable key
- the auth enable key and the admin password key
- image repository/tag keys (needed for `kind load docker-image`)
- persistence keys (storageClass, size) — Kind's default provisioner is
  `standard` / `rancher.io/local-path`
- service type and port keys

Write that mapping into the session log as a table. Any required key that does not exist
is reported as a gap, not worked around with an invented flag; a scenario that needs a
missing key is `NOT-RUN` with that reason.
