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




