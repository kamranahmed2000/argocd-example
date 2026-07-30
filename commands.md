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


