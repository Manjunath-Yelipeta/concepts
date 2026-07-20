# Kubernetes

## Why Kubernetes? — Where Docker Stops

Docker gets you as far as *one host*. That's the whole problem:

| Docker's limit | What breaks |
|----------------|-------------|
| **Single host** | You can't rely on one machine — it dies, your app dies |
| **Multiple hosts = a cluster** | A cluster needs a **master** (an orchestrator) and **nodes**. Docker has no such brain. |
| **Cross-host networking** | Containers on different hosts can't resolve each other — you need an **overlay network** |
| **No autoscaling** | Traffic doubles, nothing happens |
| **No load balancing** | Nothing spreads traffic across container copies |
| **No secrets management** | Passwords end up in plain text (as they do in the roboshop compose file) |
| **Unsafe volume management** | Data is pinned to one host's disk |

So the split from here on is:

```
Build the image  →  Docker
Run the image    →  Kubernetes
```

### Docker Swarm vs Kubernetes

**Docker Swarm** is Docker's own native orchestrator. **Kubernetes** is what the industry actually uses:

1. **From Google** — highly stable, huge community
2. **PaaS-like** — integrates with cloud services: secrets managers, load balancers, storage
3. **Better load balancing** — Swarm has native LB, but it's not close to Kubernetes + nginx
4. **Better volume management**
5. **Better networking**
6. **Deployment strategies** — blue/green, rolling update, A/B, canary
7. **Helm charts** — packaged, reusable, templated deployments

---

## Cluster Architecture

A cluster is **master + nodes**:

- **Master (control plane)** — the **orchestrator**. It decides what runs where, and keeps reality matching your declared intent.
- **Worker nodes** — the machines that actually run your containers.

You never tell Kubernetes *how* to do something. You declare **desired state** ("I want 10 copies of this image running") and the control plane's job is to continuously make the cluster match it. That single idea explains almost everything else in this document.

### Workstation tooling

Four things on the machine you drive the cluster from:

| Tool | Purpose |
|------|---------|
| **Docker** | Build the images |
| **eksctl** | Create/delete the EKS cluster itself |
| **kubectl** | Talk to the cluster — the command you'll live in |
| **aws configure** | Credentials, so eksctl/kubectl can reach AWS |

```bash
eksctl create cluster --config-file=eksctl.yaml
aws eks update-kubeconfig --name roboshop --region us-east-1   # writes cluster access config
eksctl delete cluster --config-file=eksctl.yaml                 # clusters cost money — delete them
```

Authentication and authorisation config lands in **`~/.kube/config`** — that file is what makes `kubectl` able to reach *your* cluster.

### On-demand vs spot

Worker nodes are just EC2 instances, so the usual trade-off applies:

| | **On-demand** | **Spot** |
|---|---|---|
| Cost | Full price | **70–90% discount** |
| Reliability | Yours until you stop it | AWS can **take it back with 2 minutes' notice** |
| Use for | **Production** | dev / test workloads |

Spot is the right default for learning and non-prod. Don't put production on it just because it's cheap.

---

## Everything Is a Resource

**Everything in Kubernetes is a resource**, and every resource is configured with YAML in the same four-part shape:

```yaml
apiVersion:      # which API version this resource belongs to
kind:            # the resource type (Pod, Service, Deployment...)
metadata:        # name, namespace, labels, annotations
spec:            # the desired state — what you actually want
```

Learn this shape once and every new resource type is just a new `kind` with a different `spec`.

```bash
kubectl apply -f 01-namespace.yaml     # create/update from YAML — declarative
kubectl get namespace                   # list
kubectl describe namespace roboshop     # human-readable detail + events
kubectl get namespace roboshop -o yaml  # full YAML as the cluster sees it
kubectl delete -f 01-namespace.yaml     # remove
```

`apply`, `get`, `describe`, `delete` work on **every** resource type. The nouns change; the verbs don't.

### Namespaced vs cluster-scoped

Resources come in two scopes:

| Scope | `NAMESPACED` | Meaning |
|-------|--------------|---------|
| **Namespace level** | `true` | Lives inside a namespace (Pod, Service, Deployment, ConfigMap, Secret) |
| **Cluster level** | `false` | Belongs to the whole cluster (Namespace, Node, PersistentVolume) |

```bash
kubectl api-resources        # the NAMESPACED column tells you which is which
```

The AWS parallel is exact — some things are scoped to a VPC, some aren't:

```
SG          → VPC-scoped        ≈ namespaced resource
R53         → not VPC-scoped    ≈ cluster-scoped resource
CloudFront  → not VPC-scoped    ≈ cluster-scoped resource
```

---

