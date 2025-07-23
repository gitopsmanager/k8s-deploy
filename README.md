# 🚀 Reusable GitHub Workflow: Kubernetes App Deployment with ArgoCD

This GitHub Actions reusable workflow automates Kubernetes application deployment using [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/). It renders and templates your manifest files, commits the results to your continuous deployment repo, and uses the ArgoCD REST API to create and sync applications.

> ✅ This workflow **always creates the ArgoCD Application** if it does not already exist — ensuring first-time deployments are fully automated.

---

## ⚙️ Inputs

| Name                  | Required | Type    | Description                                                                 |
|-----------------------|----------|---------|-----------------------------------------------------------------------------|
| `runner`              | ❌       | string  | GitHub runner label (default: `ubuntu-latest`)                              |
| `cd_repo`             | ✅       | string  | Continuous deployment repo that stores templated manifests                  |
| `environment`         | ✅       | string  | Logical environment name (e.g. `dev`, `prod`)                               |
| `application`         | ✅       | string  | Application name (used as part of ArgoCD app name)                          |
| `namespace`           | ✅       | string  | Kubernetes namespace to deploy into                                         |
| `repo`                | ✅       | string  | GitHub repo with source manifests (e.g. `my-org/my-app`)                   |
| `repo_path`           | ✅       | string  | Path to manifest files (e.g. `manifests/`)                                  |
| `repo_commit_id`      | ❌       | string  | Optional commit SHA to deploy (overrides `branch_name`)                     |
| `branch_name`         | ❌       | string  | Git branch to checkout (default: `main`)                                    |
| `overlay_dir`         | ✅       | string  | Subdirectory under `overlays/` to use with Kustomize                        |
| `image_tag`           | ❌       | string  | Image tag to apply to container image(s)                                    |
| `image_base_name`     | ❌       | string  | Single image base name to patch (e.g. `ghcr.io/org/app`)                    |
| `image_base_names`    | ❌       | string  | Comma-separated list of image base names                                    |
| `kustomize_version`   | ❌       | string  | Kustomize version to install (default: `5.0.1`)                              |
| `skip_status_check`   | ❌       | string  | If `'true'`, skips waiting for ArgoCD sync to finish                        |
| `insecure_argo`       | ❌       | boolean | If `true`, disables TLS verification for ArgoCD endpoints                   |
| `debug`               | ❌       | boolean | If `true`, prints debug output of generated files and paths                 |

---

## 🔐 Required Secrets

| Name                                 | Description                                         |
|--------------------------------------|-----------------------------------------------------|
| `ARGO_CD_ADMIN_USER`                 | ArgoCD admin username                               |
| `ARGO_CD_ADMIN_PASSWORD`             | ArgoCD admin password                               |
| `CONTINUOUS_DEPLOYMENT_GH_APP_ID`   | GitHub App ID used to access the CD repo            |
| `CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY` | GitHub App private key                      |

### 🧩 Optional Secrets

| Name               | Description                                                                 |
|--------------------|-----------------------------------------------------------------------------|
| `ARGOCD_CA_CERT`   | PEM-formatted CA certificate to verify ArgoCD's TLS endpoint (if not using `insecure_argo`) |

> ⚠️ **TLS Verification**:  
> You must either set `insecure_argo: true` to skip TLS validation **OR** provide a valid `ARGOCD_CA_CERT` secret. If neither is set and ArgoCD is secured with a custom/self-signed certificate, the workflow will fail to connect.

---

## 📁 Repository Layout Expectations

### Source Repository (`repo`)

This should contain a standard [Kustomize base/overlay structure](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/) like:

```
manifests/
  base/
  overlays/
    dev/
    prod/
```

### Continuous Deployment Repository (`cd_repo`)

The templated output from the deployment will be committed to:
```
${{ cd_repo }}/${cluster}/${namespace}/${application}/
```

This becomes the **ArgoCD source path**, from which the app syncs after every deployment.

---

## 🌍 Environment Mapping

Your CD repo must define:

```
${{ cd_repo }}/config/env-map.yaml
```

With contents like:
```yaml
dev:
  cluster: aks-west-europe
  dns_zone: internal.demo.affinity7software.com
prod:
  cluster: aks-prod-eu
  dns_zone: prod.example.com
```

This file maps logical environments (like `dev`, `prod`) to:
- `cluster`: used in naming the ArgoCD app and web UI
- `dns_zone`: used to compute the full ArgoCD web endpoint

> 💡 This is used to support different clusters and hosted DNS zones (Azure DNS or Route53) per environment.

---

## 🛠 Actions Used

