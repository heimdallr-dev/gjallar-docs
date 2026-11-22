# Operations

## Scheduled jobs

| CronJob | Default schedule | What it does | Needs |
|---|---|---|---|
| `gjallar-db-refresh` | `17 3 * * *` | Restarts the Grype DB mirror (pulls the newest DB image), waits for it, then publishes `db_updated`. The hub re-queues **every stored SBOM** to Grype; no images are pulled again | Role: `get`/`patch` on the mirror deployment only |
| `gjallar-analytics-maintain` | `41 2 * * *` | Writes today's `finding_daily` snapshot; in DuckLake mode also merges small files, expires snapshots and cleans up old files | Same volume as analytics (pod affinity in Parquet mode) |
| `gjallar-backup` | `23 1 * * *` (off) | `pg_dump` to a PVC, keeps N days | PVC |
| `gjallar-vunnel` | `7 1 * * *` (off) | Option B: build the Grype DB yourself with Vunnel + grype-db (needs ~100 GB, hours, an NVD API key) | RWX PVC |

Image discovery is not a CronJob: the agent publishes inventory every
`agent.interval` (default 5 minutes), and new digests are queued immediately.

Run any job on demand:

```bash
kubectl -n gjallar create job --from=cronjob/gjallar-db-refresh refresh-$(date +%s)
```

## How fresh is the vulnerability data?

- The mirror pod serves the DB the image was built with. `db-refresh`
  restarts it with `imagePullPolicy: Always`.
- Each image's `db_version` (UI → Images, or `GET /api/v1/images`) shows
  which DB build matched it.
- Grype refuses a DB older than 5 days by default. If `db-refresh` fails for
  days, vulnerability jobs start failing. Alert on the CronJob failing
  (kube-state-metrics: `kube_job_status_failed`).

Disk: each `worker-vuln` pod unpacks the DB (~3 GB) into its `/tmp`
emptyDir once and reuses it for every job; matches run with
`GRYPE_DB_AUTO_UPDATE=false` after a single, serialised `grype db update`.

## Scaling

| Component | How |
|---|---|
| API / hub | `api.replicas`. JetStream load-balances the durable consumers; Postgres claims are atomic |
| Workers | `workers.<role>.replicas` and `concurrency`, or `keda.enabled` for queue-depth autoscaling (0→N) |
| Analytics | One writer by design. Read throughput is DuckDB; for heavy notebooks, attach DuckLake directly (see analytics.md) |
| NATS | Single node by default; move to a 3-node cluster via the official chart for HA |
| Postgres | CloudNativePG with a replica for HA |

Rough sizing for 1,000 images: Syft about 2–20 s per image; Grype about
1–5 s per SBOM. Postgres stays under 1 GB. SBOMs take about 1–5 MB each in
the NATS object store (compressed).

## Multi-cluster

Run the hub in one cluster. In every other cluster install only the agent,
pointing at a **NATS leaf node** that dials out to the hub:

```yaml
# values for a spoke cluster
nats: { enabled: false, externalUrl: nats://local-leaf:4222 }
postgresql: { enabled: false }
grypeDB: { enabled: false }
analytics: { enabled: false }
web: { enabled: false }
mcp: { enabled: false }
workers: { sbom: { enabled: false }, vuln: { enabled: false }, license: { enabled: false } }
api: { replicas: 0 }
cluster: { name: dc-manila-01 }
```

Spokes then need no inbound firewall rules.

Give spokes the **agent** credentials only, never the services' token:

```sh
kubectl -n gjallar get secret gjallar-nats -o jsonpath='{.data.agent-password}' | base64 -d > agent.pw   # on the hub
kubectl -n gjallar create secret generic gjallar-nats --from-file=agent-password=agent.pw                 # on the spoke
```

The `agent` user may only report inventory and runtime data and read SBOMs,
so a leaked spoke credential cannot queue jobs, forge results or delete
streams. Upgrading from 0.2.x: update the hub first (it creates the
`agent-password` key), or keep `nats.auth: token` until every spoke runs 0.3.0.

## Backups and restore

- **Postgres** holds the operational state. Enable `cronjobs.backup`, or use
  CloudNativePG's barman backups to S3.
  Restore: `gunzip -c gjallar-<ts>.sql.gz | psql "$DATABASE_URL"`.
