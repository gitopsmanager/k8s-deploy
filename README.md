# 🚀 Reusable GitHub Workflow: Kubernetes App Deployment with ArgoCD (v2)

This reusable workflow automates Kubernetes application deployment using [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/).  

It renders manifest files using **Jinja2-style templating implemented with [Nunjucks](https://mozilla.github.io/nunjucks/)**, commits the results into your **continuous deployment (CD) repo**, and uses the **ArgoCD REST API** to create, sync, and validate applications.

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
  - `application_details` (JSON array, optional)  
    ```json
    [
      {"name":"app1","images":["repo/app1","repo/sidecar"],"path":"services/app1/overlays/prod"},
      {"name":"app2","images":["repo/app2"],"path":"apps/app2/overlays/prod"}
    ]
    ```

---

### Environment Map

The workflow needs an `env_map` that defines your environments, clusters, and related metadata.  
You can provide it in **two ways**:

1. As a workflow input (`with: env_map: "<JSON>"`) ✅ **preferred**  
2. As an environment variable (`env: ENV_MAP: "<JSON>"`) — fallback for self-hosted runners  

If neither is provided, the workflow fails.

---

#### Cluster Resolution Logic

1. If `target_cluster` is set:  
   - The workflow searches **all clusters across all environments** in the `env_map`.  
   - If it finds a matching cluster name, it selects that cluster (and its `dns_zone`, `container_registry`, `uami_map`) regardless of the `target` environment.  
   - This makes it possible to directly target a cluster by name while still requiring `target` for approvals.  

2. If `target_cluster` is not set:  
   - The workflow looks inside the `target` environment only.  
   - If that environment has a **single cluster**, it is selected automatically.  
   - If the environment has **multiple clusters**, the workflow fails with a clear error message and lists valid options.  

---

#### Example env_map JSON

```json
{
  "dev": {
    "cluster_count": 1,
    "clusters": [
      { "cluster": "aks-dev-weu", "dns_zone": "internal.dev.example.com", "container_registry": "ghcr.io/my-org", "uami_map": [] }
    ]
  },
  "prod": {
    "cluster_count": 2,
    "clusters": [
      { "cluster": "aks-prod-weu", "dns_zone": "internal.example.com", "container_registry": "ghcr.io/my-org", "uami_map": [] },
      { "cluster": "aks-prod-use", "dns_zone": "internal.example.com", "container_registry": "ghcr.io/my-org", "uami_map": [] }
    ]
  }
}
```

---

#### Example Usages

**Targeting a cluster globally (preferred for multi-cluster envs):**
```yaml
with:
  target: prod
  target_cluster: aks-prod-weu
  env_map: ${{ secrets.ENV_MAP_JSON }}
```

**Targeting a single-cluster environment (no `target_cluster` needed):**
```yaml
with:
  target: dev
  env_map: ${{ secrets.ENV_MAP_JSON }}
```

---

## 🔑 UAMI Mapping for Azure Workload Identity

When deploying to **Azure AKS**, some clusters may define **User Assigned Managed Identities (UAMIs)** in the `env_map`.  
This workflow automatically exports those UAMI client IDs as environment variables so they can be used for manifest templating.

---

### How It Works
For each `uami_map` entry in the selected cluster:

1. **Take the `uami_name`**  
   - If it starts with `<cluster>-` (the selected cluster name followed by a dash), that prefix is **removed**.  
     - Example: `prodcluster-app-uami` → `app-uami`.

2. **Normalize the name**  
   - Replace `-` with `_`.  
     - Example: `sidecar-uami` → `sidecar_uami`.  
   - If the result doesn’t start with a letter or `_`, prepend `_`.  
     - Example: `123-identity` → `_123_identity`.

3. **Export as environment variable**  
   - Variable name = normalized `uami_name`.  
   - Value = `client_id` of the UAMI.  
   - Written into `$GITHUB_ENV`, so it’s available to all subsequent steps.  

4. **Deduplicate**  
   - If two UAMIs normalize to the same name, duplicates are skipped with a warning.

---

### Example

```json
{
  "cluster": "prodcluster",
  "dns_zone": "internal.example.com",
  "container_registry": "ghcr.io/my-org",
  "uami_map": [
    {"uami_name": "prodcluster-app-uami", "uami_resource_group": "rg-demo", "client_id": "1111-aaaa"},
    {"uami_name": "sidecar-uami", "uami_resource_group": "rg-demo", "client_id": "2222-bbbb"}
  ]
}
```

Exports:

```
app_uami=1111-aaaa
sidecar_uami=2222-bbbb
```

---

### Usage in Manifests

You can reference these exported variables during **Jinja2 templating with Nunjucks**:  

```yaml
env:
  - name: APP_UAMI_CLIENT_ID
    value: {{ app_uami }}
  - name: SIDECAR_UAMI_CLIENT_ID
    value: {{ sidecar_uami }}
```

---

### Using Nunjucks in GitHub Actions without npm install

Normally, you would install Nunjucks via `npm install nunjucks`, but `actions/github-script`
does not have access to modules installed at runtime. To keep things lightweight and fast,
you can **download the prebuilt UMD bundle** of Nunjucks directly before using it.

```yaml
- name: Download Nunjucks UMD bundle
  run: |
    curl -sSL https://cdnjs.cloudflare.com/ajax/libs/nunjucks/3.2.4/nunjucks.min.js -o nunjucks.js

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
| `convert_jinja`     | ❌       | boolean | `false`  | (Legacy) Convert `{{ var }}` → `${var}` placeholders before templating. No longer required if using Jinja2 templating directly. |
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
| [`actions/checkout`](https://github.com/actions/checkout) | `v4` | Checkout repos |
| [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token) | `v2` | Generate GitHub App token |
| [`gitopsmanager/detect-cloud`](https://github.com/gitopsmanager/detect-cloud) | `main` | Detect cloud provider |
| [`actions/github-script`](https://github.com/actions/github-script) | `v7` | JSON parsing, cluster selection, templating with Nunjucks |
| [`imranismail/setup-kustomize`](https://github.com/imranismail/setup-kustomize) | `v2` | Install kustomize |
| [`actions/upload-artifact`](https://github.com/actions/upload-artifact) | `v4` | Upload artifacts |

---

## 📦 Artifacts

- **`templated-source-manifests`** – manifests after **Jinja2 templating with Nunjucks**  
- **`built-kustomize-manifest`** – final rendered YAML from `kustomize build`  

---

## 🔄 Deployment Flow

1. Checkout repos  
2. Parse `env_map`  
3. Export UAMI client IDs  
4. Template manifests with **Nunjucks (Jinja2 style)**  
5. Copy to CD repo structure  
6. Patch images  
7. Build with kustomize  
8. Upload artifacts  
9. Commit & push  
10. Authenticate with ArgoCD  
11. Delete apps if `delete_first`  
12. Create apps  
13. Sync  
14. Wait for sync (unless skipped)  

---

## 🧪 Example Usage

```yaml
jobs:
  deploy:
    uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v2
    with:
      runner: my-self-hosted
      cd_repo: my-org/continuous-deployment
      github_environment: prod
      target: prod
      target_cluster: aks-prod-weu
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

---

## 🏗 ArgoCD URL Assumption

ArgoCD ingress must be reachable at:

```
https://${cluster}-argocd-argocd-web-ui.${dns_zone}
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

- `@v2` – Latest beta  
- `@v1` – Stable v1 series

---

## 📜 License
MIT.  
Third-party actions listed in [THIRD-PARTY-ACTIONS-AND-TOOLS.md](./THIRD-PARTY-ACTIONS-AND-TOOLS.md).
