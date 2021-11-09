# My ARGO-DEMO repo
## Description
This repo is to play around and get familiar with the family or Argo products.  Initially it was to learn about Argo CD but I quickly realized there are lots of tools under the Argo umbrella and so I use this as a one stop shop to play with them all. 

Alot of this work came from watching videos from Viktor Farcic from the devopstoolkit.  
https://www.youtube.com/channel/UCfz8x0lVzJpb_dgWm9kPVrw

| App | Description |
|---| -------|
| argo cd | continuous delivery using gitops principles |
| cert-manager | |
| argo events | |
| argo image updater | |
| argo workflows | |
| sealed secrets | |


## Initial Setup

### Start minikube
minikube start --driver=hyperkit --addons="ingress"



FYI. Addon YAML files are copied into this directory /etc/kuberentes/addons/ingress-deploy.yaml
If you make any changes to them they are overwritten on each startup of the cluster.

### Run a sample deployment to see if things are working.


### Host machine is running dnscrypt-proxy or dnsmasq
If you are running these DNS services on your host machine it is likely that DNS won't work from your minikube cluster.

There is a simple workaround if you are using dnsmasq 
https://minikube.sigs.k8s.io/docs/drivers/hyperkit/#local-dns-server-conflict

The workaround for dnscrypt-proxy is a little more involved.  There could be a better solution so feel free to try other things.

In the end the easiest thing to do is change your dns resolver.
```bash
# minikube ssh 
#edit sudo vi /etc/resolv.conf and update nameserver 
nameserver 8.8.8.8 
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

I had to restart my coredns pod so you can do it by ( I should really test this again )
#### Option 1. while connected to minikube cluster/host
if you are still connected to minikube
```bash
export KUBECONFIG=/var/lib/minikube/kubeconfig 
/var/lib/minikube/binaries/*/kubectl rollout restart deployment coredns -n kube-system
```
#### Option 2. running kubectl commands from your host
```bash
kubectl rollout restart deployment coredns -n kube-system
```

The problem with the change to the /etc/resolv.conf is that the change it not permanent and won't persist if you stop your cluster or restart it.

To create a fix across reboots you can create the following script that will run each time your minikube cluster starts to set the DNS nameserver
```bash
minikube shh
sudo su -

cat << EOF >  /var/lib/boot2docker/bootlocal.sh
echo "8.8.8.8" >> /etc/systemd/resolved.conf 
systemctl restart systemd-resolved
EOF

chmod 755  /var/lib/boot2docker/bootlocal.sh
# does coredns have to be restarted too?
``` 

The file /var/lib/boot2docker/bootlocal.sh is persisted across reboots so you can add some customizations here. 

## Another way to run a command on startup of the minikube instance
minikube start && ssh -t -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip) "sudo mount --bind /hosthome/<user> /home/<user>"


Another option I came across that seemed intersting was to forward the DNS traffic using socat.  However, it didn't work for me but I will keep this here as I think it is an interesting idea.

This doesn't seem to work!!!
sudo socat UDP4-RECVFROM:53,bind=192.168.64.1,fork UDP4-SENDTO:127.0.0.1:53

NOTE: 192.168.64.1 is the default bridge that minikube creates however, on my office laptop this bridge was 192.168.78.1 and not 64.



## Getting Ingress working

I wanted to have my service avaialable on the web just for testing and to do that I needed to forwarding traffic from my router to my minikube cluster

### Option 1 Using SOCAT to pass the traffic

Since I am using my home network I need to forward from my router to my computer.
Then from my computer I need to forward the traffic the minikube cluster on the ports that my nginx-ingress is running on.

Needed to look up the IP of the minikube host as well as the node ports used for the ingress controller.

socat tcp-listen:8443,reuseaddr,fork tcp:192.168.78.9:32480  &
socat tcp-listen:9080,reuseaddr,fork tcp:192.168.78.9:31706  &

### Option 2 Using a static route
In my router network configuration page I setup a static route that uses my PC as the gatway to access my minikube cluster.

Didn't get this to work as my router wanted a host in the same subnet.


### Option 3 Use kubectl port forward
```bash
kubectl port-forward --address 192.168.78.10 service/nginx-deployment 8000:80 &
kubectl port-forward --address 192.168.78.10 service/nginx-deployment 8443:443 &
```

### Option 4 Is the load balancer working?
## Argo CD Install
With this repo we are going to use ArgoCD to manage ArgoCD.
This is currently using the App or Apps deployment however, there is a newer model called [application sets][argoas] which I will explore in the future.


Get the default password from 

kubectl get secret -n argocd argocd-initial-admin-secret  -o jsonpath={.data.password} | base64 -d

### Configure ingress controller
Once you enable the ingres controller with the addon
you need to patch it so that argocd can use the tls certificate.
src: https://github.com/kubernetes/minikube/issues/6403#issuecomment-888754010

```bash
minikube kubectl -- patch deployment -n ingress-nginx ingress-nginx-controller -p='{"spec":{"template":{"spec":{"containers":[{"name":"controller","args":["/nginx-ingress-controller","--ingress-class=nginx","--configmap=$(POD_NAMESPACE)/ingress-nginx-controller","--report-node-internal-ip-address","--tcp-services-configmap=$(POD_NAMESPACE)/tcp-services","--udp-services-configmap=$(POD_NAMESPACE)/udp-services","--validating-webhook=:8443","--validating-webhook-certificate=/usr/local/certificates/cert","--validating-webhook-key=/usr/local/certificates/key","--enable-ssl-passthrough"]}]}}}}'

# it might just be easier to edit the deployment and add 
kubectl edit deployment -n ingress-nginx ingress-nginx-controller
--enable-ssl-passthrough
# to the args list
```


## Tools
Some of the tools I have come across are 

| Name | Type | Description |
|--------- | ------------ | ------------------------|
| envsubst | template | Allows you to create a template and replace variable wth ENV Vars |
| kyml | template | Similar to envsubst |

## Debugging

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


Cert-Manager Kustomize demo
  https://www.jetstack.io/blog/kustomize-cert-manager/
  https://github.com/jetstack/kustomize-cert-manager-demo


## Sample apps to test deployments.

This is from the book Kubernetes Up and Running.
https://github.com/kubernetes-up-and-running/kuard


## My own Q&A
| Question | Answer |
| --- | ---------------------------------------------- |
| which namespace should the argo app be created in? |
| 

## Links
[argoas]: https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/ "Learn about application sets"

