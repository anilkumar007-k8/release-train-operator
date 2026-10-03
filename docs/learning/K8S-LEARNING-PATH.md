# Kubernetes learning path: 2 days

Goal: write Kubernetes YAML from scratch, troubleshoot broken workloads, and answer interview questions with confidence.

Practice cluster: the local kind cluster `release-train`, namespace `learn`. Your YAML goes in `labs/NN-topic/`.

## How every module works

1. **Learn**: short concept, with the why.
2. **Build**: you write the YAML yourself. No copying; `kubectl explain` and `--dry-run=client -o yaml` are allowed.
3. **Break and fix**: you get a broken setup and diagnose it.
4. **Module test**, which you must pass before moving on:
   - **Practical**: a timed task from a blank file
   - **Quiz**: short concept questions
   - **Interview questions**: answer out loud, as in a real interview, then type your answer
5. **Notes**: corrected answers are saved to `docs/learning/`.

**Pass mark:** the practical works first time or with one fix, and at least 4/5 on the quiz. Otherwise, redo the weak part before moving on.

**Rules for tests:** no Google, no AI, no notes. `kubectl explain` and `--dry-run` are allowed, because they are allowed in the CKA/CKAD exams too.

## Day 1: Core workloads (about 7–8 hours)

### Module 1: Pods and YAML anatomy (~1 h)
- [ ] Namespace `learn` created from your own YAML, set as default
- [ ] A-K-M-S structure, YAML syntax rules
- [ ] `kubectl explain`, `--dry-run=client -o yaml`
- [ ] Pod with env var, `exec`, `logs`, `describe`, `port-forward`
- [ ] Multi-container pod (sidecar sharing `localhost`)

**Test**
- Practical (10 min): a Pod `tools` running `busybox:1.36` with command `sleep 3600`, label `tier: debug`, env `MODE=test`. Prove the env var from inside the pod.
- Quiz:
  1. What are the four top-level fields, and which one do you never write?
  2. Why is `value: 8080` a problem for an env var?
  3. What happens to a bare Pod when its node dies?
  4. Two containers in one pod: how does one reach the other?
  5. What is the difference between `kubectl apply` and `kubectl create`?
- Interview:
  1. What is a Pod, and why does Kubernetes use pods instead of running containers directly?
  2. When would you put two containers in one pod?

### Module 2: Labels, selectors and Deployments (~1.5 h)
- [ ] Labels on objects, `--show-labels`, `-l` filtering
- [ ] Deployment from scratch; the three labels that must match
- [ ] Scale up and down; delete a pod and watch self-healing
- [ ] Rolling update (`set image`), `rollout status`, `rollout history`
- [ ] Ship a broken image, then `rollout undo`
- [ ] `maxSurge` / `maxUnavailable`

**Test**
- Practical (15 min): a Deployment `shop` with 4 replicas of `nginx:1.26`, label `app: shop`, rolling update with `maxUnavailable: 0`. Update it to `nginx:1.27`, then roll back to 1.26.
- Quiz:
  1. Which controller recreates a deleted pod?
  2. What happens if the Deployment's selector doesn't match the template labels?
  3. What does the old ReplicaSet look like after a rollout, and why is it kept?
  4. What do `maxSurge: 1` and `maxUnavailable: 0` mean together?
  5. How do you find which revision is currently running?
- Interview:
  1. Explain how a rolling update works internally.
  2. A deploy went bad in production. Walk me through your rollback.
  3. Deployment vs ReplicaSet vs StatefulSet: when would you use each?

### Module 3: Services, endpoints and DNS (~1.5 h)
- [ ] ClusterIP Service in front of a Deployment
- [ ] `port` vs `targetPort` vs `containerPort`
- [ ] Endpoints / EndpointSlices; watch them change as pods come and go
- [ ] DNS: `svc`, `svc.namespace`, full name; `nslookup` from a debug pod
- [ ] NodePort and LoadBalancer (and why LoadBalancer stays `<pending>` on kind)
- [ ] Headless Service

