Argo CD Multi-Cluster Architecture

Overview
Argo CD can be installed in a Kubernetes cluster and can manage registered target Kubernetes clusters.

A target cluster can be added using:

argocd cluster add <context-name>
The cluster context must exist in the kubeconfig, and adding the cluster connects to it and installs the resources required for Argo CD connectivity.

Architecture
                         Git Repository
                              |
                              | Desired State
                              |
                              v
                +---------------------------+
                |    MANAGEMENT CLUSTER     |
                |                           |
                |         Argo CD           |
                |                           |
                |  +---------------------+  |
                |  | API Server          |  |
                |  | Repository Server   |  |
                |  | App Controller      |  |
                |  | Redis / Dex         |  |
                |  +---------------------+  |
                +------------+--------------+
                             |
                             |
                    Kubernetes API
                             |
             +---------------+---------------+
             |               |               |
             v               v               v
      +-------------+ +-------------+ +-------------+
      | DEV Cluster | | QA Cluster  | | PROD Cluster|
      |             | |             | |             |
      | Application | | Application | | Application |
      | Workloads   | | Workloads   | | Workloads   |
      +-------------+ +-------------+ +-------------+

Where is Argo CD installed?
Argo CD can be installed in a Kubernetes cluster that acts as the central location from which registered Kubernetes clusters are managed.

       Kubernetes Management Cluster
                   |
             +-----------+
             |  Argo CD  |
             +-----------+
                   |
         +---------+---------+
         |         |         |
         v         v         v
        DEV        QA       PROD
      Cluster    Cluster    Cluster

Does Every Cluster Need Argo CD?
No.

A full Argo CD installation is not required in every target cluster.

The target cluster is registered with Argo CD. The official cluster-management command is:

argocd cluster add <context-name>
Argo CD connects to the cluster and installs the necessary resources for Argo CD to connect to it.

Conceptually:

MANAGEMENT CLUSTER
--------------------------
Argo CD
  |
  +-- API Server
  +-- Repository Server
  +-- Application Controller
  +-- Redis
  +-- Dex

              |
              | Kubernetes API
              |
       +------+------+------+
       |             |      |
       v             v      v

DEV CLUSTER     QA CLUSTER     PROD CLUSTER
-----------     ----------     ------------
Workloads       Workloads      Workloads
Access/RBAC     Access/RBAC    Access/RBAC

How Does Argo CD Know Which Cluster to Deploy To?
The Argo CD Application CRD (Custom Resource Definition) contains the source and destination information.

spec:
  source:
    repoURL: <YOUR_REPO_URL>
    path: guestbook

  destination:
    server: <YOUR_K8S_SERVER>
    namespace: guestbook-prod
Source
Defines where the desired application configuration comes from:

Git Repository
      |
      +-- Branch / Revision
      |
      +-- Path
      |
      +-- Kubernetes YAML / Helm / etc.
Destination
Defines where the application should be deployed:

Destination
    |
    +-- Kubernetes Cluster
    |
    +-- Namespace
So the overall flow is:

         SOURCE                       DESTINATION
            |                              |
            v                              v
    +----------------+              +---------------+
    | Git Repository |              | K8s Cluster   |
    |                |              |               |
    | YAML / Helm    |              | Namespace     |
    +-------+--------+              +-------^-------+
            |                               |
            |                               |
            v                               |
       +------------------------------------+
       |              Argo CD               |
       |                                    |
       | Reads Desired State                |
       | Compares Desired vs Live State     |
       | Reconciles the Application         |
       +------------------------------------+
Multi-Cluster Deployment Flow
Developer
    |
    | git push
    v
Git Repository
    |
    | Desired State
    v
Argo CD
    |
    | Determines destination
    | from Application CRD
    |
    +----------+----------+
    |          |          |
    v          v          v
   DEV         QA        PROD
 Cluster     Cluster     Cluster
Registering a Target Cluster
A target cluster can be added with:

argocd cluster add <context-name>
For example:

argocd cluster add dev-cluster
Known clusters can be viewed using:

argocd cluster list
Quick Interview Answer
Argo CD can centrally manage multiple Kubernetes clusters. Target clusters are registered with Argo CD, and Argo CD communicates with them to manage Kubernetes resources. A full Argo CD installation is not required on every target cluster. The Application CRD specifies the source repository and destination cluster/namespace for an application.

Interview Memory Trick
Git
 |
 v
Argo CD
 |
 | Kubernetes API
 |
 +-------> DEV
 |
 +-------> QA
 |
 +-------> PROD
Remember
Argo CD       = GitOps management
Git           = Desired state
Source        = Where FROM?
Destination   = Where TO?
Target Cluster= Runs application workloads
RBAC          = Controls Argo CD permissions
One-Line Summary
