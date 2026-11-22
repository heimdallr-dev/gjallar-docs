# Install: from WSL to your Kubernetes cluster

This guide takes a fresh WSL2 Ubuntu shell to Gjallar running in your cluster.
Each step has a check, so you know it worked before moving on.

## 0. Prerequisites in WSL

```bash
sudo apt update && sudo apt install -y make git curl unzip python3-pip

# Go 1.26.11+ (older Go releases carry fixed security vulnerabilities in net/http and crypto)
curl -sSfLO https://go.dev/dl/go1.26.11.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.26.11.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc && source ~/.bashrc

# DuckDB CLI 2.x (needed for the e2e analytics checks). The release asset is a tarball since 2.0.
curl -sSfL https://github.com/duckdb/duckdb/releases/download/v2.0.1/duckdb-cli-linux-amd64.tar.gz \
  | sudo tar -xz -C /usr/local/bin duckdb

# Containers: EITHER Docker (Desktop WSL integration, or Docker Engine in WSL)
docker version
# OR containerd + nerdctl (no Docker): see "Without Docker" in step 4

# Helm + kubectl for the cluster
curl -sSfL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
kubectl version --client
```

Check: `go version`, `duckdb --version`, `docker info` (or `sudo nerdctl info`),
`helm version` and `kubectl get nodes` all work.

**No sudo?** Everything except the container runtime installs into your home
folder: unpack Go to `~/.local/go`, put `helm`, `duckdb`, `kind` and
`kubectl-gadget` in `~/.local/bin`, get `make` with
`apt-get download make && dpkg -x make_*.deb mk && cp mk/usr/bin/make ~/.local/bin/`,
and add `export PATH=$PATH:$HOME/.local/go/bin:$HOME/.local/bin` to `~/.bashrc`.

> Work inside the Linux filesystem (`~/src/gjallar`), not under `/mnt/c/...`.
> Builds there are much faster. If you copied the tree from Windows or a zip
> and `make e2e` says `Permission denied`, restore the executable bits:
> `chmod +x scripts/*.sh test/e2e/fakebin/*`.

## 1. Unpack, build and test

```bash
mkdir -p ~/src && cd ~/src
unzip ~/Downloads/gjallar.zip -d .     # or: git clone https://github.com/heimdallr-dev/gjallar
cd gjallar
make vet test build
ls bin/
```

Check: all test packages print `ok`, and `bin/` has seven binaries
(`gjallar`, `gjallar-api`, `-agent`, `-analytics`, `-mcp`, `-web`, `-worker`).

Dependencies are vendored (`vendor/`), so builds work offline. To refresh
them later: `GOFLAGS=-mod=mod go get -u ./... && go mod tidy && go mod vendor`.

## 2. End-to-end test

```bash
make e2e
```

This starts NATS and Postgres in Docker and runs the real Gjallar binaries
with stand-in `syft`, `grype` and `copa` scripts (`test/e2e/fakebin`). It
walks the whole pipeline: inventory → SBOM → match → risk → API → web proxy
→ MCP → Copa patch → admission → VEX → rematch → DuckDB analytics.

The services listen on ports 18080–18083. If one is taken, the script stops
and says so; pick another base with `GJALLAR_E2E_PORT=28080 make e2e`.

Check: `--- PASS: TestEndToEnd`, with a log line like
`ranked: CVE-2023-4863=88 (urgent), CVE-2099-0001=20 (medium)`.

## 3. Try it locally with Docker Compose (optional)

```bash
make dev                                   # builds all images; first run takes a few minutes
GJALLAR_URL=http://localhost:8080 bin/gjallar submit nginx:1.27
GJALLAR_URL=http://localhost:8080 bin/gjallar findings --band urgent
```

Open http://localhost:8083 from Windows. WSL2 forwards localhost.
With Keycloak, see the comment at the bottom of `deploy/compose/docker-compose.yml`.

Stop it with `make dev-down`.

## 4. Build and push images

Pick a registry your cluster can pull from: Harbor, GHCR, or the registry in
your homelab.

```bash
docker login harbor.local                  # or ghcr.io
make push REGISTRY=harbor.local/gjallar VERSION=0.1.0
# multi-arch: make push REGISTRY=... PLATFORMS=linux/amd64,linux/arm64
```

This builds ten images from one Dockerfile: `api agent mcp web analytics cli
worker-sbom worker-vuln worker-license worker-patch`.

