# Argo CD Applications vs. Kubernetes Manifests

## The Application CRD

The **Application CRD (Custom Resource Definition)** is a declarative contract that tells Argo CD which Kubernetes manifests to manage, where the manifests are located, where the application should be deployed, and how synchronization should be performed.

---

## Quick Comparison

| Concept | Argo CD Application CRD | Kubernetes Manifests |
|---|---|---|
| **Kind** | `Application` | `Deployment`, `Service`, `ConfigMap`, etc. |
| **Purpose** | Defines a contract for Argo CD to manage a specific set of Kubernetes manifests. | Defines the components and resources that make up the running application. |
| **Information** | Where is the code? Where should the application be deployed? How should the application be synchronized? | Which container image should run? Which ports should open? How many replicas should be created? |
| **Location in cluster** | Usually created in the `argocd` namespace. | Created in the target application namespace, such as `guestbook-staging`. |
| **Managed by** | Argo CD controllers. | Kubernetes controllers, such as the Deployment controller. |

---

## Diagrammatic Representation

```text
                     ARGO CD APPLICATION CRD
                               |
              Describes how an application is managed
                               |
          +--------------------+--------------------+
          |                    |                    |
          v                    v                    v
       SOURCE              DESTINATION          SYNC POLICY
   Where is the code?    Where should it go?   How is it synced?
          |                    |                    |
          v                    v                    v
   Git repository         K8s cluster          Manual or Auto
   Branch / revision      Namespace            Prune / Self-Heal
   Manifest path
          |
          v
  KUBERNETES MANIFESTS
          |
          +-- Deployment
          +-- Service
          +-- ConfigMap
          +-- Other resources
          |
          v
  KUBERNETES CONTROLLERS
          |
          v
  RUNNING APPLICATION
```

---

## How the Two Work Together

```text
Git Repository
      |
      | Contains
      v
Kubernetes Manifests
(Deployment, Service, ConfigMap, etc.)
      |
      | Referenced by
      v
Argo CD Application CRD
      |
      | Tells Argo CD what, where, and how to deploy
      v
Argo CD Controllers
      |
      | Apply manifests through the Kubernetes API
      v
Target Kubernetes Namespace
      |
      | Managed by Kubernetes controllers
      v
Running Application
```

---

## 1. Argo CD Application CRD

The Argo CD Application CRD answers these management questions:

- **Where is the code?** Git repository, branch or revision, and manifest path.
- **Where should it be deployed?** Target Kubernetes cluster and namespace.
- **How should it be synchronized?** Manual sync, automated sync, pruning, and self-healing.

The Application resource is normally created in the `argocd` namespace and is monitored by the Argo CD Application Controller.

### Example Application CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <YOUR_REPO_URL>
    targetRevision: HEAD
    path: guestbook
  destination:
    server: <YOUR_K8S_SERVER>
    namespace: guestbook-staging
```

---

## 2. Kubernetes Manifests

Kubernetes manifests define the actual resources required to run the application.

Examples include:

- `Deployment` for Pods, container images, and replica counts.
- `Service` for network access and ports.
- `ConfigMap` for non-sensitive configuration.
- Other Kubernetes resources required by the application.

These resources are created in the target application namespace and are handled by the appropriate Kubernetes controllers.

### Example Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  namespace: guestbook-staging
spec:
  replicas: 3
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
        - name: guestbook
          image: <YOUR_CONTAINER_IMAGE>
          ports:
            - containerPort: 8080
```

---

## Key Difference

```text
Application CRD
    = Defines how Argo CD manages and deploys the application

Kubernetes Manifests
    = Define the actual Kubernetes resources that run the application
```

The Application CRD does not replace Kubernetes manifests. The Application CRD points Argo CD to the manifests and specifies where and how Argo CD should deploy them.

---

## Quick Interview Answer

> **An Argo CD Application CRD is a declarative contract that tells Argo CD where the Kubernetes manifests are stored, where to deploy them, and how to synchronize them. Kubernetes manifests define the actual resources, such as Deployments, Services, and ConfigMaps, that run the application.**

## Memory Trick

```text
Application CRD       = WHAT TO MANAGE, WHERE TO DEPLOY, HOW TO SYNC
Kubernetes Manifests  = WHAT TO RUN
```
