# Argo CD Application YAML: Separate Repository and Bootstrap Process

## Overview

Kubernetes workload manifests and the Argo CD `Application` manifest serve different purposes:

```text
Application Code Repository
└── Kubernetes manifests
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml

Argo CD Configuration Repository
└── Application manifests
    ├── my-app-dev.yaml
    ├── my-app-qa.yaml
    └── my-app-prod.yaml
```

- **Kubernetes manifests** define what runs in Kubernetes.
- **Argo CD Application manifests** tell Argo CD where the workload manifests are stored, where to deploy them, and how to synchronize them.

---

## Example Repository Structure

```text
GitHub
|
+-- my-app
|   |
|   +-- src/
|   +-- Dockerfile
|   +-- k8s/
|       +-- deployment.yaml
|       +-- service.yaml
|       +-- configmap.yaml
|
+-- argocd-config
    |
    +-- applications/
        +-- my-app-dev.yaml
        +-- my-app-qa.yaml
        +-- my-app-prod.yaml
```

---

## Example Argo CD Application YAML

File location:

```text
argocd-config/applications/my-app-dev.yaml
```

File content:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/mycompany/my-app.git
    targetRevision: main
    path: k8s

  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### What the YAML means

- `metadata.namespace: argocd` creates the `Application` resource in the Argo CD namespace.
- `source.repoURL` identifies the repository containing the Kubernetes manifests.
- `source.targetRevision` identifies the Git branch, tag, or commit.
- `source.path` identifies the directory containing the manifests.
- `destination.server` identifies the target Kubernetes cluster.
- `destination.namespace` identifies the target application namespace.
- `automated.prune` removes resources deleted from Git.
- `automated.selfHeal` corrects differences between Git and the live cluster.
- `CreateNamespace=true` allows Argo CD to create the destination namespace if it does not already exist.

---

## Initial Bootstrap Process

Something must create the first Argo CD `Application` resource in the Kubernetes cluster where Argo CD is installed. A common initial bootstrap method is `kubectl apply`.

### Step 1: Clone the Argo CD configuration repository

```bash
git clone https://github.com/mycompany/argocd-config.git
cd argocd-config
```

### Step 2: Verify the current Kubernetes context

Make sure `kubectl` points to the cluster where Argo CD is installed:

```bash
kubectl config current-context
```

Example output:

```text
management-cluster
```

To see all available contexts:

```bash
kubectl config get-contexts
```

If required, switch to the management cluster:

```bash
kubectl config use-context management-cluster
```

### Step 3: Confirm that the Argo CD namespace exists

```bash
kubectl get namespace argocd
```

### Step 4: Apply the Application manifest

```bash
kubectl apply -f applications/my-app-dev.yaml
```

Expected result:

```text
application.argoproj.io/my-app-dev created
```

### Step 5: Verify the Application resource

```bash
kubectl get applications -n argocd
```

To inspect one Application:

```bash
kubectl get application my-app-dev -n argocd
```

To view detailed status and events:

```bash
kubectl describe application my-app-dev -n argocd
```

---

## What Happens After `kubectl apply`?

```text
Argo CD Configuration Repository
              |
              | contains
              v
      my-app-dev.yaml
      kind: Application
              |
              | kubectl apply
              v
+----------------------------------+
| Argo CD Management Cluster       |
|                                  |
| Namespace: argocd                |
| Application: my-app-dev          |
+----------------+-----------------+
                 |
                 | reads spec.source
                 v
      Application Code Repository
                 |
                 | k8s/
                 v
       deployment.yaml
       service.yaml
       configmap.yaml
                 |
                 | Argo CD sync
                 v
      Target Kubernetes Cluster
                 |
                 v
        Namespace: my-app-dev
                 |
                 v
          Running Workloads
```

After the `Application` object is created:

