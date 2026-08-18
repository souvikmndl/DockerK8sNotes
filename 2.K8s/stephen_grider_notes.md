# Kubernetes — Notes

Compiled from `simplek8s/` and `complex-k8s/` in this repo, plus the `notes-*.txt`
scratch files. Every manifest here is the **correct, current version** — removed
API groups, unpinned images and the config bugs have been folded out, so what you
read is what you should write.

Companion to [DOCKER_NOTES.md](DOCKER_NOTES.md), which covers the images these
manifests run.

| Folder | What it teaches |
|---|---|
| `simplek8s/` | Pod, Deployment, NodePort Service — the minimum viable cluster |
| `complex-k8s/` | The full 5-service app: ClusterIP, Secrets, PVCs, Ingress |

---

## 1. Why Kubernetes, when Compose already works

Compose runs containers on **one** machine. It can't scale one service across many
machines, and `docker compose up --scale worker=5` still puts all five on the same
host.

Kubernetes is a **cluster** of machines (**nodes**) with a control plane that
decides which node runs what. You never say "run this container on that machine."
You declare *what should exist*, and the cluster continuously works to make
reality match.

```
        ┌──────────────── Cluster ────────────────┐
        │  Control plane (API server, scheduler)  │
        │        decides what runs where          │
        └──────┬────────────────┬─────────────────┘
               ▼                ▼
          ┌─ Node 1 ─┐     ┌─ Node 2 ─┐
          │  Pod     │     │  Pod     │
          │  Pod     │     │  Pod     │
          └──────────┘     └──────────┘
```

The practical payoff: you can scale the `client` to 3 replicas and the `worker` to
1, independently, and the cluster will reschedule anything that dies.

---

## 2. Objects and the declarative model

> With these k8s config files, we don't exactly create containers. We create
> **objects**.

Every YAML file describes one object. The two fields that identify it:

- **`apiVersion`** — which API group the object lives in. A given `apiVersion`
  gives you access to a specific set of object types.
- **`kind`** — the object type.

| `kind` | `apiVersion` | Purpose |
|---|---|---|
| `Pod` | `v1` | Run one or more containers |
| `Service` | `v1` | Networking |
| `PersistentVolumeClaim` | `v1` | Request storage |
| `Secret` | `v1` | Hold sensitive config |
| `ConfigMap` | `v1` | Hold non-sensitive config |
| `Deployment` | `apps/v1` | Manage a replicated set of identical Pods |
| `StatefulSet` | `apps/v1` | Like a Deployment, but with stable identity + storage |
| `Ingress` | `networking.k8s.io/v1` | HTTP routing into the cluster |

The core loop:

```bash
kubectl apply -f client-deployment.yaml   # change the desired configuration of the cluster
kubectl get pods                          # print the status of all running pods
kubectl get <obj_type>
```

`kubectl apply` is **declarative** — you hand over a desired end state and the
control plane reconciles toward it. Apply the same file twice and nothing happens
the second time. Prefer this to imperative `kubectl create`/`kubectl run` for
anything you want to keep.

Apply a whole directory at once:

```bash
kubectl apply -f k8s/
```

---

## 3. Pods

A **Pod** is the smallest deployable unit — one or more containers that share a
network namespace (same IP, they reach each other on `localhost`) and can share
volumes.

> The containers grouped inside a pod must be tightly coupled — integral to each
> other, like a DB, a DB logger and a backup manager. If two containers can be
> deployed and scaled independently, they belong in separate Pods.

`simplek8s/client-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: client-pod
    labels:
        component: web # acts as a selector — this is how Services find this pod
spec:
    containers: # list of containers to run inside this pod
        - name: client # name of the container, for reference, logging and networking
          image: svk72/multi-client # docker image to pull
          ports:
              - containerPort: 3000 # the port nginx listens on inside the image
```

**`containerPort` is documentation, like Docker's `EXPOSE`.** It doesn't open
anything. It tells readers (and some tooling) what the container listens on.

### Don't write bare Pods in practice

