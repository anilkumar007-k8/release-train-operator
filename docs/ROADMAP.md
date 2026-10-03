# Roadmap

Part-time pace, about 10–15 hours a week. Dates are estimates; add 30–50% if life gets busy.

| Phase | Version | When | Goal |
|---|---|---|---|
| P0 | – | Month 1 | Foundations |
| P1 | v0.1 | Month 1–2 | Deploy in order, roll back together |
| P2 | v0.2 | Month 2–3 | Multiple features at once (soft launch) |
| P3 | v0.3 | Month 3–4 | Database and GitOps safe |
| P4 | v0.4 | Month 4 | Works everywhere (public launch) |
| P5 | v0.5 | Month 5–6 | Finds dependencies you didn't declare |
| P6 | v0.6 | Month 6–7 | AI advisor |
| P7 | v1.0 | Month 7–8+ | Production grade |

## P0: Foundations

- [ ] Go basics for operator development
- [ ] Kubebuilder project skeleton
- [ ] Local kind cluster and `make test` working
- [ ] Demo shop app: frontend, order-api, payment, parts, notification

## P1 / v0.1: Deploy in order, roll back together

- [ ] `Release` CRD: services, versions, `dependsOn`
- [ ] Dependency graph, cycle detection, deploy order
- [ ] Step-by-step rollout of plain Deployments
- [ ] Health gates from Kubernetes signals: readiness, restarts, CrashLoopBackOff
- [ ] Reverse-order rollback on failure
- [ ] Release status and Events
- [ ] Helm chart, CI with kind

**Demo:** break one of four services on purpose and watch all four roll back.

## P2 / v0.2: Multiple features at once

- [ ] Detect services shared between Releases and merge linked Releases
- [ ] Run independent Releases in parallel; isolate failures
- [ ] Native canary (traffic split by replica count)
- [ ] Prometheus health gates, including errors in calling services
- [ ] Plan preview before anything deploys

## P3 / v0.3: Database and GitOps safe

- [ ] Migration steps as Kubernetes Jobs (Flyway, Atlas, Liquibase, Alembic, ...)
- [ ] Expand/contract rules and destructive SQL detection
- [ ] One-way (point of no return) steps
- [ ] Argo CD and Flux compatibility, no sync fights
- [ ] CLI, GitHub Action, Azure DevOps task

## P4 / v0.4: Works everywhere

- [ ] Rollout drivers: Argo Rollouts, Flagger, Gateway API weights
- [ ] Detect cluster capabilities at startup
- [ ] Hardening for GKE Autopilot, OpenShift, restricted Pod Security
- [ ] Test matrix on AKS, EKS and GKE (Terraform, created and destroyed per run)

## P5 / v0.5: Finds dependencies you didn't declare

- [ ] Learn service calls from service mesh and OpenTelemetry data
- [ ] Warn about undeclared dependencies
- [ ] Detect features that share a container image

## P6 / v0.6: AI advisor

- [ ] Optional, bring your own model (Azure OpenAI, Bedrock, Vertex, Anthropic, Ollama)
- [ ] Plain-English rollback explanations
- [ ] Migration risk review, Release drafts from merged PRs
- [ ] Spending limits and privacy settings

## P7 / v1.0: Production grade

- [ ] Cloud plugins: database snapshots before migrations (Azure, AWS, GCP)
- [ ] OpenFeature integration: switch off a feature instead of rolling back an image
- [ ] Write version bumps back to Git
- [ ] Security review, full docs, upgrade guarantees
- [ ] CNCF Sandbox application
