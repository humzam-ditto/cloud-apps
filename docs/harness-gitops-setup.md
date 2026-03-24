# Harness GitOps Pipeline Setup Guide

This document covers the end-to-end process for setting up a Harness GitOps pipeline for a new app. It was written based on hands-on experience setting up `hello-web` and captures lessons learned along the way.

---

## Overview

Harness GitOps uses ArgoCD under the hood. The pipeline flow is:

```
Pipeline triggered (with input variables)
        ↓
Update Release Repo → Harness writes values to git (e.g. image tag)
        ↓
Merge PR → Harness auto-merges the change into main
        ↓
Fetch Linked Apps → finds the ArgoCD Application via the ApplicationSet
        ↓
ArgoCD syncs → cluster pulls latest state from git
```

---

## Repo Structure

Each app lives under `apps/<app-name>/` and contains:

```
apps/hello-web/
├── Chart.yaml                  # Helm chart definition
├── values.yaml                 # Base defaults
├── values-dev.yaml             # Dev environment overrides
├── values-staging.yaml         # Staging environment overrides
├── values-prod.yaml            # Prod environment overrides
├── harness-release.yaml        # Written by Harness on each pipeline run
├── applicationset.yaml         # ArgoCD ApplicationSet (source of truth)
└── templates/                  # Helm templates (use common library)
```

### values stacking

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
podAnnotations:
  harness.io/version: "0.0.1"
```

Harness updates this on every pipeline run. Keeping it isolated means Harness commits never touch your real values files.

---

## Prerequisites

Before setting up a pipeline for a new app, the following must exist in Harness:

1. **GitOps Agent** — installed in the target cluster, visible and healthy in GitOps → Agents
2. **Repository** — `cloud-apps` connected in GitOps → Repositories with a GitHub PAT (not the `gh auth token` CLI token — create a dedicated PAT with `repo`, `admin:repo_hook`, and `user:email` scopes)
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

Add env values files and `harness-release.yaml`.

### 2. Create the ApplicationSet in Harness

**GitOps → Applications → Application Set → New Application Set**

This is the critical step. The ApplicationSet must be created through the Harness UI — not just as a file in git — so Harness registers it internally.

Key settings:
- **Sync Policy**: `Create-Update` (creates and updates apps, never deletes)
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
        harness.io/serviceRef: <service-id>
        harness.io/envRef: <env-id>
    spec:
      project: <argocd-project>
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

> **Note**: The labels `harness.io/serviceRef` and `harness.io/envRef` are what Harness uses to link the Application to a Service and Environment. These must match exactly.

After creating the ApplicationSet, ArgoCD will automatically generate the child Application (e.g. `hello-web-staging`).

### 3. Sync the applicationset.yaml file in the repo

After creating the ApplicationSet in the UI, copy its YAML (stripped of runtime metadata like `uid`, `resourceVersion`, `managedFields`, `status`) and update `apps/<app-name>/applicationset.yaml` in the repo to keep it as the source of truth.

### 4. Create the Harness Service

**Services → New Service**

- **Deployment type**: Kubernetes (GitOps)
- **Manifests**:
  - **Release Repo Manifest**:
    - File path: `apps/<app-name>/harness-release.yaml`
  - **Deployment Repo Manifest**:
    - File path: `apps/<app-name>/applicationset.yaml`

> **Important**: The Deployment Repo Manifest must point to the `applicationset.yaml` file. Pointing it at `Chart.yaml` or a standalone `Application` manifest will cause the `Fetch Linked Apps` step to fail.

### 5. Create the Environment

**Environments → New Environment**

- Type: Pre-Production (staging) or Production
- After creating, go to the Environment detail page → **GitOps Clusters tab** → link it to the cluster

> **Note**: Infrastructure Definitions are not used in GitOps pipelines. Cluster linkage is done via the GitOps Clusters tab on the Environment.

### 6. Create the Pipeline

**Pipelines → New Pipeline**

Add a pipeline variable for the version/tag:
- Name: `version`
- Type: String
- Value: `<+input>` (runtime input)

Add a **Deploy stage**:
- Deployment type: **Kubernetes**
- Check the **GitOps** checkbox
- **Service tab**: select the service created in step 4
- **Environment tab**: select the environment, uncheck "Deploy to all clusters", select the specific cluster

#### Execution steps (auto-generated, configure each):

**Update Release Repo**:
- Check **Allow Empty Commit** (prevents failure when value hasn't changed)
- Add variable:
  - Name: `harnessVersion`
  - Type: String
  - Value: `<+pipeline.variables.version>`

**Merge PR**: leave as default

**Fetch Linked Apps**: leave as default — it finds the Application via the ApplicationSet name from the Deployment Repo manifest

### 7. Run the pipeline

When running, enter a version string (e.g. `0.0.2`). Harness will:
1. Write `harnessVersion: "0.0.2"` to `harness-release.yaml`
2. Create a PR on a new branch
3. Auto-merge it to `main`
4. ArgoCD detects the change and syncs the cluster

---

## Key Lessons Learned

**ApplicationSet is required, not optional**
The `Fetch Linked Apps` step only works with ApplicationSets registered in Harness. Standalone Applications will always return `Selected: 0`. Create the ApplicationSet through the Harness UI, not just as a file in git.

**The Deployment Repo Manifest must point to applicationset.yaml**
Not `Chart.yaml`, not `argocd-app.yaml`. Harness parses this file to get the ApplicationSet name and find linked Applications.

**GitHub connector must use a dedicated PAT**
The `gh auth token` CLI token does not work. Create a GitHub PAT with `repo`, `admin:repo_hook`, and `user:email` scopes and store it as a Harness secret. When pasting the token into Harness, watch for trailing whitespace/newlines.

**Namespace is set on the ApplicationSet, not the pipeline**
The destination namespace is defined in the ApplicationSet template. The pipeline's Environment tab only needs the cluster — not an Infrastructure Definition.

**harness-release.yaml isolates Harness commits**
By giving Harness a dedicated file to write to, all auto-generated pipeline commits are isolated from hand-crafted values files. This keeps git history clean and prevents Harness from overwriting environment-specific configuration.

**Release Repo and Deployment Repo can be the same repo**
For simplicity, both manifests point to `cloud-apps`. In larger setups these are often split — a separate config repo receives Harness writes while the chart repo remains read-only.

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
