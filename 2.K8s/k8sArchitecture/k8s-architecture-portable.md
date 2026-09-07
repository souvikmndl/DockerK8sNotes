# Kubernetes Architecture — Detailed Notes

> Built from the "Master processes" slide (TechWorld with Nana). The slide shows the
> control plane on the right, two worker nodes on the left, and etcd as the "cluster brain."
> These notes fill in the parts the slide abstracts away and add the *why* behind each piece.

---

## 1. The one-sentence mental model

A Kubernetes cluster is a **desired-state engine**. You *declare* what you want the system to
look like (in YAML), it gets persisted, and a set of **control loops** continuously work to make
reality match that declaration — rescheduling, restarting, and recreating things whenever they
drift. Everything else in the architecture is machinery in service of that one idea.

A cluster splits cleanly into two planes:

| Plane | Also called | Job | Runs on |
|---|---|---|---|
| **Control plane** | "master" | Makes global decisions, stores state, reconciles | Dedicated master node(s) |
| **Data plane** | worker nodes | Actually runs your containers | Worker nodes |

The slide's "Master processes" title = control plane. The two boxes labelled *Node 1 / Node 2* =
the data plane.

![Kubernetes architecture overview](diagrams/01-architecture.png)

---

## 2. Control plane components (the "master")

These four boxes in the slide — API Server, Scheduler, Controller Manager, etcd — are the brain of
the cluster. A fifth one (cloud-controller-manager) isn't drawn but matters in managed clusters.

### 2.1 kube-apiserver — the front door

**What it is:** The REST API front-end of the entire cluster. Every interaction — `kubectl`, the
scheduler, controllers, the kubelets on every node, the dashboard — goes *through* the API server.
Nothing talks to etcd directly except the API server.

**Why it's designed this way (hub-and-spoke):** Funnelling everything through one component gives
you a single place to enforce:
- **Authentication** — *who are you?* (certs, bearer tokens, OIDC)
- **Authorization** — *are you allowed to do this?* (RBAC is the usual mode)
- **Admission control** — *should this be mutated or rejected?* (resource quotas, mutating/validating
  webhooks, Pod Security admission)
- **Validation** — is the object schema-correct?

Because it's the only writer to etcd, it's also the single audit and consistency chokepoint. That
decoupling is what lets every other component stay simple: they never coordinate with each other,
they only read/write through the API server.

**Scaling:** Stateless → you run **multiple replicas behind a load balancer** for HA. State lives in
etcd, not in the API server.

**The watch mechanism:** Clients don't poll. They open a **watch** against the API server and get a
stream of changes (backed by etcd's watch). This is the backbone of the informer pattern that
schedulers and controllers use.

### 2.2 etcd — the cluster brain / source of truth

**What it is:** A distributed, strongly-consistent **key-value store**. The slide labels it
"Cluster brain / Key Value Store" — that's exactly right. It holds the *entire* cluster state:
every node, pod spec, ConfigMap, Secret, Service, the desired replica counts, everything.

**Critical distinction:** etcd stores **cluster state, not application data**. Your app's database
does not live here.

**Why etcd specifically:**
- Built on the **Raft** consensus algorithm → linearizable, consistent reads/writes. In a system
  whose whole job is agreeing on "what should the world look like," you cannot tolerate split-brain.
- Deployed in **odd numbers (3, 5, 7)** so a majority quorum can be formed. A 3-node etcd tolerates
  1 failure; a 5-node tolerates 2. Even numbers waste a node without improving fault tolerance.
- Exposes a **watch API** so the API server can efficiently notify everyone about changes.

**Operational reality (interview gold):**
- etcd is the **most backup-critical** thing in the cluster. Lose etcd with no snapshot → you've lost
  the cluster's state (the workloads may keep running, but the control plane can't reason about them).
- **Topologies:** *stacked* (etcd runs on the same nodes as the control plane — simple, default in
  kubeadm) vs *external* (etcd on its own machines — more isolation, more ops overhead).
- Secrets are stored here; **encrypt etcd at rest**, because by default Secrets are only base64-encoded.

### 2.3 kube-scheduler — decides *where*, doesn't place

**What it is:** Watches for **newly created pods that have no node assigned**, and picks the best node
for each. This is exactly what the slide's "30% used" / "60% used" annotations hint at — the scheduler
is resource-aware.

**How it decides (two phases):**
1. **Filtering (predicates):** eliminate nodes that *can't* run the pod — not enough CPU/memory for the
   pod's `requests`, taints the pod doesn't tolerate, unsatisfied `nodeSelector`/affinity, no matching
   volume zone, etc.