Check: `docker manifest inspect harbor.local/gjallar/gjallar-api:0.1.0`.

### Without Docker: containerd + nerdctl

If WSL runs containerd (rootful, `/run/containerd/containerd.sock`) instead of
Docker, add `BUILDER=nerdctl` to `make images`, `push`, `e2e`, `dev` and
`dev-down`. nerdctl needs a BuildKit daemon, so start one once:

```bash
curl -sSfL https://github.com/moby/buildkit/releases/download/v0.22.0/buildkit-v0.22.0.linux-amd64.tar.gz \
  | sudo tar -xz -C /usr/local                       # installs bin/buildkitd and bin/buildctl
sudo buildkitd --oci-worker=false --containerd-worker=true &   # or a systemd unit
sudo nerdctl login harbor.local
make push BUILDER=nerdctl REGISTRY=harbor.local/gjallar VERSION=0.1.0
```

Images land in containerd namespace `default`. Pass `CTR_NAMESPACE=k8s.io`
to build straight into a local k3s or kubeadm node's image store.
Check: `sudo nerdctl images | grep gjallar`.

The Makefile calls `sudo nerdctl`, so builds need your password, or a
narrow sudoers rule (equivalent to root on that machine, since nerdctl can
mount `/`; fine on a personal WSL, not on a shared host):

```bash
echo "$USER ALL=(root) NOPASSWD: /usr/local/bin/nerdctl, /usr/local/bin/buildkitd" \
  | sudo tee /etc/sudoers.d/gjallar-nerdctl
sudo chmod 440 /etc/sudoers.d/gjallar-nerdctl && sudo visudo -cf /etc/sudoers.d/gjallar-nerdctl
```

If a build stops with `parent snapshot … does not exist`, that is a transient
containerd snapshotter race: rerun with only the failed images,
`make images BUILDER=nerdctl COMPONENTS="worker-sbom worker-vuln"`.

### Optional: a local test cluster with kind (no registry needed)

```bash
curl -sSfL -o ~/.local/bin/kind https://github.com/kubernetes-sigs/kind/releases/download/v0.29.0/kind-linux-amd64
chmod +x ~/.local/bin/kind
scripts/kind-dev.sh up            # creates cluster "gjallar", loads the images, installs the chart
```

The script runs kind with the nerdctl provider, checks the node's DNS, loads
the ten images with `kind load image-archive` and runs `helm install`. After
rebuilding images run `scripts/kind-dev.sh load`. After a WSL restart run
`up` again (see Operations → Local dev environment). It pulls NATS, Postgres
and the Grype DB mirror from the internet; the first install takes ~5 minutes.

If the registry is private, create a pull secret in the target namespace:

```bash
kubectl create namespace gjallar
kubectl -n gjallar create secret docker-registry harbor-pull \
  --docker-server=harbor.local --docker-username=... --docker-password=...
```

## 5. Write your values

`my-values.yaml`, a sensible start for a homelab cluster:

```yaml
image:
  registry: harbor.local/gjallar
  tag: "0.1.0"
imagePullSecrets: [{ name: harbor-pull }]
cluster: { name: homelab }

workers:
  registryAuthSecret: harbor-pull     # Syft pulls private images with this

analytics:
  mode: parquet                       # simplest; see docs/analytics.md for DuckLake on Ceph RGW
  storage: 20Gi

ingress:
  enabled: true
  className: nginx                    # or your Emissary/Cilium ingress class
  web: { host: gjallar.example.org }
  tls: [{ secretName: gjallar-tls, hosts: [gjallar.example.org] }]
  annotations: { cert-manager.io/cluster-issuer: letsencrypt }

networkPolicy: { enabled: true }       # works with Cilium

observability:
  otlpEndpoint: http://tempo.observability:4318
  grafanaDashboard: true
```

## 6. Deploy

```bash
make deploy REGISTRY=harbor.local/gjallar VERSION=0.1.0 HELM_ARGS="-f my-values.yaml"
kubectl -n gjallar get pods -w
```

Check: every pod is `Running`/`Ready` within a couple of minutes:
`gjallar-api` ×2, `-agent`, `-analytics`, `-grype-db`, `-mcp` ×2, `-nats-0`,
`-postgres-0`, `-web` ×2, and `-worker-sbom`, `-vuln`, `-license`.

Watch the first scan:

