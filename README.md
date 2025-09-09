# 🚀 Reusable GitHub Workflow: Kubernetes App Deployment with ArgoCD (v2 BETA)

This GitHub Actions **reusable workflow** automates Kubernetes application deployment using [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/).  
It renders and templates your manifest files, commits the results to your **continuous deployment (CD) repo**, and uses the **ArgoCD REST API** to create and sync applications.

> ✅ **ArgoCD Applications are auto-created if missing**, so first-time deployments work out-of-the-box.  
> ⚠️ **v2 is currently in BETA** — APIs and inputs may evolve slightly.

---

## ⚙️ Inputs

### Core
| Name          | Required | Type   | Description |
|---------------|----------|--------|-------------|
| `runner`      | ❌ | string | GitHub runner label (default: `ubuntu-latest`) |
| `cd_repo`     | ✅ | string | Continuous deployment repo where templated manifests are stored |
| `environment` | ✅ | string | Logical environment name (e.g. `dev`, `prod`) |
| `cluster`     | ❌ | string | Required only when `env_map[environment]` has multiple clusters |
| `namespace`   | ✅ | string | Kubernetes namespace for deployment |

### Source Repo
| Name            | Required | Type   | Description |
|-----------------|----------|--------|-------------|
| `repo`          | ✅ | string | GitHub repo containing source manifests (e.g. `my-org/my-app`) |
| `deployFilesPath` | ❌ | string | Path to manifest files inside `repo` |
| `repo_commit_id` | ❌ | string | Commit SHA to deploy (overrides `branch_name`) |
| `branch_name`    | ❌ | string | Branch to checkout (default: `main`) |

### Deployment Modes
- **Mode A (single app):**
  - `application` (string, required)
  - `deployFilesPath` (string, required if not using `applicationDetails`)
  - `image_tag` (string, optional)
  - `image_base_name` (string, optional)
  - `image_base_names` (string, optional, comma-separated list)

- **Mode B (multi app):**
  - `applicationDetails` (JSON array, optional)
    ```json
    [
      {"name":"app1","images":["repo/app1","repo/sidecar"],"path":"services/app1/overlays/prod"},
      {"name":"app2","images":["repo/app2"],"path":"apps/app2/overlays/prod"}
    ]
    ```

### Environment Map
| Name      | Required | Type   | Description |
|-----------|----------|--------|-------------|
| `env_map` | ❌ | string | JSON describing environments, clusters, registries, and UAMI mappings |

Example shape:
```json
{
  "dev": {
    "cluster_count": 1,
    "clusters": [
      {
        "cluster": "aks-west-europe",
        "dns_zone": "internal.demo.affinity7software.com",
        "container_registry": "ghcr.io/my-org",
        "uami_map": [
          {"uami_name":"aks-uami","uami_resource_group":"rg-demo","client_id":"1234-5678"}
        ]
      }
    ]
  }
}
```

### ArgoCD / Misc
| Name               | Required | Type   | Default | Description |
|--------------------|----------|--------|---------|-------------|
| `argocd_auth_token`| ❌ | string | – | Direct token for ArgoCD auth |
| `argocd_username`  | ❌ | string | – | Username (basic auth fallback) |
| `argocd_password`  | ❌ | string | – | Password (basic auth fallback) |
| `kustomize_version`| ❌ | string | `5.0.1` | Kustomize version |
| `skip_status_check`| ❌ | string | `false` | Skip waiting for ArgoCD sync |
| `insecure_argo`    | ❌ | boolean | `false` | Skip TLS verification |
| `convert_jinja`    | ❌ | boolean | `false` | Convert `{{ var }}` → `${var}` before templating |
| `debug`            | ❌ | boolean | `false` | Print debug structure |

---

## 🔐 Required Secrets

| Name                                   | Description |
|----------------------------------------|-------------|
| `CONTINUOUS_DEPLOYMENT_GH_APP_ID`      | GitHub App ID used to push to the CD repo |
| `CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY` | GitHub App private key |

### 🧩 Optional Secrets
| Name               | Description |
|--------------------|-------------|
| `ARGOCD_CA_CERT`   | PEM CA cert to verify ArgoCD endpoint |
| `ARGOCD_USERNAME`  | Fallback ArgoCD username |
| `ARGOCD_PASSWORD`  | Fallback ArgoCD password |

---

## 📦 Artifact Uploads

| Artifact Name               | Description |
|-----------------------------|-------------|
| `templated-source-manifests` | All files after envsubst templating |
| `built-kustomize-manifest`   | Final rendered output of `kustomize build` |

---

## 🔄 Deployment Flow Summary

1. Checkout **source** and **CD repos**
2. Parse **env_map JSON** and select cluster
3. Export **UAMI env vars** if provided
4. Convert Jinja → envsubst (if enabled)
5. Template manifests (`envsubst`)
6. Copy to CD repo path:
   ```
   continuous-deployment/<cluster>/<namespace>/<application>/
   ```
7. Patch image tags (if provided)
8. Run `kustomize build`
9. Commit & push to CD repo
10. Create ArgoCD app if missing
11. Trigger ArgoCD sync
12. Wait for sync (unless skipped)

---

## 🧪 Example Workflow Call

```yaml
jobs:
  deploy:
    uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v2
    with:
      environment: dev
      application: my-app
      namespace: my-namespace
      repo: my-org/my-app
      deployFilesPath: manifests/overlays/dev
      overlay_dir: dev
      image_tag: abc123
    secrets:
      CONTINUOUS_DEPLOYMENT_GH_APP_ID: ${{ secrets.CD_GH_APP_ID }}
      CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY: ${{ secrets.CD_GH_APP_KEY }}
      ARGOCD_CA_CERT: ${{ secrets.ARGOCD_CA_CERT }}
```

---

## 🏗 ArgoCD Requirements

ArgoCD ingress must match:

```
https://${cluster}-argocd-argocd-web-ui.${dns_zone}
```

Example:
```
https://aks-west-europe-argocd-argocd-web-ui.internal.demo.affinity7software.com
```

---

## 📦 ArgoCD App Details

| Field        | Value |
|--------------|-------|
| `project`    | `default` |
| `name`       | `${namespace}-${application}` |
| `source.repoURL` | `https://github.com/${cd_repo}` |
| `source.path` | `${cluster}/${namespace}/${application}/...` |
| `destination.server` | `https://kubernetes.default.svc` |
| `destination.namespace` | `${namespace}` |

---

### 🔖 Versioning

- `@v2` – Latest **beta** version  
- `@v1` – Latest stable v1 series

---

## 📜 License

Licensed under MIT. See [LICENSE](./LICENSE).  
Third-party actions are listed in [THIRD-PARTY-ACTIONS-AND-TOOLS.md](./THIRD-PARTY-ACTIONS-AND-TOOLS.md).