A Pod is a leaf object: if it dies, it stays dead, and you can't change most of
its fields after creation. `client-pod.yaml` exists to show the shape of the
object. Everything real goes through a **Deployment**, which creates Pods for you.

---

## 4. Deployments

A Deployment declares "I want N identical Pods matching this template" and keeps
that true — restarting crashed Pods, rescheduling Pods from dead nodes, and
rolling out template changes gradually.

`simplek8s/client-deployment.yaml`, corrected and fleshed out:

```yaml
apiVersion: apps/v1 # deployments live in the apps group, not core v1
kind: Deployment
metadata:
    name: client-deployment
spec:
    replicas: 1 # number of pods to create
    selector:
        matchLabels:
            component: web # tells the deployment which pods it owns.
            # Pods can carry many labels; this is the subset that must match.
    template: # ── everything below is the Pod template ──
        metadata:
            labels:
                component: web # label of the pod. Can hold multiple key/value pairs,
                # but must include every pair in matchLabels above.
        spec:
            containers:
                - name: client # name of the container
                  image: svk72/multi-client
                  ports:
                      - containerPort: 3000 # nginx inside multi-client listens here
                  resources:
                      requests: # what the scheduler reserves for this pod
                          cpu: 50m
                          memory: 64Mi
                      limits: # hard ceiling; exceeding memory gets the container OOM-killed
                          cpu: 200m
                          memory: 128Mi
                  readinessProbe: # until this passes, Services won't send traffic here
                      httpGet:
                          path: /
                          port: 3000
                      initialDelaySeconds: 5
                  livenessProbe: # if this fails, the kubelet restarts the container
                      httpGet:
                          path: /
                          port: 3000
                      initialDelaySeconds: 15
```

### `selector.matchLabels` must be a subset of `template.metadata.labels`

This is the single most common Deployment error. The Deployment finds its Pods by
label, and it stamps those labels onto the Pods it creates. If they disagree, the
API server rejects the object — or worse, on older configs, the Deployment
endlessly creates Pods it doesn't recognise as its own.

`selector` is also **immutable** after creation. Changing it means deleting and
recreating the Deployment.

### Resource requests and probes are not optional in practice

- **No `requests`** → the scheduler treats the Pod as free and will overpack a
  node until everything starts thrashing.
- **No `readinessProbe`** → the Service starts routing traffic the instant the
  container process starts, before the app can serve, so a rollout drops requests.
- **No `livenessProbe`** → a hung-but-alive process never gets restarted.

### Rollouts

Changing the Pod template triggers a **rolling update**: new Pods come up, pass
readiness, then old ones are terminated.

```bash
kubectl rollout status deployment/client-deployment
kubectl rollout history deployment/client-deployment
kubectl rollout undo deployment/client-deployment     # revert to the previous template
kubectl rollout restart deployment/client-deployment  # recreate all pods, same template
```

---

## 5. Labels and selectors — the glue

Nothing in Kubernetes references another object by name for routing. Everything is
wired by **labels**.

```
Deployment.spec.selector.matchLabels  ──┐
                                        ├── component: web ──▶ the Pods
Service.spec.selector                 ──┘
```

```yaml
# the pod carries the label
template:
    metadata:
        labels:
            component: web

# the service looks for it
spec:
    selector:
        component: web
```

Both `component` and `web` are **arbitrary strings**. They could be `tier: frontend`
or `app: banana`. What matters is that the two sides agree. The convention in this
repo is `component: <web|server|worker|redis|postgres>`.

A Service is not "attached" to a Deployment. It selects Pods. If two Deployments
produce Pods with the same label, one Service load-balances across both — which is
exactly how blue/green deploys work.

---

## 6. Services

Pod IPs are ephemeral — every restart gets a new one. A **Service** is a stable
virtual IP + DNS name in front of a changing set of Pods.

Four types:

