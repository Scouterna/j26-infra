---
name: new-app
description: Scaffold a new j26 microservice — creates all manifests, ArgoCD Application, and image updater entry.
argument-hint: [app-name]
disable-model-invocation: true
---

Ask the user for the following before creating any files. Collect all answers first, then act in one go:

1. **App name** — used as `j26-{name}` (e.g. `booking` → `j26-booking`). If provided as `$ARGUMENTS`, use that and skip asking.
2. **Container port** — what port does the container listen on?
3. **Ingress path** — path under `app.dev.j26.se` (e.g. `/_services/foo` or `/foo`)
4. **Database access** — needs PostgreSQL (`sc-postgresql531ff-secret`)? If yes: `DATABASE_URL` string (Prisma/single-env style) or individual `DATABASE_*` vars (Strapi style)?
5. **Migrations** — run an initContainer before the main container?
6. **Azure KV secrets** — env var names of secrets needed from Azure KV (e.g. `SECRET_KEY_BASE`, `JWT_SECRET`). Blank if none.
7. **Extra configmap entries** — any non-sensitive env vars for the configmap?

---

Once you have answers, create all files below. Never create `secret-provider-class.yaml` if no KV secrets were requested.

## Files to create

### `k8s/app-manifest/j26-{name}/configmap.yaml`

Include any non-sensitive env vars the user listed. Empty `data: {}` if none.

### `k8s/app-manifest/j26-{name}/secret-provider-class.yaml`

Only if KV secrets were requested. Copy the structure from `k8s/app-manifest/j26-booking/secret-provider-class.yaml`.

Fixed values shared by all apps:
- `userAssignedIdentityID: 214abfc2-fa80-44b9-89c5-f78444a16816`
- `keyvaultName: kv-j26apps-shared-sdc`
- `tenantId: 317a47ba-fd32-41b8-8ebe-310a1adc9863`

KV secret name convention: `j26-{name}-{env-var-lowercased-with-hyphens}`
(e.g. `SECRET_KEY_BASE` → `j26-{name}-secret-key-base`)

The materialised Kubernetes secret name must be `j26-{name}-secrets`.

### `k8s/app-manifest/j26-{name}/deployment.yaml`

- Only add an `initContainer` if migrations were requested.
- If KV secrets: add CSI `volumeMounts` (`/mnt/secrets-store`, readOnly) and a `volumes` block referencing `j26-{name}-kv-csi`. Reference the materialised secret via `secretRef: name: j26-{name}-secrets` in `envFrom`.
- **DATABASE_URL style**: add `secretRef: name: sc-postgresql531ff-secret` in `envFrom` and compose `DATABASE_URL` via env var interpolation (see `k8s/app-manifest/j26-platsbank/deployment.yaml`).
- **Individual DB vars style**: map each with `valueFrom.secretKeyRef` from `sc-postgresql531ff-secret` using keys `AZURE_POSTGRESQL_HOST`, `AZURE_POSTGRESQL_PORT`, `AZURE_POSTGRESQL_DATABASE`, `AZURE_POSTGRESQL_USER`, `AZURE_POSTGRESQL_PASSWORD`.

### `k8s/app-manifest/j26-{name}/service.yaml`

Standard Service: port 80 → container port.

### `k8s/app-manifest/j26-{name}/ingress.yaml`

Use `nginx.ingress.kubernetes.io/rewrite-target`, `pathType: ImplementationSpecific`, host `app.dev.j26.se`. Follow the rewrite pattern from other services.

### `k8s/app-manifest/j26-{name}/kustomization.yaml`

List all created files as resources. Include `secret-provider-class.yaml` only if it was created.

### `k8s/argocd/apps/j26-{name}.yaml`

Clone from `k8s/argocd/apps/j26-platsbank.yaml`. Set `name`, `path`, and `namespace` to `j26-{name}`.

### Update `k8s/infra-manifest/argocd-image-updater/image-updater.yaml`

Add a `namePattern` entry for `j26-{name}` tracking `ghcr.io/scouterna/j26-{name}` with `updateStrategy: newest-build`. If the app has a separate migration image, add a second entry with alias `migrate` tracking `ghcr.io/scouterna/j26-{name}-migrate`.

---

After creating all files, remind the user to **manually add any Azure Key Vault secrets** for this app before the first deploy.