- **NATS**: SBOMs can be regenerated. If you lose the bucket, run
  `gjallar rematch`; images without an SBOM are re-scanned on the next
  inventory pass.
- **Lakehouse**: Parquet files are the history. Back up the PVC or bucket.
  DuckLake also needs its catalog tables, which live in the same Postgres.

## Upgrades

1. `make push VERSION=x.y.z`
2. `make deploy VERSION=x.y.z HELM_ARGS=...`

Always pass your values file. If you upgrade with ad-hoc `--set` flags, use
`--reset-then-reuse-values`, not `--reuse-values` (that one skips new chart
defaults and renders new settings empty).

The Postgres schema migrates itself on API start; it is idempotent
`CREATE … IF NOT EXISTS`. Workers drain gracefully: unacked jobs return to the
queue.

To upgrade an engine, change `SYFT_VERSION`, `GRYPE_VERSION`, `GRANT_VERSION`,
`COPA_VERSION`, `TRIVY_VERSION` or `DUCKDB_VERSION` in the Dockerfile and
rebuild that worker image only. **Check release notes and provenance before
bumping Trivy**: 0.69.4–0.69.6 were malicious (CVE-2026-33634).

## Observability

- **Metrics:** every pod serves `/metrics` (Prometheus text). Key series:
  `gjallar_open_findings{band}`, `gjallar_open_kev_findings`,
  `gjallar_messages_total{consumer,outcome}` (`outcome="dead"` means a job
  was given up), `gjallar_http_requests_total`, `gjallar_analytics_rows_total{table}`.
- **Dashboard:** the chart ships a Grafana dashboard ConfigMap
  (`grafana_dashboard: "1"`) for the Grafana sidecar.
- **Traces:** set `observability.otlpEndpoint`. One trace follows an image
  from discovery through every worker, because the trace context travels in
  NATS message headers.
- **Logs:** JSON on stderr with `trace_id`, ready for Loki.

Suggested alerts:

```promql
increase(gjallar_messages_total{outcome="dead"}[1h]) > 0          # jobs being dropped
max(gjallar_open_findings{band="urgent"}) > 0                     # something to act on
kube_cronjob_status_last_successful_time{cronjob="gjallar-db-refresh"} < time() - 2*86400
```

## Troubleshooting

| Symptom | Check |
|---|---|
| No images appear | `kubectl logs deploy/gjallar-agent`: RBAC errors? Pods with no digest yet (`imageID` empty) are skipped until they run |
| Images stuck "pending" | `kubectl logs deploy/gjallar-worker-sbom`: registry auth (`workers.registryAuthSecret`), network to the registry |
| Vuln jobs fail with DB errors | Is the mirror up? `kubectl exec deploy/gjallar-worker-vuln -- wget -qO- http://gjallar-grype-db:8080/databases/v6/latest.json` |
| `401` everywhere | Issuer URL must match the token's `iss` exactly (scheme, host, realm); each client that signs users in (`gjallar-web`, `-cli`, `-mcp`) needs the `gjallar-api-audience` mapper (tokens must carry `aud: gjallar`) |
| Analytics "No files found" | Fixed by `Init` seeding. If you wiped the volume, restart the analytics pod |
| Patch fails | BuildKit reachable? `GJALLAR_BUILDKIT_ADDR`; push rights in `registryAuthSecret`; Copa only patches OS packages |
| Kyverno blocks too much | Policy is `failurePolicy: Ignore` and `Audit` by default; check `kubectl get policyreports -A` before enforcing |
| `kubectl apply -f gjallar-helm/policies/kyverno/` says legacy `ClusterPolicy` is removed | Kyverno ≥ 1.20 only runs `policies.kyverno.io/v1 ValidatingPolicy`; the shipped policies use it (Kyverno ≥ 1.16 required) |
| Every Deployment change fails with `failed calling webhook "vpol.validate.kyverno.svc-fail-…"` | A Kyverno policy with `failurePolicy: Fail` while Kyverno is down. Ours are `Ignore`; check your own policies |
| Pods restart a few times right after install | Fixed: services retry NATS for 5 min. If it still happens, NATS took longer (slow image pull); check `kubectl -n gjallar describe pod gjallar-nats-0` |
| Vuln worker uses a lot of disk / first match is slow | The Grype DB unpacks to ~3 GB in the worker's `/tmp` (emptyDir, 10 Gi limit). It is downloaded once per pod and shared by all jobs |
| KEDA workers stay at 0 while jobs wait | Check `kubectl -n gjallar get scaledobject`; KEDA must reach `gjallar-nats:8222`. `activationLagThreshold` is 0 so one job wakes a worker |
| Tokens accepted but every write is 403 and `/api/v1/me` is empty | The realm lacks Keycloak's built-in `basic`/`roles` scopes (tokens have no `sub` or roles). Re-import `gjallar-helm/keycloak/gjallar-realm.json` (fixed 2026-11) |
| Runtime: no "package loaded at runtime" reasons | `kubectl -n gjallar logs deploy/gjallar-agent \| grep runtime`. Needs Inspektor Gadget running, matching `agent.runtime.gadget` version, and the image's SBOM. Scores update on the next inventory pass (≤ `agent.interval`) |
| Inspektor Gadget: `couldn't initialize inotify: too many open files` | Raise `fs.inotify.max_user_instances` (512) on the node; common on kind/WSL |

