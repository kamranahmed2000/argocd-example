# Kubernetes Labels with Argo CD Repository Secret Example

## What Are Kubernetes Labels?

Kubernetes **labels** are key/value pairs attached to Kubernetes objects. Labels are commonly used to identify, organize, group, and select resources.

Examples:

```yaml
metadata:
  labels:
    environment: production
    tier: backend
    app: guestbook
```

Think of a label as a **tag attached to a Kubernetes resource**.

```text
Kubernetes Resource
       |
       +-- label: environment=prod
       +-- label: app=guestbook
       +-- label: tier=backend
```

---

## Basic Label Format

A label normally follows this format:

```text
key=value
```

Examples:

```text
environment=production
app=guestbook
tier=backend
```

A label key can also use a prefix:

```text
argocd.argoproj.io/secret-type=repository
```

Here:

```text
argocd.argoproj.io/secret-type = Label Key
repository                    = Label Value
```

---

## Adding a Label with kubectl

General syntax:

```bash
kubectl label <resource-type> <resource-name> -n <namespace> <key>=<value>
```

For example:

```bash
kubectl label pod my-pod -n default environment=dev
```

---

# Argo CD Repository Secret Example

In this example, an existing Kubernetes Secret called `private-repo-https` is labeled so Argo CD can recognize the Secret as repository configuration.

## Command from the Example

```bash
kubectl label secret private-repo-https -n argocd argocd.argoproj.io/secret-type=repository
```

### Breaking Down the Command

```text
kubectl label
    |
    +-- secret
    |      Resource type
    |
    +-- private-repo-https
    |      Secret name
    |
    +-- -n argocd
    |      Namespace
    |
    +-- argocd.argoproj.io/secret-type=repository
           Label key                     Label value
```

Or more simply:

```text
Resource Type : Secret
Resource Name : private-repo-https
Namespace     : argocd
Label Key     : argocd.argoproj.io/secret-type
Label Value   : repository
```

After running the command, `kubectl` reports:

```text
secret/private-repo-https labeled
```

---

## Verify the Label

The example verifies the Secret with:

```bash
kubectl describe secret private-repo-https -n argocd
```

The output shows the label:

```text
Name:         private-repo-https
Namespace:    argocd
Labels:       argocd.argoproj.io/secret-type=repository
Annotations:  <none>

Type:  Opaque
```

The important line is:

```text
Labels: argocd.argoproj.io/secret-type=repository
```

This confirms that the label is attached to the Secret.

---

## Visual Flow

```text
Kubernetes Secret
private-repo-https
       |
       | kubectl label
       v
+-------------------------------------------+
| Secret: private-repo-https                |
| Namespace: argocd                         |
|                                           |
| Label:                                    |
| argocd.argoproj.io/secret-type=repository |
+-------------------------------------------+
       |
       v
Argo CD recognizes the Secret
as repository configuration
```

---

## Label in YAML Form

The same label would appear under `metadata.labels` in a Kubernetes Secret manifest:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo-https
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
```

The command-line operation:

```bash
kubectl label secret private-repo-https -n argocd argocd.argoproj.io/secret-type=repository
```

adds the equivalent metadata:

```yaml
metadata:
  labels:
    argocd.argoproj.io/secret-type: repository
```

---

## Viewing Labels

Display labels when listing Secrets:

```bash
kubectl get secrets -n argocd --show-labels
```

Display the details of one Secret:

```bash
kubectl describe secret private-repo-https -n argocd
```

---

## Updating an Existing Label

If a label already exists and its value needs to be changed, use `--overwrite`:

```bash
kubectl label secret private-repo-https -n argocd <key>=<new-value> --overwrite
```

---

## Removing a Label

A Kubernetes label can be removed by placing a `-` after the label key:

```bash
kubectl label secret private-repo-https -n argocd argocd.argoproj.io/secret-type-
```

> **Warning:** Removing an Argo CD-specific label can change how Argo CD recognizes or processes that resource. Only remove it when that behavior is intended.

---

## Labels vs. Annotations

```text
Labels
  -> Identify and organize resources
  -> Can be used for resource selection

Annotations
  -> Store additional metadata
  -> Usually not used to select resources
```

For quick interview preparation:

> **Kubernetes labels are key/value pairs attached to resources to identify, organize, and select them.**

---

## Argo CD Example: Interview Answer

> **Argo CD uses specific labels on Kubernetes Secrets to identify their purpose. For example, the label `argocd.argoproj.io/secret-type=repository` identifies a Secret as repository configuration for Argo CD.**

---

## Easy Memory Trick

```text
Label = TAG on a Kubernetes object

kubectl label
     |
     v
RESOURCE + NAME + NAMESPACE + KEY=VALUE
```

Example:

```text
Secret
  |
  +-- Name      = private-repo-https
  +-- Namespace = argocd
  +-- Label     = argocd.argoproj.io/secret-type=repository
```

---

## Sample Commands at a Glance

### Add label

```bash
kubectl label secret private-repo-https -n argocd argocd.argoproj.io/secret-type=repository
```

### Verify

```bash
kubectl describe secret private-repo-https -n argocd
```

### Show labels

```bash
kubectl get secrets -n argocd --show-labels
```

### Update a label

```bash
kubectl label secret private-repo-https -n argocd <key>=<new-value> --overwrite
```

### Remove a label

```bash
kubectl label secret private-repo-https -n argocd argocd.argoproj.io/secret-type-
```

---

## One-Line Summary

**A Kubernetes label is a key/value tag attached to a resource; in this example, `argocd.argoproj.io/secret-type=repository` tells Argo CD that `private-repo-https` represents repository configuration.**

## References

- Kubernetes: Labels and Selectors — https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
- Kubernetes: kubectl label — https://kubernetes.io/docs/reference/kubectl/generated/kubectl_label/
- Argo CD: Annotations and Labels — https://argo-cd.readthedocs.io/en/stable/user-guide/annotations-and-labels/
