# Research artifacts

Files fetched from upstream during authoring and kept as ground truth. Full provenance,
including every source that was read and the keep / adapt / drop decision for each, is in
`../skill/hugegraph-helm-ktest/references/sources.md`.

| File | Source | Why it is here |
|---|---|---|
| `upstream-3pd-3store-3server.compose.yml` | `apache/hugegraph@master:docker/docker-compose-3pd-3store-3server.yml` | The reference 3 PD + 3 Store + 3 Server topology. Origin of the port map (PD 8686/8620/8610, Store 8500/8520/8510, Server 8080), the health endpoints, and the `HG_PD_*` / `HG_STORE_*` / `HG_SERVER_*` environment variable names the skill looks for when reading how a chart wires identity. |

Not kept: the full recursive tree listing of `apache/hugegraph@master` (~1.1 MB, trivially
regenerable via the GitHub trees API, and its only load-bearing conclusion — that the repo
contains no `helm/` directory — is recorded in `sources.md` and `HANDOFF.md`).

These are upstream Apache HugeGraph files, under the Apache License 2.0, retained
unmodified for reference.
