# Pac-Man on OpenShift

A two-tier Pac-Man web application deployed on **OpenShift / Kubernetes**, consisting of a Node.js front end and a MongoDB back end with persistent storage. This repo provides all the Kubernetes/OpenShift manifests needed to get it running.

---

## Architecture

```
 Browser
    │
    ▼
┌──────────────────────┐
│  Pacman (Node.js)    │  Deployment + Service + Route
│  quay.io/jpacker/    │  Port: 8080 → exposed on :80
│  nodejs-pacman-app   │
└──────────┬───────────┘
           │ ClusterIP :27017
           ▼
┌──────────────────────┐
│  MongoDB             │  Deployment + Service + PVC
│  bitnami/mongodb     │  Port: 27017
│  PVC: 8Gi (RWO)      │
└──────────────────────┘
```

All resources are deployed into the `pacman-app` namespace/project.

---

## Repository Structure

```
.
├── deploy/                        # Individual manifests (apply separately)
│   ├── pacman-deployment.yaml     # Pacman app Deployment
│   ├── pacman-service.yaml        # Pacman ClusterIP Service (80 → 8080)
│   ├── pacman-route.yaml          # OpenShift Route (HTTP ingress)
│   ├── mongo-deployment.yaml      # MongoDB Deployment (bitnami/mongodb:5.0.10)
│   ├── mongo-service.yaml         # MongoDB ClusterIP Service (:27017)
│   └── mongo-pvc.yaml             # PersistentVolumeClaim – 8Gi ReadWriteOnce
│
├── std/                           # Single-file all-in-one manifests
│   ├── deploy                     # Full stack: creates pacman-app project + all resources
│   └── deploy_within_namespace    # Full stack: no namespace — deploy into current project
│
├── docs/
│   └── README.md                  # Legacy documentation (kept for reference)
└── LICENSE
```

---

## Prerequisites

