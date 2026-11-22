# Configuration

Every service reads environment variables. The Helm chart sets them from
`values.yaml`; Docker Compose sets them in `deploy/compose/docker-compose.yml`.

## Common (all services)

| Variable | Default | Meaning |
|---|---|---|
| `GJALLAR_NATS_URL` | `nats://localhost:4222` | NATS server (or local leaf node) |
| `GJALLAR_NATS_USER` | — | NATS user (chart: `gjallar` for services, `agent` for agents) |
| `GJALLAR_NATS_PASSWORD` | — | password for `GJALLAR_NATS_USER` (falls back to `GJALLAR_NATS_TOKEN`) |
| `GJALLAR_NATS_TOKEN` | — | NATS token, or the services' password when a user is set (the chart generates Secret `<release>-nats`) |
| `GJALLAR_NATS_CA` | — | CA file for a `tls://` NATS URL |
| `GJALLAR_NATS_CERT`, `GJALLAR_NATS_KEY` | — | client certificate for mutual TLS |
| `GJALLAR_HTTP_ADDR` | per service | Listen address |
| `GJALLAR_LOG_LEVEL` | `info` | `debug`, `info`, `warn`, `error` (JSON logs to stderr) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | — | OTLP/HTTP base URL; traces go to `/v1/traces` (Collector or Tempo) |
| `OTEL_SERVICE_NAME` | `gjallar` | Service name on spans |

## gjallar-api (`:8080`)

| Variable | Default | Meaning |
|---|---|---|
| `GJALLAR_DATABASE_URL` | — | Postgres DSN. **Empty = in-memory store (dev only)** |
| `GJALLAR_OIDC_ISSUER` | — | Keycloak realm URL. **Empty = auth disabled (dev only)** |
| `GJALLAR_OIDC_CLIENT_ID` | `gjallar` | Expected audience / `azp` |
| `GJALLAR_OIDC_ADMIN_ROLE` | `gjallar-admin` | Role required for every write |
| `GJALLAR_OIDC_VIEWER_ROLE` | — | Role required for reads (empty = any signed-in user) |
| `GJALLAR_ADMISSION_OPEN` | `false` | Serve `/api/v1/admission` without a token (for Kyverno; protect it with a NetworkPolicy) |
| `GJALLAR_ANALYTICS_URL` | — | Analytics service; proxied under `/api/v1/analytics/` |
| `GJALLAR_RUNTIME_ENABLED` | `true` | Read runtime signals from the `gjallar-runtime` KV bucket |
| `GJALLAR_TENANTS` | — | JSON `{"keycloak-group": ["namespace", ...]}`; enables multi-tenancy |
| `GJALLAR_TENANCY_GLOBAL_ROLES` | admin role | Comma-separated roles that see every namespace |
| `GJALLAR_NOTIFY` | — | JSON list of webhook targets (`name, url, format slack\|teams\|json, namespaces, min_band, kev`) |
| `GJALLAR_PUBLIC_URL` | — | UI base URL used for links in notifications |
| `GJALLAR_HARBOR_URL`, `_PROJECTS`, `_USER`, `_PASSWORD` | — | Registry discovery in Harbor (robot account with pull + list) |
| `GJALLAR_HARBOR_MAX_PER_REPO` | `20` | Newest tagged artifacts per repository |
| `GJALLAR_REGISTRY_URL`, `_REPOSITORIES`, `_USER`, `_PASSWORD`, `_MAX_TAGS` | —, all, —, —, `20` | Discovery in any OCI Distribution registry (Bearer token challenge supported) |
| `GJALLAR_DISCOVERY_INTERVAL` | `1h` | How often registries are listed |
| `GJALLAR_DTRACK_URL`, `GJALLAR_DTRACK_API_KEY` | — | Opt-in: upload each new SBOM (CycloneDX 1.6) to Dependency-Track |

## gjallar-worker (`:9090` health/metrics)

`gjallar-worker sbom|vuln|license|patch`