2. **Scoring (priorities):** rank the surviving nodes — spread across failure domains, prefer less-loaded
   nodes, honor affinity/anti-affinity preferences, image locality — then pick the highest score.

**The subtle but important part:** *The scheduler does not start the pod.* It only writes the chosen
node name back onto the Pod object (via API server → etcd). The **kubelet** on that node notices the
assignment and actually launches the container. This is Kubernetes' **pull-based** model — the master
never pushes work; nodes pull work destined for them. It keeps the master decoupled from node internals.

Inputs the scheduler considers: resource requests/limits, current node utilization, taints & tolerations,
node/pod affinity & anti-affinity, topology spread constraints, `nodeSelector`, PV/zone locality.

### 2.4 kube-controller-manager — the reconciliation engine

**What it is:** A single binary bundling many **controllers**. Each controller is a **control loop**:

```
observe current state (via watch) → compare to desired state → act to close the gap → repeat forever
```

![Reconciliation loop](diagrams/03-reconciliation.png)

This loop is the heart of Kubernetes' self-healing. Some of the controllers inside it:
- **Node controller** — notices when nodes go unreachable and reacts (marks NotReady, evicts pods).
- **ReplicaSet controller** — if you asked for 3 replicas and 1 dies, it creates a new pod to get back to 3.
- **Deployment controller** — manages rollouts/rollbacks by orchestrating ReplicaSets.
- **Job / CronJob controllers**, **endpoints controller**, **service account & token controllers**, etc.

**Why "level-triggered" matters:** Controllers reconcile based on *current observed state*, not on the
*event* that caused a change. If a controller misses an event (restart, network blip), the next
reconcile still sees the actual state and fixes it. This is why Kubernetes is robust to lost messages —
it's self-correcting by design, not dependent on a perfect event stream.

### 2.5 cloud-controller-manager — not in the slide, but real

In managed/cloud clusters, cloud-specific logic is split out here so it can evolve independently of core
Kubernetes:
- **Node controller** — checks the cloud API to see if a deleted VM should be removed from the cluster.
- **Route controller** — sets up network routes in the cloud.
- **Service controller** — provisions cloud **Load Balancers** when you create a `Service type=LoadBalancer`.

---

## 3. Worker node components (the data plane)

Each *Node* box in the slide runs three essential things. The slide draws the Docker whale + a "kubelet"
badge; the third (kube-proxy) isn't labelled but is always there.

### 3.1 kubelet — the node agent

