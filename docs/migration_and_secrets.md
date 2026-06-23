# Migration to GitHub & Secrets Strategy Documentation

## Overview
This repository hosts the full **GitOps** configuration for the Argo CD deployment in the `glauciolabs.work` environment.  All Kubernetes manifests, Helm charts, and secret‑mapping definitions are version‑controlled in Git and applied automatically by Argo CD.

---

## 1. Why GitHub for Configuration?
- **Single Source of Truth** – All manifests live in a GitHub repo (`github.com/argocd-configs`).  Changes are reviewed via pull‑requests, ensuring auditability.
- **GitOps Automation** – Argo CD continuously syncs the desired state from this repo to the cluster, providing drift detection and automated roll‑outs.
- **Secret Management** – Sensitive material never lives in plain‑text in the repo.  We use a two‑step approach:
  1. **GitHub Secrets** – Store raw secrets (tokens, client secrets) in the GitHub repository's *Secrets* store.
  2. **SealedSecrets / Secret‑Mapping** – Mapping files translate those GitHub Secrets into Kubernetes `Secret` resources using the **`sealed-secrets`** controller, making the actual secret data encrypted and safe to commit.

---

## 2. Secret Mapping Architecture
### 2.1 `secret-mapping.yaml`
```yaml
secrets:
  - name: cloudflare-api-token
    ghSecret: PROD_CLOUDFLARE_TOKEN
    namespace: external-dns
  - name: cloudflare-api-token
    ghSecret: PROD_CLOUDFLARE_TOKEN
    namespace: cert-manager
  - name: cloudflared-credentials
    ghSecret: PROD_CLOUDFLARE_TUNNEL_TOKEN
    namespace: cloudflare
  - name: docker-registry-token
    ghSecret: PROD_DOCKER_REGISTRY_TOKEN
    namespace: default
  - name: oidc-azure-clientsecret
    ghSecret: ARGOCD_PROD_CLIENT_SECRET
    namespace: argocd
    key: oidc.azure.clientSecret
    extraStringData:
      oidc.azure.allowedAudiences: "f7bd2200-f109-4e71-9231-161bbc03ce39"
    labels:
      app.kubernetes.io/part-of: argocd
```
- **`ghSecret`** – The name of the secret stored in GitHub.
- **`namespace`** – Target Kubernetes namespace.
- **`key`** – The exact key that will appear inside the generated `Secret` data map.
- **`extraStringData`** – Additional key/value pairs that are not secret (e.g., allowed audience list) and can be added as plain string data.
- **`labels`** – Optional labeling for easier selection and monitoring.

The *Secret‑Mapping* action runs as part of the CI pipeline (GitHub Actions). It pulls the GitHub secret value, base‑64‑encodes it, creates a Helm values file, and finally produces a **SealedSecret** that is checked into the repo.

---

## 3. OIDC Azure SSO Integration
1. **Azure AD App Registration** – Provides `clientID` and `clientSecret`.
2. **GitHub Secret** – `ARGOCD_PROD_CLIENT_SECRET` stores the *raw* client secret (not base‑64 encoded).
3. **Mapping** – The entry above creates a secret named `oidc-azure-clientsecret` in the `argocd` namespace, with the key `oidc.azure.clientSecret`.
4. **Argo CD ConfigMap (`argocd-cm.yaml`)** references the secret using Argo CD's secret‑reference syntax:
   ```yaml
   oidc.config: |
     name: Azure
     issuer: https://login.microsoftonline.com/<tenant-id>/v2.0
     clientID: <client-id>
     clientSecret: $oidc-azure-clientsecret:oidc.azure.clientSecret
   ```
   This indirection means the secret can be rotated in GitHub without touching the ConfigMap.

---

## 4. Rotation Procedure (Summary)
1. **Generate New Client Secret** in Azure AD.
2. **Update GitHub Secret** `ARGOCD_PROD_CLIENT_SECRET` with the new value.
3. **Merge Pull‑Request** that updates `secret-mapping.yaml` (if the mapping changed) – the CI pipeline will re‑seal the secret.
4. **Argo CD Sync** – The new `SealedSecret` is applied, creating/patching `oidc-azure-clientsecret`.
5. **Rollout Restart** of `argocd-server` to pick up the updated secret:
   ```bash
   kubectl rollout restart deployment/argocd-server -n argocd
   ```
6. **Validate** – Log in via Azure SSO and verify the flow works without 307 loops.

---

## 5. Additional Security Practices
- **Least‑privilege GitHub tokens** – Use repository‑scoped tokens for CI actions.
- **RBAC** – The `sealed-secrets` controller runs with limited permissions; only namespaces listed in `secret-mapping.yaml` can receive sealed secrets.
- **Audit Logging** – All secret‑mapping changes are captured in PR history.
- **Versioned SealedSecrets** – Each rotation creates a new version, allowing rollback if needed.

---

## 6. Where This Documentation Lives
- Repository path: `github/argocd-configs/docs/migration_and_secrets.md`
- The file is part of the GitOps repo so any collaborator can read the exact steps and rationale.

---

*Document created on 2026‑06‑23.*
