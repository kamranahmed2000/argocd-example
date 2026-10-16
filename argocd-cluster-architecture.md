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

     
