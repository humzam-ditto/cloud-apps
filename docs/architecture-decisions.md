# Architecture Decisions

A running list of design areas the team needs to align on before moving to production. Each item describes the tradeoffs and options so the discussion can be informed.

---

## 1. GitOps Sync Governance: Who can trigger a deployment?

**The tension**

ArgoCD automated sync watches the repo and syncs any change pushed to `main` immediately — bypassing the Harness pipeline entirely. This means a direct push to `main` deploys to the cluster with no audit trail in Harness, no approvals, and no pipeline controls.

**Options**

- **Automated sync only** — any push to `main` deploys automatically. Simple but no governance. Fine for dev, risky for prod.
- **Pipeline only, automated sync disabled** — ArgoCD only syncs when Harness explicitly triggers it. The pipeline is the sole deployment path. Requires discipline to not push directly.
- **Pipeline only, enforced** — disable automated sync on the ApplicationSet + branch protection on `main` so only Harness PRs can merge. The pipeline is the enforced front door, no back door exists.

**Recommendation for prod**

Option 3 — disable automated sync and enforce branch protection on `main`. This ensures every deployment is traceable to a Harness pipeline execution.

**Current state**

Automated sync is enabled. Direct pushes to `main` will trigger ArgoCD syncs bypassing the pipeline. Acceptable for POC, needs to be addressed before prod.

---

## 2. ArgoCD Application Management: ApplicationSet vs standalone Application

**Decision made**

Harness GitOps pipelines require ApplicationSets — standalone ArgoCD Applications are not supported by the `Fetch Linked Apps` pipeline step. This is non-negotiable for pipeline-managed deployments.

**How it works**

- ApplicationSets are created through the **Harness GitOps UI** — this registers them in Harness's internal system, which is what the pipeline queries
- An `applicationset.yaml` file is committed to the repo as a source-of-truth copy of what's in Harness, but it is not what drives behavior
- Standalone `Application` manifests (e.g. `argocd-app.yaml`) are not used and should not be committed to the repo

**Per-app requirement**

Every app that needs pipeline-managed deployments must have:
1. An ApplicationSet registered in Harness GitOps UI
2. A corresponding `applicationset.yaml` in the repo for documentation
3. The ApplicationSet labels must match the Harness Service and Environment identifiers exactly (`harness.io/serviceRef`, `harness.io/envRef`)

---

## 3. New App Onboarding: Manual UI setup vs automated provisioning

**The problem**

Currently, onboarding a new app requires several manual steps in the Harness UI: creating the ApplicationSet, Service, Environment, and Pipeline. This doesn't scale and is error-prone as the number of apps grows.

**What needs to be decided**

How should new apps be provisioned? Options:

- **Manual UI** — what we do today. Works for a small number of apps but doesn't scale and has no audit trail for the Harness-side config.
- **Harness Terraform provider** — define all Harness resources (Services, Environments, ApplicationSets, Pipelines) as Terraform. A new app is added by writing a Terraform module and running `terraform apply`. Auditable, repeatable, and codified.
- **Harness API** — programmatically create Harness resources via the REST API, potentially triggered by a self-service workflow or a script in this repo.

**Open questions**

- Should Harness resource definitions (Services, Environments, Pipelines) live in this repo alongside the Helm charts, or in a separate infra repo?
- Can the Pipeline Template (`gitops-deploy-stage`) be provisioned via Terraform/API so it doesn't need to be manually created per project?
- How do we handle the Harness UI-created ApplicationSet being the source of truth today — can that be fully replaced by Terraform so git is the only source of truth?

**Recommendation**

Invest in the Harness Terraform provider before scaling beyond a handful of apps. The manual UI approach will become a bottleneck quickly. This should be explored as part of designing the automated app onboarding workflow.

---

## 4. Pipeline Reuse: Pipeline Templates vs Stage Templates

**Background**

Harness supports two levels of templating:

- **Pipeline Template** — the entire pipeline is a template. Each app's pipeline is an instance of it. Currently what we use (`gitops-deploy-stage` saved as a Pipeline template).
- **Stage Template** — just the Deploy stage is a template. You build a pipeline per app but drop in the reusable stage. Allows more flexibility at the pipeline level (e.g. different variables, multiple stages, app-specific pre/post steps).

**The tension**

With a Pipeline Template, all apps share the exact same pipeline structure. This is simple but inflexible — if one app needs an extra step (e.g. a database migration before deploy), it can't deviate without breaking out of the template.

