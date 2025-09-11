# 🚀 Reusable GitHub Workflow: Kubernetes App Deployment with ArgoCD (v2)

This reusable workflow automates Kubernetes application deployment using [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/).  

It renders manifest files, commits the results into your **continuous deployment (CD) repo**, and uses the **ArgoCD REST API** to create, sync, and validate applications.

> ✅ **First-time deployments are supported** – ArgoCD apps are auto-created if missing.  
> 🗑️ **Optional `delete_first`** – force ArgoCD app deletion before re-create.  
> ⚠️ **Self-hosted runner required** – must have cluster network access and tooling installed.  

---

## ⚙️ Inputs

### Core
| Name                 | Required | Type     | Default | Description |
|----------------------|----------|----------|---------|-------------|
| `runner`             | ✅       | string   | –       | Label of the self-hosted runner (must have cluster access) |
| `cd_repo`            | ✅       | string   | –       | Continuous deployment repo where templated manifests are committed |
| `github_environment` | ✅       | string   | –       | GitHub Environment for approvals and env-scoped secrets |
| `target`             | ✅       | string   | `dev`   | Logical environment (`dev`, `qa`, `prod`). Still required even if `target_cluster` is set, as it controls approvals and environment scoping. |
| `target_cluster`     | ❌       | string   | `""`    | If set, resolves the cluster **globally across all `env_map` environments** by matching the `cluster` name. If omitted, the cluster is resolved from the given `target` environment only. |
| `namespace`          | ✅       | string   | –       | Kubernetes namespace for deployment |
| `ref`                | ❌       | string   | –       | Git reference (branch, tag, or commit SHA) for source repo checkout |
| `delete_first`       | ❌       | boolean  | `false` | Delete ArgoCD app(s) before deploying |

---

### Deployment Modes
- **Mode A: Single App**
  - `application` (string)  
  - `deploy_path` (string)  
  - `image_tag` (string, optional)  
  - `image_base_name` (string, optional)  
  - `image_base_names` (comma-separated string, optional)  
  - `overlay_dir` (string, optional)

- **Mode B: Multi App**
  - `applicationDetails` (JSON array, optional)  
    ```json
    [
      {"name":"app1","images":["repo/app1","repo/sidecar"],"path":"services/app1/overlays/prod"},
      {"name":"app2","images":["repo/app2"],"path":"apps/app2/overlays/prod"}
    ]
    ```

---

### Environment Map
`env_map` must be valid JSON in this form:

```json
{
  "prod": {
    "cluster_count": 2,
    "clusters": [
      { "cluster": "aks-prod-weu", "dns_zone": "internal.example.com", "container_registry": "ghcr.io/my-org", "uami_map": [] },
      { "cluster": "aks-prod-use", "dns_zone": "internal.example.com", "container_registry": "ghcr.io/my-org", "uami_map": [] }
    ]
  }
}
```

Precedence:
1. `inputs.env_map` (preferred)  
2. `ENV_MAP` environment variable (fallback)  
3. None → workflow fails  

---

### ArgoCD / Misc
| Name                | Required | Type    | Default  | Description |
|---------------------|----------|---------|----------|-------------|
| `argocd_auth_token` | ❌       | string  | –        | Direct ArgoCD API token |
| `argocd_username`   | ❌       | string  | –        | ArgoCD username (basic auth fallback) |
| `argocd_password`   | ❌       | string  | –        | ArgoCD password (basic auth fallback) |
| `kustomize_version` | ❌       | string  | `5.0.1`  | Version of kustomize to install |
| `skip_status_check` | ❌       | string  | `false`  | Skip waiting for ArgoCD sync |
| `insecure_argo`     | ❌       | boolean | `false`  | Disable TLS verification for ArgoCD API |
| `convert_jinja`     | ❌       | boolean | `false`  | Convert `{{ var }}` → `${var}` placeholders before templating |
| `debug`             | ❌       | boolean | `false`  | Print copied directory structure |

---

## 🔐 Secrets

| Name                                   | Required | Description |
|----------------------------------------|----------|-------------|
| `CONTINUOUS_DEPLOYMENT_GH_APP_ID`      | ✅        | GitHub App ID used for CD repo pushes |
| `CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY` | ✅     | GitHub App private key |
| `ARGOCD_CA_CERT`                       | ❌        | PEM CA cert for ArgoCD endpoint |
| `ARGOCD_USERNAME` / `ARGOCD_PASSWORD`  | ❌        | Fallback basic auth credentials |

