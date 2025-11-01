# 🚀 Reusable GitHub Workflow: Kubernetes App Deployment with ArgoCD — *Open Source* & Stable `@v2`

This reusable workflow automates Kubernetes application deployment using [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/).  


It renders manifest files using **Jinja2-style templating implemented with [Nunjucks](https://mozilla.github.io/nunjucks/)**, commits the results into your **continuous deployment (CD) repo**, and uses the **ArgoCD REST API** to create, sync, and validate applications.
It also uploads artifacts for the **Kustomize build** and the **templated files** to be checked into the CD repo, which can be used for **troubleshooting any issues with Kustomize**.

🧩 **Open Source:** Maintained by **Affinity7 Consulting Ltd**, this workflow is part of the [**GitOps Manager™**](https://gitopsmanager.io) open-source toolchain — powering both community and enterprise GitOps automation across AWS and Azure.

> ✅ **First-time deployments are supported** – ArgoCD apps are auto-created if missing.  
> 🗑️ **Optional `delete_first`** – force ArgoCD app deletion before re-create (**then performs a full redeploy**).  
> ⚠️ **Self-hosted runner required** – must have cluster network access and tooling installed.

---

## 🌐 GitOps Manager™ Enterprise Platform

[**GitOps Manager™ Enterprise**](https://gitopsmanager.io) is the full platform that powers this open-source workflow.  
It’s a **turnkey GitOps automation platform** for AWS and Azure — combining open-source GitHub Actions, Kubernetes infrastructure automation, and global-scale CI/CD.

**Highlights:**
- Secure, opinionated **multi-cloud GitOps automation** for Kubernetes workloads.  
- Deep integration with **ArgoCD**, **Argo Workflows**, **Traefik**, **ECK**, and **Kubernetes Dashboard**.  
- Built for **high availability**, **autoscaling**, and **managed upgrades**.  
- Supports **Workload Identity**, **Pod Identity**, and **private, network-isolated clusters**.  
- Enables **global deployments**, **secret management**, and **production-grade infrastructure** with **zero vendor lock-in**.

🔗 Learn more: [https://gitopsmanager.io](https://gitopsmanager.io)

---

## 📊 Collection of Usage Metrics

This workflow reports **non-identifiable usage metrics** to help the maintainers understand adoption and improve open-source GitOps tooling.

Each run sends a small JSON payload like:
```json
{
  "action": "deploy",
  "org": "<hashed>",
  "repo": "<hashed>",
  "timestamp": "2025-11-01T12:00:00Z"
}
```

Data is sent to [gitopsmanager.io/github-action-metrics](https://gitopsmanager.io/github-action-metrics) and used only for **aggregate, anonymized statistics** shown publicly on [gitopsmanager.io](https://gitopsmanager.io).

> No source code, credentials, or personal information are ever transmitted.  
> Telemetry is opt-in via usage; there’s no persistent tracking or external identifiers.

---

## 🕵️‍♂️ Privacy & Metrics Summary

**GitOps Manager™** collects limited, anonymous usage metrics to understand adoption and display community activity on [gitopsmanager.io](https://gitopsmanager.io).

**What we publicly display**
- Aggregate counts of **organizations**, **repositories**, **builds**, and **deploys**  
- These totals are shown on the website as global usage summaries

**How data is handled**
- Each organization and repository is represented by an **anonymized hash key**  
- Hashes cannot be reversed or linked to any identifiable information  
- Raw telemetry records (hashed only) are stored privately and used solely to compute aggregate totals  
- Only the aggregated counts are displayed publicly — no raw data is exposed

**What we never collect**
- No personal data, source code, credentials, or identifiable metadata are ever captured or transmitted

Telemetry is an integral part of the system’s design.  
It helps measure adoption, guide improvements, and highlight how the community is using these open-source tools.

For privacy or data-related inquiries, please use the **Contact** button on [gitopsmanager.io](https://gitopsmanager.io) to submit a request using a valid corporate email address.

---

## ⚙️ Inputs

### Core
| Name                 | Required | Type     | Default | Description |
|----------------------|----------|----------|---------|-------------|
| `github_runner`      | ✅       | string   | –       | Label of the self-hosted runner (must have cluster access) |
| `namespace`          | ✅       | string   | –       | Kubernetes namespace for deployment |
| `target_environment` | ✅       | string   | `dev`   | Logical environment (`dev`, `qa`, `prod`). Still required even if `target_cluster` is set, as it controls approvals and environment scoping. |
| `target_cluster`     | ❌       | string   | `""`    | If set, resolves the cluster **globally across all `env_map` environments** by matching the `cluster` name. If omitted, the cluster is resolved from the given `target_environment` only. |
| `ref`                | ❌       | string   | `${{ github.ref || github.sha }}` | Git reference (branch, tag, or commit SHA) for source repo checkout |
| `delete_first`       | ❌       | boolean  | `false` | Delete ArgoCD app(s) before deploying (**then redeploy**). |
| `delete_only`        | ❌       | boolean  | `false` | If `true`, the workflow will **only delete ArgoCD apps** for the specified cluster/namespace and clean up their corresponding directories in the CD repo. |
| `cd_repo`            | ✅       | string   | –       | Continuous deployment repo where templated manifests are committed |
| `cd_repo_org`        | ✅       | string   | –       | Continuous deployment repo organization |
| `github_environment` | ❌       | string   | –       | GitHub Environment for approvals and env-scoped secrets |

---
> ### 🕵️‍♂️ Additional note
> The workflow **checks for presence of `ARGOCD_CA_CERT`** and reports its length for debugging purposes.

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
   - Looks inside the `target_environment` only.  
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

### Delete-Only Mode

| Name          | Required | Type     | Default | Description |
|---------------|----------|----------|---------|-------------|
| `delete_only` | ❌       | boolean  | `false` | If `true`, the workflow will **only delete ArgoCD apps** for the specified cluster/namespace and clean up their corresponding directories in the CD repo. |

---

#### How it works

When `delete_only: true`:

- **Runs**:
  - Environment resolution (`env_map` / `target_environment` / `target_cluster`)  
  - Authentication (`argocd_auth_token` or username/password)  
  - ArgoCD connection setup (`argocd_conn`)  
  - ArgoCD app deletion (API calls)  
  - Cleanup of app directories in the **CD repo** (`continuous-deployment/<cluster>/<namespace>/<app>`)  
  - Commit & push of deletions to the CD repo  

- **Skips**:
  - Manifest templating with Nunjucks  
  - Image patching  
  - `kustomize build`  
  - Uploading artifacts  
  - Creating or syncing ArgoCD apps  

This mode is useful for **cleanly tearing down apps** in both ArgoCD **and** your CD repo, leaving no trace of the app’s manifests.

> 📌 **Telemetry note:** delete-only runs still record a `deploy` event for aggregate usage totals.

---

#### Example Usage

```yaml
jobs:
  delete:
    uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v2
    with:
      github_runner: my-self-hosted
      cd_repo: continuous-deployment
      cd_repo_org: my-org
      github_environment: prod
      target_environment: prod
      target_cluster: aks-prod-weu
      namespace: my-namespace
      delete_only: true
    secrets:
      CONTINUOUS_DEPLOYMENT_GH_APP_ID: ${{ secrets.CD_GH_APP_ID }}
      CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY: ${{ secrets.CD_GH_APP_KEY }}
      ARGOCD_CA_CERT: ${{ secrets.ARGOCD_CA_CERT }}
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

1. 🗳️ Checkout repos  
2. 🧭 Parse `env_map`  
3. 🔑 Export UAMI client IDs  
4. 🧩 Template manifests with **Nunjucks (Jinja2 style)**  
5. 📁 Copy to CD repo structure  
6. 🖼️ Patch images  
7. 🏗️ Build with kustomize  
8. 📦 Upload artifacts  
9. 🌿 Commit & push  
10. 🔐 Authenticate with ArgoCD  
11. 🗑️ Delete apps if `delete_first`  
12. 🧱 Create apps  
13. 🚀 Sync  
14. ⏳ Wait for sync (unless skipped)  

---

## 🔧 Actions Used

| Action | Version | Purpose |
|--------|---------|---------|
| [`actions/checkout`](https://github.com/actions/checkout) | `v4` | Checkout source, reusable, and CD repos |
| [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token) | `v2` | Generate GitHub App token for CD repo access |
| [`gitopsmanager/detect-cloud`](https://github.com/gitopsmanager/detect-cloud) | `v1` | Detect cloud provider (AWS, Azure, GCP, unknown) |
| [`actions/github-script`](https://github.com/actions/github-script) | `v7` | Used extensively for JSON parsing, templating, and ArgoCD API calls |
| [`imranismail/setup-kustomize`](https://github.com/imranismail/setup-kustomize) | `v2` | Install kustomize for manifest builds |
| [`actions/upload-artifact`](https://github.com/actions/upload-artifact) | `v4` | Upload templated manifests and built kustomize manifests |

> 🔒 **Pinning advice:**  
> Use major tag (`@v2`) for safe auto-updates or commit SHA for full reproducibility.

---

## 📦 Artifacts Produced

| Artifact name | Contents | Notes |
|---------------|----------|-------|
| `templated-source-manifests-<cluster>` | Application manifests after **Nunjucks (Jinja2-style) templating** | Shows cluster-specific substitutions and injected UAMI vars |
| `built-kustomize-manifest-<cluster>`  | Final combined YAML output from `kustomize build` | Ready-to-apply manifests, concatenated with `---` delimiters |

> 📝 No artifacts are uploaded when **`delete_only: true`**.

---

## 🧪 Example Usage (Deploy)

```yaml
jobs:
  deploy:
    uses: gitopsmanager/k8s-deploy/.github/workflows/deploy.yaml@v2
    with:
      github_runner: my-self-hosted
      cd_repo: continuous-deployment
      cd_repo_org: my-org
      github_environment: prod
      target_environment: prod
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

> You can override this by patching the `argocd_conn` step in the workflow if your ingress differs.

---

## 🔢 Versioning Policy — Official Release

Starting with this release, all `v2` versions follow the same **stable tagging model** used across GitOps Manager™ Actions.

| Tag | Moves When | Purpose |
|------|-------------|----------|
| **`v2`** | Any new release in the `v2.x.x` series | Always points to the latest stable release (no breaking changes). |
| **`v2.3`** | New patch within that feature line (e.g. `v2.3.5 → v2.3.6`) | Tracks bug fixes and improvements only — no new required inputs. |
| **`v2.3.7`** | Never | Fully immutable, reproducible snapshot. |

All tags will now **increment forward permanently** — no re-use or re-tagging of old versions.  

---

## 📜 License
MIT License  
Copyright (c) 2025 Affinity7 Consulting Ltd

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

Third-party actions listed in [THIRD-PARTY-ACTIONS-AND-TOOLS.md](./THIRD-PARTY-ACTIONS-AND-TOOLS.md).

