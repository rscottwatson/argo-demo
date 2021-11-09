#
Image Updater will go into the argocd namespace

Using a dedicated namespace can be more difficult to get the setup right. Therfore for the sake of simplicity the deployment will go into the same namespace.


This is referencing the latest releast but eventually this should be tied to an actual release version so that upgrades can be handeled properly.

Docs: https://argocd-image-updater.readthedocs.io/en/stable/
github: https://github.com/argoproj-labs/argocd-image-updater