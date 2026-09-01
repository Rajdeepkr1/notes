# Docker & Kubernetes — Deep Dive Roadmap

We'll go from fundamentals → internals → orchestration → networking → storage → security → production → interview problems.

---

## 1. Docker Fundamentals

**Definition:** Docker is a platform for building, shipping, and running applications inside **containers** — lightweight, isolated processes that package an application together with its dependencies and runtime, guaranteeing it runs the same way regardless of the underlying host.

**Containers vs virtual machines — Definition:** a **VM** virtualizes an entire machine, including its own full OS kernel, managed by a hypervisor — heavier, slower to start (minutes), stronger isolation. A **container** virtualizes only at the OS level, sharing the host's kernel while isolating processes, filesystem, and network via kernel features (namespaces/cgroups, section 7) — much lighter, starts in milliseconds, but all containers on a host share one kernel.

```
VM:        Hardware → Hypervisor → Guest OS (full kernel) → App
Container: Hardware → Host OS (one shared kernel) → Container runtime → App
```

**Images vs containers — Definition:** an **image** is a read-only, immutable template (filesystem + metadata) describing what to run; a **container** is a running (or stopped) *instance* of an image — analogous to a class vs an object instance.

**The Docker daemon, client, registry — Definition:** the **daemon** (`dockerd`) is the background service that actually builds/runs/manages containers on the host; the **client** (`docker` CLI) is what you type commands into, which talks to the daemon over a REST API; a **registry** (Docker Hub, ECR, GHCR) is a server that stores and distributes images.

**Basic commands:**

```bash
docker run -d -p 8080:80 --name web nginx   # run a container from the "nginx" image, detached
docker ps                                    # list running containers
docker ps -a                                 # list all containers, including stopped
docker stop web                              # gracefully stop
docker rm web                                # remove a stopped container
docker images                                # list local images
```

**Container lifecycle — Definition:** `created` → `running` → (`paused` ↔ `running`) → `stopped`/`exited` → `removed`. A stopped container still exists (and its filesystem changes persist) until explicitly `rm`'d.

**Registries** — Docker Hub is the default public registry; private/cloud alternatives (Amazon ECR, GitHub Container Registry, self-hosted) are used for proprietary images, typically gated by authentication.

---

## 2. Images & Dockerfile

**Definition:** an image is built from a stack of read-only **layers**, each corresponding to one instruction in a **Dockerfile** — a text file of instructions describing how to assemble the image step by step.

```dockerfile
FROM node:20-alpine          # base image — starting layer(s)
WORKDIR /app                 # sets the working directory for subsequent instructions
COPY package*.json ./        # copy just the manifest first (see layer caching below)
RUN npm ci --omit=dev        # install dependencies — creates a new layer
COPY . .                     # copy the rest of the source code
ENV NODE_ENV=production
EXPOSE 3000                  # documents the port the app listens on (doesn't publish it)
USER node                    # drop root privileges for the running process
CMD ["node", "server.js"]    # default command when the container starts
```

