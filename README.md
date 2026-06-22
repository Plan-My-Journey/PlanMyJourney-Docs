# Plan My Journey — Documentation

Central documentation for the Plan My Journey AI Travel Planner platform.

## Repository Map

| Repository | Purpose |
|------------|---------|
| [PlanMyJourney-App](https://github.com/Plan-My-Journey/PlanMyJourney-App) | Microservices + React frontend |
| [PlanMyjourney-Terraform](https://github.com/Plan-My-Journey/PlanMyjourney-Terraform) | AWS infrastructure (Terraform) |
| [PlanMyJourney-Gitops](https://github.com/Plan-My-Journey/PlanMyJourney-Gitops) | Kubernetes manifests, Helm, Flux, ArgoCD |
| [PlanMyJourney-Workflows](https://github.com/Plan-My-Journey/PlanMyJourney-Workflows) | Reusable GitHub Actions workflows |
| [PlanMyJourney-Docs](https://github.com/Plan-My-Journey/PlanMyJourney-Docs) | Architecture and runbooks (this repo) |

## Documents

- [ARCHITECTURE.md](./ARCHITECTURE.md) — System architecture overview
- [DEPLOYMENT.md](./DEPLOYMENT.md) — Deployment procedures
- [GITOPS.md](./GITOPS.md) — GitOps with Flux and ArgoCD
- [SCALING.md](./SCALING.md) — HPA and scaling guidance
- [FINOPS.md](./FINOPS.md) — Cost optimization
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) — Common issues and fixes

## Domain & Environment

- **Production domain:** invest-iq.online
- **AWS Region:** us-east-1
- **AWS Account:** 235270183260
- **EKS Cluster:** ai-travel-prod

## Quick Start

1. Apply Terraform from `PlanMyjourney-Terraform`
2. Bootstrap Flux/ArgoCD from `PlanMyJourney-Gitops`
3. Push application code to `PlanMyJourney-App` — CI builds and deploys via GitOps
