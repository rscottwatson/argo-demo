docs: https://github.com/bitnami-labs/sealed-secrets#readme
github: https://github.com/bitnami-labs/sealed-secrets

Client Side
brew install kubeseal

# Creating a sealed secret
```bash
echo "apiVersion: v1
kind: Secret
metadata:
  name: github-token
  namespace: default
type: Opaque
data:
  token: $(echo -n $GITHUB_TOKEN | base64)" \
    | kubeseal --format yaml \
    | tee githubtoken.yaml
```


or

wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.16.0/kubeseal-linux-amd64 -O kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal



My initial attempt of using a custom namespace failed.

The controller failed to start and I got the following log messages

controller version: v0.16.0
2021/11/05 17:57:51 Starting sealed-secrets controller version: v0.16.0
2021/11/05 17:57:51 Searching for existing private keys
panic: secrets is forbidden: User "system:serviceaccount:sealed-secrets:sealed-secrets-controller" cannot list resource "secrets" in API group "" in the namespace "sealed-secrets" 
goroutine 1 [running]:
main.main()
    /home/travis/gopath/src/github.com/bitnami-labs/sealed-secrets/cmd/controller/main.go:271 +0x211

I think this may have been because my k8s version is 1.22.2 and the API version for RBAC is now
rbac.authorization.k8s.io/v1 
and the controller YAML file is using v1beta1.

So I created a patch in the kustomization file and that seems to be working.


With argocd you need to start the controller with the option
  --update-status
must be v15 or higher.

The other option is updating the configmap for argocd with

The config map needs the following for sealed secrets.
data:
  resource.customizations.health: |-
    bitnami.com_SealedSecret: |
        hs = {}
        hs.status = "Healthy"
        hs.message = "Controller doesn't report resource status"
        return hs

  https://argo-cd.readthedocs.io/en/stable/faq/#why-are-resources-of-type-sealedsecret-stuck-in-the-progressing-state