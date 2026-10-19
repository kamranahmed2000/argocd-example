# Argo CD: Connecting to Private GitHub Repositories

## Overview

Argo CD must authenticate before it can read Kubernetes manifests, Helm charts, or Kustomize files from a private GitHub repository. The two common connection methods are:

1. **HTTPS with a GitHub Personal Access Token (PAT)**
2. **SSH with a dedicated deploy key**

Argo CD stores repository credentials as Kubernetes `Secret` resources in the `argocd` namespace. Repository secrets must include the label:

```yaml
argocd.argoproj.io/secret-type: repository
```

> **Security warning:** Never commit real PATs, passwords, or private SSH keys to Git. The screenshot used for these notes appears to contain a visible GitHub token. If that token is real, revoke or rotate it immediately and remove it from terminal history, screenshots, and logs.

---

## Connection Flow

```text
Private GitHub Repository
          ^
          | HTTPS + PAT or SSH key
          |
 Argo CD Repository Server
          ^
          | reads credentials
          |
 Kubernetes Secret
 Namespace: argocd
```

The Argo CD repository server uses the stored credentials to clone or fetch the private repository and generate the desired Kubernetes manifests.

---

# Option 1: HTTPS with a GitHub PAT

## Step 1: Create a GitHub token

Create a fine-grained GitHub PAT that is limited to the required repository and has only the permissions Argo CD needs. For read-only GitOps access, repository **Contents: Read-only** is normally sufficient.

Do not place the token directly in a Git repository.

## Step 2: Create an Argo CD repository Secret

Create a local file named `github-private-repo-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-private-repo-https
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/example-org/private-app.git
  username: YOUR_GITHUB_USERNAME
  password: YOUR_GITHUB_PAT
```

Apply the Secret:

```bash
kubectl apply -f github-private-repo-secret.yaml
```

Immediately delete the local file if the file contains a real token:

```bash
rm github-private-repo-secret.yaml
```

> Using a YAML file with a plaintext token is suitable only for demonstrating the Secret structure. In production, use a secret manager such as External Secrets Operator, Sealed Secrets, or another approved secret-management system.

## Alternative: Create the HTTPS Secret with `kubectl`

The screenshot shows this general approach. Use environment variables or securely supplied values rather than exposing the token in screenshots or shell history.

```bash
kubectl -n argocd create secret generic github-private-repo-https   --from-literal=type=git   --from-literal=url=https://github.com/example-org/private-app.git   --from-literal=username=YOUR_GITHUB_USERNAME   --from-literal=password=YOUR_GITHUB_PAT
```

Add the required Argo CD repository label:

```bash
kubectl -n argocd label secret github-private-repo-https   argocd.argoproj.io/secret-type=repository
```

## Safer shell example

This prevents the token from appearing directly in the command itself, although organization-approved secret management is still preferable:

```bash
read -s GITHUB_PAT

kubectl -n argocd create secret generic github-private-repo-https   --from-literal=type=git   --from-literal=url=https://github.com/example-org/private-app.git   --from-literal=username=YOUR_GITHUB_USERNAME   --from-literal=password="$GITHUB_PAT"

kubectl -n argocd label secret github-private-repo-https   argocd.argoproj.io/secret-type=repository

unset GITHUB_PAT
```

## HTTPS using the Argo CD CLI

```bash
argocd repo add https://github.com/example-org/private-app.git   --username YOUR_GITHUB_USERNAME   --password YOUR_GITHUB_PAT
```

## Verify the HTTPS connection

```bash
argocd repo list
```

Inspect the Kubernetes Secret without printing its sensitive values:

```bash
kubectl get secret github-private-repo-https -n argocd
```

---

# Option 2: SSH with a GitHub Deploy Key

SSH authentication uses a public/private key pair:

```text
GitHub repository
└── Public key configured as Deploy Key

Argo CD namespace
└── Private key stored in Kubernetes Secret
```

## Step 1: Generate a dedicated SSH key pair

```bash
ssh-keygen -t ed25519   -C "argocd-private-app"   -f ./argocd-private-app   -N ""
```

Files created:

```text
argocd-private-app       Private key: keep secret
argocd-private-app.pub   Public key: add to GitHub
```

## Step 2: Add the public key to GitHub

In the private GitHub repository:

```text
Settings
  -> Deploy keys
  -> Add deploy key
```

Paste the contents of:

```bash
cat argocd-private-app.pub
```

Argo CD normally needs only read access, so do not enable write access unless there is a specific requirement.

## Step 3: Create the SSH repository Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-private-repo-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: git@github.com:example-org/private-app.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    REPLACE_WITH_PRIVATE_KEY_CONTENT
    -----END OPENSSH PRIVATE KEY-----
