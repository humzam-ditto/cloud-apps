# Harness GitOps Pipeline Setup Guide

This document covers the end-to-end process for setting up a Harness GitOps pipeline for a new app. It was written based on hands-on experience setting up `hello-web` and `hello-api` and captures lessons learned along the way.

---

## Overview

Harness GitOps uses ArgoCD under the hood. The pipeline flow is:

```
Pipeline triggered (with input variables)
        ↓
Update Release Repo → Harness writes values to git (e.g. version)
        ↓
Merge PR → Harness auto-merges the change into main
        ↓
Fetch Linked Apps → reads the ApplicationSet file, finds the child Application it generated, and triggers a sync
        ↓
ArgoCD syncs → cluster pulls latest state from git
```

---

## Repo Structure

Each app lives under `apps/<app-name>/` and contains:

```
apps/hello-web/
├── Chart.yaml                  # Helm chart definition
├── values.yaml                 # Base defaults (must include harnessVersion)
├── values-dev.yaml             # Dev environment overrides
├── values-staging.yaml         # Staging environment overrides
├── values-prod.yaml            # Prod environment overrides
├── harness-release.yaml        # Written by Harness on each pipeline run
├── applicationset.yaml         # ArgoCD ApplicationSet (source of truth copy)
└── templates/                  # Helm templates (use common library)
```

### Values stacking

ArgoCD applies values files left to right, last file wins:

```
values.yaml → values-staging.yaml → harness-release.yaml
```

- `values.yaml` — base defaults for all environments
- `values-<env>.yaml` — environment-specific overrides (replicas, resources, etc.)
- `harness-release.yaml` — written by Harness pipeline on each run, highest priority

### harness-release.yaml

This file is the target of Harness pipeline writes. Keep it in git with a default value:

```yaml
harnessVersion: "0.0.1"
```

Harness updates `harnessVersion` on every pipeline run. Keeping it isolated means Harness commits never touch your real values files.

### harnessVersion in values.yaml

Every app's `values.yaml` must include `harnessVersion` so the common deployment template can render it as a pod annotation:

```yaml
harnessVersion: "0.0.1"
```

The `common` library chart renders this as `harness.io/version` on the pod, which forces a pod restart on each pipeline run even if the image hasn't changed.

---

## Prerequisites

Before setting up a pipeline for a new app, the following must exist in Harness:

1. **GitOps Agent** — installed in the target cluster, visible and healthy in GitOps → Agents
2. **Repository** — `cloud-apps` connected in GitOps → Repositories (see GitHub connector note in Key Lessons below)
3. **Cluster** — registered in GitOps → Clusters, linked to the Agent

---

## Step-by-Step Setup for a New App

### 1. Add the app to the repo

Create the Helm chart under `apps/<app-name>/` following the structure above. Use the `common` library chart to avoid duplicating templates:

```yaml
# Chart.yaml
dependencies:
  - name: common
    version: "0.1.0"
    repository: "file://../../charts/common"
```

Run `helm dependency update apps/<app-name>` to package the library.

Add env values files, `harness-release.yaml`, and ensure `harnessVersion: "0.0.1"` is in `values.yaml`.

### 2. Create the Harness Service

**Services → New Service**

- **Name**: `<app-name>-service`
- **Deployment type**: Kubernetes
- Check **GitOps**
- **Manifests**:
  - **Release Repo Manifest**:
    - File path: `apps/<app-name>/harness-release.yaml`
  - **Deployment Repo Manifest**:
    - File path: `apps/<app-name>/applicationset.yaml`

Note the Service **Id** shown under the name (e.g. `helloweb service`) — you'll need it in step 3.

> **Important**: The Deployment Repo Manifest must point to `applicationset.yaml`. Pointing it at `Chart.yaml` or a standalone `Application` manifest will cause the `Fetch Linked Apps` step to fail.

### 3. Create the Environment

**Environments → New Environment**

- **Name**: `<app-name>-staging`
- **Type**: Pre-Production

After saving, go to the Environment detail page → **GitOps Clusters tab** → link it to the cluster.

Note the Environment **Id** shown under the name — you'll need it in step 4.

> **Note**: Infrastructure Definitions are not used in GitOps pipelines. Cluster linkage is done via the GitOps Clusters tab on the Environment.

### 4. Create the ApplicationSet in Harness

**GitOps → Applications → Application Set → New Application Set**

This is the critical step. The ApplicationSet **must** be created through the Harness UI — not just as a file in git — so Harness registers it internally. The `Fetch Linked Apps` pipeline step queries Harness's internal registry, not git.

Key settings:
- **Name**: `<app-name>-staging-appset`
- **GitOps Agent**: `org.mgmtdevusw2opsint`
- **Sync Policy**: `Create-Update`
- **Generator Type**: `List`
- **Generator elements**:
  ```yaml
  generators:
    - list:
        elements:
          - cluster: staging
            namespace: <app-name>-staging
  ```