**Key instructions — Definition:**
- **`FROM`** — sets the base image every subsequent layer builds on top of.
- **`WORKDIR`** — sets (and creates, if needed) the working directory for all following instructions.
- **`COPY` vs `ADD`** — `COPY` copies files/directories from the build context into the image, literally; `ADD` does the same but also supports remote URLs and auto-extracting local tar archives — `COPY` is preferred unless `ADD`'s extra behavior is specifically needed, since its extra "magic" can be surprising.
- **`RUN`** — executes a command *at build time*, its result (filesystem changes) baked into a new layer (e.g. installing packages).
- **`CMD` vs `ENTRYPOINT`** — `CMD` specifies the default command (and/or default arguments) for the container, easily overridden by `docker run <image> <other-command>`; `ENTRYPOINT` specifies the command that always runs, with `CMD` (if present) supplying only its default arguments — combining both lets you build a container that behaves like a fixed CLI tool with overridable default flags.
- **`EXPOSE`** — documents which port(s) the containerized app listens on; purely informational/metadata — it does **not** actually publish the port (that's `-p` at `docker run` time).
- **`ENV` vs `ARG`** — `ENV` sets an environment variable available both at build time and in the running container; `ARG` defines a build-time-only variable (passed via `docker build --build-arg`), not present in the final running container unless also assigned to an `ENV`.
- **`USER`** — switches the user the following instructions (and the final running container) execute as — running as a non-root user is a security best practice (section 17).

**Image layers & caching — Definition:** each Dockerfile instruction produces a new, cached layer; on rebuild, Docker reuses cached layers for any instruction whose input (and all preceding layers) hasn't changed, only re-executing from the first changed instruction onward — which is why `COPY package*.json ./` + `RUN npm ci` is ordered *before* `COPY . .` above: dependency installation is cached and skipped unless `package.json` itself changes, even if application source code changes on every build.

**Multi-stage builds — Definition:** a Dockerfile with multiple `FROM` statements, where later stages can selectively copy artifacts from earlier stages (`COPY --from=builder`) — used to keep the final image small by excluding build-only tools/dependencies (compilers, dev dependencies) that were needed to *produce* the artifact but aren't needed to *run* it.

```dockerfile
# build stage
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

# final, slim runtime stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

**`.dockerignore` — Definition:** analogous to `.gitignore` — excludes files/directories (`node_modules`, `.git`, `.env`) from being sent to the Docker build context, keeping builds faster and preventing accidental inclusion of secrets or bloat in the image.

**Tagging & versioning — Definition:** an image reference is `name:tag` (e.g. `myapp:1.2.0`); `latest` is just a conventional default tag, not automatically "the newest" — production images should be tagged with an explicit, immutable version (a semver or a git commit SHA) rather than relying on `latest`, so deployments are reproducible.

**Base image selection — Definition:** `alpine` variants use the minimal musl-libc-based Alpine Linux (smallest size, ~5MB base, but occasional compatibility quirks with glibc-dependent native modules); `slim` variants (Debian-based) are a middle ground (smaller than full, glibc-compatible); full/default images include the most tooling but are largest — the tradeoff is image size/attack surface vs compatibility/convenience.

**Image size optimization** — use multi-stage builds, a minimal base image, combine `RUN` commands where sensible to reduce layer count, clean up package-manager caches within the same `RUN` layer they were created in (cleaning in a later layer doesn't shrink the image — the earlier layer's data is still stored).

---

## 3. Containers — Deep Dive

**Container isolation (high level) — Definition:** Linux **namespaces** give a container its own isolated view of system resources (process IDs, network interfaces, mount points); **cgroups** (control groups) limit and account for the physical resources (CPU, memory) a container may consume — together, these two kernel features are what make a container feel like an isolated machine while actually being a regular process on the host. (Full depth in section 7.)

**Interactive vs detached:**

```bash
docker run -it ubuntu bash    # interactive — attaches your terminal to the container's stdin/stdout
docker run -d nginx           # detached — runs in the background, returns immediately
```

**Exec into a running container — Definition:** `docker exec` runs an additional command inside an *already-running* container's namespace — commonly used to open a debugging shell without restarting the container.

```bash
docker exec -it web sh
```

**Environment variables & secrets:**

```bash
docker run -e DATABASE_URL=postgres://... myapp
docker run --env-file .env myapp
```

Secrets passed as plain environment variables are visible via `docker inspect` and process listings — for anything sensitive in production, prefer a secrets-management mechanism (Docker Swarm secrets, or in Kubernetes, the `Secret` object, section 10) over plain `-e`.

**Restart policies — Definition:** controls whether Docker automatically restarts a stopped container: `no` (default), `on-failure[:max-retries]`, `always` (restarts even after a manual stop, unless explicitly stopped again), `unless-stopped` (like `always`, but respects an explicit manual stop across daemon restarts).

```bash
docker run --restart unless-stopped myapp
```

**Resource limits — Definition:** caps on how much CPU/memory a container may use, enforced via cgroups — prevents one misbehaving container from starving others on the same host.

```bash
docker run --memory=512m --cpus=1.5 myapp
```

**Health checks — Definition:** a command Docker periodically runs *inside* the container to determine whether the application is actually healthy (not just that the process is alive) — reported in `docker ps`'s STATUS column, and usable by orchestrators to decide whether to route traffic to this container.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:3000/health || exit 1
```

**Logs:**

```bash
docker logs web
docker logs -f web   # follow, like tail -f
```

By default, `docker logs` reads whatever a container writes to **stdout/stderr** — this is why containerized apps should log to stdout rather than to a file inside the container.

**Copying files in/out:**

```bash
docker cp web:/app/logs/error.log ./error.log
docker cp ./config.json web:/app/config.json
```

---

## 4. Docker Networking

**Default bridge network — Definition:** Docker creates a default bridge network (`bridge`) on installation; containers on it get an internal IP and can reach the outside world via NAT through the host, but (on the default bridge specifically) cannot resolve each other by container name — only by IP.

**Custom bridge networks — Definition:** a user-created bridge network, which — unlike the default bridge — provides automatic DNS resolution between containers by container/service name, and better isolation (only containers explicitly attached to it can communicate) — the recommended way to network containers together on a single host.

```bash
docker network create my-net
docker run -d --network my-net --name db postgres
docker run -d --network my-net --name api myapi   # can reach the DB at hostname "db"
```

**Host networking — Definition:** `--network host` removes network isolation entirely — the container shares the host's network stack directly (no port mapping needed/possible), trading isolation for the lowest possible network overhead.

**None networking — Definition:** `--network none` gives the container no network interface at all, fully isolating it — used for purely computational tasks that need zero network access.

**Port mapping (`-p`) — Definition:** publishes a container's internal port to the host, making it reachable from outside the host machine.

```bash
docker run -p 8080:80 nginx   # host:8080 → container:80
```

**DNS resolution between containers** — see custom bridge networks above; Docker runs an embedded DNS server that resolves container/service names to their current internal IP within a user-defined network.

**Overlay networks (brief)** — a network type spanning *multiple Docker hosts* (used in Docker Swarm, and conceptually similar to what Kubernetes' CNI plugins provide, section 14), letting containers on different physical machines communicate as if on the same network.

---

## 5. Docker Storage & Volumes

**Definition:** by default, a container's filesystem is **ephemeral** — any data written inside it is lost when the container is removed, because that data lives in a thin, container-specific writable layer on top of the image's read-only layers.

**Volumes vs bind mounts vs tmpfs — Definition:**
- **Volumes** — storage managed entirely by Docker, living outside any container's filesystem (typically under `/var/lib/docker/volumes/`), the recommended mechanism for persisting data.
- **Bind mounts** — map a specific path on the *host* filesystem directly into the container — useful for local development (live-editing source code mounted into a container) but tightly couples the container to the host's directory structure.
- **`tmpfs`** — mounts data in the host's memory only, never written to disk at all — used for temporary, sensitive data (e.g. secrets that shouldn't ever touch disk).

```bash
docker run -v mydata:/var/lib/postgresql/data postgres   # named volume
docker run -v $(pwd)/src:/app/src myapp                   # bind mount (local dev)
docker run --tmpfs /app/cache myapp                        # tmpfs
```

**Named volumes — Definition:** a volume created with an explicit name (`mydata` above), independently manageable (`docker volume ls/inspect/rm`) and reusable across multiple containers, regardless of any single container's lifecycle.

**Volume drivers — Definition:** plugins that let Docker back a volume with storage other than the local disk (e.g. NFS, a cloud block storage service) — relevant mainly in multi-host/Swarm setups.

**Data persistence patterns** — a database container should always mount its data directory to a named volume; application code in development is often bind-mounted for live reload; anything genuinely temporary can stay in the container's ephemeral layer or `tmpfs`.

**Sharing data between containers** — multiple containers can mount the *same* named volume simultaneously, a common pattern for a sidecar container that processes files another container writes.

---

## 6. Docker Compose

**Definition:** Docker Compose is a tool for defining and running multi-container applications using a single declarative YAML file, so an entire local stack (app + database + cache, etc.) can be started with one command instead of several manual `docker run`s.

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["3000:3000"]
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on: [db]
  db:
    image: postgres:16
    volumes: ["db-data:/var/lib/postgresql/data"]
    environment:
      - POSTGRES_PASSWORD=pass

volumes:
  db-data:
```

**Services, networks, volumes — Definition:** a `service` is one containerized component of the app (built from a Dockerfile or a pre-built image); Compose automatically creates a shared network so services can reach each other by service name (`db` above); `volumes` are declared and mounted the same way as `docker run -v`, just expressed declaratively.

**Environment variables & `.env` files** — Compose automatically loads a `.env` file in the project directory, whose values can be interpolated into the compose file (`${VAR}`) and passed through to services.

**`depends_on` — Definition:** controls **startup order** (Compose starts `db` before `api`) — but by default does **not** wait for the dependency to be actually *ready* (e.g. Postgres accepting connections), only for its container process to have started; combine with a health check + `condition: service_healthy` for a true readiness wait.

```yaml
depends_on:
  db:
    condition: service_healthy
```

**Compose for local development** — the standard way teams give every developer an identical, one-command local environment (app + all its infrastructure dependencies) without installing Postgres/Redis/etc. natively on their machine.

```bash
docker compose up -d      # start all services, detached
docker compose down       # stop and remove containers (add -v to also remove volumes)
docker compose build      # rebuild images
```

---

## 7. Docker Internals

This is where we go beyond normal tutorials.

**Definition:** Docker internals covers the layered stack of components — from the `docker` CLI down to raw Linux kernel features — that actually creates and runs an isolated container process.

Understand:

```
docker run
  ↓
Docker CLI → Docker daemon (dockerd) via REST API
  ↓
containerd (high-level container runtime)
  ↓
runc (low-level OCI runtime)
  ↓
Linux kernel (namespaces, cgroups, union filesystem)
```

**Namespaces — Definition:** a Linux kernel feature that partitions a specific kind of global system resource so a process sees its own isolated instance of it. Docker uses several: **pid** (a container sees only its own process tree, starting at PID 1), **net** (its own network interfaces/routing table), **mnt** (its own filesystem mount points), **uts** (its own hostname), **ipc** (isolated inter-process communication resources), **user** (optional — maps container UIDs to different host UIDs, so "root" inside the container isn't actually root on the host).

**cgroups (control groups) — Definition:** a Linux kernel feature for limiting, accounting for, and isolating the resource usage (CPU, memory, disk I/O) of a group of processes — the mechanism behind `docker run --memory`/`--cpus` (section 3), and what prevents one container from starving the host or other containers.

**Union filesystems (overlay2) — Definition:** a filesystem type that transparently layers multiple directories ("layers") into what appears as a single merged filesystem — `overlay2` is Docker's default storage driver, implementing image layers (read-only) plus a container's writable layer as a stack, with **copy-on-write**: modifying a file that exists in a lower read-only layer copies it up into the writable layer first, leaving the original layer untouched (which is exactly why multiple containers can share the same underlying image layers safely and space-efficiently).

**containerd — Definition:** a high-level container runtime (originally extracted from Docker, now a standalone CNCF project) that manages the container lifecycle — image transfer/storage, container execution, supervision — sitting between the Docker daemon and the low-level runtime. Kubernetes can also use containerd directly as its container runtime, without Docker in between at all.

**runc — Definition:** the low-level, OCI-compliant runtime that actually creates a container by making the raw kernel syscalls to set up namespaces/cgroups and start the container's process — invoked by containerd, doing the final, literal work of "creating a container."

**OCI (Open Container Initiative) — Definition:** an open standard defining the **image format** and **runtime spec** for containers, ensuring images and runtimes from different vendors (Docker, Podman, containerd, CRI-O) are interoperable — the reason an image built by `docker build` can be run by a completely different, OCI-compliant runtime.

---

## 8. Kubernetes Fundamentals

**Definition:** Kubernetes (K8s) is an open-source **container orchestration** platform that automates deploying, scaling, healing, and networking containerized applications across a cluster of machines — solving the problems that arise once you have more containers than you can manage by hand with plain `docker run`.

**Why orchestration is needed** — a single Docker host has no answer for: what happens when a container crashes (who restarts it?), how do you run more copies under load (scaling), how do containers on *different* machines find each other, how do you roll out a new version without downtime — Kubernetes provides declarative, automated answers to all of these.

**Cluster architecture overview — Definition:** a Kubernetes cluster consists of a **control plane** (the "brain" — API server, scheduler, controller manager, etcd; see section 15) and one or more **worker nodes** (machines that actually run application containers, each running a `kubelet` agent).

```
Control Plane (API Server, Scheduler, Controller Manager, etcd)
        ↓ schedules workloads onto
Worker Nodes (kubelet + container runtime + kube-proxy, running Pods)
```

**`kubectl` — Definition:** the command-line tool for interacting with a Kubernetes cluster's API server — used to create, inspect, modify, and delete resources.

```bash
kubectl get pods
kubectl describe pod my-pod
kubectl apply -f deployment.yaml
kubectl logs my-pod
kubectl exec -it my-pod -- sh
```

**Declarative vs imperative management — Definition:** **imperative** commands tell Kubernetes exactly what action to take right now (`kubectl run`, `kubectl scale`); **declarative** management describes the *desired end state* in a YAML manifest (`kubectl apply -f`), and Kubernetes' controllers continuously reconcile actual state toward it — declarative is the standard, recommended approach for anything beyond quick ad-hoc debugging, since manifests are version-controllable and repeatable.

**YAML manifests — Definition:** the declarative configuration files describing Kubernetes objects — every manifest shares a common structure: `apiVersion`, `kind`, `metadata` (name, labels), and `spec` (the desired state).

**Namespaces (Kubernetes) — Definition:** a way to partition a single cluster into multiple virtual sub-clusters — resource names must be unique within a namespace but can repeat across namespaces — commonly used to separate environments (`dev`/`staging`) or teams within one physical cluster.

---

## 9. Core Kubernetes Objects

**Pod — Definition:** the smallest deployable unit in Kubernetes — a group of one or more containers that are always scheduled together on the same node, sharing the same network namespace (localhost between them) and, optionally, storage volumes. Most Pods run a single container; multi-container Pods are used for tightly-coupled "sidecar" patterns (e.g. a log-shipping sidecar next to the main app container).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: myapp:1.2.0
      ports: [{ containerPort: 3000 }]
```

**ReplicaSet — Definition:** ensures a specified number of identical Pod replicas are running at all times, replacing any that die — rarely created directly; almost always managed indirectly through a Deployment.

**Deployment — Definition:** a higher-level object that manages ReplicaSets on your behalf, adding declarative rolling updates, rollback history, and self-healing — the standard way to run a stateless application in Kubernetes.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector: { matchLabels: { app: my-app } }
  template:
    metadata: { labels: { app: my-app } }
    spec:
      containers:
        - name: app
          image: myapp:1.2.0
```

**Rolling updates & rollbacks — Definition:** updating a Deployment's Pod template (e.g. a new image tag) triggers a **rolling update** — new Pods are created and old ones terminated gradually, respecting configured surge/unavailability limits, so the application stays available throughout; `kubectl rollout undo` reverts to a previous, automatically-tracked revision if the new version is broken.

```bash
kubectl set image deployment/my-app app=myapp:1.3.0
kubectl rollout status deployment/my-app
kubectl rollout undo deployment/my-app
```

**Service — Definition:** a stable network endpoint (a fixed virtual IP + DNS name) in front of a dynamic, changing set of Pods (selected by label), so other parts of the app don't need to track individual Pod IPs, which change every time a Pod is recreated.

```yaml
apiVersion: v1
kind: Service
metadata: { name: my-app }
spec:
  selector: { app: my-app }
  ports: [{ port: 80, targetPort: 3000 }]
  type: ClusterIP
```

- **`ClusterIP`** (default) — an internal-only virtual IP, reachable from within the cluster.
- **`NodePort`** — additionally exposes the Service on a static port on every node's IP, reachable from outside the cluster.
- **`LoadBalancer`** — provisions an external cloud load balancer (on a supported cloud provider) that routes to the Service.
- **`ExternalName`** — maps the Service to an external DNS name (a CNAME-like redirect), for referencing resources outside the cluster by an in-cluster name.

**Ingress & Ingress Controllers — Definition:** an **Ingress** is a declarative set of HTTP(S) routing rules (host/path → Service) for exposing multiple Services through a single external entry point; it requires an **Ingress Controller** (e.g. nginx-ingress, Traefik) actually running in the cluster to implement those rules — the Ingress object alone does nothing without one.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: my-app }
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: my-app, port: { number: 80 } } }
```

**Labels & selectors — Definition:** labels are arbitrary key-value pairs attached to objects (`app: my-app`); selectors query objects by label — the mechanism connecting a Deployment to its Pods, and a Service to the Pods it routes traffic to, entirely by label matching rather than direct references.

---

## 10. Configuration & Secrets

**ConfigMap — Definition:** a Kubernetes object storing non-sensitive configuration data as key-value pairs, decoupled from container images — lets the same image run with different configuration in different environments.

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  LOG_LEVEL: "info"
  API_TIMEOUT: "30"
```

**Secret — Definition:** structurally similar to a ConfigMap, but intended for sensitive data (passwords, tokens) — stored base64-encoded (not encrypted by default at rest unless the cluster has envelope encryption configured — see section 17) and handled with tighter API access controls.

```yaml
apiVersion: v1
kind: Secret
metadata: { name: db-secret }
type: Opaque
stringData:
  DB_PASSWORD: supersecret
```

**Consuming ConfigMaps/Secrets as environment variables:**

```yaml
containers:
  - name: app
    envFrom:
      - configMapRef: { name: app-config }
      - secretRef: { name: db-secret }
```

**Mounting config as volumes — Definition:** instead of (or in addition to) environment variables, a ConfigMap/Secret can be mounted as files inside a Pod's filesystem — useful for configuration formats an application expects to read as a file (e.g. a full `.json`/`.yaml` config), and updates to a mounted ConfigMap can propagate to the file without restarting the Pod (env vars cannot).

```yaml
volumes:
  - name: config-volume
    configMap: { name: app-config }
containers:
  - name: app
    volumeMounts:
      - { name: config-volume, mountPath: /etc/config }
```

**Immutable ConfigMaps/Secrets — Definition:** marking `immutable: true` prevents accidental updates to a ConfigMap/Secret after creation, and lets the API server skip watching it for changes on every consuming Pod — a minor performance optimization for large clusters, plus a safety guard against unintended drift.

---

## 11. Storage in Kubernetes

**Volumes (Pod-level) — Definition:** unlike Docker's container-scoped ephemeral storage, a Kubernetes Volume is tied to the **Pod's** lifetime, so it survives a container restart within the same Pod (but not the Pod's deletion) — the base abstraction that ConfigMap/Secret volumes (section 10) and persistent storage (below) both build on.

**PersistentVolume (PV) — Definition:** a cluster-level storage resource (backed by a cloud disk, NFS, etc.) provisioned independently of any specific Pod, with a lifecycle of its own — represents a piece of actual storage infrastructure available to the cluster.

**PersistentVolumeClaim (PVC) — Definition:** a request for storage made by a user/Pod, specifying size and access mode, which Kubernetes binds to a matching PersistentVolume — the layer of indirection that lets application manifests request "20GB of storage" without needing to know the underlying storage implementation details.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: db-pvc }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 10Gi } }
  storageClassName: standard
```

**StorageClasses & dynamic provisioning — Definition:** a StorageClass defines *how* to automatically provision a PersistentVolume on demand (which cloud storage backend/type to use) when a PVC requests it — dynamic provisioning means you rarely need to manually pre-create PVs; the cluster creates one automatically to satisfy each PVC.

**Access modes — Definition:** `ReadWriteOnce` (mountable read-write by a single node), `ReadOnlyMany` (read-only by many nodes), `ReadWriteMany` (read-write by many nodes simultaneously — requires storage backends that support it, like NFS; most cloud block storage only supports RWO).

**StatefulSets & stable storage** — see section 12; each replica in a StatefulSet gets its *own* stable, persistent PVC that follows that specific replica across rescheduling — unlike a Deployment, where Pods are fungible and typically share (or don't need) persistent identity-bound storage.

---

## 12. Workload Types

**Deployment — Definition:** for **stateless** applications where any replica is interchangeable with any other, and Pods can be freely created/destroyed/rescheduled in any order (see section 9).

**StatefulSet — Definition:** for **stateful** applications (databases, message brokers) needing stable, unique network identities (`app-0`, `app-1`, ...) and stable, per-replica persistent storage that follows a specific replica even if it's rescheduled to a different node — also guarantees ordered, sequential deployment/scaling/termination, unlike a Deployment's Pods, which have no ordering guarantees.

**DaemonSet — Definition:** ensures exactly one copy of a Pod runs on every node (or a selected subset of nodes) in the cluster — used for node-level agents: log collectors, monitoring agents, network plugins.

**Job — Definition:** runs a Pod (or several) to completion for a **one-off** task, tracking success/failure and retrying on failure, then stopping — unlike a Deployment, which keeps Pods running indefinitely.

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: db-migration }
spec:
  template:
    spec:
      containers: [{ name: migrate, image: myapp:1.2.0, command: ["npm", "run", "migrate"] }]
      restartPolicy: Never
```

**CronJob — Definition:** creates Jobs on a recurring schedule, expressed in standard cron syntax — the Kubernetes equivalent of `node-cron` running as a reliable, cluster-managed workload rather than inside a single application process.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: nightly-cleanup }
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers: [{ name: cleanup, image: myapp:1.2.0, command: ["npm", "run", "cleanup"] }]
          restartPolicy: OnFailure
```

**When to use each** — Deployment for the vast majority of application services; StatefulSet specifically for databases/queues that need stable identity+storage; DaemonSet for node-level infrastructure agents; Job/CronJob for one-off or scheduled batch work, not long-running request-serving traffic.

---

## 13. Scaling & Scheduling

**Horizontal Pod Autoscaler (HPA) — Definition:** automatically adjusts the **number of Pod replicas** in a Deployment/StatefulSet based on observed metrics (CPU/memory utilization, or custom metrics) against a target, scaling out under load and back in when demand drops.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: my-app }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: my-app }
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

