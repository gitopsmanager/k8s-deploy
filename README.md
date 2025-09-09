# 🚀 Reusable GitHub Workflow: Kubernetes App Deployment with ArgoCD (v2 BETA)

This GitHub Actions **reusable workflow** automates Kubernetes application deployment using [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/).
It renders and templates your manifest files, commits the results to your **continuous deployment (CD) repo**, and uses the **ArgoCD REST API** to create and sync applications.

> ✅ **ArgoCD Applications are auto-created if missing**, so first-time deployments work out-of-the-box.  
> ⚠️ **v2 is currently in BETA** — inputs and behavior may evolve slightly.

---

## ⚙️ Inputs

### Core
| Name                | Required | Type   | Description |
|---------------------|----------|--------|-------------|
| `runner`            | ❌       | string | GitHub runner label (default: `ubuntu-latest`) |
| `cd_repo`           | ✅       | string | Continuous deployment repo where templated manifests are stored |
| `github_environment`| ✅       | string | **GitHub Environment** name for this job (enables approvals & env-scoped secrets in repo settings) |
| `target`            | ✅       | string | **Logical environment** to deploy to (e.g. `dev`, `qa`, `prod`). If `env_map[target]` has multiple clusters, you **must** also set `target_cluster`. |
| `target_cluster`    | ❌       | string | Specific cluster to use **when** `target` maps to multiple clusters in `env_map` (e.g. `aks-prod-weu`). For single-cluster envs leave empty. |
| `namespace`         | ✅       | string | Kubernetes namespace for deployment |

> 🔎 **`github_environment` vs `target`**  
> `github_environment` ties the job to a GitHub Environment (for approvals and env-scoped secrets).  
> `target` is your **logical deployment environment** used to look up clusters via `env_map`.

### Source Repo
| Name               | Required | Type   | Description |
|--------------------|----------|--------|-------------|
| `repo`             | ✅       | string | GitHub repo containing source manifests (e.g. `my-org/my-app`) |
| `deployFilesPath`  | ❌       | string | Path to manifest files inside `repo` |
| `repo_commit_id`   | ❌       | string | Commit SHA to deploy (overrides `branch_name`) |
| `branch_name`      | ❌       | string | Branch to checkout (default: `main`) |

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
### Providing `env_map` (input vs. environment variable)

You can provide the environment map in **two ways**. The workflow will use **`inputs.env_map` first**, and if it’s empty, it will fall back to the **`ENV_MAP` environment variable**.

**Precedence:**
1. `with: env_map: "<JSON>"`  ✅ (preferred)
2. `env: ENV_MAP: "<JSON>"`   (fallback)
3. If neither is present → the workflow **fails** with a clear error.

**Examples**

**A) Pass as workflow input (preferred)**
```yaml
jobs:
  deploy:
    uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v2
    with:
      target: prod
      target_cluster: aks-prod-weu
      namespace: my-namespace
      cd_repo: my-org/continuous-deployment
      env_map: ${{ secrets.ENV_MAP_JSON }}   # JSON string
```

**B) Provide via environment variable `ENV_MAP` (fallback)**
```yaml
jobs:
  deploy:
    uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v2
    with:
      target: prod
      target_cluster: aks-prod-weu
      namespace: my-namespace
      cd_repo: my-org/continuous-deployment
    env:
      ENV_MAP: ${{ secrets.ENV_MAP_JSON }}   # JSON string
```

> ℹ️ **Format:** `env_map` must be **valid JSON** (not YAML).  
> ❗ If `target` maps to multiple clusters in `env_map` and `target_cluster` is empty, the workflow fails and lists valid cluster options.


---

## 🔑 UAMI Mapping for Azure Workload Identity

When deploying to **Azure AKS**, this workflow exports **User Assigned Managed Identity (UAMI)** client IDs as environment variables for templating.

### How it works
- Each `uami_map` entry from `env_map` is processed.
- The `uami_name` is transformed into a safe environment variable name:
  1. If it starts with `<cluster_name>-`, that prefix is **removed**.  
     - Example: `devcluster-myidentity` → `myidentity`
  2. All `-` characters are replaced with `_`.  
     - Example: `sidecar-uami` → `sidecar_uami`
  3. If the resulting name doesn’t start with `[A-Za-z_]`, an `_` is prepended.
