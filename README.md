# manifest

> The **GitOps source of truth** for a Kubernetes application deployed continuously with **Argo CD**. This repository holds the Kubernetes manifest (a `Deployment` + `Service`); Argo CD watches it and keeps the cluster in sync with whatever is committed here.

[![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D.svg?logo=argo)](https://argo-cd.readthedocs.io)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.x-326CE5.svg?logo=kubernetes)](https://kubernetes.io)
[![GitOps](https://img.shields.io/badge/Pattern-GitOps-2088FF.svg?logo=github)](https://opengitops.dev)
[![Docker Hub](https://img.shields.io/badge/Image-Docker%20Hub-2496ED.svg?logo=docker)](https://hub.docker.com)

---

## Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [How GitOps with Argo CD Works](#how-gitops-with-argo-cd-works)
- [The Manifest Explained](#the-manifest-explained)
- [Prerequisites](#prerequisites)
- [Connecting This Repo to Argo CD](#connecting-this-repo-to-argo-cd)
- [Workflow Scenarios](#workflow-scenarios)
  - [1. Update the Image (Rolling Update)](#1-update-the-image-rolling-update)
  - [2. Rollback (v2 → v1)](#2-rollback-v2--v1)
  - [3. OutOfSync and Re-Sync](#3-outofsync-and-re-sync)
- [Command Reference](#command-reference)
- [End-to-End Flow](#end-to-end-flow)

---

## Overview

This is a **GitOps repository**. Instead of running `kubectl apply` by hand, the desired state of the cluster lives here as a committed manifest. **Argo CD** is pointed at this repo (its URL + the path to the manifest) and continuously reconciles the cluster against it.

The result:

> **Every commit to this repository becomes the cluster state automatically.** Change the image, commit, push — Argo CD detects the drift and rolls the change out. Revert the commit — Argo CD rolls it back.

The sample workload is a **Tetris** web game (`shaikmustafa/mytetrisgame`) running as a 3-replica `Deployment` exposed through a `LoadBalancer` `Service`.

---

## Repository Structure

```
manifest/
├── deployment-service.yml   # The desired cluster state Argo CD watches
└── README.md
```

A single manifest file holds both objects, separated by the YAML document marker `---`:

| Object | Name | Role |
|--------|------|------|
| **Deployment** | `tetris` | Runs 3 replicas of the Tetris container |
| **Service** | `tetris-service` | Exposes the pods externally via a LoadBalancer |

---

## How GitOps with Argo CD Works

GitOps keeps **Git as the single source of truth**. A controller (Argo CD) constantly compares *what is committed* against *what is actually running* and reconciles any difference.

```
   ┌──────────────────┐        watches repo URL + path        ┌──────────────────┐
   │   GitHub repo     │  ───────────────────────────────────► │     Argo CD       │
   │  (this manifest)  │                                        │   (controller)    │
   └──────────────────┘                                        └─────────┬────────┘
            ▲                                                            │ applies desired state
            │ git commit / push                                         ▼
            │                                                  ┌──────────────────┐
   ┌──────────────────┐                                        │ Kubernetes cluster│
   │   You (developer) │  ◄──── observe pods/sync status ─────  │  Deployment + Svc │
   └──────────────────┘            on Argo CD dashboard         └──────────────────┘
```

The cycle:

1. You commit a change to `deployment-service.yml`.
2. Argo CD detects the new commit on the tracked repo + path.
3. The cluster is now **OutOfSync** (Git differs from cluster).
4. Argo CD applies the desired state (automatically, or on a manual **Sync** click).
5. The dashboard shows the resource tree — `Application → Deployment → ReplicaSet → Pod → Container` — turning healthy and **Synced**.

---

## The Manifest Explained

**`deployment-service.yml`**

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tetris
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tetris
  template:
    metadata:
      labels:
        app: tetris
    spec:
      containers:
      - name: tetris
        image: shaikmustafa/mytetrisgame:v1
        ports:
        - containerPort: 3000   # app listens on 3000

---
apiVersion: v1
kind: Service
metadata:
  name: tetris-service
spec:
  selector:
    app: tetris
  ports:
  - protocol: TCP
    port: 80                    # service exposed on 80
    targetPort: 3000            # forwarded to container's 3000
  type: LoadBalancer
```

**Deployment fields**

| Field | Purpose |
|-------|---------|
| `replicas: 3` | Maintain three identical pods at all times |
| `selector.matchLabels` | How the Deployment finds the pods it owns (`app: tetris`) |
| `template.metadata.labels` | Labels stamped on each pod — must match the selector |
| `image: …:v1` | The image **tag** is the knob you change to update or roll back |
| `containerPort: 3000` | The port the application listens on inside the container |

**Service fields**

| Field | Purpose |
|-------|---------|
| `selector: app: tetris` | Routes traffic to pods carrying this label |
| `port: 80` | The port the Service is reachable on |
| `targetPort: 3000` | The container port traffic is forwarded to |
| `type: LoadBalancer` | Provisions an external IP for browser access |

> **The image tag is the single most important value here.** Updating `v1 → v2` triggers a rollout; reverting `v2 → v1` triggers a rollback. Both are just one-line commits.

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| **Kubernetes cluster** | Minikube, kind, EKS, GKE, or any conformant cluster |
| **`kubectl`** | Configured to talk to the cluster |
| **Argo CD** | Installed in the cluster and reachable via its dashboard/CLI |
| **GitHub access** | This repo's URL, so Argo CD can pull the manifest |

Verify the basics:

```bash
kubectl get nodes
kubectl get pods -n argocd
```

---

## Connecting This Repo to Argo CD

The one-time setup that wires Git to the cluster.

**1. Create / have a cluster** that Argo CD is installed on.

**2. Register this repository as an Argo CD Application** — point it at the repo URL and the path to the manifest:

| Setting | Value |
|---------|-------|
| **Repository URL** | `https://github.com/nikhilsaishankar/manifest.git` |
| **Path** | `.` (the manifest lives at the repo root) |
| **Revision** | `main` |
| **Cluster** | `https://kubernetes.default.svc` (in-cluster) |
| **Namespace** | `default` |
| **Sync policy** | Manual or Automatic |

Equivalent CLI form:

```bash
argocd app create tetris \
  --repo https://github.com/nikhilsaishankar/manifest.git \
  --path . \
  --revision main \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

**3. Sync once** — Argo CD reads the manifest from GitHub and creates the `Deployment` and `Service` on the cluster automatically. From now on, the dashboard shows the live resource tree and any future commits are picked up.

---

## Workflow Scenarios

The three operations demonstrated with this repo. In every case **you only edit the manifest in Git** — Argo CD does the work on the cluster.

### 1. Update the Image (Rolling Update)

Change the image tag in `deployment-service.yml` and commit:

```diff
-        image: shaikmustafa/mytetrisgame:v1
+        image: shaikmustafa/mytetrisgame:v2
```

Argo CD applies the new Deployment spec, which performs a **rolling update**:

```
Deployment "tetris"
   │
   ├── ReplicaSet (v2)  ← NEW: created for the new image
   │      └── Pod ──► Container ──► application (v2)
   │
   └── ReplicaSet (v1)  ← OLD: scaled down once v2 is healthy
```

**How it works, step by step:**

1. The Deployment creates a **new ReplicaSet** for the `v2` image.
2. That ReplicaSet starts a new **Pod**, which runs a **Container**, which runs the **application**.
3. Once the new pod becomes **healthy / Ready**, the **existing (v1) pod is deleted**.
4. The Service keeps routing on the `app: tetris` label, so traffic shifts to the new pod with no manual change.

This is the core update flow: *new replicaset → new pod → healthy → old pod removed.*

### 2. Rollback (v2 → v1)

Revert the image tag back in Git (or revert the commit):

```diff
-        image: shaikmustafa/mytetrisgame:v2
+        image: shaikmustafa/mytetrisgame:v1
```

Because the **v1 ReplicaSet already exists** from the earlier rollout, Kubernetes does not rebuild it — it scales it back up:

```
Deployment "tetris"
   │
   ├── ReplicaSet (v1)  ← scaled back UP, new pod created here
   │      └── Pod ──► Container ──► application (v1)
   │
   └── ReplicaSet (v2)  ← pod deleted once v1 pod is healthy
```

**How it works:**

1. A new **Pod is created under the already-present v1 ReplicaSet**.
2. Once that v1 pod is **healthy**, the **v2 pod is deleted**.
3. The cluster is back on v1 — the reverse of the update flow.

### 3. OutOfSync and Re-Sync

**OutOfSync** means *Git and the cluster no longer match*.

A common way to reach it: you roll back on the cluster (or someone changes a resource directly) so the cluster runs **v1**, while the tracked Git revision still says **v2**. Argo CD compares the two, sees they differ, and marks the Application **OutOfSync** on the dashboard.

```
   Git revision  ──►  image: v2              ┐
                                             ├──►  mismatch  ──►  ⚠ OutOfSync
   Live cluster  ──►  image: v1 (rolled back) ┘
```

**Resolving it:**

- Click **Sync** in the Argo CD dashboard (or `argocd app sync tetris`). Argo CD re-applies the Git desired state so the cluster matches again and the status returns to **Synced**.
- To keep Git and cluster aligned permanently, also commit the rollback to this repo so the desired state itself reflects v1.
- On the dashboard you can click any node (**Deployment**, **ReplicaSet**, **Pod**) to open its **full describe** — events, conditions, and the live vs. desired diff that explains *why* it is OutOfSync.

---

## Command Reference

### Argo CD

| Command | Purpose |
|---------|---------|
| `argocd app list` | List registered applications |
| `argocd app get tetris` | Show sync status, health, and resource tree |
| `argocd app sync tetris` | Re-apply Git desired state (fixes OutOfSync) |
| `argocd app history tetris` | List previous synced revisions |
| `argocd app rollback tetris <id>` | Roll back to a previous synced revision |
| `argocd app diff tetris` | Show live-vs-desired differences |

### kubectl

| Command | Purpose |
|---------|---------|
| `kubectl get deploy tetris` | Deployment status and ready replicas |
| `kubectl get rs` | List ReplicaSets (one per image version) |
| `kubectl get pods -l app=tetris` | List the application pods |
| `kubectl describe pod <pod>` | Full describe — events, image, conditions |
| `kubectl get svc tetris-service` | Service details and external IP |
| `kubectl rollout status deploy/tetris` | Watch a rollout complete |
| `kubectl rollout history deploy/tetris` | Revision history of the Deployment |

---

## End-to-End Flow

```
1. Edit deployment-service.yml  (e.g. bump image tag)
        │
        ▼
2. git commit && git push
        │
        ▼
3. Argo CD detects the new revision  →  Application goes OutOfSync
        │
        ▼
4. Sync (auto or manual)  →  Argo CD applies the desired state
        │
        ▼
5. Rolling update: new ReplicaSet → new Pod → healthy → old Pod removed
        │
        ▼
6. Dashboard shows Synced + Healthy   →   app reachable via the LoadBalancer
```

Everything above is driven from Git. The cluster is never touched by hand — **commit the change, and Argo CD makes the cluster match.**

---

<div align="center">
  <sub>A GitOps demo repository — <strong>continuous delivery to Kubernetes with Argo CD</strong>.</sub>
</div>
