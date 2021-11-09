docs: https://argo-cd.readthedocs.io/en/stable/
github: https://github.com/argoproj/argo-cd/


Command line tool
argocd 

Commands
argocd account update-password --new-password xxxxxxxxx
( once updated you can delete the secret argocd/argocd-initial-admin-secret )


The config map needs the following for sealed secrets.
data:
  resource.customizations.health: |-
    bitnami.com_SealedSecret: |
        hs = {}
        hs.status = "Healthy"
        hs.message = "Controller doesn't report resource status"
        return hs

  https://argo-cd.readthedocs.io/en/stable/faq/#why-are-resources-of-type-sealedsecret-stuck-in-the-progressing-state