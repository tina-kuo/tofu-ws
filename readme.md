# Architecture

GitHub Repo
│
├── clusters/minikube
│   ├── gotk-components.yaml
│   ├── gotk-sync.yaml
│   ├── tofu-controller.yaml
│   └── terraform.yaml
│
└── infra
    └── main.tf

Git
 ↓
Flux
 ↓
tofu-controller
 ↓
OpenTofu
 ↓
AWS IAM Role

## Create cluster
````
minikube start --wait=all
minikube update-context
kubectl cluster-info # Verify control plane up and running
````

## Create GitHub repo
```
gh repo create <OWNER>/<REPO> --public
git clone git@github.com:<OWNER>/<REPO>.git
cd <REPO>
git remote -v # Verify origin
```

## Flux Bootstrap
```
flux bootstrap git \
  --url=ssh://git@github.com/<OWNER>/<REPO> \
  --branch=main \
  --private-key-file=<PRIVATE_KEY_PATH> \
  --path=./clusters/minikube

kubectl get pods -n flux-system # Verify source-controller, kustomize-controller, helm-controller, notification-controller running
flux get all # Verify GitRepository, Kustomization ready
```

## Install tofu-controller
```
mkdir -p clusters/minikube && touch clusters/minikube/tofu-controller.yaml
```
```
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: tofu-controller
  namespace: flux-system
spec:
  interval: 1h
  url: https://flux-iac.github.io/tofu-controller

---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: tofu-controller
  namespace: flux-system
spec:
  interval: 10m
  releaseName: tofu-controller
  targetNamespace: flux-system
  chart:
    spec:
      chart: tofu-controller
      version: ">=0.0.0"
      sourceRef:
        kind: HelmRepository
        name: tofu-controller
        namespace: flux-system
      interval: 1m
```
```
git add .
git commit -m "creating tofu controller"
git push
flux reconcile source git flux-system # Force reconcile
flux reconcile kustomization flux-system
flux get helmreleases -A # Verify tofu-controller created and ready0
```
