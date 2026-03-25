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
