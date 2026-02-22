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

## Cleanup

```bash
oc delete project pacman-app
```

---

## License

This project is licensed under the [Apache 2.0 License](./LICENSE).