- **Template**:
  ```yaml
  template:
    metadata:
      name: <app-name>-{{.cluster}}
      labels:
        harness.io/serviceRef: <service-id-from-step-2>
        harness.io/envRef: <env-id-from-step-3>
    spec:
      project: mgmt-dev-usw2-ops-int-clusters
      source:
        repoURL: https://github.com/humzam-ditto/cloud-apps
        targetRevision: main
        path: apps/<app-name>
        helm:
          valueFiles:
            - values.yaml
            - values-staging.yaml
            - harness-release.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{.namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
  ```

After creating, ArgoCD will automatically generate the child Application (e.g. `<app-name>-staging`).

### 5. Sync applicationset.yaml to the repo

After creating the ApplicationSet in the UI, copy its YAML, strip runtime metadata (`uid`, `resourceVersion`, `managedFields`, `status`), and commit it to `apps/<app-name>/applicationset.yaml`. This keeps the repo as a source-of-truth reference.

### 6. Create the Pipeline

**Option A — Use the existing Pipeline Template (recommended)**

When creating a new pipeline, select **Use Template** and pick `gitops-deploy-stage`. Supply the Service and Environment as inputs. This reuses the same execution pattern as all other apps.

**Option B — Create from scratch**

**Pipelines → New Pipeline**

Add a pipeline variable:
- Name: `version`
- Type: String
- Value: `<+input>` (runtime input)

Add a **Deploy stage**:
- Deployment type: **Kubernetes**
- Check the **GitOps** checkbox
- **Service tab**: select the service from step 2
- **Environment tab**: select the environment from step 3, uncheck "Deploy to all clusters", select the specific cluster

Configure the **Update Release Repo** step:
- Check **Allow Empty Commit**
- Add variable:
  - Name: `harnessVersion`
  - Type: String
  - Value: `<+pipeline.variables.version>`

Leave **Merge PR** and **Fetch Linked Apps** as default.

### 7. Run the pipeline

Enter a version string (e.g. `1.0.0`). Harness will:
1. Write `harnessVersion: "1.0.0"` to `harness-release.yaml`
2. Create a PR on a new branch
3. Auto-merge it to `main`
4. ArgoCD detects the change and syncs the cluster
5. Pod restarts with annotation `harness.io/version: "1.0.0"`

---

## Key Lessons Learned

**ApplicationSet is required, not optional**
The `Fetch Linked Apps` step only works with ApplicationSets registered in Harness. Standalone Applications will always return `Selected: 0`. Create the ApplicationSet through the Harness UI, not just as a file in git.

**The Deployment Repo Manifest must point to applicationset.yaml**
Not `Chart.yaml`, not a standalone `argocd-app.yaml`. Harness parses this file to get the ApplicationSet name and find linked Applications.

**Create Service and Environment before the ApplicationSet**
The ApplicationSet template requires the Service ID and Environment ID in its labels. Create the Service and Environment first so you have those IDs ready.

**GitHub connector: prefer a GitHub App over a PAT**
A GitHub App is the preferred authentication method for production use — it has finer-grained permissions, doesn't expire, and isn't tied to a personal account. For a POC a PAT works, but plan to migrate before going to prod.

If using a PAT: the `gh auth token` CLI token does not work. Create a dedicated GitHub PAT with `repo`, `admin:repo_hook`, and `user:email` scopes and store it as a Harness secret. When pasting the token into Harness, watch for trailing whitespace/newlines.

**Namespace is set on the ApplicationSet, not the pipeline**
The destination namespace is defined in the ApplicationSet template. The pipeline's Environment tab only needs the cluster — not an Infrastructure Definition.

**harness-release.yaml isolates Harness commits**
By giving Harness a dedicated file to write to, all auto-generated pipeline commits are isolated from hand-crafted values files. This keeps git history clean and prevents Harness from overwriting environment-specific configuration.

**harnessVersion must be wired through the common library chart**
Harness writes `harnessVersion` as a top-level key in `harness-release.yaml`. The `common` deployment template reads this and renders it as the `harness.io/version` pod annotation. Without this wiring, the pod will not restart on pipeline runs and the annotation will stay stale.

**Release Repo and Deployment Repo can be the same repo**
For simplicity, both manifests point to `cloud-apps`. In larger setups these are often split — a separate config repo receives Harness writes while the chart repo remains read-only.

**Reuse the Pipeline Template for new apps**
The `gitops-deploy-stage` pipeline template captures the full execution pattern. New apps should use it rather than building pipelines from scratch.

---

## Identifiers Reference

| Resource | Harness ID |
|---|---|
| Account | `vhalP_TdQWmkLSimDvY0SA` |
| Org | `default` |
| Project | `Playground` |
| Cluster | `org.mgmtdevusw2opsint` |
| ArgoCD Project | `mgmt-dev-usw2-ops-int-clusters` |
| ArgoCD Namespace | `argo` |