**Vertical Pod Autoscaler (VPA) — Definition:** automatically adjusts a Pod's CPU/memory **requests/limits** based on observed usage, rather than the replica count — complementary to (and generally not combined with, for the same metric, without care) HPA.

**Cluster Autoscaler — Definition:** automatically adds or removes **worker nodes** from the cluster itself, based on whether pending Pods can't be scheduled due to insufficient capacity (scale up) or nodes are significantly underutilized (scale down) — the layer above HPA/VPA, which only manage Pods within existing node capacity.

**Resource requests & limits — Definition:** a **request** is the amount of CPU/memory a container is guaranteed to get, used by the scheduler to decide which node has room for it; a **limit** is the maximum it's allowed to use — exceeding a memory limit gets a container killed (OOMKilled); exceeding a CPU limit just throttles it.

```yaml
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits: { cpu: "500m", memory: "512Mi" }
```

**Scheduling controls — Definition:**
- **Node selectors/affinity** — constrain which nodes a Pod *can* (or preferably should) be scheduled onto (e.g. nodes with a GPU, or in a specific AZ).
- **Pod affinity/anti-affinity** — schedule a Pod near (affinity) or away from (anti-affinity) other specific Pods — e.g. spreading replicas of the same app across different nodes/AZs for availability.
- **Taints & tolerations** — a **taint** on a node repels Pods from being scheduled there unless the Pod has a matching **toleration** — used to reserve nodes for specific workloads (e.g. dedicated GPU nodes).