With a Stage Template, each app has its own pipeline but reuses the core deploy logic. More flexible but slightly more setup per app.

**Open questions**

- Will any apps need app-specific pipeline steps that don't belong in a shared template?
- Should the pipeline template expose Service and Environment as runtime inputs so one template truly covers all apps, or is a separate pipeline per app cleaner?
- As we move toward automated onboarding (see item 3), which approach is easier to provision programmatically?

**Current state**

Using a Pipeline Template (`gitops-deploy-stage v1`). Service and Environment are currently hardcoded in the template rather than runtime inputs, which limits true reuse. This needs to be fixed before the template can be used generically across all apps.

---

## 5. Pipeline Template Versioning: How to safely roll out template changes

**Background**

Harness Pipeline Templates support versioning (`v1`, `v2`, etc.) and a `STABLE` tag that pipelines can pin to. Understanding how this works is important before rolling out template changes in production.

**How versioning works**

- Pipelines pin to either a specific version (e.g. `v1`) or to `STABLE`
- `STABLE` is a floating tag — it points to whichever version you promote to stable
- Editing a version in place causes pipelines using it to drift and require manual reconciliation
- Creating a new version leaves existing pipelines untouched

**The right workflow for template changes**

1. Save changes as a new version (e.g. `v2`) rather than overwriting the current stable version
2. Test `v2` by creating a throwaway pipeline explicitly pinned to `v2`
3. Once validated, promote `v2` to **Stable** in the template settings
4. Pipelines pinned to `STABLE` automatically use `v2` on their next run — no reconciliation needed

**Why this matters for prod**

Overwriting a version in place is the equivalent of force-pushing to main — it silently changes what all consumers are running. In prod, this could cause unexpected pipeline behavior mid-deployment. The versioned promotion flow ensures:
- No surprise changes to running pipelines
- A testable path before any change reaches prod pipelines
- A clear rollback path — if `v2` causes issues, re-promote `v1` to Stable

**Current state**

`v1` was edited in place during POC, which required manual reconciliation on all pipelines. Before going to prod, establish the convention of always creating new versions rather than editing existing ones.

---

## 6. Pipeline Rollback: When does it actually trigger?

**Background**

Harness GitOps deploy stages have a built-in rollback that reverts `harness-release.yaml` to the previous values and re-syncs ArgoCD. However, rollback only triggers when a **pipeline step fails** — not when a deployment results in unhealthy pods.

**What the current pipeline catches**

Rollback fires if a pipeline step throws an error:
- `Update Release Repo` fails (bad credentials, repo unreachable)
- `Merge PR` fails (merge conflict, branch protection block)
- `Fetch Linked Apps` fails (ApplicationSet not found, agent unreachable)
- `ArgoCD Sync` fails (invalid manifest YAML, CRD missing)

**What the current pipeline does NOT catch**

Failures that happen asynchronously after ArgoCD reports "Synced":
- Bad image tag → `ImagePullBackOff`
- App crash on startup → `CrashLoopBackOff`
- Readiness probe failing → pod never becomes Ready
- Misconfigured env vars, OOM kills, wrong config values

In these cases, all pipeline steps return green and Harness marks the deployment successful — even though the app is degraded.

**The fix: GitOps Sync step with "Wait until healthy"**

The `GitOps Sync` step (added after `Fetch Linked Apps`) has a **"Wait until healthy"** + **"Fail If Step Times Out"** option. When enabled, Harness polls the ArgoCD app health after sync and fails the step if the app does not reach `Healthy` within the timeout. This closes the loop:

```
GitOps Sync (Wait until healthy)
    ↓ healthy              ↓ unhealthy/timeout
Pipeline succeeds     Pipeline fails → Rollback stage runs
                                      → reverts harness-release.yaml
                                      → re-syncs to last good state
```

**Known limitation: GitOps Sync dynamic app targeting**

The `GitOps Sync` step initially had issues resolving apps dynamically — leaving the Application Name empty caused a fast `SocketTimeoutException` (~46s). This turned out to be an ArgoCD refresh instability issue in the cluster, not a Harness bug. After an ArgoCD tuning PR, dynamic resolution via `Fetch Linked Apps` output works correctly — leaving Application Name empty is fine and the step automatically targets the correct app.

**Known limitation: Rollback via GitOps Revert PR does not work with merge commits**

The rollback stage uses `Revert GitOps PR` to undo the `harness-release.yaml` change. This fails with:

