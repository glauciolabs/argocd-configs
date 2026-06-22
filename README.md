# argocd-configs

This repository contains the **GitOps** configuration for the K3s HA cluster. It follows the **App‑of‑Apps** pattern using Argo CD.

- **Base**: Argo CD installation and common resources.
- **Apps**: Individual components (MetalLB, Contour, Cert‑Manager, External Secrets, External DNS, Cloudflared).

All manifests are managed with **Kustomize** and Helm releases via the **HelmRelease** CRD (when applicable).
