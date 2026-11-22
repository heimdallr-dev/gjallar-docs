# Verification report, 2026-11-22 (round 2)

Follows [round 1](verification-2026-11.md). Same WSL2 machine (11 GB RAM,
rootful containerd, kind), plus an on-premises S3 (Ceph RGW), Harbor and Keycloak.

## New features, and how each was verified

| Feature | Freedom | Verified by |
|---|---|---|
| Metering: engine runs/seconds, API use per OIDC client and user, usage per namespace; Grafana "Metering" row | 3, 9 | unit test on `/metrics` output; panels in the dashboard |
| Notifications (Slack, Teams Adaptive Card, JSON), per namespace, sent once across replicas (NATS KV ledger) | 3, 7 | unit tests; e2e: exactly one message for the urgent KEV finding, none after re-scoring |
| Multi-tenancy (Keycloak groups → namespaces), every read filtered; analytics fleet-wide only | 6 | unit test; **kind + Keycloak**: viewer sees `demo` only (238 findings, 3 images, analytics 403), admin sees all (10 652, 22, 200) |
| Registry discovery: Harbor API and **any OCI Distribution registry** (Bearer challenge) | 2 | unit tests (pagination, nested names, token flow); live `registry:2`; live Harbor listing |
| Signed SBOM attestations (cosign) for your own registries | 4, 8 | unit tests (args, prefix filter, signing failure doesn't fail the scan) |
| Arrow Flight SQL on the analytics pod, OIDC + role checked | 3, 4 | unit test with a real Flight SQL client; e2e against the real DuckDB lakehouse |
| Kubescape regolibrary controls (C-xxxx) via OPA in the agent | 1, 7 | real regolibrary v2 + OPA 1.4.2: 133 rules load, privileged pod fails C-0057; agent mapping test |
| Lakehouse tables `sbom_packages`, `risk_scores` (with inputs), `runtime_loaded`, `audit_events`; reports `runtime_effect`, `package_inventory` | 4 | e2e queries each table |
| SBOMs as CycloneDX and SPDX (`/api/v1/images/{digest}/sbom?format=`) | 4 | e2e downloads all three formats |
| Any OIDC provider: flat `roles`, groups as roles, username fallback | 6 | unit test with non-Keycloak token shapes |
| Audit log of every human decision | 8 | e2e: accept call recorded in `audit_events` |
| Analytics SQL sandbox (DuckDB `allowed_directories` + `lock_configuration`) | — security | unit test bypassing the keyword check; manual test on Ceph RGW: lake readable, other bucket and `/etc/passwd` refused |
| NATS token auth (generated Secret) | — security | real NATS 2.11: rejected without token, accepted with |
| Self-hosted Grype DB ([gjallar-vunnel](https://github.com/heimdallr-dev/gjallar-vunnel)) | 2, 5 | built image; provider `alpine` → DB v6.1.10 → Grype `db update` from a local HTTP server → status valid |

## Defects found and fixed this round

| Defect | Effect | Fix |
|---|---|---|
| Consumer `BackOff` replaced `AckWait` as redelivery timeout | **every job longer than 10 s ran again in parallel** (duplicate Syft/Grype/Copa) | BackOff removed; retries via `NakWithDelay`; e2e regression test |
| Worker consumers created only by a running worker | KEDA could never wake a worker that had never run (patch worker stuck at 0) | consumers created on bus connect |
| Web CSP `connect-src <issuer>` without trailing `/` | **browser login broken in every deployment** (discovery and token calls blocked) | prefix match + test |
| Copa default `--timeout 5m` | patching an image with ~40 OS updates failed | `workers.patch.timeout` (20m) |
| Copa pushed next to the source image | impossible for public images | `workers.patch.targetRepository` |
| Syft, Grype, Grant installed via `install.sh` from `main` | unpinned code in images | release tarballs verified against checksums |
| Engines' update checks on by default outside the chart | a hidden network call (CLI) | disabled in the shared runner |
| Vunnel CronJob ran `pip install vunnel` at runtime | unpinned code, internet needed at run | gjallar-vunnel image, pinned and verified |
| `helm upgrade --reuse-values` dropped new defaults | runtime RBAC in the wrong namespace | documented `--reset-then-reuse-values` |

## Gjallar's own dependencies (found by CI on first push)

`govulncheck` in CI found **55 reachable known vulnerabilities**: Go 1.24.7
(out of support) in `net/http`, `crypto/tls`, `crypto/x509`; brotli,
klauspost/compress, gRPC and pgx. Fixed by moving to **Go 1.26.11**
(checksum-verified), brotli 1.2.7, compress 1.20.2, gRPC 1.83.2, pgx 5.9.2,
and dropping the build sandbox's `replace` directives so `golang.org/x/*` and
`gopkg.in/yaml.v3` come from the official, checksum-database-verified modules.
Result: `govulncheck` reports **no vulnerabilities**. The upgrade also exposed
a race in the Flight SQL result cleanup (fixed: the file is unlinked as soon
as it is opened).

## Release v0.1.0 (published)

| Artifact | Where | Verified |
|---|---|---|
| 10 images (amd64 + arm64) | `ghcr.io/heimdallr-dev/gjallar-*:0.1.0` | pulled and run (`gjallar-cli` version, Grype 0.122.1 in `worker-vuln`); `gh attestation verify` → SLSA v1 provenance from `release.yml@refs/tags/v0.1.0` |
| CLI binaries (linux, darwin, windows) | GitHub release `v0.1.0` | `SHA256SUMS` matches; attestation verifies |
| Helm chart | `oci://ghcr.io/heimdallr-dev/charts/gjallar:0.1.0` | `helm show` / `helm template` from GHCR (33 objects) |
| DB builder | `ghcr.io/heimdallr-dev/gjallar-vunnel:gjallar-v0.68.0` | release workflow green |

Public Sigstore signatures are added automatically once the repositories are
public (the public Rekor log would otherwise expose private repository names).

## Round 3: tools, sources and registries (2026-11-22)

| What | Result |
|---|---|
| Syft sources | Docker Hub, Quay, GHCR, private plain-HTTP registry, `docker-archive` and `oci-archive` tarballs, and the **containerd** image store: same 14 packages for the same Alpine image everywhere |
| Grype (default) | nginx:1.27 → 790 findings, EPSS/KEV |
| **Trivy** as primary (`workers.vuln.matcher: trivy`) | 161 findings on the CycloneDX copy |
| **OSV** second source (offline) | 160 findings, merged with the primary (primary wins on the same package + CVE), each marked `source: osv` with a reason |
| VEX as signed attestation | cosign attest `--type openvex` to a registry, `verify-attestation` with the public key: statement `not_affected` verified |
| Dependency-Track export | adapter tested against the documented `PUT /api/v1/bom` contract; live server not started (needs ≥ 5 GB RAM on a host already running other services) |

Interoperability problems found and handled in the adapters:

| Problem | Effect | Handling |
|---|---|---|
| Syft writes `distro=debian-12.11` in purls; OSV indexes `Debian:12` | **every Debian package matched nothing** in OSV (silent) | OSV adapter normalises Debian to the major and Alpine to major.minor |
| Syft 1.57 writes CycloneDX **1.7**; Trivy 0.69 reads ≤ 1.6 | Trivy refused the SBOM | the stored CycloneDX copy is 1.6 (widest support, incl. Dependency-Track) |
| Trivy reports nothing from Syft's SPDX | silent zero | Trivy always reads CycloneDX |

## Environment findings

| Finding | Note |
|---|---|
| Harbor blob uploads timed out | The registry's storage backend was degraded at the time; Copa → Harbor push waits on a healthy registry |
| WSL restarted three times under the full stack | likely memory (11 GB for kind + Kyverno + KEDA + Keycloak + IG + Gjallar); give WSL more memory/swap in `.wslconfig` |
| kind (nerdctl provider) can't delete a never-started node | `kind-dev.sh` removes the container first |

## Round 4: live checks without a cluster (2026-11-22, 0.3.0)

After WSL restarted under the full kind stack, every check in this round ran
as a few small containers with memory limits instead of a cluster.

| What | How | Result |
|---|---|---|
| **Dependency-Track** live | DT 4.13.2 (`-Xmx4g`, container capped at 3 GB), Automation team key, `go test ./internal/adapters/dtrack -run Live` with Syft's nginx:1.27 CycloneDX | first upload **rejected (37 schema errors)**; after the fix below: accepted, project auto-created, components listed |
| **Agent NATS user** | nats 2.11.6 with the chart's rendered `nats.conf`, `go test ./internal/adapters/natsbus -run AgentUser` | agent may publish inventory and runtime events, read/write the runtime KV and read SBOMs; **denied**: publishing jobs or results, writing SBOMs, deleting or purging streams; wrong password refused |
| **DuckLake on Ceph RGW** | analytics 0.2.0 + Postgres catalog + NATS, one `gjallar.results.scored` event | event queryable through the SQL API; Parquet files on `s3://gjallar-lake/verify-20261122/` after a flush (see below) |
| DuckLake schema evolution and flush | `go test ./internal/adapters/analytics` with DuckDB 2.0.1 | an "old" lake gains new columns on start; `Maintain` writes inlined rows to Parquet |
| **arm64 images** | QEMU user emulation (`tonistiigi/binfmt`), `--platform linux/arm64` | api and worker binaries start (0.2.0); Syft 1.57.2, Grype 0.122.1, Trivy 0.69.3, OSV-Scanner, cosign 2.4.1 run as aarch64; Syft scans alpine:3.22 |
| Public images and signatures | anonymous GHCR pulls; `cosign verify` / `verify-attestation --type cyclonedx` (keyless, public Rekor) | all 11 packages pull without login; api 0.2.0 signature and SBOM attestation verify |

Defects found and fixed:

| Problem | Effect | Fix |
|---|---|---|
| Dependency-Track 4.13 validates license IDs against its bundled SPDX list; Syft uses a newer one (`SMAIL-GPL`) | **the whole SBOM was rejected**, so nothing reached DT | the exporter retries once with only the IDs DT named moved to `license.name` (text kept) |
| DuckLake stores small batches *inline* in the catalog database | history stayed in Postgres, not in open Parquet on S3 (0 data files) | daily `Maintain` now calls `ducklake_flush_inlined_data` |
| `finding_events` had no `source` column | the lake could not tell Grype, OSV and Trivy findings apart | column added; existing lakes gain it on start (`ALTER TABLE … ADD COLUMN IF NOT EXISTS`) |
| One NATS token for every component | a leaked token from any agent (possibly in another cluster) could queue jobs, forge results or delete streams | separate users: services `gjallar`, agents `agent` with an allow-list (`nats.auth: users`, default) |
| `kind load` imports `--all-platforms` | multi-arch images pulled from GHCR failed to load into kind | `kind-dev.sh` imports only the host platform |
| Keycloak reports Ready before its admin API answers | e2e realm setup failed now and then | the setup retries the admin token |

## Round 5: real cluster deployment (2026-11-23, 0.3.1)

Installed with only the published chart (`helm install oci://ghcr.io/heimdallr-dev/charts/gjallar --version 0.3.1`)
on a 5-node Kubernetes 1.36 cluster (containerd 2.3, Ceph RBD volumes), small
profile, sign-in through the cluster's existing Keycloak 26.8.

| What | Result |
|---|---|
| Install | 11 pods Running on 3 nodes; NATS, Postgres and analytics volumes Bound on `ceph-rbd-infra-k8s` (new chart value `storageClass`) |
| Inventory | 86 workloads in 23 namespaces, 82 images |
| SBOM + match | 79 of 82 images within minutes; the other 3 are private images (needs `workers.registryAuthSecret`) |
| Noise reduction | **7,314 open findings → 11 urgent (all CISA KEV), 22 high**, 1,021 medium, 6,260 low |
| Sign-in | realm imported with `keycloak/import-realm.sh`; API 401 without a token, with a forged token and through the web proxy; web config and CSP point at the realm; Keycloak accepts the UI's PKCE (S256) authorize request for `http://localhost:8083/` and refuses other redirect URIs |
| Web UI move | UI and server now built in gjallar-web (standard library only, own tests); core no longer contains it |

Found and fixed: `make deploy` passed git's `v0.3.0` as the image tag (images
are `0.3.0`); clusters without a default StorageClass needed three settings
(now one).

## Still open

- **Copa push to Harbor**: Trivy found 72 fixable packages in alpine:3.20.0,
  but WSL restarted while BuildKit ran; needs a host with more memory.
- **gjallar-web 0.3.1 image**: the repo's workflow may not push to the
  `gjallar-web` package yet (GHCR: package settings → Manage Actions access →
  add `gjallar-web`); until then the chart runs `web.tag=0.3.0`, the same UI.
- **First browser sign-in** on the real cluster: left to the operator, since
  it replaces the temporary password.
- **Browser PKCE and device-flow pages** (Playwright script ready:
  `gjallar-e2e: browser/run-in-kind.sh`) and **multi-cluster**: both need a
  running cluster; on this 12 GB WSL use `LIGHT=1` in `kind/setup-all.sh`,
  or a bigger host.
- **NATS TLS in the chart**: clients support `tls://` with `GJALLAR_NATS_CA`
  (and mutual TLS); the bundled single-node NATS still listens in plain text
  inside the cluster. Use the official NATS chart with TLS for remote agents.
