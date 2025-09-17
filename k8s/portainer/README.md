# Portainer

*Portainer unifies Kubernetes, Docker, and Podman environments into one intuitive platform, engineered for both scale and edge deployment. From GitOps to observability, security to provisioning,  portainer removes the overhead of managing containers, so your teams move faster, operate safer, and focus on what matters: delivering business outcomes.*<br>
**https://www.portainer.io/**

## Installation

We use [Helm](https://helm.sh/) to install Portainer in the cluster. The helm chart values are in the [portainer-values.yaml](portainer-values.yaml) file.
```sh
helm install portainer portainer --repo https://portainer.github.io/k8s/ --values portainer-values.yaml --namespace portainer --create-namespace
```

The values file enables the enterprise version, enable persistent storage, sets the requested storage size to a minimum and enables Portainer to be reached at https://portainer.app.j26.se through the cluster Ingress.

## Enterprise key
To user the enterprise version, a [free three node key](https://www.portainer.io/take-3) has been requested. The key is not stored in the repo. @hakan-persson has a copy of the key. The key is entered in the Portianer GUI during installation.

## Authentication

After installation, the Portainer app is configured to use Github OAuth authentication using a client in Scouterna Github organization. An account is created in Portainer when a user connects for the first time, but all permissions must be configured manually by a Portainer admin.