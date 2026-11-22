# Status and roadmap

## Done in 0.1.0

- Hexagonal core: domain (assets, findings lifecycle, risk score, license
  policy, nine posture checks) and use cases (hub, four workers, local scan,
  admission, rematch, patch, VEX exceptions).
- Engines: Syft (SBOM, any source), Grype (matching on stored SBOMs, EPSS and
  KEV from DB v6), built-in license policy plus optional Grant, Copa + Trivy
  report + rootless BuildKit.
- NATS JetStream: work queues, event stream, SBOM object store, runtime KV,
  retries with back-off, poison-message handling, dedupe, trace propagation.
- Postgres repository with atomic job claims (multi-replica safe).
- Analytics: DuckDB over Parquet or DuckLake, curated reports, read-only SQL,
  Parquet export, daily snapshots, compaction.
- Interfaces: REST, web UI (PKCE), MCP server (9 tools), CLI (CI gate,
  device-flow login, ops).
- Kubernetes: agent, Helm chart (every component a pod), CronJobs (DB
  refresh + rematch, analytics maintenance, backup, Vunnel), KEDA,
  NetworkPolicies, PodMonitor, Grafana dashboard, Kyverno admission and
  baseline policies, Keycloak realm.
- Observability: Prometheus metrics, OTLP traces to Tempo, JSON logs.
- Runtime reachability (2026-11): agent mode that streams Inspektor Gadget
  `trace_open` and records loaded packages per image digest.
- Tooling (2026-11): containerd/nerdctl builds (`BUILDER=nerdctl`), kind dev
  cluster script, dev Keycloak manifest.

## Verified, and how

Full report with evidence: [verification-2026-11.md](verification-2026-11.md).

| Check | Result |
|---|---|
| Unit and adapter tests (8 packages) | pass |
| End-to-end: real binaries + NATS 2.11 + Postgres 17 + DuckDB 2.0.1, on containerd (`BUILDER=nerdctl`) | pass |
| `helm lint` (Helm 3.17.3); default / all-features render | pass; 32 / 50 objects |
| Image builds, nerdctl 2.4.2 + BuildKit 0.22 | all 10; engines run inside at their pinned versions |
| Real Syft 1.57.2 + Grype 0.122.1 (`gjallar scan nginx:1.27`) | 790 findings → 1 urgent, 2 high; EPSS, KEV and fixes parsed from live DB v6 |
| Helm install on kind (Kubernetes 1.33.1, containerd 2.1) | all pods Ready, 0 restarts; inventory → SBOM → Grype → risk in-cluster |
| Kyverno 1.20 admission (CEL `ValidatingPolicy`, `http.Get`) | Deny mode refused `nginx:1.27` with Gjallar's reason; baseline flags `:latest` |
| KEDA 2.21 `nats-jetstream` scaler | workers 0 when idle; one queued job wakes the SBOM worker in ~5 s |
| Keycloak 26.4 realm import + API roles | viewer: read 200 / write 403; admin: write 202; bad/no token 401; device flow issues codes |
| Inspektor Gadget 0.58 runtime producer | packages recorded; libssl3 CVE 63 → 78 (loaded), KEV brotli CVE 78 → 29 (never loaded) |

## Not yet verified

- **Browser flows:** PKCE login in the web UI and approving a device-flow
  code in Keycloak's page. Tokens, roles and audience were verified with real
  Keycloak tokens; only the interactive pages remain.
- **Copa patching** against a real registry (needs push rights and BuildKit
  in the cluster: `workers.patch.enabled`).
- **MCP with Keycloak tokens** from a real AI client (token forwarding is
  covered by unit and e2e tests without auth).
- **DuckLake mode on S3** (Parquet mode is what e2e and kind exercise).
- **Multi-cluster** with a NATS leaf node.
- **arm64 images** (`PLATFORMS=linux/arm64`).

## Done in round 2 (2026-11-22)

Notifications, multi-tenancy, registry discovery (Harbor + any OCI registry),
signed SBOM attestations, Arrow Flight SQL, Kubescape regolibrary via OPA,
metering, audit log, NATS auth, CycloneDX/SPDX export, any-OIDC identity,
pinned and checksum-verified engines, self-hosted Grype DB (gjallar-vunnel),
signed releases with SBOM and SLSA provenance, OpenSSF Scorecard.
See [verification-2026-11-22.md](verification-2026-11-22.md).

## Next

Ordered by what most improves safety and freedom per hour of work.

1. ~~Read-only SQL by construction~~ (done 2026-11-22: sessions are locked
   to the lake path and the export folder; see architecture.md).
2. **NATS TLS and per-service credentials** (NKeys/JWT), so a leaked token
   is not a key to the whole bus.
3. ~~Notification retries~~ (done: a failed send unmarks the alert and the
   bus redelivers with backoff).
4. ~~VEX as signed attestations~~ (done in 0.2.0).
   Original note: **VEX as signed attestations.** Accepted risks published as OpenVEX
   cosign attestations next to the image, so Kyverno `verifyImages` can honour
   them offline.
5. ~~More open data sources~~ (done in 0.2.0: offline OSV second source, Trivy as alternative matcher, Dependency-Track export; GUAC reads the signed attestations).
   Original note: **More open data sources.** OSV.dev as a vulnerability source next to the
   Grype DB; export to GUAC / Dependency-Track (CycloneDX).
6. **Runtime, part 2.** `trace_exec` next to `trace_open`.
7. **Verification on a bigger host** (≥ 16 GB): browser flows, MCP via a real
   AI client, Copa push, DuckLake on Ceph, multi-cluster leaf nodes, arm64.
8. **OpenSSF Best Practices badge** (bestpractices.dev): most criteria are now
   met (SECURITY.md, CVD, CI, signed releases, tests); file the self-assessment.