**Test**
- Practical (15 min): expose the `shop` Deployment as a ClusterIP Service `shop-svc` on port 8080 forwarding to container port 80. From a debug pod in another namespace, curl it using its DNS name.
- Break and fix: a Service with zero endpoints. Find and fix the cause.
- Quiz:
  1. Why do we need Services if pods have IPs?
  2. What does an empty endpoints list tell you?
  3. Give the full DNS name of Service `payment` in namespace `billing`.
  4. Which Service type does a private AKS cluster usually use to expose an app, and how?
  5. Layer 4 vs Layer 7: where do Service and Ingress fit?
- Interview:
  1. A user says "the API is unreachable". Debug it step by step.
  2. How does a request to a ClusterIP actually reach a pod? (kube-proxy)
  3. Why not create one LoadBalancer per service in production?

### Module 4: Resources and probes (~1.5 h)
- [ ] Requests vs limits; CPU millicores and memory units
- [ ] Trigger OOMKilled on purpose and read exit code 137
- [ ] Make a pod Pending with huge requests and read the scheduler event
- [ ] QoS classes: Guaranteed, Burstable, BestEffort
- [ ] Readiness, liveness and startup probes; break each and watch the effect
- [ ] LimitRange and ResourceQuota on a namespace

**Test**
- Practical (15 min): a Deployment with requests 100m/128Mi and limits 250m/256Mi, an HTTP readiness probe on `/`, and a liveness probe with a 10 s initial delay.
- Break and fix: a pod stuck in CrashLoopBackOff caused by a bad liveness probe.
- Quiz:
  1. Memory limit exceeded vs CPU limit exceeded: what happens in each case?
  2. Which value does the scheduler use: requests or limits?
  3. A pod fails readiness. Is it restarted?
  4. What is exit code 137?
  5. What QoS class does a pod with no requests or limits get, and why does it matter?
- Interview:
  1. Explain requests vs limits, and how you would size them for a new service.
  2. Tell me about a probe misconfiguration that could cause an outage.
  3. A pod keeps getting OOMKilled. What do you do?

### Module 5: ConfigMaps and Secrets (~1 h)
- [ ] ConfigMap from literals and from a file
- [ ] Use it as env vars (`envFrom`, `valueFrom`) and as a mounted volume
- [ ] Prove that env vars don't update live and mounted files do
- [ ] Secret; decode it with base64 to see it isn't encrypted
- [ ] Secrets in AKS: Key Vault with the Secrets Store CSI driver (concept)

**Test**
- Practical (15 min): a ConfigMap `app-config` with `LOG_LEVEL=debug` and a file `app.properties`, and a Secret `db-creds` with `password`. A pod reads `LOG_LEVEL` and `DB_PASSWORD` as env vars and mounts `app.properties` at `/etc/app/`.
- Quiz:
  1. Is a Secret encrypted? What is it actually?
  2. Two ways to consume a ConfigMap, and the update behaviour of each?
  3. What happens if a pod references a ConfigMap that doesn't exist?
  4. How do you protect Secrets properly in AKS?
  5. Why not bake config into the image?
- Interview:
  1. How do you manage secrets for apps on Kubernetes in production?
  2. How do you roll out a config change to running pods?

### Module 6: Troubleshooting labs (~1.5 h)
Ten broken setups, one at a time. For each one: find the cause, fix it, and explain it in one sentence.
- [ ] ImagePullBackOff
- [ ] CrashLoopBackOff (app error)
- [ ] CrashLoopBackOff (missing config)
- [ ] OOMKilled
- [ ] Pending (insufficient resources)
- [ ] Service with no endpoints
- [ ] Wrong `targetPort`
- [ ] App listening on 127.0.0.1
- [ ] Readiness probe never passes
- [ ] Deployment selector mismatch

### Day 1 final exam (45 min, CKAD style)
Five mixed tasks from a blank terminal, timed, followed by 10 rapid-fire interview questions. Graded as pass, or a list of weak spots for tomorrow morning.