**Pod Disruption Budgets (PDB) — Definition:** specifies the minimum number/percentage of a workload's Pods that must remain available during *voluntary* disruptions (node drains for maintenance, cluster upgrades) — prevents a maintenance operation from taking down too many replicas of a service at once.

---

## 14. Kubernetes Networking

**The Kubernetes networking model — Definition:** every Pod gets its own cluster-wide IP address, and every Pod can reach every other Pod's IP directly, without NAT, regardless of which node they're on — a flat networking model that's a foundational Kubernetes requirement, implemented under the hood by a CNI plugin.

**Pod-to-pod networking** — since every Pod has a routable IP within the cluster's flat network, containers in different Pods communicate directly by Pod IP (or, in practice, almost always via a stable Service, section 9, rather than a Pod's own ephemeral IP).

**Services & kube-proxy — Definition:** `kube-proxy` runs on every node and implements the virtual IP behavior of Services — it programs the node's networking rules (via iptables or IPVS) so that traffic sent to a Service's ClusterIP gets transparently load-balanced across the Service's backing Pods.

**CNI (Container Network Interface) — Definition:** a standard plugin interface Kubernetes delegates actual Pod networking setup to — plugins like Calico, Cilium, and Flannel implement the flat pod-to-pod network (and, for some, Network Policy enforcement) differently, but all satisfy the same Kubernetes networking model.

