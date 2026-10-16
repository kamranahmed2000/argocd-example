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