| Type | Reachable from | Use |
|---|---|---|
| `ClusterIP` | inside the cluster only | the default; service-to-service |
| `NodePort` | outside, on `<nodeIP>:<30000-32767>` | **dev only** |
| `LoadBalancer` | outside, via a cloud LB | one cloud LB (and bill) per service |
| `Ingress` | outside, via one shared entry point | production HTTP routing |

`Ingress` is not really a Service type — it's a separate object kind. See §12.

### NodePort — `simplek8s/client-node-port.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
    name: client-node-port
spec:
    type: NodePort
    ports:
        - port: 3050 # used for internal networking; another pod can connect on this
          targetPort: 3000 # the port inside the pod that we are exposing
          nodePort: 31515 # the port on the node (your machine) used to reach targetPort.
          # Must be in 30000-32767. Omit it and Kubernetes picks one for you.
    selector:
        component: web # find all pods labelled component=web.
        # This connects to metadata.labels on the pod template.
```

The three ports:

```
your browser ──▶ <nodeIP>:31515      nodePort
                      │
                      ▼
                 Service:3050         port      (cluster-internal)
                      │
                      ▼
                    Pod:3000          targetPort (what the container listens on)
```

NodePort is dev-only because it burns a high port on **every** node in the
cluster, gives you no TLS, no host routing, and an ugly URL.

### ClusterIP — `complex-k8s/k8s/client-cluster-ip-service.yaml`

```yaml
apiVersion: v1
kind: Service # ClusterIP is a Service-type object
metadata:
    name: client-cluster-ip-service # name of this object — also its DNS name
spec:
    type: ClusterIP # note the casing, it is a keyword
    selector:
        component: web # must match the pods we are targeting
    ports:
        - port: 3000 # port exposed to other pods in the cluster
          targetPort: 3000 # our code runs on this port inside the pod
```

`type: ClusterIP` is the default and can be omitted, but writing it is clearer.

### Service DNS is how services find each other

Every Service gets a DNS record at its **object name**. Inside the cluster:

```
redis-cluster-ip-service                              # same namespace
redis-cluster-ip-service.default                      # explicit namespace
redis-cluster-ip-service.default.svc.cluster.local    # fully qualified
```

This is the direct analogue of Compose's "service name == hostname" — except the
name is the *Service* object's name, not the Deployment's.

---

## 7. Updating a running deployment (the `:latest` trap)

You push a new image with the same tag and re-apply the Deployment. **Nothing
happens.** The manifest is byte-identical, so there is no change to reconcile —
and even if a Pod restarted, `imagePullPolicy` defaults to `IfNotPresent` for
tagged images, so the node reuses the cached layer.

Three ways out, worst to best:

```bash
# 1. Imperative — works, but the cluster now disagrees with your YAML
kubectl set image deployment/client-deployment client=svk72/multi-client:a1b2c3d

# 2. Force a restart — only helps if the tag actually moved AND
#    imagePullPolicy is Always
kubectl rollout restart deployment/client-deployment

# 3. Correct: tag images with the commit SHA in CI and put that tag in the manifest
```

**Always deploy an immutable tag.** The CI pipeline in
[DOCKER_NOTES.md §10](DOCKER_NOTES.md) tags every image with `${{ github.sha }}`
precisely so the manifest can reference `svk72/multi-client:a1b2c3d`. Changing
that string is a real template change, which triggers a real rollout, and rolling
back is just re-applying the old SHA.

Never run `:latest` in a cluster. You lose the ability to know what's deployed and
the ability to roll back.

---

## 8. Configuration: env vars, ConfigMaps, Secrets

The images read everything from environment variables (see `server/keys.js` in
[DOCKER_NOTES.md §9](DOCKER_NOTES.md)), so the same image runs under Compose and
under Kubernetes with no changes.

### Plain values

```yaml
env:
    - name: REDIS_HOST
      value: redis-cluster-ip-service # the Service object's name, resolved by cluster DNS
    - name: REDIS_PORT
      value: "6379" # must be quoted — env values are strings, and 6379 parses as an int