---

## 🔧 Actions Used

| Action | Version | Purpose |
|--------|---------|---------|
| [`actions/checkout`](https://github.com/actions/checkout) | `v4` | Checkout source repo, reusable workflow repo, and CD repo |
| [`tibdex/github-app-token`](https://github.com/tibdex/github-app-token) | `v1` | Generate GitHub App token for authenticated CD repo access |
| [`gitopsmanager/detect-cloud`](https://github.com/gitopsmanager/detect-cloud) | `main` | Detect whether runner is AWS, Azure, or unknown |
| [`actions/github-script`](https://github.com/actions/github-script) | `v7` | Inline scripting for JSON parsing, cluster selection, app management, templating, etc. |
| [`imranismail/setup-kustomize`](https://github.com/imranismail/setup-kustomize) | `v2` | Install kustomize CLI for manifest builds |
| [`actions/upload-artifact`](https://github.com/actions/upload-artifact) | `v4` | Upload built manifests and templated source manifests as workflow artifacts |

---

## 📦 Artifacts

- **`templated-source-manifests`** – manifests after envsubst templating  
- **`built-kustomize-manifest`** – final rendered YAML from `kustomize build`  

---

## 🔄 Deployment Flow

1. Checkout source and CD repos  
2. Parse `env_map`  
   - If `target_cluster` is set → select cluster globally across all envs.  
   - Otherwise → select from `target` environment only.  
3. Export Azure UAMI client IDs (if defined)  
4. Optional: Jinja → envsubst conversion  
5. Template manifests with `envsubst`  
6. Copy to CD repo structure:
   ```
   continuous-deployment/<cluster>/<namespace>/<application>/
   ```
7. Patch image tags (if provided)  
8. Build manifests with `kustomize`  
9. Upload artifacts  
10. Commit & push to CD repo  
11. Authenticate with ArgoCD (token > user/pass > secrets)  
12. (Optional) Delete apps if `delete_first` is true  
13. Create missing ArgoCD apps  
14. Trigger sync  
15. Wait for sync (unless skipped)  

---

## 🧪 Example Usage

Single-app deploy (global cluster override):
```yaml
jobs:
  deploy:
    uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v2
    with:
      runner: my-self-hosted
      cd_repo: my-org/continuous-deployment
      github_environment: prod
      target: prod
      target_cluster: aks-prod-weu     # resolves this cluster directly
      namespace: my-namespace
      application: my-app
      deploy_path: manifests/overlays/prod
      image_tag: abc123
      overlay_dir: prod
      env_map: ${{ secrets.ENV_MAP_JSON }}
    secrets:
      CONTINUOUS_DEPLOYMENT_GH_APP_ID: ${{ secrets.CD_GH_APP_ID }}
      CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY: ${{ secrets.CD_GH_APP_KEY }}
      ARGOCD_CA_CERT: ${{ secrets.ARGOCD_CA_CERT }}
```

Multi-app deploy (single-cluster target only):
```yaml
with:
  runner: my-self-hosted
  cd_repo: my-org/continuous-deployment
  github_environment: qa
  target: qa
  namespace: qa-namespace
  applicationDetails: |
    [
      {"name":"service1","images":["repo/service1"],"path":"services/service1/overlays/qa"},
      {"name":"service2","images":["repo/service2"],"path":"apps/service2/overlays/qa"}
    ]
  env_map: ${{ secrets.ENV_MAP_JSON }}
```

---

## 🏗 ArgoCD URL Assumption

ArgoCD ingress must be reachable at:

```
https://${cluster}-argocd-argocd-web-ui.${dns_zone}
```

Example:
```
https://aks-prod-weu-argocd-argocd-web-ui.internal.example.com
```

---

## 📦 ArgoCD App Details

| Field                  | Value |
|------------------------|-------|
| `project`              | `default` |
| `name`                 | `${namespace}-${application}` |
| `source.repoURL`       | `https://github.com/${cd_repo}` |
| `source.path`          | `${cluster}/${namespace}/${application}/...` |
| `destination.server`   | `https://kubernetes.default.svc` |
| `destination.namespace`| `${namespace}` |

---

### 🔖 Versioning

- `@v2` – Latest **beta** version  
- `@v1` – Latest stable v1 series

---

## 📜 License
Licensed under MIT.  
Third-party actions are listed in [THIRD-PARTY-ACTIONS-AND-TOOLS.md](./THIRD-PARTY-ACTIONS-AND-TOOLS.md).  



