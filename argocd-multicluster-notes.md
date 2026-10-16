# Argo CD Multi-Cluster Architecture

## Overview

Argo CD can be installed centrally in a Kubernetes cluster and can manage registered target Kubernetes clusters.

A target cluster can be registered using:

```bash
argocd cluster add <context-name>
```

---

## Architecture

```text
                         Git Repository
                              |
                              | Desired State
                              v
                +---------------------------+
                |    MANAGEMENT CLUSTER     |
                |                           |
                |         Argo CD           |
                |                           |
                |  +---------------------+  |
                |  | API Server          |  |
                |  | Repository Server   |  |
                |  | App Controller      |  |
                |  | Redis / Dex         |  |
                |  +---------------------+  |
                +------------+--------------+
                             |
                       Kubernetes API
                             |
             +---------------+---------------+
             |               |               |
             v               v               v
      +-------------+ +-------------+ +-------------+
      | DEV Cluster | | QA Cluster  | | PROD Cluster|
      |             | |             | |             |
      | Application | | Application | | Application |
      | Workloads   | | Workloads   | | Workloads   |
      +-------------+ +-------------+ +-------------+
```

## Where is Argo CD installed?

Argo CD can be installed in one Kubernetes cluster that acts as the central management location for registered clusters.

```text
       Kubernetes Management Cluster
                   |
             +-----------+
             |  Argo CD  |
             +-----------+
                   |
         +---------+---------+
         |         |         |
         v         v         v
        DEV        QA       PROD
      Cluster    Cluster    Cluster
```

## Does Every Cluster Need Argo CD?

**No.** A full Argo CD installation is not required on every target cluster.

The target cluster is registered with Argo CD. Argo CD then uses the registered cluster access to manage Kubernetes resources there.

```bash
argocd cluster add <context-name>
```

## How Does Argo CD Know Where to Deploy?

The **Application CRD (Custom Resource Definition)** specifies the source and destination.

- **Project:** Which Argo CD project manages the application.
- **Source:** Where the desired application configuration comes from, such as the Git repository, revision, and path.
- **Destination:** Which Kubernetes cluster and namespace receive the application.
- **Sync Policy:** How Argo CD synchronizes and maintains the application.

### Application YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <YOUR_REPO_URL>
    targetRevision: HEAD
    path: guestbook

  destination:
    server: <YOUR_K8S_SERVER>
    namespace: guestbook-prod
```

## Multi-Cluster Deployment Flow

```text
Developer
    |
    | git push
    v
Git Repository
    |
    | Desired State
    v
Argo CD
    |
    | Destination from Application CRD
    |
    +----------+----------+
    |          |          |
    v          v          v
   DEV         QA        PROD
 Cluster     Cluster     Cluster
```

## Registering a Target Cluster

```bash
argocd cluster add <context-name>
```

Example:

```bash
argocd cluster add dev-cluster
```

List clusters known to Argo CD:

```bash
argocd cluster list
```

## Quick Interview Answer

> **Argo CD can be installed centrally and manage multiple registered Kubernetes clusters. The target clusters do not require a full Argo CD installation. The Application CRD tells Argo CD where the desired application configuration comes from and which cluster and namespace it should be deployed to.**

## Interview Memory Trick

```text
Git
 |
 v
Argo CD
 |
 | Kubernetes API
 |
 +-------> DEV
 |
 +-------> QA
 |
 +-------> PROD
```

### Remember

```text
Argo CD        = GitOps management
Git            = Desired state
Source         = Where FROM?
Destination    = Where TO?
Target Cluster = Runs application workloads
RBAC           = Controls Argo CD permissions
```

## One-Line Summary

**Git stores the desired state -> Argo CD reads it -> Application CRD identifies the destination -> Argo CD manages resources in the registered Kubernetes cluster.**
