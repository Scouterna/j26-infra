This directory contains configuration files for ArgoCD.

ArgoCD was installed by following the setup guide and not using Helm: https://argo-cd.readthedocs.io/en/stable/getting_started/

ArgoCD has multiple services running on the same port and requires certificates
to be directly configured in the ArgoCD Server. cert-manager is therefore
configured to provide the certifiates to ArgoCD directly, and ssl-passthrough is
abled for the Nginx ingress.