## Namespace

A **namespace** is an **isolated project space** to create your resources in — `roboshop`, `expense`, `amazon`, `hdfc`, `flipkart`. One cluster, many projects, no collisions.

```yaml
# k8-resources/01-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: roboshop
  labels:
    environment: dev
    project: roboshop
    purpose: poc
```

Every roboshop resource then declares `namespace: roboshop` in its metadata. Switching namespaces constantly gets old fast — **kubens** (from the `kubectx` project) sets your default so you can stop typing `-n roboshop`.

---

## Pod

A **pod is the smallest deployable unit** in Kubernetes. You don't run containers directly — you run pods.

A pod contains **one or more containers**, and the containers inside a pod **share the same network space and storage**. Same network space means they reach each other on `localhost`.

```yaml
# k8-resources/02-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx        # pod name
  labels:
    project: roboshop
    environment: dev
    purpose: poc
spec:
  containers:
  - name: nginx      # container name
    image: nginx     # by default pulled from Docker Hub
```

### Multi-container pods

```yaml
# k8-resources/03-multi-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: nginx
    image: nginx
  - name: almalinux
    image: almalinux:9
    command: ["sleep", "1000"]     # overrides the image's CMD
```

Note `command:` — that's the Kubernetes way to override a container's `CMD`, same idea as passing a command to `docker run`.

### Debugging pods

Two failure states you'll hit constantly, and what they actually mean:

| Status | Meaning | Usual cause |
|--------|---------|-------------|
| **ImagePullBackOff** / **ErrImagePull** | The node **can't pull the image** | Authentication issues, or the image address is wrong |
| **CrashLoopBackOff** | The image pulled fine, but the **container won't stay up** | Check the container's command — it's exiting immediately |

`CrashLoopBackOff` is the Kubernetes echo of a Docker lesson: a container needs a foreground process that runs indefinitely, or it exits and Kubernetes restarts it, forever.

```bash
kubectl exec -it nginx -- bash    # shell into a pod (note the -- separator)
kubectl logs <pod>
kubectl describe pod <pod>        # the Events section at the bottom is where the answer usually is
```

---

## Labels and Annotations

Both attach metadata to a resource, but they exist for **opposite audiences**:

| | **Labels** | **Annotations** |
|---|---|---|
| Audience | **Kubernetes itself** | **Systems outside Kubernetes** |
| Purpose | **Selectors** — how resources find each other | Informational — URLs, build info, tooling hints |
| Special characters | **Not allowed** | Allowed |
| Max size | **63 characters** | **256 KB** |

```yaml
# k8-resources/04-labels.yaml
metadata:
  name: labels-demo
  labels:
    project: roboshop
    environment: dev
    purpose: poc
```

```yaml
# k8-resources/05-annotations.yaml
metadata:
  name: annotations
  labels:
    project: roboshop
  annotations:
    imageregistry: "https://hub.docker.com/"
    jenkins-build-url: "https://jenkins.joindevops.com/build/roboshop/job/2"
```

> **Labels are the load-bearing concept.** Services find pods by label. ReplicaSets find pods by label. Deployments find pods by label. If a Service ever selects nothing, a label mismatch is the first thing to check.

---

## Resources — Requests and Limits

Containers consume resources **dynamically**. That's a feature, but with no ceiling a container keeps taking more and more until the **other containers on that host can't get any**. One greedy pod starves its neighbours.

**So always limit resources on your containers.** Two knobs, for CPU and memory:

| | | Meaning |
|---|---|---|
| **`requests`** | **soft limit** | The guaranteed minimum. The scheduler uses this to decide which node the pod fits on. |
| **`limits`** | **hard limit** | The ceiling. The container can never exceed it. |

```yaml
# k8-resources/06-resources.yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:            # soft limit
        cpu: "100m"        # 1000m = 1 CPU
        memory: "128Mi"
      limits:              # hard limit
        cpu: "150m"
        memory: "256Mi"
```

Units worth knowing: CPU is measured in **millicores** (`1000m` = 1 full CPU), memory in `Mi`/`Gi`.

---

## Configuration: ENV, ConfigMap, Secret

Recall the Docker best practice — **never bake configuration into the image**. The same image must run in dev, stage, and prod. Kubernetes gives three ways to inject it.

### 1. Inline `env`

Fine for one-offs, but it's config trapped in the pod definition:

```yaml
# k8-resources/07-env.yaml
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: project
      value: "roboshop"
    - name: course
      value: kubernetes
```

### 2. ConfigMap

A **ConfigMap** holds application configuration as **key-value pairs**, separate from the pod:

```yaml
# k8-resources/08-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  project: roboshop
  course: kubernetes
  trainer: sivakumar
```

