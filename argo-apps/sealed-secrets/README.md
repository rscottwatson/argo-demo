docs: https://github.com/bitnami-labs/sealed-secrets#readme
github: https://github.com/bitnami-labs/sealed-secrets

Client Side
brew install kubeseal

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