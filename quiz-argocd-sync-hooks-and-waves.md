# Argo CD Sync Hooks & Sync Waves - Q&A Quiz

## Question 1: Which hook should run before application deployment?

**Question:** You need to run a database schema migration job **before** the new application deployment is applied to the cluster. If the job fails, the deployment should be aborted. Which Argo CD hook is appropriate?

**Answer:** `PreSync`

The `PreSync` hook executes before the application manifests are applied. If the hook fails, the sync operation is aborted, preventing the new application code from being deployed.

---

## Question 2: What is the default Sync Wave value?

**Question:** In Argo CD, what is the default value for the `sync-wave` annotation if the annotation is not explicitly defined on a resource?

**Answer:** `0`

If the following annotation is missing:

```text
argocd.argoproj.io/sync-wave
```

Argo CD assigns a wave value of `0`.

Resources with negative wave numbers are applied before wave `0`, and resources with positive wave numbers are applied afterward.

```text
Negative waves -> Wave 0 -> Positive waves
```

---

## Question 3: How do you delete a successful hook immediately after completion?

**Question:** A Kubernetes Job should be deleted immediately after the Job completes successfully during a sync operation. Which annotation should be added to the Job manifest?

**Answer:**

```yaml
argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

The `HookSucceeded` policy instructs Argo CD to delete the hook resource after the hook finishes successfully.

---

## Question 4: What is the execution order for Sync Waves?

**Question:** You have three resources with the following Sync Wave values:

```text
ConfigMap:  Wave -5
Service:    Wave 0 (Default)
Deployment: Wave 5
```

What is the execution order during the Sync phase?

**Answer:**

```text
ConfigMap -> Service -> Deployment
```

Argo CD orders waves from the lowest integer to the highest integer:

```text
Wave -5 -> Wave 0 -> Wave 5
```

Argo CD waits for resources in the current wave to be healthy before starting the next wave.

---

## Question 5: When does Argo CD move to the next Sync Wave?

**Question:** When using Sync Waves, what condition must be met before Argo CD proceeds from Wave 1 to Wave 2?

**Answer:** All resources defined in Wave 1 must reach a `Healthy` status in the cluster.

```text
Wave 1 applied
      |
      v
Resources become Healthy
      |
      v
Wave 2 begins
```

---

## Question 6: How does a Sync hook differ from a standard resource?

**Question:** If a resource is defined with:

```yaml
argocd.argoproj.io/hook: Sync
```

How does the resource behave compared with a standard resource without a hook annotation?

**Answer:** The hook resource is treated as **ephemeral**. Argo CD creates the hook to perform a task during the sync operation, but does not treat the hook as a standard continuously managed application resource after the sync operation completes.

The screenshot describes a standard resource as being managed by Argo CD for activities such as drift detection and pruning, while a hook is treated as a task used during the sync operation.

---

# Quick Answer Sheet

```text
Q1: Database migration before deployment? -> PreSync

Q2: Default sync-wave value? -> 0

Q3: Delete hook after successful execution?
    -> argocd.argoproj.io/hook-delete-policy: HookSucceeded

Q4: Wave -5, Wave 0, Wave 5 order?
    -> ConfigMap -> Service -> Deployment

Q5: When can the next wave begin?
    -> After resources in the current wave are Healthy

Q6: Sync hook behavior?
    -> Ephemeral task used during the sync operation
```

---

# Memory Diagram

```text
                 Argo CD Sync
                      |
          +-----------+-----------+
          |                       |
          v                       v
        Hooks                  Sync Waves
          |                       |
   PreSync / Sync          Lowest number first
          |                       |
          v                       v
 Perform sync task       -5 -> 0 -> 5
          |                       |
          v                       v
HookSucceeded can       Wait for Healthy before
remove successful hook    moving to next wave
```
