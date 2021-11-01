# argo-demo
Experiment with argo and various tools.


Get the default password from 

kubectl get secret -n argocd argocd-initial-admin-secret  -o jsonpath={.data.password} | base64 -d

Issues with minikube on work laptop.
Network manager kept trying to revert my /etc/resovl.conf to something that didn't allow me to resolv names with as my actual dns server is dnscrypt-proxy and that won't forward requests.

So change /etc/resolv.conf to 8.8.8.8 and then

sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

I was unable to resolve a hostname from a pod.  So I created a busybox pod to test.

apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "99999"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always

  Then ran
  k exec -ti -n default pod/busybox -- nslookup github.com

This showed it was not working so I killed my coredns pod and once that restarted then things started to work again.  I am guessing the core-dns pod got it's config from the old /etc/resolv.conf file.