```

The quoting on `"6379"` is a real requirement, not style. An unquoted number is a
YAML integer and the API server rejects it.

### Secrets

Create the secret imperatively — it holds a password, so it must never live in a
YAML file in git:

```bash
kubectl create secret generic pgpassword --from-literal PGPASSWORD=pa55word
#                             │                         │
#                             │                         └─ the key inside the secret
#                             └─ the secret object's name, used in secretKeyRef
```

- `generic` — an arbitrary key/value secret. The other types are `docker-registry`
  (for private image pulls) and `tls` (a cert/key pair).
- `--from-literal` — value on the command line. Use `--from-file` for certs.

Reference it in a Pod template:

```yaml
env:
    - name: PGPASSWORD # the env var name the app reads
      valueFrom:
          secretKeyRef:
              name: pgpassword # name of the secret object we created
              key: PGPASSWORD # the key inside that secret. Arbitrary —
              # it has no relation to the env var name above.
```

Inspect (but note the values come back base64-encoded):

```bash
kubectl get secrets
kubectl describe secret pgpassword
```

**Secrets are only base64-encoded, not encrypted.** Anyone with read access to the
namespace can decode them, and unless the cluster has encryption-at-rest
configured they sit in plaintext in etcd. For anything real, use a sealed-secrets
or external-secrets operator so the *encrypted* form can live in git.

### ConfigMaps

Same shape, for non-sensitive config:

```yaml
env:
    - name: PGDATABASE
      valueFrom:
          configMapKeyRef:
              name: app-config
              key: PGDATABASE
```

---

## 9. Volumes, PersistentVolumes and PersistentVolumeClaims

Three different things share the word "volume":

| Term | Lifetime | Notes |
|---|---|---|
| **Volume** (e.g. `emptyDir`) | dies with the Pod | scratch space, sidecar sharing |
| **PersistentVolume (PV)** | independent of any Pod | the actual storage on the cluster |
| **PersistentVolumeClaim (PVC)** | independent of any Pod | a *request* for a PV |

A plain Kubernetes `volume` does **not** survive the Pod. That's the opposite of a
Docker named volume. For a database you need a PV.

### The claim — `complex-k8s/k8s/database-persistent-volume-claim.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: database-persistent-volume-claim
spec:
    accessModes:
        - ReadWriteOnce # tells k8s to find storage that supports this access mode
    resources:
        requests:
            storage: 2Gi # we need 2GB
```

You write a **PVC**, not a PV. The PVC is an advertisement: "I need 2Gi that
supports RWO." A **StorageClass** with a dynamic provisioner sees it and creates
the matching PV automatically — an EBS volume on AWS, a PD on GCP, a hostPath dir
on minikube.

```bash
kubectl get storageclass    # what's available; one is marked (default)
kubectl get pv              # the volumes that actually exist
kubectl get pvc             # your claims, and whether they're Bound
```

A PVC stuck in `Pending` almost always means no default StorageClass, or no
provisioner that can satisfy the requested access mode.

Access modes:

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` | mounted read-write by a single **node** |
| `ReadOnlyMany` | mounted read-only by many nodes |
| `ReadWriteMany` | mounted read-write by many nodes (needs NFS/CephFS/EFS) |
| `ReadWriteOncePod` | mounted read-write by exactly one **Pod** |

Most cloud block storage is `ReadWriteOnce` only. This is why you can't naively
scale a Postgres Deployment past 1 replica.

### Mounting it — `complex-k8s/k8s/postgres-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: postgres-deployment
spec:
    replicas: 1
    strategy:
        type: Recreate # kill the old pod before starting the new one.
        # The default RollingUpdate deadlocks here: a ReadWriteOnce volume
        # can't be mounted by the new pod while the old one still holds it.
    selector:
        matchLabels:
            component: postgres
    template:
        metadata:
            labels:
                component: postgres
        spec:
            volumes: # connect this deployment to the PVC by name
                - name: postgres-storage
                  persistentVolumeClaim:
                      claimName: database-persistent-volume-claim
            containers:
                - name: postgres
                  image: postgres:16-alpine
                  ports:
                      - containerPort: 5432
                  volumeMounts: # connects the volume into the container
                      - name: postgres-storage # must match the volume name above
                        mountPath: /var/lib/postgresql/data # where postgres stores its data
                        subPath: postgres # store it in a `postgres/` subdirectory of the volume
                  env:
                      - name: POSTGRES_PASSWORD
                        valueFrom:
                            secretKeyRef:
                                name: pgpassword
                                key: PGPASSWORD
                  resources:
                      requests:
                          cpu: 100m
                          memory: 256Mi
                      limits:
                          memory: 512Mi
