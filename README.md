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
| `github_runner`      | ✅       | string   | –       | Label of the self-hosted runner (must have cluster access) |
| `namespace`          | ✅       | string   | –       | Kubernetes namespace for deployment |
| `target`             | ✅       | string   | `dev`   | Logical environment (`dev`, `qa`, `prod`). Still required even if `target_cluster` is set, as it controls approvals and environment scoping. |
| `target_cluster`     | ❌       | string   | `""`    | If set, resolves the cluster **globally across all `env_map` environments** by matching the `cluster` name. If omitted, the cluster is resolved from the given `target` environment only. |
| `ref`                | ❌       | string   | `${{ github.ref || github.sha }}` | Git reference (branch, tag, or commit SHA) for source repo checkout |
| `delete_first`       | ❌       | boolean  | `false` | Delete ArgoCD app(s) before deploying |
| `cd_repo`            | ✅       | string   | –       | Continuous deployment repo where templated manifests are committed |
| `cd_repo_org`        | ✅       | string   | –       | Continuous deployment repo organization |
| `github_environment` | ❌       | string   | –       | GitHub Environment for approvals and env-scoped secrets |

---

### Deployment Modes
- **Mode A: Single App**
  - `application` (string)  
  - `deploy_path` (string, default `Deployments`)  
  - `image_tag` (string, optional)  
  - `image_base_name` (string, optional)  
  - `image_base_names` (comma-separated string, optional)  
  - `overlay_dir` (string, required)

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
2. As an environment variable (`ENV_MAP` from the runner) — fallback for self-hosted runners  

If neither is provided, the workflow fails.

---

#### Cluster Resolution Logic

1. If `target_cluster` is set:  
   - Searches **all clusters across all environments** in the `env_map`.  
   - Selects by exact cluster name (case-insensitive).  

2. If `target_cluster` is not set:  
   - Looks inside the `target` environment only.  
   - If single cluster → use it.  
   - If multiple clusters → fail with error listing options.  

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


## 🔑 UAMI Mapping for Azure Workload Identity

When deploying to **Azure AKS**, some clusters may define **User Assigned Managed Identities (UAMIs)** in the `env_map`.  
These are passed as an array under `uami_map`:

```json
"uami_map": [
  {
    "uami_name": "prodcluster-app-uami",
    "uami_resource_group": "rg-demo",
    "client_id": "1111-aaaa"
  },
  {
    "uami_name": "sidecar-uami",
    "uami_resource_group": "rg-demo",
    "client_id": "2222-bbbb"
  }
]
```

---

### How UAMI vars are exported

For each entry:

1. **Take the `uami_name`**  
   - If it starts with `<cluster_name>-` (the selected cluster name followed by a dash), that prefix is **removed**.  
     - Example: `prodcluster-app-uami` → `app-uami`.

2. **Normalize the name**  
   - Replace `-` with `_`.  
     - Example: `sidecar-uami` → `sidecar_uami`.  
   - If the result doesn’t start with a letter or `_`, prepend `_`.  
     - Example: `123-identity` → `_123_identity`.

3. **Export as environment variable**  
   - Variable name = normalized `uami_name` (lowercased).  
   - Value = `client_id` of the UAMI.  
   - Written into `$GITHUB_ENV`, so it’s available to all subsequent steps.

4. **Deduplicate**  
   - If two UAMIs normalize to the same name, duplicates are skipped with a warning.

---

### Example Result

Input:

```json
"uami_map": [
  {"uami_name": "prodcluster-app-uami", "uami_resource_group": "rg-demo", "client_id": "1111-aaaa"},
  {"uami_name": "sidecar-uami", "uami_resource_group": "rg-demo", "client_id": "2222-bbbb"}
]
```

Output exported vars:

```bash
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

### ArgoCD / Misc
| Name                | Required | Type    | Default  | Description |
|---------------------|----------|---------|----------|-------------|
| `argocd_auth_token` | ❌       | string  | –        | Direct ArgoCD API token |
| `argocd_username`   | ❌       | string  | –        | ArgoCD username (basic auth fallback) |
| `argocd_password`   | ❌       | string  | –        | ArgoCD password (basic auth fallback) |
| `kustomize_version` | ❌       | string  | `5.0.1`  | Version of kustomize to install |
| `skip_status_check` | ❌       | boolean | `false`  | Skip waiting for ArgoCD sync |
| `insecure_argo`     | ❌       | boolean | `false`  | Disable TLS verification for ArgoCD API |
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

## 🔧 Actions Used

| Action | Version | Purpose |
|--------|---------|---------|
| [`actions/checkout`](https://github.com/actions/checkout) | `v4` | Checkout source, reusable, and CD repos |
| [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token) | `v2` | Generate GitHub App token for CD repo access |
| [`gitopsmanager/detect-cloud`](https://github.com/gitopsmanager/detect-cloud) | `v1` | Detect cloud provider (AWS, Azure, GCP, unknown) |
| [`actions/github-script`](https://github.com/actions/github-script) | `v7` | Used extensively for JSON parsing, cluster selection, templating, ArgoCD API calls, etc. |
| [`imranismail/setup-kustomize`](https://github.com/imranismail/setup-kustomize) | `v2` | Install kustomize for manifest builds |
| [`actions/upload-artifact`](https://github.com/actions/upload-artifact) | `v4` | Upload templated manifests and built kustomize manifests |

---

## 📦 Artifacts Produced

| Artifact name | Contents | Notes |
|---------------|----------|-------|
| `templated-source-manifests-<cluster>` | Application manifests after **Nunjucks (Jinja2-style) templating** | Shows cluster-specific substitutions and injected UAMI vars |
| `built-kustomize-manifest-<cluster>`  | Final combined YAML output from `kustomize build` | Ready-to-apply manifests, concatenated with `---` delimiters |
---

## 🧪 Example Usage

```yaml
jobs:
  deploy:
    uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v2
    with:
      github_runner: my-self-hosted
      cd_repo: continuous-deployment
      cd_repo_org: my-org
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

## 🔖 Versioning

- `@v2` – Stable v2 series  
- `@v1` – Legacy v1 series  

---

## 📜 License
MIT.  
Third-party actions listed in [THIRD-PARTY-ACTIONS-AND-TOOLS.md](./THIRD-PARTY-ACTIONS-AND-TOOLS.md).