- The transformed name is exported and set to the UAMI’s `client_id`.
- All exported variables are logged for visibility.

### Example
Given this `env_map` cluster entry:
```json
{
  "cluster": "devcluster",
  "dns_zone": "internal.demo.affinity7software.com",
  "container_registry": "ghcr.io/my-org",
  "uami_map": [
    {"uami_name": "devcluster-app-uami", "uami_resource_group": "rg-demo", "client_id": "1111-aaaa"},
    {"uami_name": "sidecar-uami", "uami_resource_group": "rg-demo", "client_id": "2222-bbbb"}
  ]
}
```

Exports:
```
app_uami=1111-aaaa
sidecar_uami=2222-bbbb
```

Use in manifests:
```yaml
env:
  - name: APP_UAMI_CLIENT_ID
    value: ${app_uami}
  - name: SIDECAR_UAMI_CLIENT_ID
    value: ${sidecar_uami}
```

---

## ArgoCD / Misc
| Name               | Required | Type   | Default | Description |
|--------------------|----------|--------|---------|-------------|
| `argocd_auth_token`| ❌       | string | –       | Direct token for ArgoCD auth |
| `argocd_username`  | ❌       | string | –       | Username (basic auth fallback) |
| `argocd_password`  | ❌       | string | –       | Password (basic auth fallback) |
| `kustomize_version`| ❌       | string | `5.0.1` | Kustomize version |
| `skip_status_check`| ❌       | string | `false` | Skip waiting for ArgoCD sync |
| `insecure_argo`    | ❌       | boolean| `false` | Skip TLS verification |
| `convert_jinja`    | ❌       | boolean| `false` | Convert `{{ var }}` → `${var}` before templating |
| `overlay_dir`      | ❌       | string | `""`     | Optional overlay subdir (used for image patching & build) |
| `debug`            | ❌       | boolean| `false` | Print debug structure |

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
| `ARGOCD_USERNAME`  | Fallback ArgoCD username (if not passing as input) |
| `ARGOCD_PASSWORD`  | Fallback ArgoCD password (if not passing as input) |

---

## 📦 Artifact Uploads

| Artifact Name                | Description |
|------------------------------|-------------|
| `templated-source-manifests` | All files after envsubst templating |
| `built-kustomize-manifest`   | Final rendered output of `kustomize build` |

---

## 🔄 Deployment Flow Summary

1. Checkout **source** and **CD repos**  
2. Parse **env_map JSON** and resolve the cluster from `target` + optional `target_cluster`  
3. Export **UAMI env vars** (normalized) if provided  
4. Optional: Convert Jinja → envsubst placeholders  
5. Template manifests with `envsubst`  
6. Copy to CD repo path:
   ```
   continuous-deployment/<cluster>/<namespace>/<application>/
   ```
7. Patch image tags (if provided)  
8. Run `kustomize build` and upload artifact  
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
      # Core
      cd_repo: my-org/continuous-deployment
      github_environment: prod
      target: prod                 # logical environment
      target_cluster: aks-prod-weu # REQUIRED here because prod has >1 clusters
      namespace: my-namespace

      # Source
      repo: my-org/my-app
      deployFilesPath: manifests/overlays/prod
      branch_name: main

      # App mode A
      application: my-app
      overlay_dir: prod
      image_tag: abc123

      # Env map (JSON string)
      env_map: ${{ secrets.ENV_MAP_JSON }} # or pass inline JSON

    secrets:
      CONTINUOUS_DEPLOYMENT_GH_APP_ID: ${{ secrets.CD_GH_APP_ID }}
      CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY: ${{ secrets.CD_GH_APP_KEY }}
      ARGOCD_CA_CERT: ${{ secrets.ARGOCD_CA_CERT }}
```

**Single-cluster example (no `target_cluster` needed):**
```yaml
with:
  cd_repo: my-org/continuous-deployment
  github_environment: dev
  target: dev
  target_cluster: ""
  namespace: my-namespace
  repo: my-org/my-app
  deployFilesPath: manifests/overlays/dev
  application: my-app
  overlay_dir: dev
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

Licensed under MIT. See [LICENSE](./LICENSE).  
Third-party actions are listed in [THIRD-PARTY-ACTIONS-AND-TOOLS.md](./THIRD-PARTY-ACTIONS-AND-TOOLS.md).
