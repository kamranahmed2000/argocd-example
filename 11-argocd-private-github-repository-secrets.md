# Argo CD: How Repository Secrets Are Identified and Used for Private GitHub Repositories

## Overview

When an Argo CD `Application` points to a **private GitHub repository**, Argo CD needs credentials to read that repository.

The important distinction is:

- **Kubernetes stores the Secret.**
- **Argo CD interprets the Argo CD-specific label on the Secret.**
- The label tells Argo CD whether the Secret is a repository definition or a reusable repository credential template.
- Argo CD then matches the repository URL against the URL stored in the Secret to determine which credentials apply.

```text
Application CR
    |
    | spec.source.repoURL
    v
Private GitHub Repository
    ^
    |
Argo CD needs credentials
    |
    v
Secrets in the Argo CD namespace
    |
    | Argo CD identifies relevant Secrets by label
    v
argocd.argoproj.io/secret-type
    |
    +-- repository  -> credentials/config for a specific repository
    |
    +-- repo-creds  -> reusable credentials matched by repository URL prefix
```

---

## 1. Application Points to the Private Repository

Example Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mycompany/private-app.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
```

The important field is:

```yaml
repoURL: https://github.com/mycompany/private-app.git
```

Argo CD must authenticate to GitHub before the repository server can fetch manifests from this private repository.

---

## 2. Kubernetes Does Not Understand the Argo CD Label Semantics

A Kubernetes Secret is simply an API object stored in Kubernetes.

For example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-app-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
```

Kubernetes stores the Secret and its labels. Kubernetes itself does **not** interpret
`argocd.argoproj.io/secret-type: repository` as meaning "GitHub credentials."

That label belongs to Argo CD's configuration convention. Argo CD watches/reads the relevant Secrets in its installation namespace and uses the label to determine their purpose.

```text
Kubernetes API
      |
      | stores
      v
Secret + labels + credential data
      |
      | read/interpreted by
      v
Argo CD
```

---

## 3. Repository Secret for One Specific Repository

Use this form when credentials/configuration are associated with one repository URL.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-app-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/mycompany/private-app.git
  username: <GITHUB_USERNAME>
  password: <GITHUB_TOKEN>
```

The key label is:

```yaml
argocd.argoproj.io/secret-type: repository
```

This tells Argo CD that the Secret represents repository configuration.

The URL:

```yaml
url: https://github.com/mycompany/private-app.git
```

identifies the repository associated with that Secret.

### Flow

```text
Application
    |
    | repoURL
    v
https://github.com/mycompany/private-app.git
    |
    v
Argo CD repository configuration
    |
    | Secret label identifies repository Secret
    v
private-app-repo Secret
    |
    | credentials
    v
GitHub Private Repository
```

---

## 4. Repository Credential Template (`repo-creds`)

If several repositories use the same credentials, Argo CD can use a repository credential template.

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-mycompany-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
type: Opaque
stringData:
  url: https://github.com/mycompany
  username: <GITHUB_USERNAME>
  password: <GITHUB_TOKEN>
```

The important label is now:

```yaml
argocd.argoproj.io/secret-type: repo-creds
```

This tells Argo CD that the Secret is a **repository credential template** rather than a repository entry for only one exact repository.

The URL acts as a prefix for repositories that can use the credentials.

For example:

```text
Credential URL prefix:
https://github.com/mycompany

Can apply to repositories such as:
https://github.com/mycompany/app-one.git
https://github.com/mycompany/app-two.git
https://github.com/mycompany/platform-config.git
```

---

## 5. How the Label Filtering Works

Conceptually, Argo CD classifies repository-related Secrets using this label:

```text
argocd.argoproj.io/secret-type
```

Two important values are:

```text
repository  -> repository-specific configuration
repo-creds  -> reusable repository credential template
```

Think of the Argo CD logic as:

```text
Secrets in argocd namespace
         |
         v
Check Argo CD secret-type label
         |
   +-----+------+
   |            |
   v            v
repository   repo-creds
   |            |
   |            +--> URL-prefix credential matching
   |
   +--> repository-specific configuration
```

This is **Argo CD behavior**, not a generic Kubernetes feature.

