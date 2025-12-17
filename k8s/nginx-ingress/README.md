# Ingress Controller and Certificate Manager

## Nginx Ingress 
An [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) makes it possible to route traffic to different pods based on hostname and path values using a single entry point. The entry point is typical an external IP-address retrieved from a Load balancer. The ingress controller exposes ports 80 and 433.

Apps can register with the Nginx ingress controller using an [ingress manifest](https://kubernetes.io/docs/concepts/services-networking/ingress/) with the "ingressClassName" set to "nginx".

### Installation
The Nginx ingress controller is installed using [Helm](hhttps://kubernetes.github.io/ingress-nginx/deploy/).

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --version 4.14.0 \
  --set-string controller.config.enable-access-log-for-default-backend="true" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
  --set controller.enableSnippets=true \
  --namespace ingress-nginx \
  --create-namespace
```

## Certificate manager
Typically one wants to use TLS (https) to communicate with the app. It is possible to have TLS terminate locally in the pod. But it's more common to terminate TLS in the ingress controller proxy and use plain http from the proxy to the pod. Either way, a Certificate Manager can be used to issue the TLS certificate used. 

We use [cert-manager](https://cert-manager.io/) to issue certificates and integrates it with the ingress controller to terminate TLS in the ingress.


### Installation
We also use [Helm](https://cert-manager.io/docs/installation/helm/) here:
```bash
helm install cert-manager cert-manager \
  --repo https://charts.jetstack.io \
  --version v1.19.1 \
  --set crds.enabled=true \
  --namespace cert-manager
```

To issue valid certificates automatically, we install a [Cluster Issuer](https://cert-manager.io/docs/concepts/issuer/) that use [Let's Encrypt](https://letsencrypt.org/).

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: "hakan.persson@scouterna.se"
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - http01:
        ingress:
          class: nginx
          podTemplate:
            spec:
              nodeSelector:
                "kubernetes.io/os": linux
EOF
```

Let's Encrypt certificates in the ingress is requested using the annotation _cert-manager.io/cluster-issuer: "letsencrypt"_ in the apps ingress manifest.