- OpenShift cluster (or [Developer Sandbox](https://developers.redhat.com/developer-sandbox))
- `oc` CLI installed and logged in
- A default StorageClass available in your cluster (for the MongoDB PVC)

---

## Deployment

### Option 1 — Single-file (recommended)

**Create the `pacman-app` project and deploy everything in one shot:**
```bash
oc apply -f std/deploy
```

**Deploy into an existing / pre-created namespace:**
```bash
oc project <your-namespace>
oc apply -f std/deploy_within_namespace
```

---

### Option 2 — Individual manifests

Apply each manifest in the following order:

```bash
# 1. Create the project
oc new-project pacman-app

# 2. MongoDB storage
oc apply -f deploy/mongo-pvc.yaml

# 3. MongoDB deployment and service
oc apply -f deploy/mongo-deployment.yaml
oc apply -f deploy/mongo-service.yaml

# 4. Pacman deployment and service
oc apply -f deploy/pacman-deployment.yaml
oc apply -f deploy/pacman-service.yaml

# 5. OpenShift Route (exposes the app externally)
oc apply -f deploy/pacman-route.yaml
```

---

## Verifying the Deployment

```bash
# Check all pods are Running
oc get pods -n pacman-app

# Get the public URL
oc get route pacman -n pacman-app
```

Navigate to the route URL in your browser to start playing.

---

## Key Details

| Component | Image | Port |
|-----------|-------|------|
| Pacman (frontend) | `quay.io/jpacker/nodejs-pacman-app:latest` | 8080 |
| MongoDB | `bitnami/mongodb:5.0.10` (deploy/) / `bitnami/mongodb:latest` (std/) | 27017 |

| Resource | Name | Namespace |
|----------|------|-----------|
| PVC | `mongo-storage` | `pacman-app` |
| MongoDB Service | `mongo` | `pacman-app` |
| Pacman Service | `pacman` | `pacman-app` |
| Route | `pacman` | `pacman-app` |

---

## Deploying under ACM Regional-DR (with ODF + Argo CD)

This app is used as a stateful test workload for ACM/ODF Regional-DR (Submariner + Ceph RBD mirroring + Ramen). A few of the manifests in this repo carry settings that are not obvious from the YAML alone — they encode lessons learned from running real failover and relocate cycles. If you copy this app's pattern for other DR-protected workloads, replicate these settings.

### MongoDB image and mount path

- **Image**: `bitnamilegacy/mongodb:5.0.10`. As of 2025-08-28, Bitnami removed nearly all tagged images from `docker.io/bitnami/*`. The free read-only archive of the old tags now lives at `docker.io/bitnamilegacy/*`. Pulling `bitnami/mongodb:5.0.10` returns `manifest unknown`.
- **PVC mount path**: `/bitnami/mongodb` (not `/data/db`). Bitnami's MongoDB image overrides the upstream `dbPath` to `/bitnami/mongodb/data/db`. If the PVC is mounted at `/data/db`, mongod ignores the mount and writes to the container's overlay filesystem — data is lost on every pod restart and never replicated by ODF, even though everything appears to "work".

### Namespace SCC annotation pinning

`mongo-deployment.yaml` defines a `Namespace` resource with three pinned annotations:

```yaml
metadata:
  annotations:
    openshift.io/sa.scc.uid-range: "1000840000/10000"
    openshift.io/sa.scc.supplemental-groups: "1000840000/10000"
    openshift.io/sa.scc.mcs: "s0:c29,c14"
```

Why: OpenShift's namespace lifecycle controller auto-assigns these per cluster, **independently**. So the same namespace name on two clusters gets different UID ranges and SELinux MCS levels. After ODF Regional-DR replicates a Ceph RBD volume from the source cluster to the target, the on-disk file ownership (frozen from the source) does not match the auto-assigned identity of the pod on the target, and pods like mongod fail with `Operation not permitted` on `O_NOATIME`-style opens.

Pinning the namespace annotations to identical values across both managed clusters makes the default `restricted-v2` SCC assign the **same** UID, fsGroup, and MCS level on both sides, so file ownership matches the running pod's identity on every failover/relocate. The Deployment stays vanilla — no `runAsUser`, no custom SCC, no chown init container.

> Adapt the values in the YAML to your own clusters. Pick **one** cluster's allocations as canonical and copy them into the manifest:
> ```bash
> oc get namespace pacman-app -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}{"\n"}{.metadata.annotations.openshift\.io/sa\.scc\.supplemental-groups}{"\n"}{.metadata.annotations.openshift\.io/sa\.scc\.mcs}{"\n"}'
> ```

### Argo CD Application sync policy (critical for Regional-DR)

If you deploy this app via an Argo CD `ApplicationSet` driven by an ACM `Placement`, the **default** Argo sync policy will destroy your data during `Relocate`. Apply the settings below before adding the workload to a `DRPlacementControl`.

What goes wrong with defaults: during a Relocate, ACM's Placement controller transiently empties the `PlacementDecision` while it coordinates the handoff. The ApplicationSet generator sees zero matched clusters and **deletes its child Argo `Application`** (which carries `resources-finalizer.argocd.argoproj.io`). The cascade then deletes the `Namespace`, which deletes the PVC, which — because `ocs-storagecluster-ceph-rbd.reclaimPolicy: Delete` — destroys the underlying Ceph RBD image. Replication has nothing left to mirror, and the relocate deadlocks at `RunningFinalSync` indefinitely. Failover (different code path) does not drain the Placement decision and is not affected — but Relocate is.

#### Required Application syncPolicy

| Argo CD UI option | YAML field | Setting | Why |
|---|---|---|---|
| Delete resources that are no longer defined in the source repository | `automated.prune` | **`false`** | Stops Argo from pruning when the Placement decision transiently empties during Relocate. |
| Allow applications to have empty resources | `automated.allowEmpty` | **`true`** | Lets the Application stay valid through the empty-decision window instead of erroring out. |
| Automatically sync when cluster state changes | `automated.selfHeal` | **`true`** | selfHeal only re-applies missing resources from Git; it does not prune. Keep on. |
| Automatically create namespace if it does not exist | `syncOptions: CreateNamespace=true` | **on** | Lets Argo create the namespace on the relocate target. The annotated `Namespace` resource in this repo will reconcile annotations on top. |
| Prune propagation policy | `syncOptions: PrunePropagationPolicy=orphan` | **`orphan`** | If anything is ever deleted (including the Application object itself), `orphan` leaves managed resources behind instead of cascade-deleting them. |

Equivalent YAML on the Application:

```yaml
spec:
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
      allowEmpty: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=orphan
```

#### Required ApplicationSet setting

The Application sync policy above protects against pruning during sync. The **ApplicationSet** can still delete the child Application when the Placement empties; to prevent that cascade, set on the `ApplicationSet`:

```yaml
spec:
  syncPolicy:
    preserveResourcesOnDeletion: true
```

This is documented in ODF's GitOps DR reference architecture and is mandatory for any DR-protected ApplicationSet.

### Storage class and default

The PVC requests `ocs-storagecluster-ceph-rbd`, which is the only StorageClass mirrored by ODF Regional-DR. On both managed clusters, ensure `ocs-storagecluster-ceph-rbd` is the **single** default StorageClass — having `gp3-csi` also marked default leads to non-deterministic provisioning and silent loss of DR coverage for any PVC created without an explicit `storageClassName`.

### Validating an end-to-end DR cycle

Once the above is in place, the data-plane outcome you should expect:

1. Deploy via the ApplicationSet on cluster A. PVC `mongo-storage` binds to a fresh `pvc-<uuid>` Ceph RBD image. mongo writes data files into `/bitnami/mongodb/data/db` (which `df` should show as `/dev/rbdN ext4`, not `overlay`).
2. Play a game / insert documents into `db.highscore`.
3. Issue a `Failover` from cluster A to cluster B via the DRPC. End-to-end completion should be ~1 minute.
4. On cluster B, the *same* PVC UID rebinds to the same Ceph RBD image (mirrored). mongo starts with the `db.highscore` documents present.
5. Issue a `Relocate` back to cluster A. With the Argo settings above, the source workload is preserved through the Placement-decision window, replication completes, and the data ends up back on cluster A intact.

---

## Cleanup

```bash
oc delete project pacman-app
```

---

## License

This project is licensed under the [Apache 2.0 License](./LICENSE).
