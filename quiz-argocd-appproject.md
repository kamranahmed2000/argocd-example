# Argo CD AppProject - Question & Answer Quiz

This quiz covers the key `AppProject` concepts shown in the provided training screenshots. It is designed for quick revision and interview preparation.

---

## Question 1: What is the primary purpose of AppProject in Argo CD?

**Question:** What is the primary operational purpose of the `AppProject` Custom Resource Definition (CRD) in Argo CD?

**Answer:** `AppProject` acts as a logical grouping for Argo CD Applications and allows administrators to enforce restrictions on:

- Source Git repositories
- Destination Kubernetes clusters and namespaces
- Kubernetes resource types that applications are allowed to deploy

### Quick Memory

```text
AppProject
   |
   +-- Which Git repos can be used?
   +-- Which clusters/namespaces can be targeted?
   +-- Which Kubernetes resources can be deployed?
```

**Interview answer:**

> **AppProject logically groups Argo CD Applications and provides security boundaries around source repositories, destination clusters/namespaces, and allowed Kubernetes resources.**

---

## Question 2: What happens when cluster-scoped resources are not allowed?

**Question:** An Application tries to deploy a `ClusterRoleBinding`, which is a cluster-scoped resource, but the assigned Project has an empty `clusterResourceWhitelist`. What happens?

**Answer:** The deployment fails because the Project does not explicitly allow the creation of that cluster-scoped resource.

The Project must explicitly permit the required cluster-scoped resource through its cluster resource allow-list configuration.

### Example Concept

```text
Application
     |
     | tries to deploy
     v
ClusterRoleBinding
     |
     v
AppProject checks cluster resource permissions
     |
     +-- Allowed --> Deploy
     |
     +-- Not Allowed --> Deployment blocked
```

### Example AppProject configuration

```yaml
spec:
  clusterResourceWhitelist:
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding
```

**Interview answer:**

> **If the AppProject does not allow a cluster-scoped resource such as ClusterRoleBinding, Argo CD prevents the Application from deploying that resource.**

---

## Question 3: What happens if the Application does not specify a project?

**Question:** A developer creates an Argo CD `Application` manifest but does not specify the `project` field. Which project is used?

**Answer:** The Application is associated with the `default` project.

### Example

An Application might explicitly contain:

```yaml
spec:
  project: default
```

### Quick Memory

```text
No project specified
        |
        v
  default project
```

**Interview answer:**

> **If no project is specified, the Application uses Argo CD's default project.**

---

## Question 4: Why is sourceRepos an important security control?

**Question:** Why is the `sourceRepos` field in an `AppProject` considered an important security control?

**Answer:** `sourceRepos` controls which Git repositories Applications in the Project are allowed to use as sources.

By restricting the allowed repositories, administrators can prevent Applications from referencing unauthorized or untrusted repositories.

### Example

```yaml
spec:
  sourceRepos:
    - https://github.com/mycompany/approved-apps.git
```

Conceptually:

```text
Application
     |
     | requests Git repository
     v
AppProject sourceRepos
     |
     +-- Approved repository --> Allowed
     |
     +-- Unapproved repository --> Blocked
```

### Quick Memory

```text
sourceRepos = WHERE applications are allowed to get manifests FROM
```

**Interview answer:**

> **sourceRepos provides a security boundary by restricting Argo CD Applications to approved Git repositories.**

---

## Question 5: How can specific namespace resources be denied?

**Question:** In a multi-tenant environment, the Data team should be able to deploy most namespace-scoped resources, but should not be allowed to create `ResourceQuota` or `LimitRange` resources. Which AppProject configuration should be used?

**Answer:** Use the namespace resource deny-list field. The screenshot refers to the field as:

```text
spec.namespaceResourceBlacklist
```

Some newer terminology/documentation may describe blacklist/whitelist concepts as deny-list/allow-list behavior.

### Example

```yaml
spec:
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
    - group: ""
      kind: LimitRange
```

Conceptually:

```text
Data Team Application
        |
        v
     AppProject
        |
        +-- Deployment    --> Allowed
        +-- Service       --> Allowed
        +-- ConfigMap     --> Allowed
        +-- ResourceQuota --> Blocked
        +-- LimitRange    --> Blocked
```

### Quick Memory

```text
namespaceResourceBlacklist
        = namespace-scoped resources that are NOT allowed
```

**Interview answer:**

> **Use the namespace resource deny-list/blacklist configuration in AppProject to block specific namespace-scoped resource kinds while allowing other resources.**

---

# AppProject Quick Revision

```text
                         AppProject
                             |
          +------------------+------------------+
          |                  |                  |
          v                  v                  v
     sourceRepos         destinations       resources
          |                  |                  |
          v                  v                  v
     Which Git repos?   Which clusters?    Which K8s kinds?
                       Which namespaces?
```

---

# Quick-Glance Q&A

## Q: What does AppProject do?

**A:** Groups Applications and defines security boundaries for repositories, destinations, and Kubernetes resources.

## Q: What does sourceRepos control?

**A:** Which Git repositories Applications are allowed to use.

## Q: What do destinations control?

**A:** Which Kubernetes clusters and namespaces Applications can deploy to.

## Q: What does clusterResourceWhitelist control?

**A:** Which cluster-scoped Kubernetes resources an Application can deploy.

## Q: What does namespaceResourceBlacklist control?

**A:** Which namespace-scoped resource kinds Applications are prevented from deploying.

## Q: What project is used when no project is specified?

**A:** The `default` project.

---

# Interview Memory Trick

```text
AppProject = SECURITY BOUNDARY

SOURCE       --> Which Git repos?
DESTINATION  --> Which clusters/namespaces?
RESOURCES    --> Which Kubernetes resource kinds?
```

## One-Line Interview Summary

> **AppProject is Argo CD's logical security boundary for Applications, controlling where manifests can come from, where applications can deploy, and which Kubernetes resource types they are permitted to create.**
