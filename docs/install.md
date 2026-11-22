# Install on Kubernetes

Everything comes signed from GHCR: the chart and every image. Nothing to build
or clone. A small cluster is enough; the default install asks for about 1 CPU
and 2 GB of memory in total, plus 15 GB of volumes.

## 1. Install

```bash
helm install gjallar oci://ghcr.io/heimdallr-dev/charts/gjallar --version 0.3.1 \
  -n gjallar --create-namespace \
  --set cluster.name=my-cluster
```

| Your cluster | Add |
|---|---|
| no default StorageClass | `--set storageClass=<name>` (used by NATS, Postgres and analytics) |
| small or shared nodes | `-f values-kind.yaml` from [gjallar-helm](https://github.com/heimdallr-dev/gjallar-helm/blob/main/charts/gjallar/values-kind.yaml) (one replica each, lower requests) |
| private registries | `--set workers.registryAuthSecret=<docker-registry Secret>`, or images from them stay without an SBOM |
| air-gapped | mirror the images and set `image.registry`; see [network egress](operations.md#network-egress-what-leaves-your-network) |

Within minutes the agent reports every running workload; Syft, Grype and the
risk score follow per image. Watch with `kubectl -n gjallar logs deploy/gjallar-api -f`.

Without `auth.issuer`, sign-in is **off**: anyone who can reach the API is an
admin. Keep it that way only on a laptop; otherwise continue with step 2.

## 2. Sign-in with your identity provider

Gjallar works with any OIDC provider (Keycloak, Entra ID, Authentik, Dex,
GitLab). It needs a public client for the browser (`gjallar-web`, PKCE), the
API audience `gjallar`, and the roles `gjallar-admin` and `gjallar-viewer`.

**Keycloak already in the cluster:** import the realm with
[`keycloak/import-realm.sh`](https://github.com/heimdallr-dev/gjallar-helm/blob/main/keycloak/import-realm.sh).
It logs in with Keycloak's bootstrap-admin Secret, imports the realm, and
creates the users `admin` and `viewer` with random **temporary** passwords
(stored in Secret `gjallar/gjallar-initial-users`, never printed):

```bash
KC_NS=keycloak KC_SVC=keycloak-http KC_PORT=8443 \
KC_ADMIN_SECRET=keycloak-bootstrap-admin KC_CA_SECRET=keycloak-server-tls \
  keycloak/import-realm.sh
helm upgrade gjallar oci://ghcr.io/heimdallr-dev/charts/gjallar --version 0.3.1 -n gjallar \
  --reuse-values --set auth.issuer=https://keycloak.example.org/realms/gjallar
```

The issuer must be the URL in the tokens' `iss` (Keycloak's public hostname),
reachable from the browser and from the API pod.

## 3. Open it

```bash
kubectl -n gjallar port-forward svc/gjallar-web 8083
kubectl -n gjallar get secret gjallar-initial-users -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Open <http://localhost:8083>, sign in as `admin` and choose a new password.
For a public URL set `ingress.enabled=true` and `ingress.web.host`, and add
`https://<host>/*` to the `gjallar-web` client's redirect URIs.

## Upgrade and remove

```bash
helm upgrade gjallar oci://ghcr.io/heimdallr-dev/charts/gjallar --version <new> -n gjallar --reuse-values
helm uninstall gjallar -n gjallar        # volumes stay; delete the namespace to remove all data
```

Your data stays readable without Gjallar: Postgres (plain SQL), Parquet or
DuckLake files, and SBOM/VEX exports in CycloneDX, SPDX and OpenVEX.

For a laptop or a local kind cluster, see [install-wsl.md](install-wsl.md).