1. The Argo CD Application Controller detects the new `Application` resource.
2. Argo CD reads the Git repository, revision, and path from `spec.source`.
3. Argo CD generates or reads the Kubernetes manifests.
4. Argo CD compares the desired Git state with the live cluster state.
5. Argo CD applies the resources to the destination cluster and namespace.
6. With automated sync enabled, Argo CD continues reconciling changes.

---

## Local Cluster vs. External Cluster

### Deploying to the cluster where Argo CD runs

Use the internal Kubernetes API address:

```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: my-app-dev
```

### Deploying to an external registered cluster

Register the external cluster with Argo CD:

```bash
argocd cluster add <cluster-context-name>
```

Then use the registered cluster name:

```yaml
destination:
  name: production-cluster
  namespace: my-app-prod
```

Use either `destination.server` or `destination.name`, not both.

---

## Important Prerequisites

Before applying the `Application` YAML, confirm the following:

- Argo CD is installed in the management cluster.
- The Argo CD `Application` CRD exists.
- The `argocd` namespace exists.
- The source Git repository is accessible to Argo CD.
- Private repository credentials are configured in Argo CD.
- The destination cluster is registered if it is external.
- Argo CD has the required RBAC permissions on the destination cluster.
- The source path matches the actual directory in Git.

Verify the `Application` CRD:

```bash
kubectl get crd applications.argoproj.io
```

---

## Common Errors

### Error: Resource mapping not found for kind `Application`

Possible cause: The Argo CD CRDs are not installed, or `kubectl` is connected to the wrong cluster.

Check:

```bash
kubectl get crd applications.argoproj.io
kubectl config current-context
```

### Error: Repository not accessible

Possible cause: The repository is private or the repository URL is incorrect.

Check the configured repositories:

```bash
argocd repo list
```

### Error: Application remains `OutOfSync`

Possible causes include an incorrect Git path, invalid manifests, unavailable destination cluster, or insufficient RBAC permissions.

Check:

```bash
kubectl describe application my-app-dev -n argocd
argocd app get my-app-dev
```

---

## Scaling Beyond One Application

For a few applications, individual `Application` YAML files can be applied manually:

```bash
kubectl apply -f applications/my-app-dev.yaml
kubectl apply -f applications/my-app-qa.yaml
kubectl apply -f applications/my-app-prod.yaml
```

To apply every manifest in the directory:

```bash
kubectl apply -f applications/
```

For many applications or clusters, use one of these patterns:

- **App of Apps:** A parent Argo CD Application manages multiple child Application manifests stored in Git.
- **ApplicationSet:** Generates multiple Argo CD Applications from a template using Git, cluster, list, or other generators.

```text
First Bootstrap
      |
      | kubectl apply
      v
Bootstrap Application or ApplicationSet
      |
      | reads the Argo CD config repository
      v
+-------------+-------------+-------------+
|             |             |             |
v             v             v             v
App DEV       App QA        App PROD      Other Apps
```

This makes `kubectl apply` primarily a first-time bootstrap action. Future Application changes can then be managed through Git.

---

## Quick Interview Answer

> **I keep Argo CD Application manifests in a separate GitOps configuration repository. For the initial bootstrap, I connect `kubectl` to the cluster where Argo CD is installed and run `kubectl apply` on the Application YAML. This creates the Application resource in the `argocd` namespace. Argo CD then reads the source repository and path from the Application specification and deploys the Kubernetes manifests to the configured destination cluster and namespace.**

---

## Easy Memory Flow

```text
Application YAML in Git
          |
          | kubectl apply
          v
Application CR in argocd namespace
          |
          | Argo CD reads spec.source
          v
Kubernetes manifests in application repo
          |
          | Argo CD sync
          v
Target cluster and namespace
          |
          v
Running application
```

## One-Line Summary

**Store the Application YAML in a GitOps configuration repo, bootstrap it into the Argo CD cluster with `kubectl apply`, and let Argo CD deploy the referenced workload manifests to the target cluster.**

---

## Official Reference

- [Argo CD Declarative Setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