**Network Policies — Definition:** a Kubernetes object that acts like a firewall for Pod-to-Pod traffic, specifying which Pods/namespaces are allowed to communicate with which — without any NetworkPolicy defined, all Pods can reach all other Pods by default; policies are how you restrict that.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-from-api-only }
spec:
  podSelector: { matchLabels: { app: db } }
  ingress:
    - from: [{ podSelector: { matchLabels: { app: api } } }]
```

**DNS in Kubernetes (CoreDNS) — Definition:** CoreDNS runs as a cluster add-on providing DNS resolution for Services and Pods by name (`my-service.my-namespace.svc.cluster.local`), so applications can reference each other by stable DNS name instead of IP.

**Ingress vs Service vs Gateway API** — a Service load-balances *within* the cluster (or exposes one app externally, at L4 for `LoadBalancer`); an Ingress provides L7 HTTP routing for *multiple* Services behind one entry point; the newer **Gateway API** is a more expressive, extensible successor to Ingress (separating infrastructure-provider concerns from route definitions), gradually being adopted as Ingress's long-term replacement.

---

## 15. Kubernetes Internals

Going beyond normal tutorials.

**Definition:** Kubernetes internals covers how the control-plane components work together to continuously reconcile a cluster's actual state toward the desired state declared in manifests.

Understand:

```
kubectl apply
  ↓
