Argo CD Multi-Cluster Architecture

Argo CD is installed in a management cluster and can manage multiple registered Kubernetes clusters.

Architecture

                 Git Repository
                       |
                       v
              +------------------+
              |     Argo CD      |
              | Management Cluster|
              +--------+---------+
                       |
                Kubernetes API
                       |
          +------------+------------+
          |            |            |
          v            v            v
     DEV Cluster   QA Cluster   PROD Cluster

---

Does every cluster need Argo CD?

No. A full Argo CD installation is not required on every target cluster.

The target clusters are registered with Argo CD.

argocd cluster add <context-name>
``

---