| Action                                                                 | Version | Description                                                                 |
|------------------------------------------------------------------------|---------|-----------------------------------------------------------------------------|
| [actions/checkout](https://github.com/actions/checkout)               | `v4` | Used to clone both source and CD repositories                              |
| [tibdex/github-app-token](https://github.com/tibdex/github-app-token) | `v1`    | Generates a GitHub App token to push to the CD repo                        |
| [imranismail/setup-kustomize](https://github.com/imranismail/setup-kustomize) | `v2` | Installs the desired version of Kustomize                                  |
| [actions/upload-artifact](https://github.com/actions/upload-artifact) | `v4`    | Stores rendered and built outputs for debugging                            |

---

## 🔄 Deployment Flow Summary

1. **Clone the source repo and CD repo**
2. **Read cluster and DNS zone info** from `${{ cd_repo }}/config/env-map.yaml`
3. **Template all `${}` variables** using `envsubst` (only if present in supported file types)
4. **Copy** templated manifests to:
   ```
   ${{ cd_repo }}/${cluster}/${namespace}/${application}/
   ```
5. **Patch image tags** in the `overlay_dir` if `image_tag` is provided
6. **Run `kustomize build .`** on the overlay
   - The result is saved as `build-output.yaml` (uploaded as an artifact)
   - If build fails, the action will stop before contacting ArgoCD
7. **Commit and push** the rendered code to the CD repo
8. **ArgoCD app is created if it does not exist**, using the rendered template:
   ```
   gitopsmanager/k8s-deploy/templates/argocd-application-template.json
   ```
   The file is rendered using `envsubst` with variables from the environment.
9. **ArgoCD sync is triggered via REST API**
10. **Sync status is polled**, unless `skip_status_check: true`

---

## 📦 Artifact Uploads

Two useful artifacts are uploaded after every run:

| Artifact Name               | Description                                                 |
|-----------------------------|-------------------------------------------------------------|
| `templated-source-manifests` | All files after `envsubst` templating                      |
| `built-kustomize-manifest`   | Final rendered output of `kustomize build`                 |

> These allow you to troubleshoot errors with ArgoCD sync or validate the rendered output locally.

---

## 🧪 Example Workflow Call

```yaml
jobs:
  deploy:
    uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v1
    with:
      environment: dev
      application: my-app
      namespace: my-namespace
      repo: my-org/my-app
      repo_path: manifests
      overlay_dir: dev
    secrets:
      ARGO_CD_ADMIN_USER: ${{ secrets.ARGO_CD_ADMIN_USER }}
      ARGO_CD_ADMIN_PASSWORD: ${{ secrets.ARGO_CD_ADMIN_PASSWORD }}
      CONTINUOUS_DEPLOYMENT_GH_APP_ID: ${{ secrets.CD_GH_APP_ID }}
      CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY: ${{ secrets.CD_GH_APP_KEY }}
      ARGOCD_CA_CERT: ${{ secrets.ARGOCD_CA_CERT }}
```

---

## 🏗 ArgoCD Requirements

You **must** install ArgoCD with the following ingress hostname format:

```
https://${cluster}-argocd-argocd-web-ui.${dns_zone}
```

For example:
```
https://aks-west-europe-argocd-argocd-web-ui.internal.demo.affinity7software.com
```

> 🧠 This is how the workflow constructs the URL to talk to the ArgoCD API. If your ingress differs, you'll need to update the URL logic in `deploy.yaml`.

---

## 📦 ArgoCD App Details

The created ArgoCD Application will have:

| Field        | Value                                                                 |
|--------------|-----------------------------------------------------------------------|
| `project`    | `default`                                                             |
| `name`       | `${namespace}-${application}`                                         |
| `source.repoURL` | `https://github.com/${cd_repo}`                                   |
| `source.path` | `${cluster}/${namespace}/${application}/overlays/${overlay_dir}`     |
| `destination.server` | `https://kubernetes.default.svc`                             |
| `destination.namespace` | value from `inputs.namespace`                             |

---
### 🔖 Versioning

This workflow is published with Git tags for stable releases.

- `@v1` – Tracks the latest stable version in the v1.x series (recommended)
- Semantic tags like `@v1.0.0` will be published in the future for reproducible builds

#### Example:
```yaml
uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v1
```

> ✅ Use `@v1` for stability and latest updates. Semantic version tags (e.g., `v1.0.0`) will be available as additional pinning options in future releases.

---

## 📄 License

© 2025 Affinity7 Consulting Ltd  
Licensed under the [MIT License](LICENSE) — free to use, modify, and distribute with attribution.