API Server (validates, persists to etcd)
  ↓
etcd (cluster state store)
  ↓
Controller Manager (reconciliation loops)
  ↓
Scheduler (assigns Pods to Nodes)
  ↓
kubelet (on each node, runs containers via container runtime)
  ↓
kube-proxy (implements Service networking)
```

**Control plane components — Definition:**
- **API Server** — the single entry point for all cluster interaction (kubectl, controllers, kubelets all talk only to it); validates and persists object state, and is the only component that talks directly to etcd.
- **etcd — Definition:** a distributed, consistent key-value store that holds the entire cluster's state — every object (Pods, Deployments, Secrets, everything) is ultimately a record in etcd; losing etcd without a backup means losing the cluster's entire configuration state.
- **Controller Manager** — runs the built-in **controllers**, each responsible for one type of object, continuously watching the API server and reconciling reality toward the declared desired state (e.g. the Deployment controller ensures the right number of ReplicaSet-managed Pods exist).
- **Scheduler** — watches for newly-created Pods with no assigned node, and decides which node to place each on, based on resource requests, constraints (affinity/taints), and current node capacity.

**The reconciliation/controller pattern — Definition:** the core design pattern underlying essentially all of Kubernetes — a controller runs an infinite loop: **observe** current state → **compare** against desired state → **act** to reduce the difference — repeated continuously (not just once at creation), which is why Kubernetes self-heals: if a Pod is manually deleted, the Deployment controller notices the discrepancy on its very next reconciliation pass and creates a replacement.

**etcd & the watch mechanism — Definition:** rather than controllers polling etcd repeatedly, the API server exposes a **watch** API — controllers subscribe to a stream of change events for the objects they care about, reacting immediately when something changes, rather than on a fixed polling interval.

**Custom Resource Definitions (CRDs) & Operators — Definition:** a CRD lets you define your *own* Kubernetes object types beyond the built-in ones (Pod, Deployment, etc.), extending the API; an **Operator** is a custom controller (following the same reconciliation pattern) that manages instances of a CRD — the standard pattern for encoding operational knowledge about a complex application (e.g. a database operator that knows how to safely perform backups/failover) directly into Kubernetes-native automation.

**The scheduler's decision process (brief)** — filters nodes down to those that satisfy hard constraints (enough resources, matching node selectors/taints), then scores the remaining candidates (spreading load, affinity preferences) to pick the best fit.

**kubelet responsibilities — Definition:** the agent running on every node, responsible for making sure the containers described in Pods assigned to its node are actually running (via the container runtime), reporting node/Pod status back to the API server, and executing liveness/readiness probes (section 18).

**Container Runtime Interface (CRI) — Definition:** a standard plugin interface letting kubelet talk to different container runtimes (containerd, CRI-O) interchangeably — since Kubernetes 1.24+, Docker itself is no longer used directly as the runtime (Docker-built *images* still work fine, since they're OCI-compliant — only the Docker daemon specifically was removed from the node's runtime path).

---

## 16. Helm & Package Management

**Definition:** Helm is the de facto package manager for Kubernetes — it packages a set of related Kubernetes manifests (a full application, with all its Deployments/Services/ConfigMaps) into a versioned, reusable, configurable unit called a **chart**.

**Charts, releases, repositories — Definition:** a **chart** is the package itself (templated YAML + metadata); installing a chart into a cluster creates a **release** (a named, tracked instance of that chart's resources, which can be upgraded/rolled back independently); a **repository** is a collection of charts published for others to install (analogous to npm's registry, for Kubernetes app packages).

**Templating with Helm — Definition:** chart manifests use Go templating syntax with placeholders filled in from configurable values, letting one chart definition produce different output for different environments/configurations.

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**Values files — Definition:** a YAML file (`values.yaml`, or a custom override file) supplying the actual values substituted into a chart's templates — the mechanism for one chart to be deployed differently per environment (`values-prod.yaml` vs `values-staging.yaml`) without duplicating the underlying manifests.

```bash
helm install my-release ./my-chart -f values-prod.yaml
helm upgrade my-release ./my-chart --set image.tag=1.3.0
helm rollback my-release 1
```

**Chart dependencies — Definition:** a chart can declare other charts as dependencies (e.g. an application chart depending on a bundled PostgreSQL chart), which Helm fetches and installs together as part of one release.

---

## 17. Security

**Container image security — Definition:** scanning images for known vulnerabilities (via tools like Trivy, or a registry's built-in scanning) before deployment, and preferring minimal base images (section 2) to reduce the attack surface (fewer installed packages = fewer potential CVEs).

**Running as non-root — Definition:** by default, a container process may run as root inside its own user namespace — since root-in-container is a meaningfully weaker security boundary than a real non-root user, best practice is to explicitly run application containers as a non-root user (`USER` in the Dockerfile, section 2, and/or `securityContext` in the Pod spec).

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
```