Inject the whole thing with `envFrom` — every key becomes an environment variable:

```yaml
# k8-resources/09-pod-config.yaml
spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
    - configMapRef:
        name: nginx-config
```

This is exactly how roboshop wires up its services:

```yaml
# k8-roboshop/catalogue/manifest.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalogue
  namespace: roboshop
data:
  MONGO: "true"
  MONGO_URL: "mongodb://mongodb:27017/catalogue"
```

### 3. Secret

A **Secret** is for sensitive values. Same shape as a ConfigMap, but the values are **base64 encoded**:

```yaml
# k8-resources/10-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
type: Opaque
data:
  username: YWRtaW4K
  password: YWRtaW4xMjMK
```

```yaml
# k8-resources/11-pod-secret.yaml
    envFrom:
    - secretRef:
        name: nginx-secret
```

### Encoding is not encryption

**A Kubernetes Secret is not confidential.** This is the single most misunderstood thing about them.

| | **Encoding** | **Encryption** |
|---|---|---|
| Reversible by anyone | **Yes** — no key needed | No — needs the key |
| Purpose | Safe *transport* of data | **Secrecy** |
| Example | `sivakumar` → `saiavaaakauamara` | `sivakumar` → `gfdhgfdjg95q095745jkdgfhaf49354390` |

Base64 is just a **reversible transformation** — anyone can undo it in one command:

```bash
echo YWRtaW4K | base64 -d      # → admin
```

So a Secret keeps a password out of *casual sight* in a YAML file. It does **not** protect it. Anyone who can read the Secret can read the password. Real protection means an external secrets manager (AWS Secrets Manager, Vault) plus RBAC and encryption at rest.

> This is why roboshop's `mysql` Secret and plain-text `MYSQL_ROOT_PASSWORD` are fine for learning and **not** fine for production.

---

## Services

### Why services exist

**Pods are ephemeral.** They get created and destroyed constantly, and each new pod gets a new IP. So you **can't use a pod IP as an address** — it won't be there tomorrow.

A **Service** is the stable front door. It solves two problems at once:

1. **Pod-to-pod communication** — a stable name that always resolves
2. **Load balancing** — spreads traffic across all matching pods

```
pod  →  service  →  endpoints (podip:container-port)
```

The Service tracks its pods **by label selector**, and maintains the live list of matching pods as its **endpoints**. Pods come and go; the Service name doesn't.

```yaml
# k8-resources/12-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:              # the Service finds pods with THESE labels
    project: roboshop
    tier: frontend
    component: frontend
  ports:
    - protocol: TCP
      port: 80           # service port
      targetPort: 80     # container port
```

This is why roboshop's nginx config can say `proxy_pass http://catalogue:8080/` — `catalogue` is a Service name, and it resolves no matter which pod is alive behind it.

### The three service types

| Type | Reachable from | How |
|------|---------------|-----|
| **ClusterIP** | **Inside the cluster only** (default) | Internal virtual IP + DNS name |
| **NodePort** | **Outside** — the internet | Opens a port on **every worker node** |
| **LoadBalancer** | **Outside** — properly | Provisions a real cloud load balancer |

#### ClusterIP

The default. Internal only — perfect for backend services and databases that should never be publicly reachable. Every roboshop backend (`catalogue`, `user`, `cart`, `mongodb`, `mysql`) uses it.

#### NodePort

Exposes the pod to the outside world:

```
http://<ec2-ip>:<nodePort>  →  ClusterIP  →  pod
```

Kubernetes opens the **same ephemeral port on all worker nodes** — hit *any* node on that port and traffic is forwarded to the ClusterIP and on to a pod. The allowed range is **30000–32767**.

```yaml
# k8-resources/13-service-np.yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007
```

#### LoadBalancer

Provisions a real cloud load balancer in front of the service — the production way to expose something:

```yaml
# k8-resources/14-service-lb.yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
```

Each type builds on the one before: **LoadBalancer → NodePort → ClusterIP → pod**. In roboshop only `frontend` is a LoadBalancer; everything behind it is ClusterIP.

---

## Sets — Managing Pods at Scale

A bare pod is a pet: if it dies, it's gone. Nobody runs bare pods in production. Four controllers manage pods for you:

| Kind | Purpose |
|------|---------|
| **ReplicaSet** | Keep N identical pods running |
| **Deployment** | ReplicaSet + version/rollout management ← **the one you use** |
| **StatefulSet** | Stateful apps needing stable identity and storage (databases) |
| **DaemonSet** | Exactly one pod **per node** (log collectors, monitoring agents) |

### ReplicaSet

