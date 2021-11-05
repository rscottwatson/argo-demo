# My ARGO-DEMO repo
Experiment with argo and various tools.

minikube start --driver=hyperkit --addons="ingress"

Once you enable the ingres controller with the addon
you need to patch it so that argocd can use the tls certificate.
src: https://github.com/kubernetes/minikube/issues/6403#issuecomment-888754010

minikube kubectl -- patch deployment -n ingress-nginx ingress-nginx-controller -p='{"spec":{"template":{"spec":{"containers":[{"name":"controller","args":["/nginx-ingress-controller","--ingress-class=nginx","--configmap=$(POD_NAMESPACE)/ingress-nginx-controller","--report-node-internal-ip-address","--tcp-services-configmap=$(POD_NAMESPACE)/tcp-services","--udp-services-configmap=$(POD_NAMESPACE)/udp-services","--validating-webhook=:8443","--validating-webhook-certificate=/usr/local/certificates/cert","--validating-webhook-key=/usr/local/certificates/key","--enable-ssl-passthrough"]}]}}}}'

Looks like this get changed on each restart of the minikube instance and the change it not persisted.
So just edit the deployment and add the option
--enable-ssl-passthrough
to the arguments of the deployment ingress-nginx-controller
in the file /etc/kuberentes/addons/ingress-deploy.yaml

you can also just edit the deployment and change the args list

Get the default password from 

kubectl get secret -n argocd argocd-initial-admin-secret  -o jsonpath={.data.password} | base64 -d

Issues with minikube on work laptop.
Network manager kept trying to revert my /etc/resovl.conf to something that didn't allow me to resolv names with as my actual dns server is dnscrypt-proxy and that won't forward requests.


So change /etc/resolv.conf to 8.8.8.8 and then
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

I had to restart my codedns deployment
kubectl rollout restart deployment coredns -n kube-system
I also restarted the kubelet for good measure but I am not actually sure I had to do this.
sudo systemctl restart kubelet


Another solution
sudo su -
cat << EOF >  /var/lib/boot2docker/bootlocal.sh
echo "8.8.8.8" >> /etc/systemd/resolved.conf 
systemctl restart systemd-resolved
EOF
chmod 755  /var/lib/boot2docker/bootlocal.sh

Need to see if I need to restart coredns.

# Another way to run a command on startup of the minikube instance
minikube start && ssh -t -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip) "sudo mount --bind /hosthome/<user> /home/<user>"

# TO run kubectl command while ssh'd into the minikube VM
export KUBECONFIG=/var/lib/minikube/kubeconfig 
/var/lib/minikube/binaries/v1.22.2/kubectl apply -f /etc/kubernetes/addons/ingress-deploy.yaml

If you are using dnsmasq you can fix the problem by adding a listen-address to your dnsmasq.conf file
https://minikube.sigs.k8s.io/docs/drivers/hyperkit/#local-dns-server-conflict

This doesn't seem to work!!!
Another option is to use socat like this
sudo socat UDP4-RECVFROM:53,bind=192.168.64.1,fork UDP4-SENDTO:127.0.0.1:53
( 192.168.64.1 is the bridge that minikube is using not sure how to find out which bridge this is 
 on my office laptop this bridge was 192.168.78.1 )

 sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.mDNSresponder.plist
 sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.mDNSresponder.plist

unload and load are deprecated should use enable or disable 


Since I am using my home network I need to forward from my router to my computer.
The from my computer I need to forward the traffic the minikube cluster on the ports that my
nginx-ingress is running on.

socat tcp-listen:8443,reuseaddr,fork tcp:192.168.78.9:32480  &
socat tcp-listen:9080,reuseaddr,fork tcp:192.168.78.9:31706  &

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