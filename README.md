# Grafana Installation and Configuration

Grafana is the visualization layer for the in-cluster **Prometheus**. It is delivered by **FluxCD**
GitOps like the other infrastructure components: the Prometheus datasource, the dashboards, and
Keycloak OIDC login are all provisioned from values — there is no manual UI setup.

## Repository layout

| File | Purpose |
|---|---|
| `base.values.yaml` | Shared config: Prometheus datasource, `[auth.generic_oauth]` (Keycloak), dashboards sidecar, `root_url`. |
| `lcl.values.yaml` / `snbx.values.yaml` | Environment overlays (local / sandbox). Local adds `tls_skip_verify_insecure` for self-signed certs. |
| `dashboards/base/*.json` | Dashboard JSON files (raw data only — **no `kustomization.yaml` here**). |
| `lcl.values.yaml` | Legacy pre-overlay file — ignored, kept for reference. |

## FluxCD delivery (current)

Flux reconciles Grafana via:
- `clusters/base/infrastructure/fluxcd-grafana.yaml` — `HelmRepository` + `HelmRelease`
  `grafana-helmrelease`, with per-environment patches under
  `clusters/{local,sandbox}/infrastructure/fluxcd-grafana.yaml`.
- `grafana-base-values` / `grafana-level-values` ConfigMaps generated from `base.values.yaml` +
  `lcl`/`snbx.values.yaml` by `configMapGenerator` in `clusters/{local,sandbox}/config/kustomization.yaml`.
- **Dashboards** are generated from `dashboards/base/*.json` as ConfigMaps labeled
  `grafana_dashboard: "1"` by `configMapGenerator` entries in the same `config/kustomization.yaml`,
  each annotated `kustomize.toolkit.fluxcd.io/substitute: "disabled"` (dashboard JSON is full of
  `${...}` Grafana template vars that Flux substitution would otherwise blank out — same pattern as
  `keycloak-users-realm`). Grafana's **dashboards sidecar** (`sidecar.dashboards.enabled: true`)
  imports them automatically.

> Adding a dashboard: drop the JSON in `dashboards/base/`, add a file entry to one of the
> `grafana-dashboards-*` `configMapGenerator` groups in `config/kustomization.yaml` (keep each
> ConfigMap < 1 MB), and ensure its title is unique (duplicate titles break sidecar provisioning).

## Prerequisite — Keycloak OIDC client secret (before reconcile)

Grafana logs in via Keycloak OIDC against the **`master`** realm, **reusing Kiali's `kiali-client`**
(see `base.values.yaml` → `grafana.ini` `[auth.generic_oauth]`). Two manual steps are required
(secret management is manual for now, mirroring the Kiali setup):

1. In Keycloak → realm `master` → Clients → `kiali-client`, add Grafana's redirect URI:
   `https://grafana.<CLUSTER_DOMAIN>/login/generic_oauth`.
2. The client secret is delivered by External Secrets, not created by hand. The
   `boucio-grafana-oauth` ExternalSecret in
   `infrastructure/external-secrets/externalsecrets/infra/<env>/` writes the
   `grafana-oauth` Secret (key `client-secret`) from the **same** source value Kiali
   uses (`boucio-kiali-client-secret`), so the two can no longer drift apart.

   Set it once: on local in the gitignored `boucio-local-external-credentials.yaml`,
   on sandbox in GCP Secret Manager. The value comes from Keycloak, realm `master`,
   Clients, `kiali-client`, Credentials. See
   `infrastructure/external-secrets/README.md`.

   For a standalone `helm install` outside this GitOps setup, create it manually:

```shell
kubectl create secret generic grafana-oauth \
  --from-literal=client-secret='<kiali-client secret>' \
  -n grafana
```

Grafana injects it as `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET` via `envValueFrom`. Restart Grafana
after changing the secret (`kubectl rollout restart deploy/grafana -n grafana`) — the env var is read
at pod start.

> Log in as a **non-`admin`** Keycloak user with an email set; the username `admin` collides with
> Grafana's built-in admin and the OAuth user sync fails. OAuth users currently land as `Viewer`
> (role mapping from Keycloak roles is a known backlog item).

## Access

The chart ingress is intentionally **disabled** (`ingress.enabled: false`); external exposure is an
Istio `VirtualService` managed in a separate GitOps repo, serving `grafana.<CLUSTER_DOMAIN>`
(matching `server.root_url`). For local access without it:

```shell
kubectl -n grafana port-forward svc/grafana 3000:80   # http://localhost:3000
```

Built-in admin password (until secret management lands):
```shell
kubectl get secret grafana -n grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

## Manual install (reference, non-GitOps)

```shell
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install grafana grafana/grafana -n grafana --create-namespace \
  -f base.values.yaml -f lcl.values.yaml
```

## Sandbox

Mirrors Local but uses `snbx.values.yaml` (TLS verification on; no `tls_skip_verify_insecure`).

## References

1. https://github.com/grafana/helm-charts/tree/main/charts/grafana
2. https://github.com/dotdc/grafana-dashboards-kubernetes
3. https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/generic-oauth/
4. https://istio.io/latest/docs/ops/integrations/grafana/
