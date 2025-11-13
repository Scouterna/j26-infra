## Initial deployment

Create a project for each application/repository:
```bash
# Do this for every project
argocd proj create -f projects/j26-auth.yaml
```

Then deploy the app-of-apps:
```bash
argocd app create apps \
    --dest-namespace argocd \
    --dest-server https://kubernetes.default.svc \
    --repo https://github.com/scouterna/j26-infra.git \
    --path k8s/argocd-apps/apps

argocd app sync apps
```
