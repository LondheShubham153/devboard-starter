# DevBoard on Kubernetes (Kind + Gateway API)

You already know this app from `docker compose`: three containers — Postgres,
the Go backend, the React frontend — talking to each other by name on one
Docker network. This doc runs the *same* app on Kubernetes, on a local
[Kind](https://kind.sigs.k8s.io/) (Kubernetes IN Docker) cluster, and explains
every manifest in `k8s/` in terms of the compose concepts you already know.

**Pinned versions** (set in `k8s/kind-up.sh`, so everyone in the class gets
the exact same cluster):

| Tool | Version |
|---|---|
| Kubernetes (Kind node image) | `v1.36.1` |
| Kind | `v0.32.0` |
| Gateway API | `v1` (standard channel) |
| Envoy Gateway | `v1.9.1` |

## 0. Which Kubernetes version, and why it matters

Kubernetes ships a new minor version roughly every 4 months, each with a
name/theme. What's currently out:

| Version | Codename | Released | Why it matters here |
|---|---|---|---|
| **1.35** | Timbernetes (The World Tree Release) | Dec 2025 | Introduced native **Gang Scheduling** (a `Workload` API for all-or-nothing Pod groups) and graduated **in-place Pod resource updates** (change CPU/memory without restarting the Pod) and **Structured Authentication Configuration** to GA. |
| **1.36** ← *we pin this* | ハル / Haru | Apr 2026 | 18 enhancements graduated to stable, including **User Namespaces for Pods** (better container isolation) and **Mutating Admission Policies**, plus 4 Dynamic Resource Allocation (DRA) features reaching GA. This is the version this repo's Kind cluster runs. |
| **1.37** | Garhwal | Aug 2026 | Newest at time of writing. Adds **KYAML** output for `kubectl` and stable **pod-level resources**, graduates **device-level taints/tolerations** for DRA to GA, and moves rootless kubelet (user-namespace kubelet) and cgroups v2 Memory QoS toward beta. |

Why pin a specific version instead of "whatever Kind defaults to":
- **Reproducibility** — everyone in the class hits the same API surface, the
  same defaults, the same bugs (or lack of them). "Works on my cluster" stops
  being version roulette.
- **Kind's default tracks upstream's latest** and moves whenever you upgrade
  the `kind` CLI — pinning the node image (`kindest/node:v1.36.1`) decouples
  your cluster version from your `kind` binary version.
- **Feature availability** — e.g. Gang Scheduling (1.35+) or Mutating
  Admission Policies (GA in 1.36) simply don't exist, or are alpha/beta with
  a feature-gate flag, on older versions. If a manifest uses a field from a
  newer release, running it against an older cluster just fails silently or
  is rejected by the API server.
- **Skew rules** — in real clusters `kubectl` and the control plane can be up
  to one minor version apart, but staying aligned removes a whole class of
  "why doesn't this flag exist" confusion while learning.

## 1. Why Kubernetes, and how it maps to compose

Compose is one host running containers from a fixed list you write by hand.
Kubernetes is a control plane that keeps a *desired state* running across one
or more nodes, healing and rescheduling things itself. The concepts you used
in `docker-compose.yml` all have a Kubernetes equivalent:

| Compose | Kubernetes | Why it's different |
|---|---|---|
| a `service:` block | a `Deployment` (or `StatefulSet` for Postgres) | K8s manages a *set* of identical Pods, restarts crashed ones, and can run several replicas |
| `depends_on` + `healthcheck` | readiness/liveness `probes` | K8s doesn't just wait for a container to start — it continuously checks it's still healthy, and only sends traffic once a probe passes |
| `.env` values | `ConfigMap` / `Secret` | non-secret config vs. secret config get separate objects, injected as env vars the same way |
| a named `volume:` | `PersistentVolumeClaim` | same idea (durable storage independent of the container), but requested per-Pod and provisioned by the cluster |
| `ports:` on a service | a `Service` | gives Pods a stable name and IP other Pods can call — this is *why* `http://backend:8080` still works unchanged in `vite.preview.config.js` |
| exposing to your laptop | a `Gateway` + `HTTPRoute` | compose just published a host port; a real cluster needs a routing layer in front of many possible backends — that's what Gateway API is |

## 2. Kind basics

Kind runs an entire Kubernetes cluster (control plane + worker nodes) as
Docker containers on your machine — fast to create and delete, good for
learning and CI, not for production.

`kind/kind-config.yaml` defines the cluster shape: **1 control-plane + 2
workers**, so Pods actually get scheduled across nodes instead of all
landing in one place — closer to a real cluster than Kind's single-node
default, and the smallest topology where "which node is this Pod on"
becomes a real question.

Two Kind-specific things you'll hit in this project:

- **No real cloud load balancer.** A `Service`/`Gateway` of type
  `LoadBalancer` never gets an external IP on Kind — there's no cloud
  provider to hand one out. This project uses `NodePort` instead (see
  `07-gateway.yaml`), paired with `extraPortMappings` in
  `kind-config.yaml` that forwards host port `8080` → the control-plane
  container's port `30080`. That's the same NodePort Envoy's Service listens
  on, so `http://localhost:8080` reaches it directly, no `port-forward`
  needed. (`8080` instead of the usual `80` because a privileged port needs
  root and is often already taken on a dev laptop — pick whichever's free on
  yours.)
- **Cross-node NodePort routing can be flaky on Kind under Docker Desktop.**
  A `NodePort` is *supposed* to work from any node regardless of which one
  the Pod actually landed on — kube-proxy forwards it. In practice, hairpin
  routing through Kind's control-plane container to a Pod on a *different*
  node can silently hang on Docker Desktop's networking. The fix in
  `07-gateway.yaml`'s `EnvoyProxy` resource pins the Envoy Pod to the
  control-plane node (`nodeSelector` + a toleration for its taint) — the
  same trick the official Kind + ingress-nginx guide uses for exactly this
  reason, so the mapped port and the Pod are always on the same node.
- **No image registry by default.** `kubectl` can only run images the
  cluster can pull. Since we build our images locally, we hand them
  directly to the cluster with `kind load docker-image` instead of pushing
  to Docker Hub and pulling them back down.

## 3. Walking through `k8s/`, in apply order

### `00-namespace.yaml`
Everything for this app lives in a `devboard` Namespace — a label boundary so
`kubectl get pods` and RBAC rules can scope to just this app instead of the
whole cluster.

### `01-config.yaml`
A `ConfigMap` (`POSTGRES_USER` — not sensitive) and a `Secret`
(`POSTGRES_PASSWORD`, `POSTGRES_DB` — same idea, base64-obscured at rest,
plus stricter access rules). Both get referenced by later manifests with
`valueFrom.configMapKeyRef` / `secretKeyRef` — this is the K8s version of
loading `.env` into a container.

### `02-postgres-init.yaml`
Your existing `init/postgres/01_schema.sql` and `02_seed.sql`, unchanged,
just wrapped in a `ConfigMap`. Postgres's own container image runs anything
mounted at `/docker-entrypoint-initdb.d` on first boot — Kubernetes doesn't
know or care that's a Postgres convention, it's just mounting files.

### `03-postgres.yaml`
Postgres is a `StatefulSet`, not a `Deployment`. The difference matters here:
a StatefulSet gives each replica a **stable identity and its own PVC** (here,
`volumeClaimTemplates` requests 1Gi per replica). A Deployment's Pods are
interchangeable and share nothing — fine for stateless backend/frontend, wrong
for a database where losing "which disk was mine" means losing data.

The matching `Service` has `clusterIP: None` — a **headless** Service. It
doesn't load-balance; it just gives DNS to the StatefulSet's Pods directly.
For a single-replica Postgres this mostly matters for the pattern, not the
behavior — but it's how you'd address `postgres-0`, `postgres-1`, etc.
individually if you scaled it.

### `04-backend.yaml`
A `Deployment` (stateless — any replica can handle any request) +
`Service`. Notice `POSTGRES_URL` is built from the *other* env vars using
`$(VAR)` interpolation, right in the manifest — Kubernetes expands
already-defined env vars into later ones in the same list, so the connection
string is assembled without hardcoding the password twice.

The `livenessProbe` hits `GET /health` — the same endpoint `make smoke`
already checks. If it starts failing, Kubernetes kills and restarts the
container automatically; compose can't do that on its own.

### `05-frontend.yaml`
Same Deployment/Service pattern, plus two things Postgres/backend don't
have:
- **`resources.requests/limits`** — a hint (requests) and a hard cap
  (limits) for CPU/memory, so the scheduler knows how to pack Pods onto
  nodes and one runaway container can't starve its neighbors.
- A `HorizontalPodAutoscaler` targeting 10% average CPU (deliberately low,
  so it's easy to trigger and *watch* scale up in this repo). It scales the
  frontend Deployment between 1 and 5 replicas based on load — something
  `docker compose` has no concept of at all.

### `06-rbac.yaml`
Three RBAC objects that don't run anything — they control *who can do what*
inside the `devboard` namespace:
- `ServiceAccount devboard-intern` — an identity a Pod or user can act as.
- Two `Role`s scoped to `devboard`: `devboard-admin-role` can create/update/
  delete Deployments, Services and Pods; `devboard-intern-role` can only
  `get/list/watch/delete` them (no create/update).
- A `RoleBinding` grants the intern Role to the intern ServiceAccount.

This is the smallest possible example of Kubernetes' access-control model:
permissions are never global by default, they're granted per-namespace,
per-verb, per-resource-type.

### `07-gateway.yaml` — Gateway API
Ingress (the older way to expose HTTP routes) is being phased out in favor
of the **Gateway API**. Four objects here:
- `GatewayClass` — infrastructure-level: "here's an implementation" (we use
  [Envoy Gateway](https://gateway.envoyproxy.io/)).
- `EnvoyProxy` — Envoy Gateway–specific tuning for *how* it deploys the data
  plane: exposes it as a `NodePort` pinned to `30080` (matching
  `kind-config.yaml`'s port mapping) and schedules its Pod onto the
  control-plane node (see the Kind-specifics note above).
- `Gateway` — "open a listener" (here, HTTP on port 80, using the
  `EnvoyProxy` config above via `infrastructure.parametersRef`). This is the
  object a platform team usually owns.
- `HTTPRoute` — "route matching traffic to this Service" (here, everything
  → `frontend:4173`). This is the object an app team usually owns — the
  split is the whole point of the API: infra and app concerns don't have to
  live in the same YAML anymore, unlike a single Ingress resource.

We only route to `frontend` — the frontend container already proxies
`/api/*` to the `backend` Service internally (see
`frontend/vite.preview.config.js`), exactly like it does in compose.

## 4. Run it

```bash
./k8s/kind-up.sh
```

This creates the 3-node Kind cluster (from `kind/kind-config.yaml`), builds
and loads both images, installs Envoy Gateway, and applies everything in
`k8s/`. Once it's done, open:

```
http://localhost:8080
```

Useful checks while it's running:

```bash
kubectl get nodes                            # 1 control-plane + 2 workers
kubectl get pods -n devboard                 # everything should be Running
kubectl get gateway,httproute -n devboard    # Gateway/route status
kubectl get hpa -n devboard                  # autoscaler's current view
kubectl get pods -n envoy-gateway-system -o wide   # Envoy Pod on the control-plane node
```

Tear it down:

```bash
./k8s/kind-down.sh
```

## 5. What's next

This is a local Kind cluster on your laptop. A follow-up doc will cover
running the same manifests on a Kind cluster hosted on an EC2 instance —
same `k8s/` folder, different networking (a real public IP and security
group instead of `port-forward`).
