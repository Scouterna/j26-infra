# j26-keycloak-scoutid

Keycloak deployment for J26, running the upstream ScoutID image
(`ghcr.io/scouterna/scoutid-keycloak:latest`) with the `jamboree26` realm
configured for Scoutnet login.

## Architecture

```
                    ┌─────────────────────────────────┐
                    │  Traefik (IngressRoute)          │
                    │  id.dev.j26.se                   │
                    │    /           → 301 Scouterna   │
                    │    /realms/*   → Keycloak :8080  │
                    │    /resources/* → Keycloak :8080 │
                    │  admin.id.dev.j26.se             │
                    │    /*          → Keycloak :8080  │
                    └───────────────┬─────────────────┘
                                    │
                    ┌───────────────▼─────────────────┐
                    │  Keycloak pod (Deployment)       │
                    │  ghcr.io/scouterna/              │
                    │    scoutid-keycloak              │
                    │  :8080  HTTP + realms            │
                    │  :9000  /health + /metrics       │
                    └──────┬──────────────┬───────────┘
                           │              │
          ┌────────────────▼──┐    ┌──────▼──────────────────┐
          │  Azure Postgres   │    │  Azure Key Vault         │
          │  psql-j26apps-    │    │  kv-j26apps-shared-sdc   │
          │  shared-sdc       │    │  (via CSI driver)        │
          │  j26_dev_scoutid  │    └─────────────────────────-┘
          └───────────────────┘
```

## Secrets (Azure Key Vault via CSI)

All secrets are sourced from `kv-j26apps-shared-sdc` via the Secrets Store CSI
driver (`15-secret-provider-class.yaml`). Mounting the CSI volume in a pod is
what triggers materialization of the k8s Secrets — no manual `kubectl create
secret` needed.

| KV secret name | k8s Secret | Key |
|---|---|---|
| `j26-dev-scoutid-postgres-user` | `keycloak-db-csi` | `KC_DB_USERNAME` |
| `j26-dev-scoutid-postgres-password` | `keycloak-db-csi` | `KC_DB_PASSWORD` |
| `j26-dev-scoutid-keycloak-bootstrap-admin-username` | `keycloak-admin-csi` | `KC_BOOTSTRAP_ADMIN_USERNAME` |
| `j26-dev-scoutid-keycloak-bootstrap-admin-password` | `keycloak-admin-csi` | `KC_BOOTSTRAP_ADMIN_PASSWORD` |
| `j26-dev-scoutid-keycloak-master-admin-username` | `keycloak-admin-csi` | `MASTER_ADMIN_USERNAME` |
| `j26-dev-scoutid-keycloak-master-admin-password` | `keycloak-admin-csi` | `MASTER_ADMIN_PASSWORD` |

**Bootstrap admin** (`KC_BOOTSTRAP_ADMIN_*`): used only during initial startup and
by the `keycloak-config` Job. Can be disabled in the Keycloak admin console once
`j26-keycloak-admin` is in place.

**Master admin** (`MASTER_ADMIN_*`): the permanent `j26-keycloak-admin` service
account created by the `ensure-master-admin` Job. Use this for day-to-day admin
console access at `https://admin.id.dev.j26.se`.

To retrieve the master admin password:
```bash
az keyvault secret show \
  --vault-name kv-j26apps-shared-sdc \
  --name j26-dev-scoutid-keycloak-master-admin-password \
  --query value -o tsv
```

## Files

| File | What it does |
|---|---|
| `00-namespace.yaml` | `j26-keycloak-scoutid` namespace |
| `15-secret-provider-class.yaml` | CSI SecretProviderClass — maps KV secrets to k8s Secrets |
| `20-keycloak.yaml` | Service + Deployment (upstream image, init container waits for Postgres) |
| `30-ingressroute.yaml` | Traefik IngressRoutes, TLS certs, root-redirect Middleware |
| `40-config-cli.yaml` | ArgoCD Sync hook Job — applies realm config via keycloak-config-cli |
| `45-keycloak-config-configmap.yaml` | ConfigMap containing the realm config YAML files (see below) |
| `50-servicemonitor.yaml` | Prometheus ServiceMonitor (scrapes `:9000/metrics`) |
| `55-ensure-master-admin-job.yaml` | ArgoCD PostSync hook Job — creates `j26-keycloak-admin` if absent |

## ArgoCD sync order

ArgoCD applies all resources in a single sync wave, with two hook Jobs:

1. **Sync wave (default)**: namespace, CSI class, Deployment, Ingress, ConfigMap,
   ServiceMonitor all applied in parallel.
2. **`keycloak-config` Job** (`hook: Sync`): runs on every sync. Polls until
   Keycloak is ready (up to 300 s via `KEYCLOAK_AVAILABILITYCHECK_ENABLED`), then
   applies the four config files from the ConfigMap.
3. **`ensure-master-admin` Job** (`hook: PostSync`): runs after all resources are
   healthy. Creates the `j26-keycloak-admin` master-realm admin user if it doesn't
   exist yet (idempotent).

Both Jobs have `hook-delete-policy: BeforeHookCreation` so they re-run on every
sync and are cleaned up (`ttlSecondsAfterFinished: 600`) afterwards.

## Realm config (ConfigMap)

The `jamboree26` realm is configured declaratively via
[keycloak-config-cli](https://github.com/adorsys/keycloak-config-cli). The four
source files live in
[j26-keycloak/keycloak-config/](https://github.com/Scouterna/j26-keycloak/tree/main/keycloak-config):

| File | Content |
|---|---|
| `01-realm.yaml` | Realm settings, token lifetimes, user profile attributes |
| `02-authentication.yaml` | ScoutID browser login flow (scoutnet-authenticator) |
| `03-scopes.yaml` | OIDC scopes (profile + scoutnet_member_no, email, phone, memberships) |
| `04-clients.yaml` | Clients: account console, scout-test-client |

`45-keycloak-config-configmap.yaml` is **generated** from those files. When any
source file changes, regenerate it:

```bash
kubectl create configmap keycloak-config \
  --namespace j26-keycloak-scoutid \
  --from-file=keycloak-config \
  --dry-run=client -o yaml \
  > k8s/app-manifest/j26-keycloak-scoutid/45-keycloak-config-configmap.yaml
```

(Run this from the root of `j26-keycloak-scoutid`, then commit the result here.)

## Monitoring

Keycloak exposes Prometheus metrics on port `9000` at `/metrics`
(`KC_METRICS_ENABLED=true`, baked into the upstream image). The `ServiceMonitor`
picks these up via the `kube-prometheus-stack` Prometheus instance (label
`release: kps`).

Per-user-event metrics (`keycloak_logins_total` etc.) require the
`user-event-metrics` build-time feature — not yet in `:latest`. Once upstream PR
[scoutid-keycloak#1](https://github.com/Scouterna/scoutid-keycloak/pull/1) lands
and the image is rebuilt, add to `20-keycloak.yaml`:

```yaml
- name: KC_EVENT_METRICS_USER_ENABLED
  value: "true"
- name: KC_EVENT_METRICS_USER_TAGS
  value: realm,idp,clientId
```

## Ingress / hostnames

| Hostname | Purpose | Accessible paths |
|---|---|---|
| `id.dev.j26.se` | Member-facing login | `/` (→ 301 Scouterna), `/realms/*`, `/resources/*` |
| `admin.id.dev.j26.se` | Admin console | `/*` |

TLS is issued by cert-manager (`letsencrypt-traefik` ClusterIssuer). The admin
host can be IP-gated by enabling the `admin-ip-allowlist` Middleware defined in
`30-ingressroute.yaml`.