```bash
kubectl -n gjallar logs deploy/gjallar-agent | head      # "inventory published"
kubectl -n gjallar logs deploy/gjallar-api -f            # "queued sbom ..."
kubectl -n gjallar port-forward svc/gjallar-api 8080:8080 &
GJALLAR_URL=http://localhost:8080 bin/gjallar summary
GJALLAR_URL=http://localhost:8080 bin/gjallar findings --band urgent
```

Run the first DB refresh now instead of waiting for 03:17:

```bash
kubectl -n gjallar create job --from=cronjob/gjallar-db-refresh db-refresh-now
```

## 7. Turn on Keycloak

1. Import the realm: in the Keycloak admin console choose **Create realm →
   Browse** and pick `gjallar-helm/keycloak/gjallar-realm.json`. Or put it in your
   Keycloak's import folder. Change the two seeded passwords; both are
   marked temporary.
2. In client `gjallar-web`, set **Valid redirect URIs** to `https://gjallar.example.org/*`.
3. Upgrade:
   ```bash
   make deploy ... HELM_ARGS="-f my-values.yaml --set auth.issuer=https://keycloak.example.org/realms/gjallar"
   ```
4. CLI: `bin/gjallar login https://keycloak.example.org/realms/gjallar`
   (device flow; open the printed link and confirm the code).

Check: the UI shows **Sign in**. A `viewer` user can browse but gets 403 on
Scan/Patch/Accept. `admin` can do everything. `GET /api/v1/me` with your
token shows your username and roles; if it shows empty fields, the realm was
imported from an older copy of the file (see Operations → Troubleshooting).

No Keycloak yet? `gjallar-helm/keycloak/keycloak-dev.yaml` runs a throwaway one
(dev mode, in-memory) with the realm imported; the header of that file has
the four commands.

## 8. Turn on enforcement with Kyverno (optional)

Needs **Kyverno ≥ 1.16**. The policies are CEL `ValidatingPolicy`
(`policies.kyverno.io/v1`); Kyverno 1.20 no longer runs the old
`kyverno.io/v1 ClusterPolicy`.

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
make kyverno                         # applies gjallar-helm/policies/kyverno/*.yaml in Audit mode
kubectl get policyreports -A         # see what *would* be blocked
```

When the reports look right, change `validationActions: [Audit]` to `[Deny]`
in `gjallar-helm/policies/kyverno/gjallar-admission.yaml`. Both policies use
`failurePolicy: Ignore`, so the cluster keeps working if Gjallar or Kyverno is
down. Unknown images are allowed (and queued for a scan).

Check (Deny mode): `kubectl run t --image=<an image with urgent findings>` is
refused with `Gjallar: blocked: 1 urgent and 1 known-exploited open findings`.

## 9. Turn on patching with Copa (optional)

```yaml
workers:
  patch: { enabled: true, push: true }
  registryAuthSecret: harbor-push      # needs push rights
```

Then **Images → Patch** in the UI, or `gjallar patch sha256:...`. The result
is `repo:tag-patched` in your registry. Rescan it, then roll it out with your
normal GitOps flow.

## 10. Scale to zero with KEDA (optional)

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda -n keda --create-namespace
make deploy ... HELM_ARGS="-f my-values.yaml --set keda.enabled=true"
```

Workers then sit at 0 replicas and wake up when the JetStream queue has work.
Check: `kubectl -n gjallar get scaledobject` shows `READY True`; after
`gjallar submit alpine:3.20` the SBOM worker goes 0 → 1 within seconds.

## 11. Runtime reachability with Inspektor Gadget (optional)

Ranks vulnerabilities in packages that actually load above ones that never
do (×1.0 vs ×0.3).

```bash
# kubectl-gadget must match the agent's gadget version (v0.58.0 by default)
curl -sSfL https://github.com/inspektor-gadget/inspektor-gadget/releases/download/v0.58.0/kubectl-gadget-linux-amd64-v0.58.0.tar.gz \
  | tar -xz -C ~/.local/bin kubectl-gadget
kubectl gadget deploy
make deploy ... HELM_ARGS="-f my-values.yaml --set agent.runtime.enabled=true"
```

Check: `kubectl -n gjallar logs deploy/gjallar-agent | grep runtime` shows
`runtime packages recorded`, and within one `agent.interval` findings carry
`package loaded at runtime` / `package never loaded at runtime` reasons.
On kind or WSL, raise `fs.inotify.max_user_instances` to 512 first.

## Uninstall

```bash
make undeploy                        # PVCs are kept on purpose
kubectl -n gjallar delete pvc --all  # only if you really want the data gone
```
