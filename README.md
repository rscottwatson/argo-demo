# argo-demo
Experiment with argo and various tools.


Get the default password from 

kubectl get secret -n argocd argocd-initial-admin-secret  -o jsonpath={.data.password} | base64 -d

Issues with minikube on work laptop.
Network manager kept trying to revert my /etc/resovl.conf to something that didn't allow me to resolv names with as my actual dns server is dnscrypt-proxy and that won't forward requests.

So change /etc/resolv.conf to 8.8.8.8 and then

sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved