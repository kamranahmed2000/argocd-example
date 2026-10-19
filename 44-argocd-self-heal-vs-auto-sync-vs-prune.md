# Argo CD: Self-Heal vs. Auto Sync vs. Prune

## Overview

Argo CD continuously compares the desired state stored in Git with the live state in Kubernetes. **Auto Sync**, **Self-Heal**, and **Prune** control how Argo CD responds when it finds differences.

---

## Quick Comparison

| Feature | Trigger | Argo CD Action | Easy Meaning |
|---|---|---|---|
| **Auto Sync** | A desired-state change is committed to Git | Automatically applies the new Git state to Kubernetes | Git changed, so update Kubernetes |
| **Self-Heal** | A live Kubernetes resource is manually changed and differs from Git | Automatically restores the resource to match Git | Kubernetes changed, so restore from Git |
| **Prune** | A previously managed resource is removed from Git | Automatically deletes the corresponding resource from Kubernetes during sync | Deleted from Git, so delete from Kubernetes |

---

## 1. Auto Sync

Auto Sync automatically synchronizes an application when Argo CD detects that the desired state in Git differs from the live state in Kubernetes.

### Example

Git initially specifies:

```yaml
spec:
  replicas: 2
```

A developer changes Git to:

```yaml
spec:
  replicas: 4
```

With Auto Sync enabled, Argo CD updates Kubernetes automatically:

```text
Git changes from 2 replicas to 4 replicas
                    |
                    v
                 Argo CD
                    |
                    v
Kubernetes changes from 2 replicas to 4 replicas
```

### Configuration

```yaml
spec:
  syncPolicy:
    automated:
      enabled: true
```

A commonly supported shorter form is:

```yaml
spec:
  syncPolicy:
    automated: {}
```

### Easy Meaning

```text
Git changed -> Update Kubernetes
```

---

## 2. Self-Heal

Self-Heal corrects configuration drift when someone or something changes a live Kubernetes resource directly and that resource no longer matches Git.

### Example

Git specifies:

```yaml
spec:
  replicas: 2
```

Someone manually changes the live Deployment:

```bash
kubectl scale deployment my-app --replicas=5
```

The states now differ:

```text
Git desired state        Kubernetes live state
replicas = 2             replicas = 5
       \                     /
        \---- OutOfSync ----/
```

With Self-Heal enabled, Argo CD restores the live value to `2` because Git is the desired source of truth.

```text
Git: replicas = 2
        |
        v
     Argo CD
        |
        | Self-Heal
        v
Kubernetes: replicas = 2
```

### Configuration

```yaml
spec:
  syncPolicy:
    automated:
      enabled: true
      selfHeal: true
```

### Easy Meaning

```text
Kubernetes changed -> Restore from Git
```

> **Important:** Self-Heal corrects configuration drift. It does not mean Argo CD repairs application-code defects or directly fixes a crashed application.

---

## 3. Prune

Prune removes a live Kubernetes resource when the corresponding manifest is removed from Git.

### Example

Initially, Git contains:

```text
k8s/
├── deployment.yaml
├── service.yaml
└── configmap.yaml
```

A developer removes `configmap.yaml` from Git:

```text
k8s/
├── deployment.yaml
└── service.yaml
```

With Prune enabled, Argo CD deletes the previously managed ConfigMap from Kubernetes during synchronization.

```text
Resource removed from Git
           |
           v
        Argo CD
           |
           | Prune
           v
Resource removed from Kubernetes
```

### Configuration

```yaml
spec:
  syncPolicy:
    automated:
      enabled: true
      prune: true
```

### Easy Meaning

```text
Deleted from Git -> Delete from Kubernetes
```

---

## Combined Configuration

A typical automated synchronization policy is:

```yaml
spec:
  syncPolicy:
    automated:
      enabled: true
      prune: true
      selfHeal: true
```

### Combined Flow

```text
                         Git
                          |
                    Desired State
                          |
                          v
                       Argo CD
                 /         |         \
                /          |          \
               v           v           v
          Auto Sync    Self-Heal      Prune
               |           |           |
        Git changed   Live K8s      Resource removed
               |      changed          from Git
               v           v           v
         Apply change   Restore     Delete resource
         to Kubernetes  from Git    from Kubernetes
```

---

## Full Argo CD Application Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/mycompany/my-app.git
    targetRevision: main
    path: k8s

  destination:
    server: https://kubernetes.default.svc
    namespace: my-app

  syncPolicy:
    automated:
      enabled: true
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Practical Scenarios

### Scenario 1: A manifest is changed in Git

```text
Auto Sync handles it.
```

Argo CD automatically applies the updated desired state to Kubernetes.

### Scenario 2: Someone runs `kubectl edit` and changes the live resource

```text
Self-Heal handles it.
```

Argo CD restores the live resource to match Git.

### Scenario 3: A manifest is deleted from Git

```text
Prune handles it.
```

Argo CD deletes the corresponding managed resource from Kubernetes.

### Scenario 4: A Pod crashes but the Deployment specification still matches Git

```text
Kubernetes handles it.
```

The Kubernetes Deployment controller recreates the Pod. This is not an Argo CD Self-Heal action because there may be no configuration drift from Git.

---

## Important Safety Notes

- Enable **Prune** carefully because it permits Argo CD to delete managed resources that are no longer defined in Git.
- Protect important branches and require pull-request reviews because Git becomes the desired source of truth.
- Use Argo CD Projects and Kubernetes RBAC to limit which clusters, namespaces, and resource types an application can manage.
- Test automated synchronization policies in a non-production environment before enabling the same behavior in production.

---

## Interview Answer

> **Auto Sync automatically applies changes from Git to Kubernetes. Self-Heal corrects live-cluster drift by restoring resources to match Git. Prune deletes Kubernetes resources that have been removed from Git.**

---

## Quick Memory Trick

```text
Auto Sync = Git changed     -> Update Kubernetes
Self-Heal = K8s changed     -> Restore from Git
Prune     = Deleted in Git  -> Delete from Kubernetes
```

## One-Line Summary

**Auto Sync applies Git changes, Self-Heal reverses unauthorized live-cluster changes, and Prune deletes resources removed from Git.**

---

## Official Reference

- [Argo CD Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
