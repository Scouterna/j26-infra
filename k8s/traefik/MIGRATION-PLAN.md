# ingress-nginx → Traefik Migration Plan

## Background

ingress-nginx was officially retired in March 2026:
https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/

No further releases, bug fixes, or security updates will be made. We need to migrate to a
supported ingress controller.

**Chosen replacement: Traefik v3**

Traefik is a fully conformant Gateway API implementation that also supports the classic
Ingress API. This means we can migrate ingresses one at a time while both controllers run
in parallel. The same migration has already been completed on the webservices cluster
(Traefik v3.6.13, Helm chart 39.0.8).

---

## Current State

### ingress-nginx
- Version: 4.14.0
- Namespace: `ingress-nginx`
- Installed via Helm manually (not ArgoCD managed)
- Documented in `k8s/nginx-ingress/README.md`

### cert-manager
- Version: v1.19.1
- Installed via Helm manually (not ArgoCD managed)
- ClusterIssuer `letsencrypt` created manually, uses `class: nginx` as http01 solver

### Ingresses (8 total in git + 2 in cluster but missing from git)

| File | Namespace | Host | Notes |
|------|-----------|------|-------|
| `k8s/argocd/ingress.yaml` | argocd | argocd.dev.j26.se | ssl-passthrough, needs IngressRoute |
| `k8s/infra-manifest/monitoring/helm-values.yaml` | monitoring | grafana.j26.se | inside Helm values |
| `k8s/app-manifest/j26-app/ingress.yaml` | j26-app | app.dev.j26.se | simple, no rewrite |
| `k8s/app-manifest/j26-auth/ingress.yaml` | j26-auth | app.dev.j26.se/auth | path rewrite |
| `k8s/app-manifest/j26-booking/ingress.yaml` | j26-booking | app.dev.j26.se/_services/booking | path rewrite |
| `k8s/app-manifest/j26-cms/ingress.yaml` | j26-cms | app.dev.j26.se/_services/cms | path rewrite + proxy-body-size |
| `k8s/app-manifest/j26-map/ingress.yaml` | j26-map | app.dev.j26.se/_services/map | path rewrite |
| `k8s/app-manifest/j26-platsbank/ingress.yaml` | j26-platsbank | app.dev.j26.se/_services/platsbank | path rewrite (keeps prefix) |
| `k8s/app-manifest/j26-signupinfo/ingress.yaml` | j26-signupinfo | app.dev.j26.se/_services/signupinfo | path rewrite + x-forwarded-prefix |
| missing from git | j26-notifications | app.dev.j26.se/notifications | needs new file |
| missing from git | j26-infratest | app.dev.j26.se/infratest | needs new file |

---

## Migration Steps

### Step 1 — Install Traefik alongside ingress-nginx

Traefik is installed the same way as ingress-nginx — manually via Helm, not ArgoCD managed.
Document in `k8s/traefik/README.md` (to be created).