```

Two details that bite:

**`mountPath` must be `/var/lib/postgresql/data`, not `/var/lib/postgresql`.**
That's the container's `PGDATA`. Mounting the parent directory looks like it
works — Postgres starts, you can write rows — but the data directory itself is
still on the container's ephemeral layer, so everything vanishes on restart. The
symptom is a database that silently resets, which is far worse than one that
fails loudly.

**`subPath: postgres` is required.** A freshly provisioned ext4 volume contains a
`lost+found` directory. Postgres refuses to `initdb` into a non-empty directory
and crash-loops. `subPath` mounts a subdirectory *of* the volume instead of its
root, sidestepping this.

### Postgres really wants a StatefulSet

A Deployment gives its Pods random names and no stable storage identity. For
anything stateful, `StatefulSet` is the right kind — stable names
(`postgres-0`), and a `volumeClaimTemplates` block that gives each replica its
own PVC. The Deployment above works for a single replica and is fine for
learning, but don't scale it.

In production, the genuinely correct answer for most teams is a managed database
(RDS, Cloud SQL) reached over the network, with no Postgres Pod at all — exactly
what the Elastic Beanstalk compose file in [DOCKER_NOTES.md §9](DOCKER_NOTES.md)
does.

---

## 10. The complete app — `complex-k8s/`

The same five services from the Compose project, now as 11 objects.

```
                    ┌──────────── Ingress ────────────┐
   browser ────────▶│  /      → client-cluster-ip:3000│
                    │  /api/* → server-cluster-ip:5000│
                    └─────────────────────────────────┘
                            │                │
                            ▼                ▼
                  client-deployment    server-deployment
                     (3 replicas)         (3 replicas)
                    component: web      component: server
                                              │
                          ┌───────────────────┴────────────────┐
                          ▼                                    ▼
              redis-cluster-ip:6379              postgres-cluster-ip:5432
                          │                                    │
                    redis-deployment                  postgres-deployment
                    component: redis                  component: postgres
                          ▲                                    │
                          │                                    ▼
                  worker-deployment                    PVC (2Gi, RWO)
                  component: worker
```

Every arrow crossing a box boundary goes through a **Service**. Pods never address
Pods directly.

### `server-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: server-deployment
spec:
    replicas: 3
    selector:
        matchLabels:
            component: server
    template:
        metadata:
            labels:
                component: server
        spec:
            containers:
                - name: server
                  image: svk72/multi-server:<commit-sha>
                  ports:
                      - containerPort: 5000
                  env:
                      # every hostname here is a Service object name, resolved by cluster DNS
                      - name: REDIS_HOST
                        value: redis-cluster-ip-service
                      - name: REDIS_PORT
                        value: "6379"
                      - name: PGUSER
                        value: postgres
                      - name: PGHOST
                        value: postgres-cluster-ip-service
                      - name: PGPORT
                        value: "5432"
                      - name: PGDATABASE
                        value: postgres
                      - name: PGPASSWORD
                        valueFrom:
                            secretKeyRef:
                                name: pgpassword
                                key: PGPASSWORD
                  resources:
                      requests:
                          cpu: 100m
                          memory: 128Mi
                      limits:
                          memory: 256Mi
                  readinessProbe:
                      httpGet:
                          path: /
                          port: 5000
                      initialDelaySeconds: 5
```

Compare the `env` block to `docker-compose-dev.yml` — it's the same seven
variables. Only the *values* changed: Compose service names became Kubernetes
Service names.

### `worker-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: worker-deployment
spec:
    replicas: 1
    selector:
        matchLabels:
            component: worker
    template:
        metadata:
            labels:
                component: worker
        spec:
            containers:
                - name: worker
                  image: svk72/multi-worker:<commit-sha>
                  env:
                      - name: REDIS_HOST
                        value: redis-cluster-ip-service
                      - name: REDIS_PORT
                        value: "6379"
                  resources:
                      requests:
                          cpu: 100m
                          memory: 128Mi
                      limits:
                          memory: 256Mi
```

**The worker has no Service and no `containerPort`.** Nothing ever connects *to*
it — it only subscribes to Redis and pushes results back. A Service would be dead
weight. This is the clearest illustration that Services are for inbound traffic
only.

### `redis-deployment.yaml` / `redis-cluster-ip-service.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: redis-deployment
spec:
    replicas: 1
    selector:
        matchLabels:
            component: redis
    template:
        metadata:
            labels:
                component: redis
        spec:
            containers:
                - name: redis
                  image: redis:7-alpine
                  ports:
                      - containerPort: 6379
                  resources:
                      requests:
                          cpu: 50m
                          memory: 64Mi
                      limits:
                          memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
    name: redis-cluster-ip-service
spec:
    type: ClusterIP
    selector:
        component: redis
    ports:
        - port: 6379
          targetPort: 6379
```

Redis gets no PVC here on purpose — it's a cache and a pub/sub bus, and losing it
costs a recomputation, not data. `---` separates multiple objects in one file, if
you prefer grouping a Deployment with its Service.

### Applying it all

```bash
kubectl create secret generic pgpassword --from-literal PGPASSWORD=pa55word
kubectl apply -f k8s/
kubectl get pods
kubectl get services
```

The secret must exist **before** the Deployments that reference it, or their Pods
sit in `CreateContainerConfigError`.

---

## 11. Ingress

Full walkthrough — annotations, the regex rewrite, `pathType` semantics, path
precedence and TLS — is in **[DOCKER_NOTES.md §11](DOCKER_NOTES.md)**, since it
shares the nginx reverse-proxy material. The short version:

**Ingress is only routing *rules*.** Something has to implement them: an **Ingress
Controller**, a pod in your cluster that watches Ingress objects and reconfigures
itself. This repo uses **ingress-nginx** (the Kubernetes community project — not
F5's confusingly-named `nginx-ingress`; their annotations are incompatible).

```bash
minikube addons enable ingress          # minikube
kubectl get ingressclass                # should list `nginx`
```

`complex-k8s/k8s/ingress-service.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: ingress-service
    annotations:
        # treat the `path` fields below as regular expressions
        nginx.ingress.kubernetes.io/use-regex: "true"
        # rewrite the proxied path to capture group 1 — this is what strips /api
        nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
    ingressClassName: nginx # selects which controller implements this Ingress
    rules:
        - http:
              paths:
                  - path: /?(.*)
                    pathType: ImplementationSpecific
                    backend:
                        service:
                            name: client-cluster-ip-service
                            port:
                                number: 3000
                  - path: /api/?(.*)
                    pathType: ImplementationSpecific
                    backend:
                        service:
                            name: server-cluster-ip-service
                            port:
                                number: 5000
```

Most Ingress tutorials online are written against the API that was **removed in
Kubernetes 1.22**. What changed:

| Old (`extensions/v1beta1`) | Current |
|---|---|
| `apiVersion: extensions/v1beta1` | `apiVersion: networking.k8s.io/v1` |
| `annotations: kubernetes.io/ingress.class: nginx` | `spec.ingressClassName: nginx` |
| `backend: { serviceName: x, servicePort: 3000 }` | `backend.service.name` + `backend.service.port.number` |
| no `pathType` | `pathType` is **required** |

The `/api` rewrite does in the cluster exactly what the nginx `rewrite` directive
did under Compose:

```
GET /api/values/all  →  $1 = "values/all"  →  server-cluster-ip-service gets /values/all
```

The Ingress replaces both the `nginx` container **and** the NodePort Service. One
entry point, TLS terminated in one place, and adding a route is a YAML edit rather
than a rebuilt image.

---

## 12. kubectl cheat sheet

```bash
# ---- applying ----
kubectl apply -f client-deployment.yaml   # create or update one object
kubectl apply -f k8s/                     # everything in a directory
kubectl delete -f k8s/                    # remove what those files declare
kubectl diff -f k8s/                      # what would change if I applied this?

# ---- inspecting ----
kubectl get pods
kubectl get pods -o wide                  # + node and pod IP
kubectl get all                           # pods, services, deployments, replicasets
kubectl get deployments,services,pvc
kubectl describe pod <name>               # events at the bottom — always read these first
kubectl get pod <name> -o yaml            # the full object as the server sees it

# ---- debugging ----
kubectl logs <pod>
kubectl logs <pod> -f                     # follow
kubectl logs <pod> --previous             # logs from the crashed instance
kubectl logs -l component=server          # by label, across pods
kubectl exec -it <pod> -- sh              # shell inside a running pod
kubectl port-forward svc/postgres-cluster-ip-service 5432:5432  # reach a ClusterIP locally
kubectl get events --sort-by=.metadata.creationTimestamp

# ---- rollouts ----
kubectl rollout status deployment/client-deployment
kubectl rollout history deployment/client-deployment
kubectl rollout undo deployment/client-deployment
kubectl rollout restart deployment/client-deployment
kubectl scale deployment/client-deployment --replicas=5

# ---- secrets & config ----
kubectl create secret generic pgpassword --from-literal PGPASSWORD=pa55word
kubectl get secrets
kubectl describe secret pgpassword

# ---- storage ----
kubectl get storageclass
kubectl get pv
kubectl get pvc

# ---- minikube ----
minikube start
minikube ip                               # the address to hit in your browser
minikube addons enable ingress
minikube service list
minikube dashboard
```

`kubectl describe` before `kubectl logs`. If a Pod never started, there are no
logs — the reason is in the **Events** section at the bottom of `describe`.

### Reading Pod states

| State | Usual cause |
|---|---|
| `Pending` | no node has room (check `requests`), or a PVC isn't Bound |
| `ImagePullBackOff` | wrong image name/tag, or a private registry with no pull secret |
| `CrashLoopBackOff` | the container starts and exits — read `logs --previous` |
| `CreateContainerConfigError` | a referenced Secret or ConfigMap doesn't exist |
| `Running` but not `Ready` | the readiness probe is failing |

---

## 13. Conventions worth keeping

- **Deployments, never bare Pods.** A Pod that dies stays dead.
- **`selector.matchLabels` ⊆ `template.metadata.labels`**, and `selector` is
  immutable — get it right the first time.
- **Deploy immutable image tags** (the commit SHA), never `:latest`. Re-applying
  an identical manifest does nothing, so `:latest` gives you no way to ship and no
  way to roll back.
- **Pin base images** in the manifests too: `postgres:16-alpine`, `redis:7-alpine`.
- **Always set `resources.requests`** — without them the scheduler thinks your
  Pods are free and will overpack nodes.
- **Always set a `readinessProbe`** — otherwise a rollout sends traffic to Pods
  that aren't serving yet.
- **Quote numeric env values** (`value: "6379"`), or the API server rejects them.
- **Secrets are base64, not encryption.** Create them imperatively, keep them out
  of git, and reach for sealed-secrets or external-secrets when it's real.
- **`strategy: Recreate`** on any Deployment holding a ReadWriteOnce volume, or
  the rollout deadlocks.
- **Mount Postgres at `/var/lib/postgresql/data` with a `subPath`** — the parent
  path silently fails to persist, and no `subPath` crash-loops on `lost+found`.
- **Stateful workloads want a StatefulSet**, or better, a managed database outside
  the cluster.
- **Ingress over NodePort** for anything but local poking.
