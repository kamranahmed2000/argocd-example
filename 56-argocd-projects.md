# Argo CD Projects

## Projects

**Manage and segregate multiple Argo CD applications at scale.**

## What Is a Project?

A **Project** is a logical grouping object that **exists only inside Argo CD**. A Project is not a cluster-wide resource like Kubernetes Namespaces.

## Projects Enable and Support

### Organization and Multi-Tenancy

Provide a simple and effective way to group related applications, such as applications belonging to a team or business unit.

### Access Control (RBAC)

Projects are the central point for configuring **Role-Based Access Control (RBAC)** in Argo CD, allowing granular permissions to be set for users.

### Security Guardrails

Define guardrails for which source repositories, destinations, and Kubernetes resources the applications belonging to the Project can access.

## Example Project YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-fin
  namespace: argocd
spec:
  description: "Project for Finance team"
  sourceRepos:
    - "<REPO_URL_1>"
    - "!<REPO_URL_2>"

  destinations:
    - namespace: finance
      server: 'https://kubernetes.default.svc'

  clusterResourceWhitelist:
    - group: ''
      kind: 'Namespace'
```