**A pod is a subset of a ReplicaSet.** Its one job: make sure the **desired number of pods is running at all times**. Kill a pod and it's replaced immediately.

Pods it creates are named `<replicaset-name>-<random-chars>`.

```yaml
# k8-resources/15-replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 10
  selector:
    matchLabels:              # how the RS finds its pods
      project: roboshop
      tier: frontend
      component: frontend
  template:                   # the pod definition it stamps out
    metadata:
      labels:                 # MUST match the selector above
        project: roboshop
        tier: frontend
        component: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:trixie-otel
```

**The catch:** a ReplicaSet **does not care about image version changes**. Change the image and nothing happens — it sees N pods running and is satisfied. That limitation is the entire reason Deployments exist.

### Deployment

Think about what a release actually meant on a traditional server:

```
1. remove old code
2. download new code
3. restart the server
```

That doesn't work here, because **pods and containers are immutable**. You never change a running container. Instead:

```
1. build a new version of the image
2. change the image version in the pod definition
3. apply it
```

A **Deployment** manages that transition. The hierarchy:

```
Deployment  →  creates ReplicaSet  →  creates Pods
```

```yaml
# k8-resources/16-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 10
  selector:
    matchLabels:
      project: roboshop
      tier: frontend
      component: frontend
      purpose: deployment
  template:
    metadata:
      labels:
        project: roboshop
        tier: frontend
        component: frontend
        purpose: deployment
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### Rolling update

The default strategy, and the payoff for using a Deployment. Going from **v1 → v2** with 4 pods running:

```
create 1 new pod (v2)  →  delete 1 old pod (v1)
create 2nd new pod     →  delete 1 old pod
create 3rd new pod     →  delete 1 old pod
create 4th new pod     →  delete 1 old pod
```

One in, one out — so there's **never a moment with zero pods serving traffic**. Zero downtime, and if the new version is broken you still have old pods up while you roll back.

```bash
kubectl rollout status deployment/frontend -n roboshop
kubectl rollout undo deployment/frontend -n roboshop      # back to the previous ReplicaSet
```

This is where `latest` bites you again: rolling back means pointing at a *specific* previous version. If everything is `latest`, there's nothing to roll back **to** — which is why roboshop pins `joindevops/catalogue:4.0.0`.

---

## ConfigMaps as Files, and Volumes

A ConfigMap holds key-value pairs — but **a file is just a key whose value is the file's contents**. That's how you get a config file into a pod without rebuilding the image.

**The problem it solves:** `nginx.conf` lives inside the frontend image. If it changes, you'd have to rebuild the image, push it, and update the manifest's image version — a full release cycle for a config tweak.

Instead, put the file in a ConfigMap and **mount it as a volume**:

```yaml
# k8-roboshop/frontend/manifest.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx
  namespace: roboshop
