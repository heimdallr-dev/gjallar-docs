# Verification report, 2026-11-21

The first run of Gjallar 0.1.0 outside the build sandbox: a WSL2 machine
(Ubuntu 26.04, 11 GB RAM, rootful containerd, no Docker) and a kind cluster
on that containerd. This report lists what was checked, what broke, what was
fixed, and what is still open. Status at a glance lives in
[roadmap.md](roadmap.md).

## Environment

| Piece | Version |
|---|---|
| Go | 1.24.7 |
| containerd / nerdctl / BuildKit | rootful containerd, nerdctl 2.4.2, BuildKit 0.22.0 |
| Kubernetes | kind 0.29 (nerdctl provider), Kubernetes 1.33.1, containerd 2.1.1 |
| Helm | 3.17.3 |
| Engines in images | Syft 1.57.2, Grype 0.122.1, Grant 0.8.0, Copa 0.15.0, Trivy 0.69.3, DuckDB 2.0.1 |
| Add-ons | Kyverno 1.20.0 (chart 3.10.0), KEDA 2.21.1, Keycloak 26.4, Inspektor Gadget 0.58.0 |

## Results

| # | Check | Result |
|---|---|---|
| 1 | `go vet`, `go test ./...` (8 packages; e2e runs separately) | pass |
| 2 | `make e2e BUILDER=nerdctl` | pass: `CVE-2023-4863=88 (urgent), CVE-2099-0001=20 (medium)` |
| 3 | `helm lint`, default / all-features render | pass; 32 / 50 objects |
| 4 | `make images BUILDER=nerdctl` | 10/10 images; every engine runs inside at its pinned version; DuckDB extensions baked in |
| 5 | `gjallar scan nginx:1.27` with real Syft + Grype | 790 findings → urgent 1 · high 2 · medium 59 · low 728; top: CVE-2026-111497 [KEV] libbrotli1, risk 78. ~3 min including the first DB download |
| 6 | Helm install on kind | all pods Ready with 0 restarts; agent → SBOM → Grype → risk in-cluster; same top finding as #5 |
| 7 | Kyverno admission | Audit: PolicyReports per pod. Deny: `kubectl run web2 --image=nginx:1.27` refused with `Gjallar: blocked: 1 urgent and 1 known-exploited open findings`; `busybox:1.36` allowed |
| 8 | Kyverno baseline | `nginx:latest` fails GJ-007; pinned images pass |
| 9 | KEDA | ScaledObjects Ready; workers 0 when idle; `gjallar submit alpine:3.20` woke the SBOM worker in ~5 s, then license and vuln |
| 10 | Keycloak | realm imports; no/bad token 401; viewer read 200, write 403; admin write 202; `/api/v1/me` shows user + roles; device authorization issues user codes |
| 11 | Inspektor Gadget runtime | agent recorded loaded packages for 3 images; nginx: 88 findings "loaded", 3 "never loaded" |

What runtime data changed for nginx:1.27 (the point of Gjallar):

| Finding | Static | With runtime | Why |
|---|---|---|---|
| CVE-2025-15467 libssl3 | 63 high | **78 urgent** | nginx loads libssl |
| CVE-2026-111497 [KEV] libbrotli1 | 78 urgent | **29 medium** | nginx never loads brotli |

Packages seen loading: libc6, libssl3, libpcre2-8-0, zlib1g, libacl1, nginx.

## Defects found and fixed

| # | Defect | Effect | Fix |
|---|---|---|---|
| 1 | DuckDB 2.0 renamed its CLI download to `duckdb-cli-linux-<arch>.tar.gz` | analytics image build failed (404) | `Dockerfile` |
| 2 | Services exited when NATS wasn't reachable yet | 5–6 crash-loop restarts on every fresh install | `natsbus.Connect` retries for 5 minutes |
| 3 | Each concurrent Grype job downloaded and unpacked its own DB | 2 × 2.8 GB per pod, risk of filling the 10 Gi emptyDir with higher concurrency | one serialised `grype db update`, matches run with `GRYPE_DB_AUTO_UPDATE=false` (+ test) |
| 4 | Kyverno policies were `kyverno.io/v1 ClusterPolicy` | rejected by Kyverno 1.20 (legacy types removed); admission feature unusable | rewritten as CEL `ValidatingPolicy` (`policies.kyverno.io/v1`) |
| 5 | Baseline policy had no `failurePolicy` (default `Fail`) | while Kyverno restarted, **every** Deployment/StatefulSet change in the cluster was rejected | `failurePolicy: Ignore` |
| 6 | KEDA `activationLagThreshold: "1"` | KEDA activates on lag *greater than* the threshold, so a single queued job never woke a scaled-to-zero worker | `"0"` |
| 7 | Realm declared its own `clientScopes` list | Keycloak skipped its built-in scopes: tokens had no `sub`, username or roles, so **admins got 403 on every write** | built-in scopes kept; audience mapper on each client |
| 8 | e2e used fixed ports 18080–18083 | collided with another local service and hung for 3 minutes | `GJALLAR_E2E_PORT`, fails fast on a busy port |
| 9 | Executable bits lost when the tree was copied into WSL | `make e2e`: `Permission denied` | restored; noted in the install guide |
| 10 | Makefile was Docker-only | no way to build on containerd | `BUILDER=nerdctl` for images, push, e2e, dev |

## Environment findings (not Gjallar bugs)

| Finding | Workaround |
|---|---|
| kind node gets a new IP after a WSL restart (containerd CNI allocates round-robin); the control plane can't survive it | `scripts/kind-dev.sh up` recreates the cluster |
| kind node kept an old WSL resolver (the WSL host) that doesn't answer container traffic | `kind-dev.sh` checks DNS at create time; WSL uses `1.1.1.1` with `generateResolvConf = false` |
| `fs.inotify.max_user_instances=128` too low for Inspektor Gadget on kind | raise to 512 (resets on reboot) |
| Kyverno lost leader election under load (load average 20 on 11 GB) | transient; it recovers. Single-node dev clusters need headroom |
| `parent snapshot … does not exist` during one nerdctl build | transient containerd race; rerun the failed images |
| `helm upgrade --reuse-values` ignores new chart defaults (runtime RBAC rendered into the wrong namespace) | use `--reset-then-reuse-values` |
| Inspektor Gadget leaves `runtime.containerImageDigest` empty on containerd | agent resolves digests from pod status |

## Still open

See [roadmap.md → Not yet verified](roadmap.md#not-yet-verified): browser
PKCE and device-flow approval pages, Copa against a real registry, MCP with
real tokens, DuckLake on S3, multi-cluster, arm64.

## Reproduce

```bash
make vet test
make images BUILDER=nerdctl
GJALLAR_E2E_PORT=28080 make e2e BUILDER=nerdctl
scripts/kind-dev.sh up
helm install kyverno kyverno/kyverno -n kyverno --create-namespace && kubectl apply -f gjallar-helm/policies/kyverno/
helm install keda kedacore/keda -n keda --create-namespace
kubectl create ns keycloak
kubectl -n keycloak create configmap gjallar-realm --from-file=gjallar-helm/keycloak/gjallar-realm.json
kubectl apply -f gjallar-helm/keycloak/keycloak-dev.yaml
kubectl gadget deploy
helm upgrade gjallar gjallar-helm/charts/gjallar -n gjallar --reset-then-reuse-values \
  --set keda.enabled=true --set agent.runtime.enabled=true \
  --set auth.issuer=http://keycloak.keycloak.svc:8080/realms/gjallar
```