```
MultipleParentsNotAllowedException: Cannot revert commit because it has 2 parents
```

The `Merge PR` step creates a **merge commit** (2 parents). `git revert` on a merge commit requires specifying a parent (`-m 1`), which the `Revert GitOps PR` step does not support.

Attempted workaround: disable merge commits on the GitHub repo to force squash merges. This broke the `Merge PR` step entirely (`405: Merge commits are not allowed`), confirming the step hardcodes regular merge strategy.

**Correct rollback flow**

The full rollback section requires three steps:

```
Revert GitOps PR   →   GitOps Merge PR   →   GitOps Sync
(creates revert PR)    (merges it to main)    (syncs ArgoCD to reverted state)
```

- `Revert GitOps PR` uses the commit ID from `Update Release Repo` (not `Merge PR`) — expression: `<+execution.steps.updateReleaseRepo.updateReleaseRepoOutcome.commitId>`
- `GitOps Merge PR` merges the revert PR to main
- `GitOps Sync` re-syncs ArgoCD to pick up the reverted `harness-release.yaml`

**Current state**

All three rollback steps are configured and validated end-to-end. A bad image tag (`bad-tag`) correctly triggers the full rollback flow — the revert PR is created, merged, and ArgoCD returns to a healthy state automatically.

**Recommendation**

Ensure the rollback `GitOps Sync` step also has `Wait until healthy` enabled so the rollback itself is verified before the pipeline marks the rollback as complete.

---

## 7. Service and Environment Modeling: Scope and granularity

**Background**

In Harness, a **Service** represents what you're deploying (the application) and an **Environment** represents where you're deploying (the infrastructure target). How you define these has downstream effects on ApplicationSet labels, pipeline reuse, and onboarding complexity.

**Best practices**

- **One Service per application** — `hello-web`, `hello-api`, etc. Never create per-environment variants of a service.
- **One Environment per deployment target, shared across apps** — `staging` is a single environment used by all apps deploying there. It is not app-specific.
- The combination of Service × Environment defines a unique deployment target. The ApplicationSet labels wire this:
  ```yaml
  harness.io/serviceRef: helloapiservice   # identifies the Service
  harness.io/envRef: staging               # identifies the Environment
  ```

**Current state and issue**

The POC was set up with app-specific environment IDs (`hellowebstaging`, `helloapistaging`). This is an anti-pattern — it means every new app requires a new Environment to be created in Harness, which doesn't scale.

**Recommendation**

Consolidate to a single `staging` environment (and later `prod`) shared across all apps before scaling beyond the current 2 apps. When paired with automated onboarding (see item 3), adding a new app should only require:
1. Creating a new Service in Harness
2. Creating an ApplicationSet that references the existing shared Environment
3. No new Environment needed

**Impact on existing setup**

Refactoring requires updating the `harness.io/envRef` labels in both ApplicationSets and re-registering them in the Harness GitOps UI. Low effort now, high effort after scaling.

---

## 8. Deployment Model: GitOps vs Native Kubernetes (Hybrid)

**Background**

Harness supports multiple deployment models. This repo uses Harness GitOps (ArgoCD-based) as its primary model. However, not all workload types are a good fit for GitOps — specifically, ephemeral workloads on dynamically created clusters.

**Decision: Hybrid model**

Use the right deployment model per workload type:

| Workload type | Model | Rationale |
|---|---|---|
| Long-lived environments (dev, staging, prod) | Harness GitOps | Drift detection, audit trail, fleet management via ApplicationSets |
| Ephemeral clusters (preview environments, load test clusters, CI integration) | Native Kubernetes | No per-cluster agent overhead; Delegate connects dynamically via kubeconfig |

Both models are supported within the same Harness account/org/project. This is not an either/or architectural commitment.

**Why GitOps is a poor fit for ephemeral clusters**

Every GitOps cluster requires bootstrapping the GitOps Agent (ArgoCD components) onto the cluster, registering the cluster in Harness, and registering ApplicationSets through the Harness UI (or Terraform). For a short-lived cluster, this overhead is pure waste — it gets torn down before the investment pays off. The agent also consumes cluster resources (ArgoCD controller, repo server, Redis) that add no value for temporary workloads.

**How Native Kubernetes handles ephemeral clusters**

The Harness Delegate does not live inside the target cluster — it connects to the Kubernetes API server using credentials (kubeconfig, service account token, IAM role). A single existing Delegate in the same VPC/region can reach any new cluster's API endpoint without additional networking setup.

