# Argo CD Sync Phases & Hooks

## Overview

Argo CD Sync Phases and Hooks allow custom code to run during an Argo CD sync operation.

## Sync Flow

```text
Sync operation triggered
          |
          v
     PreSync phase
          |
          v
       Success?
       /           Yes       No
      |         |
      v         v
 Sync phase   SyncFail phase
      |
      v
Apply application manifests
      |
      v
Success and Healthy?
       /           Yes       No
      |         |
      v         v
PostSync     SyncFail
 phase        phase
```

---

## 1. PreSync Phase

The **PreSync** phase runs before application manifests are applied.

The hook annotation is:

```yaml
argocd.argoproj.io/hook: PreSync
```

The image shows several Kubernetes Jobs running during the PreSync phase.

### PreSync Job Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-pre-sync
  annotations:
    argocd.argoproj.io/hook: PreSync
spec:
  ...
```

### PreSync Result

```text
PreSync succeeds
      |
      v
Continue to Sync phase

PreSync fails
      |
      v
Run SyncFail phase
```

---

## 2. Sync Phase

The **Sync** phase applies the application manifests.

A Sync hook uses this annotation:

```yaml
argocd.argoproj.io/hook: Sync
```

### Sync Phase Flow

```text
Sync phase
    |
    +-- Run Sync hooks
    |
    +-- Apply application manifests
    |
    v
Check whether the sync is successful and healthy
```

### Sync Result

```text
Successful and Healthy
        |
        v
Run PostSync phase

Not Successful or Healthy
        |
        v
Run SyncFail phase
```

---

## 3. PostSync Phase

The **PostSync** phase runs after the Sync phase is successful and healthy.

The hook annotation is:

```yaml
argocd.argoproj.io/hook: PostSync
```

### PostSync Flow

```text
Sync successful and healthy
            |
            v
       PostSync phase
            |
            v
      Run PostSync hooks
```

---

## 4. SyncFail Phase

The **SyncFail** phase runs when the PreSync or Sync phase fails.

The hook annotation is:

```yaml
argocd.argoproj.io/hook: SyncFail
```

### SyncFail Flow

```text
PreSync failure
      |
      +------------------+
                         v
                   SyncFail phase
                         ^
      +------------------+
      |
Sync is not successful and healthy
```

---

## Additional Argo CD Hooks

### Skip

The **Skip** hook indicates to Argo CD that the manifest should not be applied.

```text
Skip = Do not apply the manifest
```

### PostDelete

The **PostDelete** hook executes after all Application resources are deleted.

The image notes that PostDelete is available starting in **Argo CD v2.10**.

```text
Application resources deleted
              |
              v
       PostDelete hook
```

---

## Hook Summary

| Hook | When It Runs |
|---|---|
| **PreSync** | Before application manifests are applied |
| **Sync** | During the Sync phase |
| **PostSync** | After the sync is successful and healthy |
| **SyncFail** | When PreSync or Sync fails |
| **Skip** | Tells Argo CD to skip applying a manifest |
| **PostDelete** | After all Application resources are deleted; available starting in v2.10 |

---

## Quick Memory Diagram

```text
PreSync
   |
   v
Sync and apply manifests
   |
   +-- Success and Healthy --> PostSync
   |
   +-- Failure -------------> SyncFail

Skip       = Do not apply the manifest
PostDelete = Run after Application resources are deleted
```

## One-Line Summary

**Argo CD hooks run custom Kubernetes resources at specific stages of a sync operation: before sync, during sync, after a successful sync, after a failed sync, or after Application resource deletion.**
