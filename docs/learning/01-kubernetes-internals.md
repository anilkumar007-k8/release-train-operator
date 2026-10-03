# Learning notes 01: How Kubernetes works inside

The parts of a cluster, how they communicate, and what happens when you create a Deployment. Every command here was run on the local kind cluster.

## 1. The parts

A cluster has two halves: the **control plane** (the brain) and the **nodes** (the workers).

```
                  CONTROL PLANE
 ┌───────────────────────────────────────────────────────┐
 │   ┌──────────┐      ┌────────────────────┐            │
 │   │   etcd   │◄────►│   kube-apiserver   │◄─── kubectl│
 │   │(database)│      │  (the only door)   │            │
 │   └──────────┘      └─────────▲──────────┘            │
 │          ┌────────────────────┼──────────────┐        │
 │   ┌──────┴──────┐   ┌─────────┴────────┐  ┌──┴─────┐  │
 │   │  scheduler  │   │controller-manager│  │  our   │  │
 │   │             │   │ (built-in loops) │  │operator│  │
 │   └─────────────┘   └──────────────────┘  └────────┘  │
 └───────────────────────────────▲───────────────────────┘
                                 │ HTTPS (watch)
 ┌───────────────────────────────┴───────────────────────┐
 │  NODE                                                 │
 │   kubelet ──► containerd ──► containers               │
 │   kube-proxy (Service networking)                     │
 └───────────────────────────────────────────────────────┘
```

| Part | Job |
|---|---|
| **kube-apiserver** | The front door. Every read and write goes through it |
| **etcd** | Key-value database that stores every object |
| **scheduler** | Picks a node for each new pod |
| **controller-manager** | About 40 built-in reconcile loops: Deployment, ReplicaSet, Job, Node, Endpoints... |
| **kubelet** | Agent on each node. Starts and stops containers, reports pod status |
| **containerd** | The container runtime that actually runs containers |
| **kube-proxy** | Programs the node so Service IPs reach the right pods |
| **CoreDNS** | Resolves names like `order-api` to Service IPs |

On kind, all of these run as pods: `kubectl get pods -n kube-system`.

## 2. The golden rule of communication

> Components never talk to each other. Everyone talks only to the API server. Only the API server talks to etcd.

Each component **watches** the API server for the objects it cares about, does its one job, and **writes the result back**. The next component sees that write and takes over. Our operator follows the same rule: it only ever talks to the API server.

### Watch, not polling

A component opens one long-lived HTTPS request, e.g. `GET /api/v1/pods?watch=true`, and the API server **pushes** `ADDED`, `MODIFIED` and `DELETED` events down it as they happen. `kubectl get pods -w` uses the same mechanism.

### resourceVersion and conflicts

Every object has a `metadata.resourceVersion`. If two clients update the same object at the same time, the one holding the older version gets a **409 Conflict** and must re-read and retry. Operator code has to handle this.

### The API is just HTTPS

`kubectl get nodes -v=6` shows the real request:

```
"Response" verb="GET" url="https://127.0.0.1:43793/api/v1/nodes?limit=500" status="200 OK"
```

Our own API is served the same way: `kubectl get --raw /apis/delivery.releasetrain.io/v1alpha1`.

## 3. The journey of a request

```
kubectl ──► 1. Authentication   Who are you?           (certificate / token)
            2. Authorization    Are you allowed?       (RBAC)
            3. Admission        Change or reject it?   (defaults, policies, schema validation)
            4. Stored in etcd
            5. Watchers notified ──► controllers react
```

Our sample Release was rejected at step 3 (`spec: Required value`) by the CRD schema. Our operator never saw it.

## 4. Creating a Deployment: a relay race

`kubectl create deployment web --image=nginx --replicas=2`

1. **API server** stores the Deployment in etcd.
2. **Deployment controller** sees it and creates a **ReplicaSet**.
3. **ReplicaSet controller** sees the ReplicaSet and creates **2 Pods** (no node yet, `Pending`).
4. **Scheduler** sees pods with no node, picks one, writes `spec.nodeName`.
5. **kubelet** on that node sees a pod assigned to it and asks containerd to pull and start nginx.
6. **kubelet** reports status back: `Running`, pod IP.
7. **Endpoints controller** adds ready pods behind any matching Service.

Seven handovers, all through the API server. Replay them with `kubectl get events --sort-by=.metadata.creationTimestamp`.

## 5. Deployment → ReplicaSet → Pod

```
Deployment  web                       "I want nginx, 2 copies, roll out safely"
   └── ReplicaSet  web-5fc9f4bf66     "keep exactly 2 pods of THIS template"
          ├── Pod  web-5fc9f4bf66-m7l78
          └── Pod  web-5fc9f4bf66-w7qx7
```

- A pod name tells the story: `web` (Deployment) + `5fc9f4bf66` (ReplicaSet, a hash of the pod template) + `m7l78` (random).
- When a pod is deleted, the **ReplicaSet controller** recreates it. The Deployment doesn't change at all; the ReplicaSet sees 1 pod where it wants 2.
- Each pod's `metadata.ownerReferences` points to its ReplicaSet, and each ReplicaSet's to its Deployment.

**Why two layers?** For rollouts. Changing the image makes the Deployment create a **new** ReplicaSet and shift pods from the old one to the new one step by step. The old ReplicaSet is kept with 0 replicas, so `kubectl rollout undo` just scales it back up. This matters a lot for Release Train: every rollout and rollback we do is built on this mechanism.

## 6. What if etcd goes down?

- **Running pods keep running.** kubelet and containerd run them locally, and kubelet still restarts crashed containers.
- **Service traffic keeps flowing.** kube-proxy's rules are already programmed on the node.
- **Nothing can change.** The API server can't read or write state: no new pods, no scaling, no deploys, no rescheduling if a node dies.

That's why etcd backups are a core ops task on self-managed clusters. On AKS, EKS and GKE, the cloud runs and backs up etcd for you.

## 7. Look inside etcd

```bash
kubectl exec -n kube-system etcd-release-train-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry --prefix --keys-only | grep -E "deployments|releasetrain"
```

Output on our cluster:

```
/registry/apiextensions.k8s.io/customresourcedefinitions/releases.delivery.releasetrain.io
/registry/apiregistration.k8s.io/apiservices/v1alpha1.delivery.releasetrain.io
/registry/deployments/default/web
/registry/deployments/kube-system/coredns
```

Every object lives under `/registry/<resource>/<namespace>/<name>`, including our CRD.

## 8. Review questions

1. **Who recreated the deleted pod?** The ReplicaSet controller (not the Deployment controller, not the scheduler).
2. **etcd down?** Existing pods keep running; no changes are possible.
3. **Who does our operator talk to?** Only the API server. It changes a Deployment's image; the Deployment controller, ReplicaSet controller, scheduler and kubelet do the rest.

## Next

Lesson 3: rollouts in practice (`kubectl set image`, `rollout status`, `rollout undo`), the building blocks our operator automates.