For fully automated ephemeral cluster lifecycles, the pipeline owns the full flow:

```
Terraform/script provisions cluster
    → cluster endpoint + credentials output as pipeline variables
    → Harness Delegate uses those credentials to deploy
    → (optional) teardown stage destroys cluster after use
```

Infrastructure Definition cluster endpoint and namespace can be marked as runtime inputs, or a provisioner step earlier in the pipeline can output the values and pass them downstream.

**Deployment strategy support**

Native Kubernetes supports Rolling, Canary, and Blue/Green natively — no additional cluster components required. If ephemeral workloads need progressive delivery (e.g. load-testing a canary before promoting), this is available without adding Argo Rollouts.

**What remains GitOps-only**

Drift detection and self-healing are not available in Native Kubernetes. For long-lived environments where cluster state integrity matters, GitOps remains the correct model. Ephemeral clusters do not need drift detection — they are destroyed rather than corrected.

**Current state**

Not yet implemented. GitOps is the only active deployment model. Native Kubernetes pipeline support should be built when the first ephemeral workload use case is identified.

---

## 9. Deployment Health Gating: Harness CV vs Argo Rollouts AnalysisTemplates

**Background**

When using progressive delivery (canary, blue/green), something needs to decide whether a deployment is healthy enough to promote to the next traffic percentage — or whether to abort and roll back. Two tools can fill this role: Harness Continuous Verification (CV) and Argo Rollouts AnalysisTemplates. They are complementary, not competing — but understanding what each does is important before designing rollout pipelines.

**What each tool does**

Argo Rollouts controls **traffic splitting mechanics**. It manages how traffic is shifted between stable and canary pods, integrating with ingress controllers and service meshes (Istio, NGINX, ALB) to do weighted routing. It defines the rollout steps and cadence in a `Rollout` manifest. It does not evaluate whether the canary is healthy — it only manages the percentages.

Harness CV evaluates **whether the deployment is healthy** by querying your observability stack (Datadog, Prometheus, New Relic, Splunk, etc.). It runs as a pipeline step, compares canary metrics against an ML-derived baseline from prior deployments, and makes a pass/fail judgement. It does not control traffic routing.

**How they combine**

They are complementary. Argo Rollouts handles the mechanics of traffic shifting; CV provides the intelligence about whether to proceed:

```
Argo Rollouts shifts 10% traffic to canary pods
    → Harness CV queries Datadog for error rate, latency, log anomalies
    → CV: healthy ✓ → promote
Argo Rollouts shifts to 50%
    → CV: p99 latency spiked 3x vs baseline ✗ → rollback
    → Argo Rollouts shifts 100% traffic back to stable
```

**What are AnalysisTemplates**

AnalysisTemplates are Argo Rollouts' built-in alternative to CV — a Kubernetes CRD that defines metric queries and pass/fail thresholds directly in YAML. Argo Rollouts runs them during the rollout to gate promotion without needing Harness.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
  - name: error-rate
    interval: 1m
    successCondition: result[0] < 0.05
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"5.."}[1m]))
          /
          sum(rate(http_requests_total[1m]))
```

**Comparison**

| | AnalysisTemplates | Harness CV |
|---|---|---|
| Defined as | Kubernetes CRD in Git | Harness pipeline step |
| Metric sources | Prometheus, Datadog, webhooks, custom jobs | Same + Splunk, ELK, AppDynamics, and more |
| Threshold logic | Manually written (`result < 0.05`) | ML-derived baseline from prior deployments |
| Baseline comparison | Static — you define what healthy means | Dynamic — Harness learns what normal looks like |
| Kubernetes-native | Yes — lives alongside the Rollout manifest | No — lives in Harness outside the cluster |
| Works without Harness | Yes | No |

**Recommendation**

Use Harness CV as the primary health gating mechanism. The ML-based baseline comparison is more powerful than manually tuned thresholds — it catches regressions that are hard to anticipate upfront (e.g. a latency spike that's only anomalous relative to time-of-day patterns). CV also integrates directly into pipeline rollback, so no additional wiring is needed.

Reserve AnalysisTemplates for cases where health check logic must live in Git alongside the rollout definition, or where Harness is unavailable in the execution path.

**Current state**

Not yet implemented. No CV health sources have been configured. This should be designed alongside the first canary or blue/green pipeline.