A command you can use to inspect repository-labeled Secrets is:

```bash
kubectl get secrets -n argocd \
  -l argocd.argoproj.io/secret-type=repository
```

To inspect reusable credential-template Secrets:

```bash
kubectl get secrets -n argocd \
  -l argocd.argoproj.io/secret-type=repo-creds
```

These commands demonstrate normal Kubernetes label selection. Argo CD uses the same label values as part of its repository configuration model.

---

## 6. `repository` vs. `repo-creds`

### `repository`

```yaml
labels:
  argocd.argoproj.io/secret-type: repository
```

Use when defining/configuring a **specific repository**.

```text
Secret
  |
  +--> https://github.com/mycompany/private-app.git
```

### `repo-creds`

```yaml
labels:
  argocd.argoproj.io/secret-type: repo-creds
```

Use as a **credential template** that can provide credentials to matching repository URLs.

```text
Credential Template
https://github.com/mycompany
          |
          +--> app-one.git
          +--> app-two.git
          +--> platform-config.git
```

---

## 7. End-to-End Example

Assume the Application contains:

```yaml
source:
  repoURL: https://github.com/mycompany/private-app.git
  targetRevision: main
  path: k8s
```

And this credential template exists:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
type: Opaque
stringData:
  url: https://github.com/mycompany
  username: <GITHUB_USERNAME>
  password: <GITHUB_TOKEN>
```

Conceptually:

```text
Application CR
    |
    | repoURL
    v
https://github.com/mycompany/private-app.git
    |
    | matches credential prefix
    v
https://github.com/mycompany
    |
    v
repo-creds Secret
    |
    | Git credentials
    v
Argo CD Repository Server
    |
    | authenticated Git request
    v
Private GitHub Repository
    |
    | manifests
    v
Argo CD Application Controller
    |
    v
Target Kubernetes Cluster
```

---

## 8. Applying the Secret

If the Secret YAML is created outside Git:

```bash
kubectl apply -f github-repo-secret.yaml
```

Verify the Secret exists:

```bash
kubectl get secret github-creds -n argocd
```

Check the Argo CD-specific label without displaying credential values:

```bash
kubectl get secret github-creds -n argocd \
  --show-labels
```

Or filter directly:

```bash
kubectl get secrets -n argocd \
  -l argocd.argoproj.io/secret-type=repo-creds
```

---

## 9. Security Note

Do **not** commit real GitHub tokens, passwords, or private SSH keys as plaintext into Git.

The examples in this document intentionally use placeholders:

```text
<GITHUB_USERNAME>
<GITHUB_TOKEN>
```

In production, use your organization's approved secret-management approach. Argo CD supports private Git repositories over HTTPS and SSH, and also supports GitHub App credentials.

---

## 10. GitHub App Credentials

Argo CD can also authenticate using a GitHub App. A credential-template Secret can contain GitHub App information instead of a username/token.

Simplified example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-app-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
type: Opaque
stringData:
  url: https://github.com/mycompany
  githubAppID: "<APP_ID>"
  githubAppInstallationID: "<INSTALLATION_ID>"
  githubAppPrivateKey: |
    <PRIVATE_KEY>
```

Keep the real private key outside source control and provide it through an approved secret-management mechanism.

---

## Quick Interview Answer

> **Kubernetes stores the repository Secret, but Argo CD interprets it. Argo CD uses the `argocd.argoproj.io/secret-type` label to distinguish repository Secrets from reusable `repo-creds` credential templates. Argo CD then uses the repository URL or matching URL prefix to select credentials for a private Git repository.**

---

## Easy Memory Trick

```text
Kubernetes = Stores the Secret
Label      = Tells Argo CD what type of Secret it is
URL        = Tells Argo CD which repository/repositories it applies to
Credentials= Let Argo CD authenticate to the private repository
```

```text
repository = specific repository configuration
repo-creds = reusable credentials by URL prefix
```

---

## Official References

- Argo CD private repositories: https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/
- Argo CD repository credential Secret example: https://argo-cd.readthedocs.io/en/stable/operator-manual/argocd-repo-creds-yaml/