| Variable | Role | Default | Meaning |
|---|---|---|---|
| `GJALLAR_CONCURRENCY` | all | `2` | Jobs handled at once per pod |
| `GJALLAR_MAX_DELIVER` | all | `4` | Attempts before a job is given up |
| `GJALLAR_SYFT_SOURCE` | sbom | `registry` | `registry`, `docker`, `podman`, `containerd`, `oci-archive`... |
| `GJALLAR_SYFT_SCOPE` | sbom | squashed | `all-layers` also catalogs files deleted in later layers |
| `DOCKER_CONFIG` | sbom, patch | — | Folder with `config.json` for private registries |
| `GRYPE_DB_UPDATE_URL` | vuln | Anchore CDN | Your mirror, e.g. `http://gjallar-grype-db:8080/databases` |
| `GJALLAR_LICENSE_POLICY` | license | built-in | JSON file `{"allow":[...],"deny":[...]}` (SPDX IDs, `*` suffix glob) |
| `GJALLAR_GRANT_CONFIG` | license | — | `.grant.yaml`; when set, Grant runs as a second opinion |
| `GJALLAR_BUILDKIT_ADDR` | patch | — | e.g. `tcp://gjallar-buildkit:1234` (required) |
| `GJALLAR_COPA_SCANNER` | patch | `trivy` | Report format Copa reads. `grype` needs a `copa-grype` plugin |
| `GJALLAR_COPA_TAG_SUFFIX` | patch | `-patched` | New tag = old tag + suffix |
| `GJALLAR_COPA_PUSH` | patch | `true` | Push the patched image |
| `GJALLAR_MATCHER` | vuln | `grype` | `grype` (EPSS/KEV, DB mirror) or `trivy` (reads the CycloneDX copy) |
| `GJALLAR_OSV`, `GJALLAR_OSV_CACHE` | vuln | `false`, `/tmp/osv-db` | Offline OSV-Scanner as a second source (downloads the OSV DB; never sends your packages) |
| `TRIVY_DB_REPOSITORY` | vuln, patch | Trivy default | Internal Trivy DB mirror |
| `GJALLAR_COPA_TARGET_REPO` | patch | — | Push patched images here (e.g. `harbor.example.com/gjallar`); required for public source images |
| `GJALLAR_COPA_TIMEOUT` | patch | `20m` | `copa --timeout` (Copa's own 5 m fails on images with many OS updates) |
| `GJALLAR_ATTEST_REPOS` | sbom | — | Comma-separated repository prefixes whose SBOMs get a cosign attestation |
| `GJALLAR_COSIGN_KEY` | sbom | `env://COSIGN_PRIVATE_KEY` | cosign key reference (with `COSIGN_PASSWORD`) |
| `GJALLAR_COSIGN_INSECURE` | sbom | `false` | Plain-HTTP registries (labs only) |

## gjallar-agent (`:9090` metrics)

| Variable | Default | Meaning |
|---|---|---|
| `GJALLAR_CLUSTER` | `default` | Cluster name in workload IDs |
| `GJALLAR_INTERVAL` | `5m` | How often to publish inventory |
| `GJALLAR_SKIP_NAMESPACES` | `kube-system` | Comma-separated |
| `GJALLAR_RUNTIME_GADGET` | `false` | `true` = stream Inspektor Gadget `trace_open` and record loaded packages |
| `GJALLAR_GADGET_BIN` | `kubectl-gadget` | Shipped in the agent image |
| `GJALLAR_GADGET_IMAGE` | `trace_open:v0.58.0` | Keep in step with the deployed Inspektor Gadget |
| `GJALLAR_RUNTIME_FLUSH` | `1m` | How often observed opens become KV updates |
| `GJALLAR_REGOLIBRARY` | — | Folder with Kubescape regolibrary `rules`, `controls`, `default_config_inputs` (shipped at `/usr/local/share/regolibrary`) |
| `GJALLAR_OPA_BIN` | `opa` | OPA CLI used for regolibrary |

Every service retries its first NATS connection for up to 5 minutes before
exiting, so a cold start (NATS still pulling its image) doesn't crash-loop.

## gjallar-analytics (`:8081`)

`gjallar-analytics serve` (writer + query API) or `gjallar-analytics maintain` (CronJob).

| Variable | Default | Meaning |
|---|---|---|
| `GJALLAR_LAKE_MODE` | `parquet` | `parquet` or `ducklake` |
| `GJALLAR_LAKE_DATA` | `/data/lake/` | Folder or `s3://bucket/prefix/` |
| `GJALLAR_LAKE_CATALOG` | — | DuckLake catalog, e.g. `postgres:dbname=gjallar host=pg user=… password=…` |
| `GJALLAR_LAKE_FLUSH` | `30s` | How often batches become Parquet |
| `GJALLAR_LAKE_RETAIN` | `720h` | DuckLake snapshot retention (`maintain`) |
| `GJALLAR_S3_ENDPOINT` / `_REGION` / `_ACCESS_KEY` / `_SECRET_KEY` / `_USE_SSL` | — | S3-compatible storage (Ceph RGW, MinIO, SeaweedFS) |
| `GJALLAR_DUCKDB_BIN` | `duckdb` | Path to the DuckDB CLI |
| `GJALLAR_FLIGHT_ADDR` | `:8815` | Arrow Flight SQL listener; `off` disables it |
| `GJALLAR_OIDC_ISSUER`, `GJALLAR_OIDC_CLIENT_ID` | — | Flight SQL bearer-token checks (empty = unauthenticated, dev only) |
| `GJALLAR_FLIGHT_ROLES` | admin role | Roles allowed to use Flight SQL |

## gjallar-web (`:8083`) and gjallar-mcp (`:8082`)

| Variable | Service | Meaning |
|---|---|---|
| `GJALLAR_API_URL` | both | e.g. `http://gjallar-api:8080` |
| `GJALLAR_OIDC_ISSUER` | both | Enables the login (web) or required bearer tokens (MCP) |
| `GJALLAR_OIDC_WEB_CLIENT_ID` | web | Public PKCE client, default `gjallar-web` |
| `GJALLAR_MCP_PUBLIC_URL` | mcp | Advertised in `/.well-known/oauth-protected-resource` |

## CLI

| Variable | Meaning |
|---|---|
| `GJALLAR_URL` | Hub API (default `http://localhost:8080`) |
| `GJALLAR_TOKEN` | Bearer token. Overrides the token cached by `gjallar login` |
| `GJALLAR_OIDC_ISSUER`, `GJALLAR_OIDC_CLI_CLIENT_ID` | Used by `gjallar login` (client default `gjallar-cli`) |

## Build and test (Makefile / scripts)

| Variable | Default | Meaning |
|---|---|---|
| `BUILDER` | `docker` | `nerdctl` builds and runs compose on containerd (needs `buildkitd`) |
| `CTR_NAMESPACE` | `default` | containerd namespace for `BUILDER=nerdctl` (`k8s.io` = a local node's image store) |
| `NERDCTL` | `sudo nerdctl --namespace $(CTR_NAMESPACE)` | Override if your containerd is rootless |
| `GJALLAR_E2E_PORT` | `18080` | Base port for the four e2e services (base … base+3) |
| `KIND_CLUSTER`, `REGISTRY`, `VERSION` | `gjallar`, `ghcr.io/heimdallr-dev`, `0.1.0` | `scripts/kind-dev.sh` |

## Helm values worth knowing

Upgrading to a newer chart? Use `helm upgrade --reset-then-reuse-values`
(Helm ≥ 3.14), not `--reuse-values`: the latter ignores new chart defaults,
so new settings render empty.


| Value | Default | Notes |
|---|---|---|
| `postgresql.enabled` | `true` | Simple StatefulSet. For production use CloudNativePG and `postgresql.existingSecret` with key `DATABASE_URL` |
| `nats.enabled` | `true` | Single node. For a 3-node JetStream cluster use the official NATS chart and `nats.externalUrl` |
| `grypeDB.enabled` | `true` | Mirrors Anchore's daily DB inside the cluster (Kubescape's offline-DB image) |
| `workers.patch.enabled` | `false` | Also deploys rootless BuildKit |
| `analytics.mode` | `parquet` | `ducklake` needs `analytics.s3.dataPath` |
| `keda.enabled` | `false` | Workers scale 0→N on JetStream consumer lag; one queued job wakes a worker within seconds |
| `agent.runtime.enabled` | `false` | Runtime reachability via Inspektor Gadget (install it first: `kubectl gadget deploy`). Adds DaemonSet read and `pods/portforward` in `agent.runtime.gadgetNamespace` |
| `auth.issuer` | `""` | Keycloak realm URL; empty disables auth (dev only) |
| `tenancy.groups`, `tenancy.globalRoles` | `{}`, `[gjallar-admin]` | Multi-tenancy |
| `notify.existingSecret` | `""` | Secret with key `GJALLAR_NOTIFY` (webhook URLs are credentials) |
| `discovery.harbor.*` | off | Harbor URL, projects, robot Secret (`username`, `password`) |
| `discovery.registry.*` | off | Any OCI registry: `url`, `repositories`, `maxTags`, `existingSecret` |
| `nats.auth` | `users` | `users`: services and agents get separate NATS users, agents can only report; `token`: one shared token (older releases, external NATS) |
| `nats.tokenSecret` | generated | Secret with keys `token` (services) and `agent-password` (agents); spokes copy only `agent-password` |
| `nats.caSecret` | — | Secret with `ca.crt`, mounted into the agent for a `tls://` hub |
| `workers.patch.trivyDBRepository` | `""` | Internal Trivy DB mirror (air-gapped) |
| `workers.patch.targetRepository`, `.timeout` | `""`, `20m` | Where Copa pushes; Copa timeout |
| `workers.vuln.matcher`, `workers.vuln.osv` | `grype`, `false` | Primary matcher; offline OSV second source |
| `export.dependencyTrack.*` | off | `url`, `existingSecret` (key `api-key`) |
| `workers.sbom.attest.*` | off | `repositories`, `keySecret` (`cosign.key`, `cosign.password`), `insecure` |
| `buildkit.config` | `""` | `buildkitd.toml`, e.g. plain-HTTP or mirror registries |
| `agent.regolibrary.enabled` | `false` | Kubescape regolibrary controls via OPA |
| `web.replicas`, `mcp.replicas` | `2` | `gjallar-helm/charts/gjallar/values-kind.yaml` sets 1 for one-node clusters |
| `networkPolicy.enabled` | `false` | Recommended on Cilium or Calico |
| `cronjobs.*` | see `values.yaml` | DB refresh, analytics maintenance, backup, Vunnel |