Install Gateway API CRDs first (required for Traefik's Gateway provider):
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

Install Traefik:
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

kubectl create namespace traefik

helm install traefik traefik/traefik \
  -n traefik \
  -f k8s/traefik/traefik-values.yaml \
  --version 39.0.8
```

Key settings in `traefik-values.yaml` (file to be created in this PR or a follow-up):
- `ingressClass.isDefaultClass: false` — don't steal existing nginx ingresses
- Both classic Ingress provider and Gateway API provider enabled
- `service.type: LoadBalancer`
- HTTP → HTTPS redirect enabled globally

Note the new LoadBalancer IP after deploy — needed for DNS update.

### Step 2 — Update ClusterIssuer

Add a `traefik` http01 solver alongside the existing `nginx` solver so cert-manager
can issue certificates for both controllers during the transition period:

```bash
kubectl edit clusterissuer letsencrypt
```

Add under `spec.acme.solvers`:
```yaml
- http01:
    ingress:
      class: traefik
      podTemplate:
        spec:
          nodeSelector:
            "kubernetes.io/os": linux
```

cert-manager automatically selects the correct solver based on the ingress class of
each certificate request.

### Step 3 — Update DNS

Point DNS records at the new Traefik LoadBalancer IP. Both controllers run simultaneously
so there is no downtime — existing nginx ingresses keep working on the old IP until
each one is migrated.

Records to update:
- `app.dev.j26.se`
- `argocd.dev.j26.se`
- `grafana.j26.se`

### Step 4 — Migrate ingresses (commit to git, ArgoCD applies)

Migrate in this order — commit and verify each change before proceeding to the next.

#### 4a. `monitoring/grafana` — easy

Edit `k8s/infra-manifest/monitoring/helm-values.yaml`:
```yaml
# Change:
ingressClassName: nginx
# To:
ingressClassName: traefik
```
Remove any nginx-specific annotations. No path rewrite needed.

#### 4b. `j26-app` — easy

Edit `k8s/app-manifest/j26-app/ingress.yaml`:
```yaml
# Change:
ingressClassName: nginx
# To:
ingressClassName: traefik
```
Remove `nginx.ingress.kubernetes.io/backend-protocol` annotation (not needed for Traefik).

#### 4c. Path-rewrite services — medium

The following services use nginx's `rewrite-target` annotation which has no direct
equivalent in the standard Ingress API. Each needs a Traefik `Middleware` CRD resource
and a reference to it in the ingress annotation.

**Services with simple strip-prefix rewrite** (`rewrite-target: /$2`):
`j26-auth`, `j26-booking`, `j26-map`, `j26-notifications`, `j26-infratest`

For each, add a `Middleware` resource in the same namespace:
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: strip-prefix
  namespace: j26-auth          # change per service
spec:
  stripPrefix:
    prefixes:
      - /auth                  # change to the path prefix for this service
```

And update the ingress:
```yaml
ingressClassName: traefik
annotations:
  traefik.ingress.kubernetes.io/router.middlewares: j26-auth-strip-prefix@kubernetescrd
# Remove nginx annotations
```
Change `path` from `/auth(/|$)(.*)` to `/auth`, `pathType: Prefix`.

**Services that keep the prefix in the rewrite** (`rewrite-target: /_services/cms/$2`):
`j26-cms`, `j26-platsbank`

Use `ReplacePathRegex` middleware instead:
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: replace-path
  namespace: j26-cms
spec:
  replacePathRegex:
    regex: ^/_services/cms/(.*)
    replacement: /_services/cms/$1
```

**`j26-signupinfo`** — has both a rewrite AND `x-forwarded-prefix` header injection.
Needs two middlewares chained together:
1. `stripPrefix` for the path rewrite
2. `headers` middleware to inject `X-Forwarded-Prefix: /_services/signupinfo`

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: signupinfo-headers
  namespace: j26-signupinfo
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Prefix: /_services/signupinfo
```

Reference both in the ingress annotation (comma-separated):
```yaml
traefik.ingress.kubernetes.io/router.middlewares: >
  j26-signupinfo-strip-prefix@kubernetescrd,
  j26-signupinfo-signupinfo-headers@kubernetescrd
```

**`j26-cms`** — also has `proxy-body-size: 50m`. Add a `buffering` middleware:
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: cms-buffering
  namespace: j26-cms
spec:
  buffering:
    maxRequestBodyBytes: 52428800   # 50 MiB
```

#### 4d. ArgoCD — replace Ingress with IngressRoute CRD

Per the official ArgoCD docs for Traefik v3:
https://argo-cd.readthedocs.io/en/latest/operator-manual/ingress/#traefik-v30

Traefik v3 can terminate both TCP and HTTP on the same port, so ssl-passthrough is not
needed. Instead, ArgoCD runs in insecure mode (Traefik handles external TLS) and Traefik
uses separate routes for HTTPS and gRPC traffic.

**Step 4d-1:** Enable insecure mode in ArgoCD by adding `--insecure` to the argocd-server
args, or setting in the argocd-cm ConfigMap:
```yaml
data:
  server.insecure: "true"
```

**Step 4d-2:** Replace `k8s/argocd/ingress.yaml` with an `IngressRoute` CRD:
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`argocd.dev.j26.se`) && Headers(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argocd-server
          port: 80
          scheme: h2c
    - kind: Rule
      match: Host(`argocd.dev.j26.se`)
      priority: 10
      services:
        - name: argocd-server
          port: 80
  tls:
    secretName: argocd-server-tls
```

Also keep cert-manager managing the certificate by retaining a minimal `Certificate`
resource (or keep a stripped-down Ingress just for the ACME challenge).

#### 4e. Add missing ingress manifests to git

`j26-notifications` and `j26-infratest` have ingresses in the live cluster but no
manifest files in the repo. Before migrating them, check the live cluster:
```bash
kubectl get ingress -n j26-notifications -o yaml
kubectl get ingress -n j26-infratest -o yaml
```
Create `k8s/app-manifest/j26-notifications/ingress.yaml` and
`k8s/app-manifest/j26-infratest/ingress.yaml` with Traefik annotations directly
(no need to first create nginx versions).

### Step 5 — Remove ingress-nginx

Once all ingresses are migrated and verified:

```bash
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
```

Update `k8s/nginx-ingress/README.md` to note that nginx has been replaced by Traefik,
and update the ClusterIssuer to remove the `nginx` solver.

---

## Annotation mapping reference

| nginx annotation | Traefik equivalent |
|---|---|
| `ingressClassName: nginx` | `ingressClassName: traefik` |
| `rewrite-target: /$2` | `stripPrefix` Middleware |
| `rewrite-target: /prefix/$2` | `replacePathRegex` Middleware |
| `proxy-body-size: 50m` | `buffering` Middleware (`maxRequestBodyBytes`) |
| `x-forwarded-prefix: /path` | `headers` Middleware (`customRequestHeaders`) |
| `backend-protocol: http` | not needed (Traefik default) |
| `ssl-passthrough: true` | IngressRoute with ArgoCD in insecure mode |
| `cert-manager.io/cluster-issuer: letsencrypt` | unchanged |

---

## Verification checklist

- [ ] Traefik pod running, LoadBalancer IP assigned
- [ ] ClusterIssuer updated with `traefik` solver
- [ ] DNS updated to Traefik IP
- [ ] `grafana.j26.se` loads with valid TLS
- [ ] `app.dev.j26.se` loads (j26-app frontend)
- [ ] Path routing works: `/auth`, `/_services/booking`, `/_services/cms`, etc.
- [ ] `argocd.dev.j26.se` loads in browser
- [ ] ArgoCD CLI connects via gRPC (`argocd login argocd.dev.j26.se`)
- [ ] cert-manager issues new certificates for all hosts
- [ ] `j26-notifications` and `j26-infratest` ingress files added to git
- [ ] ingress-nginx uninstalled
- [ ] ClusterIssuer `nginx` solver removed
