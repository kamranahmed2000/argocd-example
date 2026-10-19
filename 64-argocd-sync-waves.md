# Argo CD Sync Waves

## Overview

**Sync Waves** define the order in which Argo CD applies Kubernetes resources within each synchronization phase.

A Sync Wave is assigned to a resource using this annotation:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "<wave-number>"
```

Argo CD processes waves from the **lowest number to the highest number**.

```text
-5 -> -2 -> -1 -> 0 -> 1 -> 2 -> 3 -> 10
```

If a resource does not have a Sync Wave annotation, the resource uses **Wave 0**.

---

## Sync Phases and Sync Waves

The images show three major phases:

1. **PreSync Phase**
2. **Sync Phase**
3. **PostSync Phase**

Argo CD completes the waves within one phase before moving to the next phase.

```text
PreSync Phase
     |
     | Lowest wave to highest wave
     v
Sync Phase
     |
     | Lowest wave to highest wave
     v
PostSync Phase
     |
     | Lowest wave to highest wave
     v
Sync operation completed
```

### Phase and Wave Example from the Image

```text
PreSync Phase
  Wave -5
  Wave -2
  Wave  2
  Wave 10
       |
       v
Sync Phase
  Wave -2
  Wave -1
  Wave  1
  Wave  2
       |
       v
PostSync Phase
  Wave -5
  Wave  3
  Wave  4
  Wave  5
```

> Wave ordering restarts inside each phase. For example, PostSync Wave `-5` still runs after the entire Sync phase because phases take precedence over wave numbers.

---

## Order of Execution

The order shown in the images moves from top to bottom:

```text
PHASE 1: PreSync
  -5 -> -2 -> 2 -> 10

PHASE 2: Sync
  -2 -> -1 -> 1 -> 2

PHASE 3: PostSync
  -5 -> 3 -> 4 -> 5
```

### Important Rule

```text
Phase first, then Wave number
```

Argo CD first determines the phase and then processes resources in ascending wave order within that phase.

---

## How Sync Waves Work

When a sync operation is triggered, Argo CD applies resources wave by wave.

```text
Sync operation triggered
          |
          v
Apply resources in lowest wave
          |
          v
Check resource health
          |
          v
Proceed to next wave
          |
          v
Apply resources in next wave
          |
          v
Check resource health
          |
          v
Continue until all waves are complete
```

The image illustrates this example:

```text
Sync Wave -1
    |
    +-- Apply application manifests
    +-- Check: Healthy?
             |
             v
Sync Wave 1
    |
    +-- Apply application manifests
    +-- Check: Healthy?
             |
             v
Sync Wave 2
    |
    +-- Apply application manifests
    +-- Check: Healthy?
```

This ordering is useful when later resources depend on earlier resources being available and healthy.

---

## Example Resource Ordering

Assume an application requires the following order:

1. Create the Namespace.
2. Create the database StatefulSet.
3. Deploy the main application.

The resources can be assigned these waves:

```text
Wave -1 -> Namespace
Wave  1 -> Database StatefulSet
Wave  2 -> Application Deployment
```

### Namespace: Wave -1

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-ns
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

### Database StatefulSet: Wave 1

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-db
  namespace: my-ns
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  serviceName: my-db
  replicas: 1
  selector:
    matchLabels:
      app: my-db
  template:
    metadata:
      labels:
        app: my-db
    spec:
      containers:
        - name: database
          image: <DATABASE_IMAGE>
```

### Application Deployment: Wave 2

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application
  namespace: my-ns
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-application
  template:
    metadata:
      labels:
        app: my-application
    spec:
      containers:
        - name: application
          image: <APPLICATION_IMAGE>
```

### Resulting Execution

```text
Wave -1
  Create Namespace
        |
        v
Wave 1
  Create Database StatefulSet
        |
        v
Wave 2
  Create Application Deployment
```

---

## Negative, Zero, and Positive Waves

### Negative Waves

Negative waves run before Wave 0.

```text
Wave -5 -> Wave -2 -> Wave -1 -> Wave 0
```

Common uses include foundational or prerequisite resources.

### Wave 0

Wave 0 is the default when no Sync Wave annotation is present.

```yaml
metadata:
  name: my-resource