data:
  nginx.conf: |              # key = filename, value = the whole file
    user nginx;
    worker_processes auto;
    ...
    location /api/catalogue/ { proxy_pass http://catalogue:8080/; }
    location /api/user/ { proxy_pass http://user:8080/; }
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: frontend
        image: joindevops/frontend:4.0.0
        volumeMounts:                        # container level: where it appears
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf                # mount just this file, not the whole dir
          readOnly: true
      volumes:                               # pod level: attaching the "disk" to the pod
      - name: nginx-config
        configMap:
          name: nginx
          items:
          - key: nginx.conf
            path: nginx.conf
```

Two levels, and the distinction matters:

| | Where | Meaning |
|---|---|---|
| **`volumes`** | **Pod** level | Attach the storage to the pod — like adding a disk |
| **`volumeMounts`** | **Container** level | Where that storage appears inside *this* container |

The Docker parallel is direct:

```
Docker:      volumes: [mongodb]  +  volumeMounts: mongodb:/data/db
Kubernetes:  volumes: (pod)      +  volumeMounts: (container)
```

`subPath` is the detail worth remembering — without it, mounting into `/etc/nginx/` would replace the **entire directory**. `subPath` places just the one file.

---

## Storage — Volumes, PV & PVC

A ConfigMap-as-volume solves *config*. Real **data** — a database's files, uploaded images, logs — is a different problem, and it's the one interviewers push on. Start from the pain:

> **Everything is ephemeral, all the way down.** A pod dies → its data dies. But it's worse than that: on EKS the **nodes themselves are ephemeral** (spot reclaim, autoscaling, upgrades all replace them). So "just store it on the node" doesn't save you either — the node is as disposable as the pod. Durable data has to live **outside the cluster**, on real cloud storage.

### First, know your AWS storage — this *is* an interview question

You can't reason about Kubernetes storage without knowing what's underneath it. Three services, and *when* you'd pick each:

| | **EBS** | **EFS** | **S3** |
|---|---|---|---|
| What it is | **Block** storage — a raw disk | **File** storage — a shared filesystem (NFS) | **Object** storage |
| Mental model | A hard disk you plug in | A network drive everyone maps | A bucket reached over HTTP(S) |
| Attach to how many instances | **One at a time** | **Many at once** | n/a — accessed over HTTP/HTTPS |
| Size | **Fixed** — you provision N GB | **Grows automatically** | Effectively infinite |
| Speed | **Fastest** | Slower | Not a filesystem |
| Location constraint | **Same AZ** as the instance | Anywhere in the network | Region |
| Security group | Not needed | **Required** | n/a |
| Use it for | **OS disks, databases** | Shared *normal* files across many pods | Backups, artifacts, static assets |

The one-liners that make it stick:

- **EBS is like a hard disk; EFS is like a shared/mapped network drive.**
- **Block vs file:** block storage stores data as raw 4 KB blocks on a disk — a *filesystem* sits on top to turn those blocks into files (CRUD on blocks). That's why EBS is a three-step ritual: **format the disk → create a filesystem → mount it.** EFS skips all that — AWS already fixed the filesystem as NFS, so your only step is **mount**.
- **Databases → EBS, always.** EFS can't back a database (latency + file-locking semantics). Normal files that many pods must share → EFS.

### Two families of Kubernetes volume

```
Volumes
├── Ephemeral   → live and die with the pod   (emptyDir, hostPath)
└── Persistent  → outlive the pod             (PV + PVC on EBS/EFS)
```

Same `volumes` (pod level) + `volumeMounts` (container level) wiring you already know — what changes is what sits behind the volume.

### Ephemeral: `emptyDir`

An **empty directory created when the pod is scheduled, deleted when the pod dies.** It's not for keeping data — it's **scratch space shared between containers in the same pod** (remember: containers in a pod share storage).

The classic use is the **sidecar logging pattern**: the app writes logs to the shared dir, a second container reads them and ships them to ELK.

```yaml
# k8-resources/volumes/01-emptyDir.yaml
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /var/log/nginx        # app writes logs here
      name: nginx-logs
  - name: almalinux                     # sidecar
    image: almalinux:9
    command: ["sleep", "1000"]
    volumeMounts:
    - mountPath: /mnt/nginx-logs        # sidecar reads the SAME volume
      name: nginx-logs
      readOnly: true                    # sidecar only reads — good hygiene
  volumes:
  - name: nginx-logs
    emptyDir:                           # empty dir on the node, gone when the pod is
      sizeLimit: 500Mi
```

Two containers, one volume, mounted at *different paths* — that's the whole point of `emptyDir`.

### Ephemeral: `hostPath` — powerful and dangerous

**`hostPath` mounts a directory from the node itself into the pod.** Sounds useful; it's a trap for application data, for one blunt reason:

> A pod is **not pinned to a node**. Delete it and it may reschedule onto a **different** node — where your `hostPath` data doesn't exist. Your data is stranded on the old node.

So `hostPath` is **never for app data.** Its legitimate use is the mirror image: not putting data *in*, but reading the **node's own files out** — shipping **host** logs/metrics to ELK. And you do it **`readOnly: true`** as a guardrail, because giving a pod write access to the host filesystem is a serious security hole.

This is where **DaemonSet** clicks into place — the fourth "set" from earlier:

> **DaemonSet = exactly one pod on every node.** Pair it with `hostPath` and you have an agent on each node reading that node's `/var/log` and shipping it out. That's *the* canonical log/metrics-collector pattern.

```yaml
# k8-resources/volumes/02-host-path.yaml
apiVersion: apps/v1
kind: DaemonSet                          # one per node
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system                 # an ADMIN namespace, not yours
spec:
  template:
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true                 # read the host's logs, never write
      volumes:
      - name: varlog
        hostPath:
          path: /var/log                 # the node's own log directory
```

Note the namespace: `kube-system`. **`hostPath` is an administrator activity**, not something app teams do in their own namespace — it reaches outside the Kubernetes abstraction and touches the host, which is a cluster-admin concern.

### Persistent storage: the org chart is the real lesson

Here's the mental model interviews are actually testing. In an on-prem cluster, getting a disk is a **process**, not a command:

```
1. raise a ticket, get manager approval
2. storage team gets the ticket — you state size, reason
3. their lead approves, a member provisions the disk, hands you the address
4. you email the admin team to wire it in
```

Kubernetes admins don't know storage internals, and storage teams don't know Kubernetes. So Kubernetes put **wrapper objects on top of the raw disk** to divide the responsibility cleanly — **PV, PVC, SC**:

| Object | Full name | Who owns it | Analogy |
|--------|-----------|-------------|---------|
| **PV** | PersistentVolume | **Storage/cluster admin** | The **wallet** (the actual disk) |
| **PVC** | PersistentVolumeClaim | **App developer** | The **request** for spending money |
| **SC** | StorageClass | Admin (enables automation) | The rule that **creates wallets on demand** |

The family chain from the notes nails the separation of concerns:

```
son  →  mother  →  father  →  wallet
Pod  →   PVC    →   PV     →  storage
```

The **son (Pod) never touches the wallet (disk) directly.** He asks his mother (PVC), who goes to the father (PV), who holds the wallet (real EBS/EFS). Each layer only talks to its neighbour — so the app developer writes a **PVC** ("I need 2Gi, read-write") and never has to know the volume ID, the AZ, or the CSI driver. That decoupling *is* the reason PV/PVC exist. If someone asks "why not mount the disk straight into the pod?" — this is the answer: **it separates the person who needs storage from the person who provisions it.**

- **PV** = the logical Kubernetes representation of a real disk. The physical disk, as a K8s object.
- **PVC** = a claim against a PV — "give me this much, with these access rights." The pod references the *claim*, never the PV.

### Static vs dynamic provisioning

| | **Static** | **Dynamic** |
|---|---|---|
| Who creates the disk | **You**, manually (create the EBS volume, then the PV) | **Kubernetes**, automatically via a **StorageClass** |
| When | Ahead of time | The moment a PVC asks |
| Real-world use | Rare / learning | The norm in production |

The example below is **static** — the disk already exists (`volumeHandle: vol-...`) and you're describing it to Kubernetes by hand. Getting there needs plumbing worth naming because it's classic interview/debug territory:

1. Install the **EBS CSI driver** (the thing that lets K8s attach EBS)
2. The **EKS nodes need IAM permission** (`EBSCSIDriverPolicy`) to attach disks — miss this and volumes hang in `Pending`
3. Create the EBS volume **in the same AZ as a node** (EBS is AZ-locked)
4. Write the PV, then the PVC, then reference the PVC in the pod

```yaml
# k8-resources/volumes/03-ebs-static.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-static
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 2Gi
  csi:
    driver: ebs.csi.aws.com              # the EBS CSI driver
    fsType: ext4
    volumeHandle: vol-0298e8f2bbbde0f28  # the pre-created EBS volume (static)
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-static
spec:
  storageClassName: ""                   # "" = don't use the default SC; bind to MY PV
  volumeName: ebs-static                 # explicitly bind to the PV above
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-static
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: persistent-storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-static              # the pod references the CLAIM, not the PV
  nodeSelector:
    topology.kubernetes.io/zone: us-east-1d   # pin the pod to the disk's AZ
```

Two details that trip people up:

- **`storageClassName: ""`** is deliberate. An empty string means "do **not** fall back to the default StorageClass" — you want this PVC to bind to *your* named PV, not have K8s dynamically provision a new disk.
- **`nodeSelector` on the AZ.** EBS is AZ-locked, so the pod *must* land in the same AZ as the disk (`us-east-1d`) or it can never mount it. This is the single most common EBS-on-K8s failure.

### Access modes — asked constantly

Who can mount the volume, and how. Directly reflects EBS-vs-EFS reality (EBS = one node; EFS = many):

| Mode | Meaning |
|------|---------|
| **ReadWriteOnce** (RWO) | One **node** mounts read-write; pods *on that node* can use it → typical **EBS** |
| **ReadOnlyMany** (ROX) | Many **nodes** mount, read-only |
| **ReadWriteMany** (RWX) | Many **nodes** mount read-write → needs **EFS** (EBS can't do this) |
| **ReadWriteOncePod** | Exactly **one pod** read-write — stricter than RWO (node-wide) |

Interview trap: RWO is *once per **node**,* not per pod — several pods on the same node can share an RWO volume. If you truly need one-pod exclusivity, that's **ReadWriteOncePod**.

### Reclaim policy — what happens to the data when the PVC is deleted

| Policy | PV object | The actual data |
|--------|-----------|-----------------|
| **Delete** | Removed | **Destroyed** — the underlying disk is deleted too |
| **Retain** | Kept | **Preserved** — data survives even after PVC/pod are gone (you clean up manually) |
| **Recycle** | Kept | **Wiped** but the disk stays (deprecated) |

The one that matters in production: **use `Retain` for anything you can't afford to lose.** `Delete` is convenient and will happily take your database down with the PVC. This is a real blast-radius question — "what happens to the data if someone `kubectl delete pvc`?" — and the honest answer is "depends on the reclaim policy," which is exactly the point.

### Where the scheduler comes in

The **scheduler** decides which node a pod runs on, weighing many factors — and storage is one of them. That EBS `nodeSelector` above is you *overriding* the scheduler to force the pod into the disk's AZ. Worth knowing the scheduler is steerable (`nodeSelector`, affinity, taints/tolerations) precisely because AZ-locked storage sometimes forces your hand.

---

## Health Checks — Probes

Kubernetes can't know whether your app is *actually working* — a running process isn't the same as a healthy app. **Probes** are how you tell it. Three of them, answering three different questions:

| Probe | Question | Runs | On failure |
|-------|----------|------|-----------|
| **startupProbe** | *Has the container finished booting?* | **Once**, at startup | Keeps waiting; other probes don't start until this passes |
| **readinessProbe** | *Is the container ready to accept traffic?* | **Continuously** | **Removes the pod from the Service endpoints** — no traffic, but no restart |
| **livenessProbe** | *Is the application still alive?* | **Continuously** | **Restarts** the pod |

The distinction that matters:

- **Readiness failing** → "don't send it traffic right now" (it might be busy or warming up). Stop routing, wait.
- **Liveness failing** → "it's wedged and won't recover." Long-running apps get stuck holding locks; restarting is the only fix.

**startupProbe** exists so slow-booting apps don't get killed by a liveness probe before they've even started.

```yaml
# k8-roboshop/payment/manifest.yaml
containers:
- name: payment
  image: joindevops/payment:4.0.0
  ports:
  - name: liveness-port
    containerPort: 8080
  startupProbe:
    httpGet:
      path: /health
      port: liveness-port
    failureThreshold: 12        # 12 × 10s = 5 minutes to boot before giving up
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /health
      port: liveness-port
    periodSeconds: 10
  livenessProbe:
    httpGet:
      path: /health
      port: liveness-port
    periodSeconds: 10
```

`failureThreshold: 12` × `periodSeconds: 10` = **the app gets 5 minutes to start**. That's the startup probe's whole job: buy slow starters enough time.

The app needs an endpoint to answer — roboshop's nginx config defines one:

```nginx
location /health {
  stub_status on;
  access_log off;
}
```

---

## The Standard Resource Set

Putting it together — what a real service in roboshop is made of:

| # | Resource | Why |
|---|----------|-----|
| 1 | **Namespace** | `roboshop` — the project's isolated space |
| 2 | **Deployment** | **Not** a bare Pod, **not** a ReplicaSet — you want rollouts |
| 3 | **ConfigMap** | Non-sensitive configuration |
| 4 | **Secret** | Sensitive values |
| 5 | **Service** | ClusterIP for internal, LoadBalancer to expose |

That's the pattern every `k8-roboshop/*/manifest.yaml` follows. The shape of the app in Kubernetes:

```
                      LoadBalancer Service
                              │
                          frontend  (nginx + configMap-mounted nginx.conf)
                              │  proxy_pass
        ┌──────────┬──────────┼──────────┬──────────┐
    catalogue    user       cart     shipping   payment      ← ClusterIP services
        │          │          │          │          │
     mongodb   mongodb     redis      mysql    rabbitmq      ← ClusterIP services
                +redis
```

Compare it to the Compose file from the Docker sessions and it's the same application — the same names, the same DNS-by-name wiring, the same dependencies. What changed is that Kubernetes now handles the scaling, the health, the rollouts, and the load balancing that Compose couldn't.

---

## Quick Reference

| Concept | One-liner |
|---------|-----------|
| Build vs run | Build the image → **Docker**; run the image → **Kubernetes** |
| Docker's limits | Single host, no orchestrator, no autoscaling/LB/secrets, unsafe volumes |
| **Cluster** | Master (orchestrator/control plane) + worker nodes |
| **Declarative** | You declare desired state; the control plane makes reality match |
| eksctl / kubectl | Create the cluster / talk to the cluster |
| `~/.kube/config` | Cluster authentication + authorisation config |
| **Spot vs on-demand** | Spot = 70–90% off but 2-min reclaim notice → dev/test, never prod |
| **Everything is a resource** | Every resource is YAML: `apiVersion`, `kind`, `metadata`, `spec` |
| `kubectl apply/get/describe/delete -f` | The four verbs — they work on every resource type |
| Namespaced vs cluster-scoped | `NAMESPACED: true` (Pod, Service) vs `false` (Namespace, Node) |
| `kubectl api-resources` | Lists every resource type + whether it's namespaced |
| **Namespace** | Isolated project space (`roboshop`, `expense`) |
| **kubens** | Set your default namespace; stop typing `-n roboshop` |
| **Pod** | Smallest deployable unit; 1+ containers sharing network space and storage |
| **ImagePullBackOff** | Node can't pull the image — auth issue or wrong image address |
| **CrashLoopBackOff** | Container won't stay up — check the container command |
| `kubectl exec -it <pod> -- bash` | Shell into a pod (note the `--`) |
| **Labels** | Metadata **for Kubernetes** — selectors; no special chars, max 63 chars |
| **Annotations** | Metadata **for external systems** — URLs, build info; special chars ok, max 256 KB |
| `requests` | **Soft limit** — guaranteed minimum; the scheduler places pods by this |
| `limits` | **Hard limit** — the ceiling a container can never exceed |
| CPU units | `1000m` = 1 CPU; memory in `Mi`/`Gi` |
| Why limit resources | Unlimited containers starve every other container on the host |
| **ConfigMap** | Key-value application config, kept out of the image |
| **Secret** | Same as ConfigMap but base64-encoded; `type: Opaque` |
| `envFrom: configMapRef/secretRef` | Inject every key as an environment variable |
| **Encoding ≠ encryption** | `base64 -d` reverses a Secret with no key — Secrets are **not confidential** |
| **Why services exist** | Pods are ephemeral — their IPs change, so you need a stable name |
| pod → service → endpoints | The Service tracks matching pods **by label** as its endpoints |
| **ClusterIP** | Default; internal to the cluster only |
| **NodePort** | Opens the same port (**30000–32767**) on every worker node → ClusterIP → pod |
| **LoadBalancer** | Provisions a real cloud load balancer; the production way to expose |
| `port` vs `targetPort` | Service port vs container port |
| **ReplicaSet** | Keeps N pods running; pods named `<rs-name>-<random>`; **ignores image changes** |
| **Deployment** | ReplicaSet + rollouts — what you actually use |
| **StatefulSet** / **DaemonSet** | Stable identity for stateful apps / one pod per node |
| Deployment → RS → Pods | The ownership chain |
| **Pods are immutable** | Never change a running pod — build a new image version and apply |
| **Rolling update** | Add one new pod, remove one old — zero downtime |
| `kubectl rollout status/undo` | Watch a rollout / roll back to the previous ReplicaSet |
| ConfigMap as a file | Key = filename, value = file contents; mount it as a volume |
| `volumes` vs `volumeMounts` | Pod level (attach the disk) vs container level (where it appears) |
| `subPath` | Mount a single file without replacing the whole directory |
| Everything is ephemeral | Pod dies → data dies; on EKS **nodes** are ephemeral too → durable data lives outside |
| **EBS vs EFS vs S3** | Block (1 node, fast, DBs) / File-NFS (many nodes, shared files) / Object (HTTP) |
| EBS ritual | Block storage → **format → filesystem → mount**; EFS is NFS, only **mount** |
| Ephemeral volumes | `emptyDir` (pod scratch, shared between containers) & `hostPath` (node's dir) |
| `emptyDir` | Empty dir born/dies with the pod; sidecar-logging scratch space |
| `hostPath` | Mounts a **node** dir; never for app data (pod may reschedule elsewhere); `readOnly` |
| **DaemonSet + hostPath** | One pod per node reading its `/var/log` → the log/metrics-collector pattern |
| **PV / PVC / SC** | Disk-as-K8s-object / the developer's claim / dynamic-provisioning rule |
| Pod → PVC → PV → storage | son → mother → father → wallet; app never touches the disk directly |
| Static vs dynamic | You create the disk + PV / a StorageClass creates it when a PVC asks |
| EBS static gotchas | CSI driver + node IAM (`EBSCSIDriverPolicy`) + same-AZ disk + `nodeSelector` |
| `storageClassName: ""` | Bind to my named PV; do **not** fall back to the default StorageClass |
| **Access modes** | RWO (1 node) / ROX (many RO) / RWX (many RW → EFS) / RWOncePod (1 pod) |
| **Reclaim policy** | Delete (disk destroyed) / **Retain** (data survives) / Recycle (wiped, deprecated) |
| Scheduler | Picks the node; steer it with `nodeSelector`/affinity — needed for AZ-locked EBS |
| **startupProbe** | Has it booted? Runs **once**; buys slow starters time |
| **readinessProbe** | Ready for traffic? Fails → **removed from Service endpoints** (no restart) |
| **livenessProbe** | Still alive? Fails → **pod restarted** |
| Standard resource set | Namespace + Deployment + ConfigMap + Secret + Service |

---
