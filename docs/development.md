# Development

## Layout

```
cmd/
  gjallar/              CLI (local scan, hub client, login, ops)
  gjallar-api/          hub: REST + event consumers
  gjallar-worker/       sbom | vuln | license | patch
  gjallar-agent/        per-cluster inventory
  gjallar-analytics/    lakehouse writer + query API, `maintain`
  gjallar-mcp/          MCP server
  gjallar-web/          UI server (static/ is embedded)
internal/
  domain/               pure model: assets, findings, risk, policy, events   ← no imports outside stdlib
  app/                  use cases + ports (interfaces)                       ← imports domain only
  adapters/
    scanner/            Syft, Grype, Grant, Copa (subprocess); SBOM file→package index
    gadget/             Inspektor Gadget trace_open stream (kubectl-gadget subprocess)
    natsbus/            JetStream bus, consumers, SBOM object store, runtime KV
    postgres/           repository + embedded schema.sql
    memory/             in-memory repository/bus/store (tests, dev)
    analytics/          DuckDB/Parquet/DuckLake
    oidc/               Keycloak token verification + HTTP middleware
    k8s/                stdlib Kubernetes client (inventory, rollout restart)
    httpapi/            REST driving adapter (+ tenancy scoping, audit, SBOM download)
    flightsql/          Arrow Flight SQL driving adapter (analytics)
    rego/               Kubescape regolibrary controls via the OPA CLI
    notify/             Slack / Teams / JSON webhooks
    harbor/, registry/  registry discovery: Harbor API, any OCI Distribution registry
    mcp/                MCP driving adapter
  client/               Go API client (CLI, MCP)
  platform/             telemetry (logs, metrics, OTLP traces), httpx helpers
deploy/
  compose/              docker-compose (dev) and e2e infra
(chart, Kyverno policies, Keycloak realm: github.com/heimdallr-dev/gjallar-helm)
  keycloak/             realm export + throwaway dev Keycloak (keycloak-dev.yaml)
  kyverno/              admission policies
test/e2e/               end-to-end test + fake engines
scripts/e2e.sh          runs the e2e test against real binaries
scripts/kind-dev.sh     local kind cluster on containerd: up | load | down
```

The dependency rule is enforced by convention and review: `domain` imports
nothing from Gjallar; `app` imports only `domain`; adapters import `app` and
`domain`; `cmd/*` wires everything together.

Third-party Go dependencies are deliberately few: `nats.go`, `pgx`, and their
transitive deps. Everything else (OIDC, OTLP, Prometheus, Kubernetes client,
MCP) uses the standard library.

## Tests

```bash
make test        # unit + adapter tests (DuckDB tests run if `duckdb` is on PATH)
make e2e         # real binaries + NATS + Postgres + DuckDB, fake engines
make e2e BUILDER=nerdctl GJALLAR_E2E_PORT=28080   # on containerd, other ports
```

| Package | What is covered |
|---|---|
| `domain` | Risk ordering, bounds, fallback; image refs; lifecycle reconcile; license expressions; posture checks; JSON |
| `app` | Full pipeline in memory: discovery → SBOM → match → risk → licenses → rematch → patch → admission |
| `scanner` | Real Syft/Grype JSON shapes (licenses as strings/objects, related-vuln CVSS/EPSS/KEV, dedupe), CLI arguments, Grype DB update-once, SBOM file ownership (merged /usr, SONAME symlinks) |
| `gadget` | Real Inspektor Gadget v0.58 `trace_open` events: keeps successful pod opens only |
| `app` (runtime) | RuntimeRecorder: path→package, merge with known, pending until the SBOM exists, pods gone |
| `oidc` | RS256 signature, issuer, expiry, audience; middleware role checks |
| `k8s` | Pods → workloads (replica collapse, owner resolution, exposure via Service/Ingress, security context) |
| `analytics` | SQL guard; real DuckDB Parquet append → report → daily snapshot |
| `mcp` | initialize, notifications, tools/list, tools/call, auth forwarding, tool errors |
| `test/e2e` | Everything above wired over the network |

## Adding an engine

Example: Trivy as an alternative matcher.

1. Implement the port in `internal/adapters/scanner/trivy.go`:
   ```go
   type Trivy struct{ Run Runner }
   func (t Trivy) Match(ctx context.Context, sbomRaw []byte) (string, []domain.Finding, error) { ... }
   ```
2. Add a fixture-based test next to it (see `scanner_test.go`).
3. Select it in `cmd/gjallar-worker` (e.g. `GJALLAR_MATCHER=trivy`).
4. Add a Dockerfile target with the binary. Nothing else changes: the hub,
   API, UI, analytics and Helm chart are engine-agnostic.

## Adding a runtime source

Write the package names seen loading for an image to the NATS KV bucket
`gjallar-runtime` (key: digest with `:` replaced by `_`, value: JSON array of
package names), or call `natsbus.RuntimeKV.Record`. The hub picks it up on the
next scoring pass: loaded packages score ×1.0, packages never loaded ×0.3.

The built-in source is Inspektor Gadget (`agent.runtime.enabled`). To add
another eBPF tool (Tetragon, Falco, Kubescape node-agent), write an adapter
that calls `app.RuntimeRecorder.Observe(ns, pod, container, path)` for each
file the container opens or executes. The recorder already resolves digests,
maps paths to packages through the SBOM and merges into the KV. Write
package names, not paths, if you bypass it: they must match `finding.package`.

## Release checklist

- `make vet test e2e` (or `BUILDER=nerdctl`)
- `helm lint gjallar-helm/charts/gjallar && helm template gjallar gjallar-helm/charts/gjallar -f <profile>`
- `make images` and `scripts/kind-dev.sh up` once: catches moved download URLs (DuckDB renamed its CLI asset in 2.0)
- On the kind cluster: `kubectl apply -f gjallar-helm/policies/kyverno/` and `gjallar-helm/keycloak/keycloak-dev.yaml`, check `/api/v1/me` roles
- Review engine version bumps in the Dockerfile (Trivy provenance!). Keep `GADGET_VERSION` and `agent.runtime.gadget` in step
- `make push VERSION=vX.Y.Z PLATFORMS=linux/amd64,linux/arm64`
- Bump `version`/`appVersion` in `Chart.yaml`