## Day 2: Platform topics (about 7–8 hours)

A 3-node kind cluster (`learn-multi`) is created for scheduling. `release-train` stays untouched for the project.

### Module 7: Storage (~1.5 h)
- [ ] Volumes: `emptyDir`, `hostPath` (and why it's dangerous)
- [ ] PersistentVolume, PersistentVolumeClaim, StorageClass, dynamic provisioning
- [ ] Access modes; reclaim policies
- [ ] StatefulSet with `volumeClaimTemplates`; stable names `db-0`, `db-1`
- [ ] AKS: `managed-csi` (Azure Disk) vs `azurefile-csi` (Azure Files)

**Test**: a StatefulSet with 2 replicas, each with its own 1Gi PVC; data survives a pod deletion. Plus quiz and interview questions.

### Module 8: Ingress and Gateway API (~1.5 h)
- [ ] Install a controller on kind
- [ ] Host- and path-based routing to two Services
- [ ] TLS with a self-signed certificate
- [ ] The same routing with Gateway API (`Gateway` and `HTTPRoute`)
- [ ] AKS: Application Routing add-on, Application Gateway for Containers

**Test**: route `/shop` and `/pay` to two services through one entry point. Plus quiz and interview questions.

### Module 9: Scheduling (~1.5 h)
- [ ] `nodeSelector`, node affinity (required vs preferred)
- [ ] Pod affinity and anti-affinity (spread replicas across nodes)
- [ ] Taints and tolerations; `NoSchedule` vs `NoExecute`
- [ ] Topology spread constraints
- [ ] `cordon`, `drain`, PodDisruptionBudgets (and how PDBs block AKS upgrades)

**Test**: place a workload only on labelled nodes, spread across nodes, with a PDB; drain a node safely. Plus quiz and interview questions.

### Module 10: Jobs, CronJobs and DaemonSets (~45 min)
- [ ] Job: `completions`, `parallelism`, `backoffLimit`
- [ ] CronJob schedules, `concurrencyPolicy`
- [ ] DaemonSet: one pod per node (log agents, monitoring)

**Test**: a CronJob every 2 minutes that keeps the last 3 successful jobs; a DaemonSet on every node. Plus quiz and interview questions.

### Module 11: RBAC and ServiceAccounts (~1 h)
- [ ] Role, ClusterRole, RoleBinding, ClusterRoleBinding
- [ ] ServiceAccount for a pod; call the API from inside a pod
- [ ] `kubectl auth can-i`, impersonation with `--as`
- [ ] AKS: Entra ID integration, Workload Identity

**Test**: a ServiceAccount that can only list pods in `learn`; prove it can't delete them. Plus quiz and interview questions.

### Module 12: Helm basics (~45 min)
- [ ] Install a chart, override values, upgrade, roll back
- [ ] Turn your Day 1 Deployment and Service into a small chart

**Test**: a chart with different `values-dev.yaml` and `values-prod.yaml` (replicas, image tag).

### Day 2 final: mock interview (~1 h)
A 45-minute DevOps interview covering everything, with scenario questions mixed with your AKS experience, followed by written feedback: strengths, gaps, and what to revise.

## Progress log

| Module | Practical | Quiz | Notes |
|---|---|---|---|
| Diagnostic quiz | – | – | Done 2026-10-03: strong troubleshooting instincts; gaps in YAML, labels, config, resources |
| 1 Pods | | | |
| 2 Deployments | | | |
| 3 Services | | | |
| 4 Resources & probes | | | |
| 5 Config & Secrets | | | |
| 6 Troubleshooting | | | |
| Day 1 exam | | | |
| 7 Storage | | | |
| 8 Ingress / Gateway | | | |
| 9 Scheduling | | | |
| 10 Jobs / DaemonSets | | | |
| 11 RBAC | | | |
| 12 Helm | | | |
| Mock interview | | | |
