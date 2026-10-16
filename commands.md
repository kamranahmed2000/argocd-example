Installing Argo CD CLI
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.8/argocd-linux-amd64
chmod +x /usr/local/bin/argocd


Verify that CLI is installed
argocd

https://argo-cd.readthedocs.io/en/stable/cli_installation/

------------------------------------------------------------------------------------------------------------------------

Argocd admin user password to login in web.
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

------------------------------------------------------------------------------------------------------------------------

kubectl get svc -n argocd
Form the service list, use argocd-server for port forwarding.
kubectl port-forward svc/argocd-server -n argocd 8080:80

Post which, use https://127.0.0.1:8080 to access argocd UI from the browser.

For login credentials, 
username : admin
password: 

password is available in a secret. 
kubectl get secret -n argocd

use the secret named argocd-initial-admin-secret 
kubectl get argocd-initial-admin-secret -o jsonpath="{.data.password}" -n argocd

the password that we get is base 64 encoded. to decode and get the raw password, use the below command
kubectl get argocd-initial-admin-secret -o jsonpath="{.data.password}" -n argocd | base64 -d 

------------------------------------------------------------------------------------------------------------------------

To install argocd CLI
for mac use, brew install argocd

which argocd -- command to get the argocd installation path

argocd version -- command to get the argocd version. 

------------------------------------------------------------------------------------------------------------------------
to connect argocd from cli to the server.
argocd login localhost:8080 --name local
press y to proceed and enter your username and password to login to argocd cli.

argocd context
argocd account get-user-info -- command to get the logged in user information.
argocd account update-password -- command to update your current logged in user password. 
argocd app help -- command to get help for argocd

argocd app list -- command to list the argocd app from the cluster. 

------------------------------------------------------------------------------------------------------------------------
The Argo CD Components
Cluster --> argocd(namespace) --> 
3 Components -- API Server, Application Controller, Repository Server.
We access the UI using the argocd API Server. we also access the argocd cli that interacts with API Server. 
The API server interacts with all argocd components for request/response.

we can also connect to the argocd API server using gRPC/REST.

ArgoCD communicates with Git Repository using the argocd Repository Server. Interacts with the Git repository and maintains a local cache of it. Also responsible for generating the final k8s manifest for a specific application.

The Argo CD Application Controller continuously compares the desired state in Git with the live Kubernetes state and reconciles differences to keep applications in sync.

argocd-redis: Provides caching to improve Argo CD performance by storing frequently accessed application and cluster state information.
argocd-dex-server: Provides SSO authentication, integrating Argo CD with identity providers using OIDC/SAML.
argocd-applicationset-controller: Automatically creates and manages multiple Argo CD Applications, especially useful for deploying across multiple clusters/environments.

Easy interview memory: Redis = Cache | Dex = Authentication/SSO | ApplicationSet = Manage many applications.


------------------------------------------------------------------------------------------------------------------------
Argo CD Components
API Server → Entry point for UI, CLI, REST/gRPC requests.
Repository Server → Connects to Git, caches the repo, and generates Kubernetes manifests.
Application Controller → Compares desired state in Git vs. live state in Kubernetes and reconciles differences.
Redis → Cache for manifest and cluster/application state.
Dex Server → Handles SSO/authentication using OIDC/SAML.
ApplicationSet Controller → Creates/manages multiple Argo CD applications at scale.
Lifecycle Hooks → Run actions during different stages of application synchronization.
⭐ 10-second memory trick

API = Access | Repo = Manifests | Controller = Sync | Redis = Cache | Dex = SSO | ApplicationSet = Scale

------------------------------------------------------------------------------------------------------------------------
![Uploading image.png…]()

The Application CRD

PROJECT      → WHO manages it?
SOURCE       → WHERE does configuration come from?
DESTINATION  → WHERE should it be deployed?
SYNC POLICY  → HOW should Argo CD manage/sync it?                 

------------------------------------------------------------------------------------------------------------------------
The Argo CD Application CRD is a declarative definition that tells Argo CD which project owns an application, where to get the desired state from, where to deploy it, and how synchronization should be handled.

Application CRD = Project + Source + Destination + Sync Policy
Or even easier:
WHO → WHERE FROM → WHERE TO → HOW

What each section means
Project: Defines which Argo CD project manages the application.
Source: Defines where the application comes from, such as Git repo, branch, and path.
Destination: Defines where the application is deployed, meaning Kubernetes cluster and namespace.
Sync Policy: Defines how Argo CD deploys and maintains it, such as Auto Sync, Prune, and Self-Heal.

Quick memory: Project = WHO | Source = FROM WHERE | Destination = TO WHERE | Sync = HOW

------------------------------------------------------------------------------------------------------------------------

Argo CD Application CRD
The Application CRD is a declarative definition that tells Argo CD what application to manage, where to get it from, where to deploy it, and how to synchronize it.

                 Argo CD Application CRD
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
       PROJECT           SOURCE         DESTINATION
    "Who manages?"    "Where from?"     "Where to?"
          |                |                |
          v                v                v
    Argo CD Project    Git Repository   K8s Cluster
                           |                |
                       Branch/Tag        Namespace
                           |
                       Folder/Chart
                           |
                           v
                  Kubernetes Manifests
                           |
                           v
                      SYNC POLICY
                         "How?"
                           |
                  +--------+--------+
                  |        |        |
                  v        v        v
                Auto     Prune   Self-Heal


What each section means
Project: Defines which Argo CD project manages the application.
Source: Defines where the application comes from, such as Git repo, branch, and path.
Destination: Defines where the application is deployed, meaning Kubernetes cluster and namespace.
Sync Policy: Defines how Argo CD deploys and maintains it, such as Auto Sync, Prune, and Self-Heal.
Quick Interview Memory
Project = WHO | Source = FROM WHERE | Destination = TO WHERE | Sync = HOW

```
Application YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <YOUR_REPO_URL>
    targetRevision: HEAD
    path: guestbook

  destination:
    server: <YOUR_K8S_SERVER>
    namespace: guestbook-prod
```

------------------------------------------------------------------------------------------------------------------------    
    


