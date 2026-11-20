# argocd-configs

This repository contains the **GitOps** configuration for the K3s HA cluster. It follows the **App‑of‑Apps** pattern using Argo CD.

- **Base**: Argo CD installation and common resources.
- **Apps**: Individual components (MetalLB, Contour, Cert‑Manager, External Secrets, External DNS, Cloudflared).

All manifests are managed with **Kustomize** and Helm releases via the **HelmRelease** CRD (when applicable).

## Finance Open Finance

The `apps` chart discovers `finance-openfinance-develop` through a static
ApplicationSet. The generated Application uses manual sync.

Before the first sync, the operator must supply the published image by digest in
the ApplicationSet template at `spec.template.spec.source.kustomize.images`,
using `ghcr.io/glauciolabs/finance-openfinance@sha256:<digest>`. The develop
overlay has an intentionally invalid placeholder digest. The publishing pipeline
does not manage this configuration.

The `family-finance` namespace needs the existing `ghcr-secret`,
`family-finance-secrets`, and `finance-bff-tls` Secrets. Provision the
`finance-openfinance-secrets` Secret with the three Pluggy keys referenced by
the Deployment. The Traefik `websecure` entry point serves the exact
`POST /api/v1/integrations/pluggy/webhook` route on `finance.glauciolabs.work`
using the existing finance TLS certificate.
