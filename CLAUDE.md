# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Infrastructure-as-code for Jamboree26 (Scouterna). All application deployments are managed via ArgoCD reading from this Git repository — there are no build or test commands. Changes take effect when pushed to `main` and ArgoCD syncs.

## Repository Structure

```
k8s/
  app-manifest/       # Per-app Kubernetes manifests (Kustomize)
  infra-manifest/     # Cluster infrastructure (monitoring, image updater)
  argocd/
    apps/             # ArgoCD Application CRDs for each microservice
    infra-apps/       # ArgoCD Applications for infrastructure components
    projects/         # ArgoCD AppProject definitions (apps, infra)
    infra-root-app.yaml  # App-of-apps root for infrastructure
```

Each app in `app-manifest/` has its own namespace and typically contains: `deployment.yaml`, `service.yaml`, `ingress.yaml`, `configmap.yaml`, `secret-provider-class.yaml`, `kustomization.yaml`.

## Naming Conventions

- Apps: `j26-{service-name}` (e.g. `j26-booking`, `j26-strapi`)
- Images: `ghcr.io/scouterna/{app-name}:latest`
- Migration images: `ghcr.io/scouterna/{app-name}-migrate:latest`
- ConfigMaps: `{app-name}-config`
- App secrets (from KV): `{app-name}-secrets`
- SecretProviderClass: `{app-name}-kv-csi`
- PostgreSQL secrets (pre-created, shared): `sc-postgresql{id}-secret`

Each app is deployed into its own namespace matching its app name.

## Secrets: Azure Key Vault via CSI Driver

All app secrets come from Azure Key Vault (`kv-j26apps-shared-sdc`) using the Secrets Store CSI Driver. Every app with secrets needs:

1. A `SecretProviderClass` that pulls named KV secrets and materialises them as a Kubernetes `Opaque` secret
2. A CSI volume mounted at `/mnt/secrets-store` in the pod (this mount is what triggers secret sync)
3. `secretRef` in `envFrom` pointing to `{app-name}-secrets`

Common parameters across all `SecretProviderClass` resources:
```yaml
useVMManagedIdentity: "true"
userAssignedIdentityID: 214abfc2-fa80-44b9-89c5-f78444a16816
keyvaultName: kv-j26apps-shared-sdc
tenantId: 317a47ba-fd32-41b8-8ebe-310a1adc9863
```

KV secret names follow the pattern `{app-name}-{env-var-name-lowercased-with-hyphens}`.

## Database Access

PostgreSQL secrets are pre-created in each namespace by Azure Service Connector — they are not managed in this repo. To set up database access for a new app:

1. Create the database in Azure Portal (on the PostgreSQL flexible server)
2. Create a PostgreSQL user via DBeaver and grant full access to the `public` schema on the specific database only (not server-wide)
3. In the AKS cluster → Service Connector, create a new connection using type **Connection string**, targeting the new app's namespace

Apps reference the resulting secret by name and key:

- `sc-postgresql531ff-secret` — used by j26-platsbank; keys: `AZURE_POSTGRESQL_HOST`, `AZURE_POSTGRESQL_PORT`, `AZURE_POSTGRESQL_DATABASE`, `AZURE_POSTGRESQL_USER`, `AZURE_POSTGRESQL_PASSWORD`
- `sc-postgresql7aaa4-secret` — used by j26-strapi; same keys as above
- `sc-postgresqla5566-secret` — used by j26-booking; keys use `AZURE_POSTGRESQL_USERNAME` (not `USER`)

Apps that use a `DATABASE_URL` (e.g. Prisma) compose it via env var interpolation:
```yaml
value: "postgresql://$(AZURE_POSTGRESQL_USER):$(AZURE_POSTGRESQL_PASSWORD)@$(AZURE_POSTGRESQL_HOST):$(AZURE_POSTGRESQL_PORT)/$(AZURE_POSTGRESQL_DATABASE)?sslmode=require"
```

Apps that take individual DB vars (e.g. Strapi) map them with `valueFrom.secretKeyRef`.

## Ingress

All apps are served under `app.dev.j26.se` using path-based routing with `nginx.ingress.kubernetes.io/rewrite-target`. The `j26-app` service handles the root `/`. Other services use `/_services/{name}(/|$)(.*)` paths.

## ArgoCD Image Updater

`k8s/infra-manifest/argocd-image-updater/image-updater.yaml` — custom `ImageUpdater` CRD. When adding a new app with an auto-updated image, add a `namePattern` entry here. All apps use `updateStrategy: "newest-build"`.

## Adding a New App

1. Create `k8s/app-manifest/{app-name}/` with `kustomization.yaml`, `deployment.yaml`, `service.yaml`, `ingress.yaml`, `configmap.yaml`
2. Add `secret-provider-class.yaml` if the app needs secrets from Azure KV
3. Create `k8s/argocd/apps/{app-name}.yaml` (ArgoCD Application pointing at the manifest path)
4. Add a `namePattern` entry to the image updater config
5. KV secrets must be manually created in Azure Key Vault before the first deploy

## Database Migrations

Apps that need migrations use an `initContainer` running before the main container. See `j26-platsbank/deployment.yaml` (Prisma) or `j26-booking/deployment.yaml` for reference. The init container uses the same image and mounts the same secrets as the main container.
