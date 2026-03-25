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
