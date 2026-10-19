# Argo CD Repository Authentication - Q&A Quiz

Use these questions as a quick interview and revision quiz for Argo CD repository authentication.

---

## Question 1 - Kubernetes Secret Discovery

You manually created a Kubernetes Secret to authenticate Argo CD with a private repository using `kubectl create secret generic`. The username/password data is correct, but Argo CD still shows **Authentication required**.

**Question:** What is the most likely missing step?

**Answer:** The Secret is missing the required label:

```text
argocd.argoproj.io/secret-type=repository
```

Argo CD uses this label to discover that the Secret contains repository connection information.

**Memory tip:** Correct credentials are not enough. Argo CD must also recognize the Secret as a **repository Secret**.

---

## Question 2 - SSH Deploy Key Pair

When configuring repository access using an SSH Deploy Key pair, where should the public and private keys be placed?

**Answer:**

- **Public key** -> Add it to the Git provider as a repository Deploy Key.
- **Private key** -> Store it in the Kubernetes Secret used by Argo CD.

```text
Git Provider / Repository
        |
        +-- Public Key

Argo CD Kubernetes Secret
        |
        +-- Private Key
```

**Memory tip:** **Public goes to Git; Private stays with Argo CD.**

---

## Question 3 - Deploy Keys vs. Personal Access Tokens

From an operational and security perspective, what is an important advantage of using SSH Deploy Keys instead of user Personal Access Tokens for Argo CD repository access?

**Answer:** Deploy Keys are tied to a **repository** rather than a specific user account. This helps avoid repository access being coupled to an individual user's account lifecycle.

**Memory tip:** **Deploy Key = Repository identity; PAT = User-associated credential.**

---

## Question 4 - Fine-Grained GitHub PAT Permission

When generating a fine-grained GitHub Personal Access Token for Argo CD to read manifests from a repository, which repository permission is required for basic read-only GitOps access?

**Answer:**

```text
Contents: Read-only
```

Argo CD needs to read repository content containing manifests. Basic read-only repository access does not require administration, pull-request write, webhook write, or delete-repository permissions.

**Memory tip:** **Argo CD reads desired state -> Contents: Read-only.**

---

## Question 5 - Repository Configuration Through the Argo CD UI

When a repository connection is configured through **Argo CD -> Settings -> Repositories -> Connect Repo**, what happens in Kubernetes?

**Answer:** The Argo CD API server stores the repository configuration as a Kubernetes Secret in the Argo CD namespace with the repository credentials/configuration and the required Argo CD Secret metadata.

```text
Argo CD Web UI
      |
      v
Argo CD API Server
      |
      v
Kubernetes Secret
      |
      v
argocd namespace
```

**Memory tip:** **UI is the frontend; repository configuration is represented in Kubernetes as a Secret.**

---

## Question 6 - SSH URL vs. HTTPS URL

You configure SSH credentials for this repository:

```text
git@github.com:myorg/myrepo.git
```

But the Argo CD Application contains:

```yaml
spec:
  source:
    repoURL: https://github.com/myorg/myrepo.git
    targetRevision: HEAD
    path: .
```

**Question:** Why can repository authentication fail?

**Answer:** The Application uses the **HTTPS repository URL**, while the configured credentials correspond to the **SSH repository URL**. The repository URL and the authentication configuration need to match the repository connection Argo CD is expected to use.

### Correct SSH example

```yaml
spec:
  source:
    repoURL: git@github.com:myorg/myrepo.git
    targetRevision: HEAD
    path: .
```

**Memory tip:** **SSH credentials -> SSH repo URL. HTTPS credentials -> HTTPS repo URL.**

---

## Question 7 - Creating a Repository Secret with kubectl

You create a Secret like this:

```bash
kubectl create secret generic repo-creds \
  --from-literal=url="git@github.com:user/repo.git" \
  --from-literal=sshPrivateKey="$(cat ~/.ssh/id_ed25519)" \
  -n argocd
```

**Question:** What important Argo CD discovery step is missing?

**Answer:** The Secret needs the Argo CD repository Secret label:

```text
argocd.argoproj.io/secret-type=repository
```

One way to add the label is:

```bash
kubectl label secret repo-creds \
  -n argocd \
  argocd.argoproj.io/secret-type=repository
```

### Resulting concept

```text
Kubernetes Secret
      |
      +-- Repository URL
      +-- SSH Private Key
      +-- Argo CD repository label
                    |
                    v
             Discovered by Argo CD
```

**Memory tip:** **Create Secret + Add repository label.**

---

# Quick-Glance Answer Sheet

1. **Secret not recognized?** -> Check `argocd.argoproj.io/secret-type=repository`.
2. **SSH key placement?** -> Public key in Git provider; private key with Argo CD.
3. **Deploy Key advantage?** -> Repository-scoped rather than tied to one user's account lifecycle.
4. **Fine-grained PAT permission?** -> `Contents: Read-only` for basic read access.
5. **UI repository setup?** -> Repository configuration is stored as a Kubernetes Secret in the Argo CD namespace.
6. **SSH credentials but HTTPS `repoURL`?** -> Authentication configuration and repository URL do not match.
7. **Manual Secret ignored?** -> Add the Argo CD repository Secret label.

---

# Interview Memory Diagram

```text
                    PRIVATE GIT REPOSITORY
                            |
             +--------------+--------------+
             |                             |
             v                             v
           SSH                           HTTPS
             |                             |
      Deploy Key                    PAT / Credentials
             |                             |
     Public -> Git                   Argo CD Secret
     Private -> Argo CD                    |
             |                             |
             +--------------+--------------+
                            |
                            v
                  Kubernetes Secret
                   in argocd namespace
                            |
                            v
                         Argo CD
                            |
                            v
                 Reads manifests from Git
```

---

## One-Line Interview Summary

> **Argo CD repository authentication is commonly configured using Kubernetes Secrets; SSH uses the private key on the Argo CD side and the public Deploy Key on the Git provider, while HTTPS can use repository credentials such as a suitably scoped token.**
