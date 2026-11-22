# Interfaces: REST, MCP, CLI

## REST API (`gjallar-api`, port 8080)

Authentication: `Authorization: Bearer <Keycloak access token>`. `GET` needs
the viewer role (if configured); everything else needs `gjallar-admin`.

**Multi-tenancy** (`tenancy.groups`): callers who are not in a global role
only see findings, images, workloads, posture, licenses and patches in the
namespaces their Keycloak groups map to; other teams' images answer 404, and
`/api/v1/analytics/*` answers 403 (raw SQL can't be filtered per namespace).
`GET /api/v1/me` shows `all_namespaces` and `namespaces`.

**Identity providers:** any OIDC provider works. Roles are read from
Keycloak's `realm_access`/`resource_access`, a flat `roles` claim (Entra ID,
Authentik), and group names (Dex, GitLab). The username falls back to
`email`, then `sub`.

**Audit:** every non-read call is logged (`"msg":"audit"`) with user, OIDC
client, method, path and status, and kept in the lakehouse table
`audit_events`. The CLI, web UI and
MCP inherit this because they use the caller's token.

| Method & path | Purpose |
|---|---|
| `GET /healthz`, `/readyz`, `/metrics` | Probes and Prometheus metrics (no auth) |
| `GET /api/v1/version`, `/api/v1/me` | Build version; the caller's identity and roles |
| `GET /api/v1/summary` | Counts by band and severity, KEV, fixable, posture, licenses, median time to fix |
| `GET /api/v1/findings?band=&namespace=&package=&digest=&state=open\|fixed\|accepted\|all&limit=` | Risk-ranked findings with repository and workloads |
| `POST /api/v1/findings/{id}/accept` `{"reason": "...", "expires": "RFC3339?"}` | Accept a risk (stored exception; appears in VEX) |
| `GET /api/v1/images`, `/api/v1/images/{digest}` | Images; one image with all its findings |
| `GET /api/v1/images/{digest}/sbom?format=syft-json\|cyclonedx-json\|spdx-json` | The stored SBOM in an open standard format |
| (on accept) | Accepting a risk also queues a **signed OpenVEX attestation** next to the image when `workers.sbom.attest` covers its registry |
| `GET /api/v1/images/{digest}/vex` | OpenVEX document of accepted findings (`grype --vex`) |
| `POST /api/v1/images/{digest}/patch` | Queue a Copa patch (409 if nothing fixable at OS level) |
| `POST /api/v1/scan` `{"image": "repo:tag"}` | Queue an SBOM + match for any image |
| `POST /api/v1/rematch` | Re-match every stored SBOM (no pulls) |
| `GET /api/v1/workloads`, `/posture`, `/licenses?verdict=denied`, `/patches` | Inventory and results |
| `GET /api/v1/admission?image=repo:tag[@digest][&deny_unknown=true]` | `{allowed, urgent, kev, reason}` for Kyverno |
| `GET /api/v1/analytics/reports`, `/reports/{name}` | Curated reports (proxied to analytics) |
| `POST /api/v1/analytics/query` `{"sql": "...", "max_rows": 1000}` | Read-only DuckDB SQL |
| `POST /api/v1/analytics/export` `{"sql": "..."}` | Query result as a Parquet download |
| Flight SQL `gjallar-analytics:8815` (gRPC) | Same SQL as Arrow record batches (see analytics.md) |

## MCP server (`gjallar-mcp`, `POST /mcp`)

Transport: MCP Streamable HTTP with JSON responses, protocol `2025-06-18`.
With `auth.issuer` set, every call needs a bearer token. The server
advertises Keycloak at `/.well-known/oauth-protected-resource` (RFC 9728), so
MCP clients can discover where to log in. Use the `gjallar-mcp` public client
from the realm export.

| Tool | Read-only | Does |
|---|---|---|
| `security_summary` | ✓ | Headline numbers |
| `top_risks` | ✓ | Ranked findings (filters: band, namespace, limit) with reasons and locations |
| `find_package` | ✓ | "Where does log4j / openssl run?" |
| `image_report` | ✓ | Everything about one digest |
| `posture_violations` | ✓ | Misconfigured running workloads |
| `license_denials` | ✓ | Packages with denied licenses |
| `analytics_report` | ✓ | One of the curated lakehouse reports |
| `analytics_query` | ✓ | One read-only DuckDB query (500 rows) |
| `request_patch` | ✗ | Queue a Copa patch (admin only, enforced by the API) |

Example client config (Claude Code / any MCP client):

```json
{ "mcpServers": { "gjallar": { "type": "http", "url": "https://gjallar.example.com/mcp" } } }
```

Try it with curl:

```bash
curl -s localhost:8082/mcp -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"top_risks","arguments":{"band":"urgent"}}}'
```

## CLI (`gjallar`)

```
Local:  gjallar scan <image> [--json] [--fail-on urgent|high|medium|low]
                             [--source registry|docker|podman|containerd] [--scope all-layers]
Hub:    gjallar summary | findings [--band b --namespace ns --package p --json] | images | posture | licenses | patches
        gjallar submit <image> | rematch | patch <digest> | accept <finding-id> <reason>
        gjallar report <name> | query "<sql>"
Auth:   gjallar login <keycloak-realm-url>     (device flow; token cached in ~/.config/gjallar)
Ops:    gjallar ops refresh-db                  (used by the in-cluster CronJob)
```

**CI gate** (GitHub Actions, GitLab, Jenkins): needs `syft` and `grype` on
PATH, no hub. Exit codes: `0` ok, `3` findings at or above `--fail-on`,
`1` error, `2` usage.

```yaml
- run: gjallar scan ${{ env.IMAGE }} --fail-on urgent
```

The local scan has no runtime or exposure data, so it uses neutral context
(runtime unknown, not exposed). The same image may rank higher in the cluster
once Gjallar knows it is behind an Ingress.