## Local dev environment on WSL (containerd, kind)

What needs attention after a **WSL restart** (`wsl --shutdown`, Windows reboot):

| Thing | What happens | Do |
|---|---|---|
| `buildkitd` | Not a service; it stops | `sudo buildkitd --oci-worker=false --containerd-worker=true &` before `make images BUILDER=nerdctl` |
| kind cluster | The node container stays `Created`, and when started it gets a new IP (containerd CNI hands out IPs round-robin), which a kubeadm control plane cannot survive | `scripts/kind-dev.sh up` (recreates the cluster and reinstalls, ~5 min) |
| inotify limits | Reset to 128 instances | `sudo sysctl fs.inotify.max_user_instances=512` (or via `sudo nerdctl run --rm --privileged busybox sysctl -w …`) before Inspektor Gadget |
| DNS in kind | The node copies WSL's `/etc/resolv.conf` at create time; `kind-dev.sh up` fails fast if the node cannot resolve `quay.io` | Make WSL's resolver reachable from containers (e.g. `nameserver 1.1.1.1` with `generateResolvConf = false`) |

Routine:

```bash
make images BUILDER=nerdctl                 # rebuild all images into containerd
scripts/kind-dev.sh load                    # push them into kind and restart pods
GJALLAR_E2E_PORT=28080 make e2e BUILDER=nerdctl
sudo nerdctl image prune -f                 # reclaim disk from old builds
```

The `gjallar-e2e` compose project and the kind node live in containerd
namespace `default`; they don't touch other namespaces on the same daemon.

## Network egress: what leaves your network

Nothing phones home: there is no telemetry, licence check or update check
(engine update checks are disabled in code). These are the only outbound
connections, all for data you asked for, and each can point at a mirror for
air-gapped sites.

| Component | Connects to | Why | Air-gapped alternative |
|---|---|---|---|
| `grype-db` mirror | `quay.io` (image pull) | Daily vulnerability DB | Build it yourself: `cronjobs.vunnel` / gjallar-vunnel, or mirror the image |
| `worker-vuln` | `gjallar-grype-db` (in cluster) | DB download | (already internal) |
| `worker-sbom` | your registries | Pull images for Syft | (your registries) |
| `worker-patch` | Trivy DB registry (`mirror.gcr.io`, `ghcr.io`) | Trivy report for Copa | `workers.patch.trivyDBRepository` |
| `agent` (runtime mode) | `ghcr.io` gadget images, via Inspektor Gadget | eBPF gadgets | Mirror `trace_open` and set `agent.runtime.gadget` |
| `api` (notifications) | Your webhook URLs | Alerts you configured; failed sends are retried with backoff | Internal chat (Mattermost, Rocket.Chat) |
| `api` (discovery) | Your Harbor / OCI registry | Registry discovery | (your registries) |
| `analytics` | Your S3 endpoint (DuckLake mode) | Lakehouse data | MinIO / Ceph RGW in cluster |
| All pods | Your OTLP endpoint, if set | Traces | Off by default |

DuckDB extensions are baked into the analytics image; nothing is fetched at runtime.
