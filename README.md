```markdown
# Internal Developer Platform (IDP) — GitOps PoC

An internal developer platform built on **Backstage + ArgoCD + Crossplane**, providing self-service environment provisioning, a service catalog, and governance guardrails — all driven by GitOps.

## Table of Contents

- [Architecture](#architecture)
- [Personas](#personas)
- [Tools](#tools)
- [End-to-End Flow](#end-to-end-flow)
- [Environments](#environments)
- [Prerequisites](#prerequisites)
- [PoC Setup — Step by Step](#poc-setup--step-by-step)
- [Success Criteria](#success-criteria)
- [Roadmap](#roadmap)
- [Open Decisions](#open-decisions)

---

## Architecture

```
Layer 1: Developer Interface   (Backstage)
Layer 2: Delivery / GitOps     (ArgoCD)
Layer 3: Infrastructure        (Crossplane)
Layer 4: Governance & Policy   (OPA/Kyverno + GitHub Rulesets)
Layer 5: Integrations          (Jira — optional, Catalog Graph)
```

Core rule: nothing is created via direct API calls. Every change is a Git commit — auditable, revertible, traceable.

## Personas

| Persona | Responsibilities |
|---|---|
| **Developer** | Consumes templates, tracks status and catalog entries in Backstage |
| **DevOps / Platform Team** | Defines templates, policies, pipelines; operates the platform; approves environment PRs |

## Tools

| Tool | Role |
|---|---|
| **Backstage** | Developer Portal + Service Catalog |
| **GitHub** | Source repos + GitOps repo + Rulesets |
| **ArgoCD** | GitOps continuous delivery |
| **Crossplane** | Infrastructure as CRDs (targets AKS/EKS) |
| **OPA / Kyverno** | Policy enforcement |
| **Jira** | Optional — tracking/approval trail |

## End-to-End Flow

1. Developer opens a Software Template in Backstage ("Provision Environment") and selects: service, branch (dropdown via GitHub API), and target tier
2. Scaffolder action generates the ArgoCD `Application` (and Crossplane claim) manifest and commits it to the GitOps repo
3. PR opened — requires approval from the DevOps team (CODEOWNERS)
4. ArgoCD detects the change and syncs
5. Crossplane reconciles and provisions the underlying infrastructure
6. Backstage catalog updates automatically (component status, relations)

Environment teardown follows the same manual pattern — a "Remove Environment" action in Backstage that reverts the GitOps commit.

## Environments

| Tier | Composition | Isolation |
|---|---|---|
| **dev** | Standalone | Dedicated cluster/namespace |
| **hml + qa** | Shared cluster | Namespace suffix (`-hml`, `-qa`) |
| **stg + prd** | Shared cluster | Namespace suffix (`-stg`, `-prd`) |

**PoC:** 1 kind cluster (extensible to 2), tiers simulated via namespace.
**Production:** 6 clusters — 3 AKS + 3 EKS, one full tier ladder (dev / hml+qa / stg+prd) per cloud.

## Prerequisites

- Docker
- [kind](https://kind.sigs.k8s.io/)
- kubectl, helm
- Node.js 18+ and Yarn
- A test GitHub org/repo + PAT with `repo` and `admin:org` scopes

## PoC Setup — Step by Step

### 1. Create the kind cluster
```bash
kind create cluster --name idp-poc
kubectl cluster-info --context kind-idp-poc
```

### 2. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 3. Install Crossplane
```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane -n crossplane-system --create-namespace
kubectl crossplane install provider crossplane-contrib/provider-kubernetes:main
```
> Uses `provider-kubernetes` (not real AWS/Azure) — validates Composition/Claim mechanics with zero cloud cost.

### 4. Create tier namespaces
```bash
for env in dev hml qa stg prd; do kubectl create namespace $env; done
```

### 5. Scaffold the GitOps repo
```
gitops-repo/apps/<service>/
├── base/
└── overlays/{dev,hml,qa,stg,prd}/
```

### 6. Bootstrap Backstage
```bash
npx @backstage/create-app@latest --skip-install
cd <app-name>
yarn install
yarn dev
```

### 7. Configure GitHub integration
In `app-config.yaml`:
```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

### 8. Configure the Kubernetes and ArgoCD plugins
Enable `@backstage/plugin-kubernetes` and `@roadiehq/backstage-plugin-argo-cd` (or equivalent) so component pages show live status.

### 9. Build the "New Service" template
Scaffolder steps: `fetch:template` → `publish:github` → `catalog:register`

### 10. Build the "Provision Environment" template
Form fields: `branch` (dropdown via GitHub API), `tier` (enum: dev/hml/qa/stg/prd) → action generates the `Application` + `Claim` manifests and opens a PR on the GitOps repo.

### 11. Set up CODEOWNERS
```
/apps/ @your-org/platform-devops-test
```

### 12. Run an end-to-end test
Create a service → run the "Provision Environment" template → approve the PR → confirm ArgoCD sync → confirm Crossplane reconciliation → confirm catalog update.

### 13. Validate the catalog relations graph
Check that `System`/`Component` relations render correctly in the Catalog Graph plugin.

## Success Criteria

- Developer creates a new service in under 5 minutes via Backstage
- Selecting branch + tier automatically generates a visible `Application` in ArgoCD
- Environment PRs require CODEOWNERS approval before merge
- Crossplane reconciles the claim without manual intervention
- Catalog displays the System's relations diagram correctly

## Roadmap

| Phase | Scope |
|---|---|
| **0 — PoC** | 1-2 kind clusters, namespace-simulated tiers, full stack validation |
| **1 — Real Foundation** | Provision the 6 clusters (3 AKS + 3 EKS), register in ArgoCD, configure Crossplane ProviderConfigs |
| **2 — Self-Service** | Backstage templates: new service + GitHub repo creation + automatic catalog registration |
| **3 — Governance** | Finalized approval group, CODEOWNERS, GitHub Rulesets, Kyverno/OPA |
| **4 — Optional** | Jira integration, FinOps/cost attribution |

## Open Decisions

1. ArgoCD: single control plane for all 6 clusters, or one per cloud?
2. Approval group: native GitHub Team or AD/Entra ID group (depends on current SSO setup)
3. Environment teardown: always manual, or optional TTL as a safety net?
4. Isolation guardrails between stg and prd sharing a cluster (NetworkPolicy, ResourceQuota, node pools)
```