**Pod Security Standards — Definition:** Kubernetes' built-in policy levels (`Privileged`, `Baseline`, `Restricted`) defining increasingly strict sets of allowed Pod configurations (no privileged containers, no host namespace access, etc.), enforceable per-namespace — the modern replacement for the deprecated PodSecurityPolicy.

**RBAC (Role-Based Access Control) — Definition:** governs what actions a user or service account may perform against the Kubernetes API — a `Role`/`ClusterRole` defines a set of permitted verbs (get/list/create/delete) on specific resource types, and a `RoleBinding`/`ClusterRoleBinding` grants that role to a specific subject (user, group, or ServiceAccount).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { namespace: default, name: pod-reader }
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

**Service Accounts — Definition:** an identity that a **Pod** (rather than a human) uses to authenticate to the Kubernetes API — every Pod runs under some ServiceAccount (a `default` one if none specified), and RBAC bindings to that ServiceAccount control what the application running inside the Pod is itself permitted to do against the cluster API.

**Network Policies (recap)** — see section 14; the primary tool for limiting lateral movement (Pod-to-Pod traffic) within the cluster.

**Secrets management (Kubernetes Secrets vs external)** — native Kubernetes Secrets are base64-encoded, not encrypted, unless the cluster is configured with envelope encryption at etcd level; for stronger guarantees (rotation, audit trail, avoiding secrets living in etcd at all), teams often integrate an external secrets manager (AWS Secrets Manager, HashiCorp Vault) via an operator that syncs external secrets into the cluster.

**Supply chain security (signed images) — Definition:** cryptographically signing container images at build time (e.g. via Sigstore/Cosign) and verifying that signature before allowing deployment (via an admission controller) — ensures only images actually produced by your trusted CI pipeline can run in the cluster, preventing a tampered or unauthorized image from being deployed.

---

## 18. Observability

**Logging — Definition:** containers should log to stdout/stderr (same convention as Docker, section 3); in a cluster, those logs are typically collected by a node-level agent (a DaemonSet, e.g. Fluentd/Fluent Bit) and shipped to a centralized logging backend, since individual container logs are lost once a Pod is deleted.

**Metrics (`kubectl top`, metrics-server) — Definition:** `metrics-server` is a lightweight cluster add-on that collects basic CPU/memory usage per Pod/node, powering `kubectl top` and feeding the Horizontal Pod Autoscaler — it provides only current-point-in-time resource metrics, not historical data or custom application metrics.