```

Conceptually, the resource is processed as:

```text
Wave 0
```

### Positive Waves

Positive waves run after Wave 0.

```text
Wave 0 -> Wave 1 -> Wave 2 -> Wave 10
```

Common uses include application workloads and resources that depend on earlier resources.

---

## Combining Sync Phases and Sync Waves

Sync phases and Sync Waves can be combined.

- **Sync phases** define the major stage of execution.
- **Sync Waves** define the ordering of resources inside each phase.

```text
PHASE determines the stage
          |
          v
WAVE determines the order inside that stage
```

### Example: PreSync Hook in Wave -1

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: database-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "-1"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: <MIGRATION_IMAGE>
```

### Example: Main Resource in Sync Wave 1

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-application
  template:
    metadata:
      labels:
        app: my-application
    spec:
      containers:
        - name: application
          image: <APPLICATION_IMAGE>
```

### Example: PostSync Hook in Wave 3

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/sync-wave: "3"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: smoke-test
          image: <TEST_IMAGE>
```

### Combined Flow

```text
PreSync Phase
  Wave -1 -> Run database migration
        |
        v
Sync Phase
  Wave 1  -> Deploy application
        |
        v
PostSync Phase
  Wave 3  -> Run smoke test
```

---

## Sync Phase vs. Sync Wave

| Concept | Purpose | Example |
|---|---|---|
| **Sync Phase** | Defines the major lifecycle stage | PreSync, Sync, PostSync |
| **Sync Wave** | Defines resource order inside a phase | `-1`, `0`, `1`, `2` |

### Easy Memory

```text
Phase = WHEN in the sync lifecycle
Wave  = WHICH ORDER inside the phase
```

---

## Practical Use Cases

### Use Case 1: Namespace Before Workloads

```text
Wave -1 -> Namespace
Wave  0 -> ConfigMap and Secret
Wave  1 -> Deployment and Service
```

### Use Case 2: Database Before Application

```text
Wave 0 -> Database
Wave 1 -> Backend application
Wave 2 -> Frontend application
```

### Use Case 3: Migration Before Deployment

```text
PreSync Wave -1 -> Database migration Job
Sync    Wave  1 -> Application Deployment
PostSync Wave 3 -> Smoke-test Job
```

### Use Case 4: Infrastructure Before Dependent Resources

```text
Wave -2 -> Custom Resource Definition
Wave -1 -> Operator
Wave  1 -> Custom Resource
```

---

## Quick Interview Questions and Answers

### Q1. What is a Sync Wave in Argo CD?

**Answer:** A Sync Wave controls the order in which Argo CD applies Kubernetes resources within a synchronization phase.

### Q2. How is a Sync Wave assigned?

**Answer:** Add the `argocd.argoproj.io/sync-wave` annotation to the Kubernetes resource.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

### Q3. In what order are waves processed?

**Answer:** From the lowest number to the highest number, including negative numbers.

```text
-2 -> -1 -> 0 -> 1 -> 2
```

### Q4. What wave is used when the annotation is missing?

**Answer:** Wave 0.

### Q5. Can Sync Waves and hooks be combined?

**Answer:** Yes. Argo CD uses phases such as PreSync, Sync, and PostSync, and follows Sync Wave order within each phase.

### Q6. Which runs first: PreSync Wave 10 or Sync Wave -5?

**Answer:** PreSync Wave 10 runs first because the PreSync phase is completed before the Sync phase begins. Phase has priority over the wave number.

### Q7. Why are negative waves useful?

**Answer:** Negative waves allow prerequisite resources to run before resources in the default Wave 0.

---

## Interview-Ready Answer

> **Sync Waves in Argo CD define the execution order of Kubernetes resources within a sync phase. Resources are assigned wave numbers using the `argocd.argoproj.io/sync-wave` annotation, and Argo CD processes them from the lowest wave number to the highest. Sync Waves can be combined with PreSync, Sync, and PostSync phases to control the complete deployment sequence.**

---

## Quick Memory Trick

```text
PHASE = Major stage
WAVE  = Order inside the stage

Lowest wave number runs first
No wave annotation means Wave 0
```

## One-Line Summary

**Argo CD Sync Waves apply Kubernetes resources from the lowest wave number to the highest within each sync phase.**

---

## Reference

- [Argo CD Sync Phases and Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
