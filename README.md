# Release Train Operator

**Ship features, not services.**

Argo Rollouts deploys one *service* safely. Release Train deploys one *feature* safely: across every service it touches, in the right order, and if anything breaks it rolls them all back together.

> **Status:** Phase 0 (foundations). Project skeleton only; nothing deploys yet. See [docs/ROADMAP.md](docs/ROADMAP.md) and the [learning notes](docs/learning/).

## The problem

A feature like "split the bill" often needs changes in several services owned by different teams:

```
db schema -> payment-service v2.4 -> order-api v3.1 -> notification v1.9 -> frontend v5.0
```

Kubernetes, Argo Rollouts and Flagger understand one service at a time. Nothing knows that these five changes belong together. So teams coordinate by hand with runbooks, release captains, Slack threads and CI scripts. That leads to:

- **Outages from wrong order.** The frontend goes live before its API and users get 404s.
- **Half-deployed releases.** One service fails after three are live, and nobody knows what is safe to roll back.
- **Canaries that look in the wrong place.** order-api looks healthy while the frontend that calls it is failing.
- **Slow delivery.** Coordination is painful, so releases get batched weekly.

## The idea

Describe the feature once:

```yaml
kind: Release
metadata:
  name: split-the-bill
spec:
  services:
    - { name: payment-service, version: v2.4 }
    - { name: order-api,       version: v3.1, dependsOn: [payment-service] }
    - { name: notification,    version: v1.9, dependsOn: [order-api] }
    - { name: frontend,        version: v5.0, dependsOn: [order-api, notification] }
  rollback: all-or-nothing
# Illustrative sketch. The final API will change.
```

The operator then:

1. **Plans** a safe order from the dependency graph.
2. **Rolls out** each service as a canary, one step at a time.
3. **Checks** the changed service and the services that call it.
4. **Gates** each step on health before moving on.
5. **Rolls back** everything already promoted, in reverse order, if a step fails.
6. **Reports** one status for the whole feature.

Several Releases can run at once. Linked features are merged, independent ones run in parallel, and a failure rolls back only what it affects.

## Design principles

- **Runs on any conformant cluster.** The core uses only the Kubernetes API: AKS, EKS, GKE, OpenShift, k3s, kubeadm. Cloud-specific features are optional plugins.
- **Works with what you have.** Plain Deployments by default; Argo Rollouts, Flagger and Gateway API traffic weights when present.
- **GitOps friendly.** Releases are plain YAML. Coexisting with Argo CD and Flux is a day-one requirement.
- **Safe database changes.** Migrations run as Kubernetes Jobs with any tool. Expand/contract is enforced and destructive SQL is flagged.
- **Predictable decisions.** Deploy and rollback logic is deterministic. An optional AI advisor can explain failures, but it never decides.

## Known limits

- If two features ship in the same container image, they can't be rolled back separately. Release Train warns about this; feature flags are the real fix.
- Code rolls back easily; data written in a new format does not.
- Dependencies learned from traffic miss rare calls and queue-based links.

## Contributing

The project is in its design phase. Feedback on the problem and the design is very welcome: open an issue.

## License

[Apache License 2.0](LICENSE)