```

Apply the Secret:

```bash
kubectl apply -f github-private-repo-ssh.yaml
```

## Alternative: Create the SSH Secret with `kubectl`

```bash
kubectl -n argocd create secret generic github-private-repo-ssh   --from-literal=type=git   --from-literal=url=git@github.com:example-org/private-app.git   --from-file=sshPrivateKey=./argocd-private-app
```

Add the required label:

```bash
kubectl -n argocd label secret github-private-repo-ssh   argocd.argoproj.io/secret-type=repository
```

Remove the private key from the workstation after storing it securely:

```bash
rm -f ./argocd-private-app
```

## SSH using the Argo CD CLI

```bash
argocd repo add git@github.com:example-org/private-app.git   --ssh-private-key-path ./argocd-private-app
```

## Verify the SSH connection

```bash
argocd repo list
```

---

# Repository Secret vs. Credential Template

## Repository Secret

Use `repository` when credentials apply to one specific repository:

```yaml
metadata:
  labels:
    argocd.argoproj.io/secret-type: repository
```

Example URL:

```yaml
stringData:
  url: https://github.com/example-org/private-app.git
```

## Credential Template

Use `repo-creds` when the same credentials should apply to multiple repositories sharing a URL prefix:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-org-https-credentials
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
type: Opaque
stringData:
  url: https://github.com/example-org
  username: YOUR_GITHUB_USERNAME
  password: YOUR_GITHUB_PAT
```

This template can match repositories underneath the URL prefix:

```text
https://github.com/example-org/app-one.git
https://github.com/example-org/app-two.git
https://github.com/example-org/platform-config.git
```

SSH credential template example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-org-ssh-credentials
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
type: Opaque
stringData:
  url: git@github.com:example-org
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    REPLACE_WITH_PRIVATE_KEY_CONTENT
    -----END OPENSSH PRIVATE KEY-----
```

---

# Referencing the Private Repository from an Application

The Argo CD `Application` uses the same repository URL format configured in the repository Secret.

## HTTPS example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: private-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example-org/private-app.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: private-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## SSH example

```yaml
spec:
  source:
    repoURL: git@github.com:example-org/private-app.git
    targetRevision: main
    path: k8s
```

The `repoURL` in the Application should match the URL configured for the repository credentials.

---

# Important Security Practices

- Never commit PATs, passwords, private keys, or unencrypted Kubernetes Secrets to Git.
- Immediately revoke any credential accidentally exposed in a screenshot, command history, log, or commit.
- Prefer fine-grained, read-only, repository-scoped access.
- Use a dedicated credential for Argo CD instead of a personal everyday credential.
- For SSH, use a dedicated deploy key and leave write access disabled unless required.
- Rotate credentials regularly and immediately when exposure is suspected.
- Restrict access to Secrets in the `argocd` namespace using Kubernetes RBAC.
- Avoid passing secrets directly on a shared command line because commands may be saved in shell history or visible to other processes.
- Use External Secrets, Sealed Secrets, or an approved vault solution for production GitOps workflows.
- Remember that Base64 encoding in a normal Kubernetes Secret is not encryption.

---

# Troubleshooting

## Repository shows authentication failure

Check the configured repository:

```bash
argocd repo list
```

Test repository details:

```bash
argocd repo get https://github.com/example-org/private-app.git
```

Check repository server logs:

```bash
kubectl logs -n argocd deployment/argocd-repo-server
```

## Secret is ignored by Argo CD

Confirm the namespace and label:

```bash
kubectl get secret github-private-repo-https -n argocd   --show-labels
```

Expected label:

```text
argocd.argoproj.io/secret-type=repository
```

## HTTPS repository does not connect

Check the following:

- The repository URL is correct.
- The PAT has access to the selected repository.
- The PAT has not expired or been revoked.
- The username and token are placed in the correct fields.
- The Application `repoURL` matches the registered URL.

## SSH repository does not connect

Check the following:

- The public key is added to the correct GitHub repository.
- The Kubernetes Secret contains the matching private key.
- The SSH URL uses the correct format: `git@github.com:ORG/REPO.git`.
- The private key is in valid OpenSSH format.
- SSH host-key verification is correctly configured.

---

# HTTPS vs. SSH: Quick Choice

```text
HTTPS + PAT
- Simple to configure
- Easy to rotate
- Token permissions and expiration must be managed

SSH Deploy Key
- Repository-specific access
- No PAT required
- Key lifecycle and known-host verification must be managed
```

---

# Interview Answer

> **Argo CD can connect to a private GitHub repository using HTTPS with a personal access token or SSH with a deploy key. The credentials are stored in a Kubernetes Secret in the `argocd` namespace and labeled as an Argo CD repository or credential-template Secret. The repository server reads those credentials to clone the private repository and generate the desired manifests.**

## Easy Memory

```text
HTTPS = Repository URL + Username + PAT
SSH   = SSH Repository URL + Private Key

GitHub keeps:
- PAT authorization, or
- SSH public deploy key

Argo CD keeps:
- PAT in a repository Secret, or
- SSH private key in a repository Secret
```

---

## Official References

- [Argo CD Private Repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)
- [Argo CD Repository Credential Secret Example](https://argo-cd.readthedocs.io/en/stable/operator-manual/argocd-repo-creds-yaml/)