**What it is:** The primary agent on every node. Responsibilities:
- **Registers the node** with the API server (advertises capacity: CPU, memory, etc.).
- **Watches the API server** for pods scheduled onto *its* node.
- Talks to the **container runtime** to actually start/stop containers so the running set matches the PodSpec.
- Runs **liveness / readiness / startup probes** and restarts unhealthy containers.
- **Reports pod & node status** back to the API server (this is the feedback that keeps etcd's view current).

**Boundary:** the kubelet only manages containers created *by Kubernetes*. Random Docker containers you
start by hand on the node are invisible to it.

### 3.2 kube-proxy — the service networking layer

**What it is:** A per-node network component that implements the **Service** abstraction. A Service gives a
stable virtual IP in front of a changing set of pod IPs; kube-proxy programs the kernel so traffic to that
VIP is load-balanced across the healthy backend pods.

**Modes:** `iptables` (default, rule-based DNAT) or `IPVS` (kernel-level load balancing, scales better to
thousands of services). Newer clusters may use eBPF-based dataplanes (e.g. Cilium) that replace kube-proxy
entirely.

### 3.3 Container runtime — where containers actually run

**What it is:** The software that pulls images and runs containers. The kubelet talks to it over the
**Container Runtime Interface (CRI)**.

> **Note on the slide:** it draws **Docker** directly. This is slightly dated. As of Kubernetes 1.24 the
> built-in Docker shim (`dockershim`) was **removed**. Modern clusters use **containerd** or **CRI-O**
> directly via CRI. (Docker images still work — they're OCI-compliant — you just don't run the Docker
> daemon as the cluster's runtime anymore.)

---

## 4. The smallest unit: the Pod

The little "Pod" boxes wrapping the Docker whales in the slide are **Pods**, not containers — the
distinction matters.

- A **Pod** is the **smallest deployable unit** in Kubernetes — one or more containers that are always
  co-scheduled on the same node.
- Containers in a Pod **share a network namespace** (same Pod IP, can reach each other on `localhost`)
  and can share **storage volumes**.
- Pods are **ephemeral and disposable** — they get a new IP each time they're recreated. You almost never
  create bare Pods; you let a **Deployment → ReplicaSet** manage them so the desired count self-heals.
- Typical pattern: one main app container per pod, optionally plus **sidecars** (logging, proxy, etc.).

---

## 5. How it all fits together — `kubectl apply` end to end

This is the flow the slide implies with "Update / Query" arrows from the Client. Worth memorizing —
it ties every component together:

![kubectl apply end-to-end flow](diagrams/02-deploy-flow.png)

1. You run `kubectl apply -f deployment.yaml`.
2. **API server** authenticates you, authorizes (RBAC), runs admission controllers, validates the object,
   and **persists the Deployment to etcd**.
3. The **Deployment controller** (in controller-manager) sees a new Deployment via its watch, creates a
   **ReplicaSet**; the ReplicaSet controller creates the required number of **Pod** objects — but with
   **no node assigned yet**.
4. The **scheduler** watches for unscheduled pods, runs filter+score, and **binds** each pod to a node
   (writes the node name back through the API server → etcd).
5. The **kubelet** on the chosen node sees a pod assigned to it, pulls the image via the **container
   runtime**, and starts the container(s).
6. **kube-proxy** updates network rules so the pod is reachable through its Service.
7. The kubelet **reports status** back to the API server; etcd's view now matches reality.

If a pod later dies, the ReplicaSet controller notices the count is below desired and the loop runs again —
**no human involved.** That's the self-healing model in action.

---

## 6. The two ideas that make Kubernetes "click"

1. **Declarative + desired state.** You describe the target ("3 replicas of this image"), not the steps.
   Contrast with imperative ops where you script each action and own the drift. In K8s, drift is the
   controllers' problem, not yours.
2. **Reconciliation loops, level-triggered.** Every controller endlessly compares current vs desired and
   nudges toward desired, judging on *state* rather than *events*. Missed an event? The next loop still
   fixes it. This is why the system is resilient to restarts, network blips, and node failures.

Everything in sections 2–5 is just the plumbing that makes these two ideas real.

---

## 7. High availability of the control plane

The slide shows a single "Master 1" — real production clusters run more:
- **≥3 control-plane nodes**, each running its own API server / scheduler / controller-manager.
- **API servers**: all active behind a load balancer (stateless, so this is easy).
- **scheduler & controller-manager**: run as multiple instances but use **leader election** — only one is
  active at a time to avoid two schedulers fighting over the same pod.
- **etcd**: an odd-sized quorum (3/5) — the real constraint on how many failures you can survive.

---

## 8. Common interview questions (quick-fire)

- **Q: Why does everything go through the API server instead of talking to etcd?**
  Single point for auth/authz/admission/audit, and it decouples every component — they never coordinate
  directly, only via the API server's watch/read/write.

- **Q: The scheduler assigned a pod — did it start the container?**
  No. It only records the node binding. The **kubelet** on that node pulls the work and starts the
  container. Pull-based, not push-based.

- **Q: What's the difference between a Pod and a container?**
  A Pod is a co-scheduled group of ≥1 containers sharing network + storage; it's the smallest schedulable
  unit. Containers are what run inside it.

- **Q: Why must etcd have an odd number of members?**
  Raft needs a majority quorum; odd sizes maximize fault tolerance per node (3→tolerate 1, 5→tolerate 2).

- **Q: A node dies — what happens to its pods?**
  Node controller marks it NotReady, evicts the pods after a grace period; ReplicaSet controller recreates
  them elsewhere; scheduler places them on healthy nodes.

- **Q: "Level-triggered vs edge-triggered" — why does K8s prefer level?**
  It reconciles on observed state, so a missed event self-corrects on the next loop → resilient to lost
  messages and restarts.

- **Q: Where did dockershim go?**
  Removed in v1.24. Kubelet talks to containerd/CRI-O over CRI now. OCI images still run fine.

---

## 9. One-glance component cheat sheet

| Component | Plane | One-liner | Talks to |
|---|---|---|---|
| **kube-apiserver** | control | Front door; auth/authz/admission; only etcd writer | everything |
| **etcd** | control | Consistent KV store; cluster source of truth | API server only |
| **kube-scheduler** | control | Picks a node for unscheduled pods | API server |
| **controller-manager** | control | Runs reconciliation loops (replicas, nodes, jobs…) | API server |
| **cloud-controller-manager** | control | Bridges to cloud APIs (LBs, routes, nodes) | API server + cloud |
| **kubelet** | worker | Node agent; runs & reports pods | API server + runtime |
| **kube-proxy** | worker | Programs Service networking / load balancing | API server + kernel |
| **container runtime** | worker | Pulls images, runs containers (containerd/CRI-O) | kubelet (via CRI) |

---

## 10. kubectl command reference (the ones you actually use)

`kubectl` is the CLI that talks to the **API server**. Mental model: it's a thin client — it builds a
REST call, the API server does authn/authz/admission, and the result comes from etcd. Almost everything
below is a verb (`get`, `describe`, `apply`, `delete`, `logs`, `exec`…) against a resource type.

### 10.1 Setup, context & cluster info

```bash
kubectl version --short                 # client + server version
kubectl cluster-info                    # control plane endpoints
kubectl config get-contexts             # list all contexts (clusters/users)
kubectl config current-context          # which cluster am I pointed at?
kubectl config use-context <ctx>        # switch cluster
kubectl config set-context --current --namespace=<ns>   # set default ns for current context
kubectl api-resources                   # every resource type + shortname + apiGroup
kubectl api-versions                    # supported API group/versions
kubectl explain pod.spec.containers     # built-in docs for any field (great for writing YAML)
```

> **Tip:** most people alias `k=kubectl` and enable shell completion. Nearly every resource has a
> short name — `po` (pods), `svc` (services), `deploy` (deployments), `rs` (replicasets),
> `ns` (namespaces), `no` (nodes), `cm` (configmaps), `ing` (ingress), `sa` (serviceaccounts).

### 10.2 Inspecting resources — `get` / `describe`

```bash
kubectl get pods                        # pods in current namespace
kubectl get pods -A                     # across ALL namespaces (--all-namespaces)
kubectl get pods -n <ns>                # in a specific namespace
kubectl get pods -o wide                # + node, pod IP, nominated node
kubectl get pods -w                     # watch: stream changes live
kubectl get pods -l app=web             # filter by label selector
kubectl get pods --field-selector status.phase=Running
kubectl get all                         # pods, svc, deploy, rs in one shot (current ns)
kubectl get deploy,svc,ing              # multiple types at once

kubectl describe pod <pod>              # full detail + Events (first stop for debugging)
kubectl describe node <node>            # capacity, allocatable, taints, running pods
kubectl get events --sort-by=.lastTimestamp   # cluster events, newest last
```

**Output formatting** (huge for scripting & debugging):

```bash
kubectl get pod <pod> -o yaml           # full manifest as stored in etcd
kubectl get pod <pod> -o json
kubectl get pods -o name                # just resource names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'      # pull specific fields
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
kubectl get pods --sort-by=.metadata.creationTimestamp
```

### 10.3 Creating & applying (declarative is the norm)

```bash
kubectl apply -f manifest.yaml          # create OR update to match file (idempotent, preferred)
kubectl apply -f ./dir/                 # apply every manifest in a directory
kubectl apply -f https://.../thing.yaml # apply straight from a URL
kubectl diff -f manifest.yaml           # preview what apply WOULD change vs live state
kubectl create -f manifest.yaml         # imperative create (errors if it already exists)

# Imperative one-liners (fast for testing / interviews)
kubectl run tmp --image=nginx --restart=Never          # a bare pod
kubectl create deployment web --image=nginx --replicas=3
kubectl create namespace dev
kubectl create configmap app-cfg --from-literal=ENV=prod
kubectl create secret generic db --from-literal=password=s3cr3t
```

**Generate YAML without applying** (the `--dry-run` + `-o yaml` trick — write this in interviews):

```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run tmp --image=nginx --dry-run=client -o yaml > pod.yaml
```

### 10.4 Editing live objects

```bash
kubectl edit deploy/web                 # opens the live manifest in $EDITOR; save = apply
kubectl set image deploy/web nginx=nginx:1.27      # change one container's image (triggers rollout)
kubectl set env deploy/web LOG_LEVEL=debug
kubectl scale deploy/web --replicas=5
kubectl autoscale deploy/web --min=2 --max=10 --cpu-percent=70   # creates an HPA
kubectl patch deploy/web -p '{"spec":{"replicas":4}}'           # surgical JSON/strategic-merge patch
kubectl label pod <pod> tier=frontend                          # add/overwrite a label (--overwrite)
kubectl annotate pod <pod> note="handle with care"
```

### 10.5 Rollouts (Deployments)

```bash
kubectl rollout status deploy/web       # block until rollout completes (or fails)
kubectl rollout history deploy/web      # revision history
kubectl rollout undo deploy/web         # roll back to previous revision
kubectl rollout undo deploy/web --to-revision=3
kubectl rollout restart deploy/web      # recreate all pods (e.g. to pick up a new ConfigMap/Secret)
kubectl rollout pause deploy/web        # batch several edits, then resume for a single rollout
kubectl rollout resume deploy/web
```

### 10.6 Logs, exec & debugging (the day-to-day core)

```bash
kubectl logs <pod>                      # stdout/stderr of the (single) container
kubectl logs <pod> -c <container>       # a specific container in a multi-container pod
kubectl logs -f <pod>                   # follow / tail live
kubectl logs <pod> --previous           # logs from the CRASHED previous instance (debugging restarts)
kubectl logs -l app=web --tail=100      # aggregate logs across pods by label
kubectl logs --since=1h deploy/web

kubectl exec -it <pod> -- /bin/sh       # shell into a running container
kubectl exec <pod> -- env               # run a one-off command
kubectl cp <pod>:/path/file ./file      # copy files out of (or into) a container
kubectl attach -it <pod>                # attach to the main process's streams
kubectl debug <pod> -it --image=busybox --target=<container>   # ephemeral debug container (1.25+)
kubectl debug node/<node> -it --image=ubuntu                    # debug a node via a privileged pod
```

**"Why won't this pod start?" triage order:**
```bash
kubectl get pod <pod> -o wide           # 1. status: Pending / CrashLoopBackOff / ImagePullBackOff?
kubectl describe pod <pod>              # 2. read the Events at the bottom
kubectl logs <pod> --previous           # 3. if it crashed, see why the last run died
```

### 10.7 Networking access

```bash
kubectl port-forward pod/<pod> 8080:80          # local:pod — hit the pod from your laptop
kubectl port-forward svc/web 8080:80            # forward to a Service
kubectl get svc                                  # ClusterIP / NodePort / LB + ports
kubectl get endpoints <svc>                      # which pod IPs a Service actually routes to
kubectl expose deploy/web --port=80 --target-port=8080 --type=ClusterIP   # quick Service
```

### 10.8 Namespaces & resources

```bash
kubectl get ns
kubectl create ns team-a
kubectl delete ns team-a                 # deletes EVERYTHING inside it — careful
kubectl get resourcequota -n team-a
kubectl get limitrange -n team-a
kubectl top pod                          # live CPU/memory per pod (needs metrics-server)
kubectl top node                         # per-node usage
```

### 10.9 Node operations (ops / maintenance)

```bash
kubectl get nodes -o wide
kubectl cordon <node>                    # mark unschedulable (no new pods land here)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data   # evict pods before maintenance
kubectl uncordon <node>                  # put it back into rotation
kubectl taint nodes <node> key=value:NoSchedule       # repel pods that don't tolerate it
kubectl taint nodes <node> key:NoSchedule-            # remove that taint (trailing -)
```

### 10.10 Deleting

```bash
kubectl delete -f manifest.yaml          # delete exactly what the file defines
kubectl delete pod <pod>
kubectl delete pod <pod> --grace-period=0 --force        # force-kill a stuck/terminating pod
kubectl delete pods -l app=web           # by label selector
kubectl delete deploy/web                # deleting the Deployment cascades to its RS + pods
```

### 10.11 Handy flags & shortcuts worth memorizing

| Flag / pattern | What it does |
|---|---|
| `-A` / `--all-namespaces` | operate across every namespace |
| `-o wide` | extra columns (node, IP) |
| `-o yaml` / `-o json` | dump the full object |
| `-w` / `--watch` | stream changes live |
| `-l key=value` | label selector filter |
| `--dry-run=client -o yaml` | generate a manifest without touching the cluster |
| `-c <container>` | target a specific container in a multi-container pod |
| `--previous` | logs of the last crashed instance |
| `-- <cmd>` | everything after `--` runs *inside* the container (exec/run) |
| `kubectl explain <resource>.<field>` | inline schema docs while writing YAML |

> **Two most useful "I'm stuck" commands:** `kubectl describe <resource> <name>` (read the Events) and
> `kubectl logs <pod> --previous` (see why the last run died). They resolve the large majority of
> "why is this broken" questions before you touch a dashboard.