**Prometheus & Grafana — Definition:** **Prometheus** is a metrics collection and time-series database that periodically **scrapes** metrics endpoints exposed by applications/infrastructure, with a powerful query language (PromQL) for alerting and analysis; **Grafana** is the visualization layer typically paired with it, building dashboards from Prometheus (and other) data sources — together, the de facto standard observability stack for Kubernetes.

**Liveness, readiness, and startup probes — Definition:**
- **Liveness probe** — periodically checks whether a container is still functioning; if it fails repeatedly, kubelet **restarts** the container.
- **Readiness probe** — checks whether a container is ready to serve traffic; if it fails, the Pod is removed from Service endpoints (stops receiving traffic) *without* being restarted — used for temporary unreadiness (still warming up, temporarily overloaded).
- **Startup probe** — used for slow-starting containers, disabling liveness/readiness checks until the startup probe itself first succeeds, preventing a slow-starting (but healthy) container from being killed prematurely by an impatient liveness probe.

```yaml
livenessProbe:
  httpGet: { path: /health, port: 3000 }
  initialDelaySeconds: 10
  periodSeconds: 10
readinessProbe:
  httpGet: { path: /ready, port: 3000 }
  periodSeconds: 5
```

**Distributed tracing basics** — same concept as AWS X-Ray (AWS notes, section 13): following a single request across multiple services/Pods, typically via OpenTelemetry instrumentation exporting to a backend like Jaeger or Tempo, essential once an app is split into multiple communicating services.

---

## 19. CI/CD for Containers

**Building & pushing images in CI — Definition:** a CI pipeline builds a Docker image from the current commit, tags it (commonly with the git SHA, for full traceability), and pushes it to a registry — the standard first stage of any containerized deployment pipeline.

```yaml
# example CI step
- run: docker build -t myrepo/myapp:${{ github.sha }} .
- run: docker push myrepo/myapp:${{ github.sha }}
```

**Image tagging strategies** — tagging by immutable git SHA (traceable, reproducible) is generally preferred over floating tags like `latest` for anything deployed, since it makes exactly which code is running in any environment unambiguous and lets you `kubectl rollout undo` to a specific, known-good image.

**GitOps (ArgoCD, Flux) — Definition:** a deployment model where a Git repository is the single source of truth for a cluster's *desired* state (the Kubernetes manifests themselves, or Helm values), and a controller running inside the cluster (ArgoCD or Flux) continuously watches that repo and automatically reconciles the live cluster to match it — deployments happen by merging a Git change, not by a CI job running `kubectl apply` directly against the cluster.

**Rolling deployments in Kubernetes** — the default Deployment update strategy (section 9); configurable via `maxSurge`/`maxUnavailable` to control how aggressively old Pods are replaced with new ones.

**Blue/green & canary on Kubernetes — Definition:** neither is natively built into a plain Deployment object — **blue/green** is typically achieved by running two full Deployments simultaneously and shifting a Service's selector (or an Ingress) between them; **canary** by running a small second Deployment alongside the main one and controlling traffic split via an Ingress/service-mesh weight — both concepts are the same as in the AWS CI/CD notes, implemented via Kubernetes-native or service-mesh (Istio/Linkerd) primitives rather than a managed AWS deployment service.

---

## 20. Production Engineering

**Multi-environment cluster strategy — Definition:** two common approaches: **namespaces within one cluster** (cheaper, simpler, but a weaker isolation boundary — a cluster-wide misconfiguration or noisy-neighbor resource contention can cross environments) vs **separate clusters per environment** (stronger isolation, matching the AWS notes' "separate accounts per environment" reasoning, at higher operational/cost overhead) — the right choice scales with how much isolation the organization actually needs between dev/staging/prod.

**Resource quotas & limit ranges — Definition:** a **ResourceQuota** caps the total resources (CPU, memory, object counts) a namespace may consume in aggregate; a **LimitRange** sets default/min/max resource requests-and-limits for individual containers within a namespace — together they prevent one team/namespace from exhausting a shared cluster's capacity.

**Cluster upgrades — Definition:** Kubernetes control-plane and node versions must be upgraded periodically (following the project's version-skew support policy); managed cluster services (EKS, GKE, AKS) automate much of the control-plane upgrade, while node upgrades typically proceed via a rolling replacement of nodes to avoid downtime.

**Backup & disaster recovery (etcd backups) — Definition:** since etcd (section 15) holds the entire cluster's state, regular etcd snapshots are essential for disaster recovery — losing etcd without a backup means having to manually recreate every cluster resource from scratch (or from your own separately-version-controlled manifests, which is precisely why GitOps, section 19, doubles as a disaster-recovery strategy).

**Cost optimization in Kubernetes** — right-sizing resource requests/limits (over-requesting wastes cluster capacity you're paying for), using the Cluster Autoscaler to scale nodes down during low demand, and using Spot/preemptible nodes (with appropriate `tolerations` and workload tolerance for interruption) for fault-tolerant batch workloads.

**Common pitfalls & anti-patterns:**
- No resource requests/limits set — a single misbehaving Pod can starve an entire node.
- Treating stateful workloads (databases) as Deployments instead of StatefulSets.
- Using `latest` image tags in production manifests, making rollbacks/traceability unreliable.
- No readiness probes — traffic gets routed to Pods that aren't actually ready yet.
- Storing sensitive data in plain ConfigMaps instead of Secrets (or an external secrets manager).
- No Pod Disruption Budgets, letting routine node maintenance take down too many replicas at once.
