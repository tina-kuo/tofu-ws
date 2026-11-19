```
Git
 ↓
Flux
 ↓
tofu-controller
 ↓
OpenTofu
 ↓
AWS IAM Role
```
## Repository Structure
```
.
├── clusters
│   └── minikube
│       ├── flux-system
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── terraform.yaml
│       └── tofu-controller.yaml
├── infra
│   └── main.tf
└── README.md
```

## Create local cluster
````
minikube start --wait=all
minikube update-context

# Verify Minikube context
kubectl config current-context

# Verify control plane up and running
kubectl cluster-info 
````

## Create GitHub repo
```
gh repo create <OWNER>/<REPO> --public
git clone git@github.com:<OWNER>/<REPO>.git
cd <REPO>

# Verify origin
git remote -v 
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

# Verify tofu-controller created and ready
flux get helmreleases -A 

# Verify CRDs terraforms.infra.contrib.fluxcd.io
kubectl get crds | grep terraform 
```

## Add OpenTofu code and a Terraform resource
```
mkdir -p infra && touch infra/main.tf
```
```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

variable "aws_region" {
  type = string
}

provider "aws" {
  region = var.aws_region
}

data "aws_iam_policy_document" "this" {
  statement {
    actions = ["sts:AssumeRole", "sts:TagSession"]

    principals {
      type        = "Service"
      identifiers = ["pods.eks.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "this" {
  name               = "tofu-ctrl-test"
  assume_role_policy = data.aws_iam_policy_document.this.json
}
```
### Deploy an AWS secret using you local capability credentials (required to already be logged in with go-aws-sso)
```
aws sts get-caller-identity

aws_creds="$(aws configure export-credentials --format env-no-export)"

printf '%s\n' "$aws_creds" |
  kubectl create secret generic aws-credentials \
    --namespace flux-system  \
    --from-env-file=/dev/stdin \
    --dry-run=client -o yaml | kubectl apply -f -
```
### Add Terraform resource
```
touch clusters/minikube/terraform.yaml
```
```
apiVersion: infra.contrib.fluxcd.io/v1alpha2
kind: Terraform
metadata:
  name: test
  namespace: flux-system
spec:
  approvePlan: "auto"
  interval: 2m
  destroyResourcesOnDeletion: true
  storeReadablePlan: human
  path: ./infra
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  vars:
  - name: aws_region
    value: "eu-west-1"
  runnerPodTemplate:
    spec:
      envFrom:
        - secretRef:
            name: aws-credentials
```
```
git add .
git commit -m "Add OpenTofu code and a Terraform resource"
git push

# Force reconcile
flux reconcile source git flux-system 
flux reconcile kustomization flux-system
kubectl get terraform -n flux-system 
# Check Terraform resource
```

### Verify AWS change
```
aws iam get-role --role-name tofu-ctrl-test
```
### Check where state is stored
```
kubectl get secrets -n flux-system
```

## Cleanup
```
# Delete the Terraform custom resource
mv clusters/minikube/terraform.yaml clusters/minikube/terraform.yaml.bk
git add .
git commit -m "Cleanup workshop"
git push

flux reconcile source git flux-system
flux reconcile kustomization flux-system

# Wait for deletion to finish
kubectl get terraform -n flux-system -w
kubectl get pods -n flux-system

# Verify AWS resource was destroyed
aws iam get-role --role-name tofu-ctrl-test
# Should return NoSuchEntity or similar error

# Remove tofu-controller
mv clusters/minikube/tofu-controller.yaml clusters/minikube/tofu-controller.yaml.bk

git add .
git commit -m "Remove tofu-controller"
git push

flux reconcile source git flux-system
flux reconcile kustomization flux-system

# Verify HelmRelease is gone
flux get helmreleases -A

# Verify Terraform CRD is gone (optional)
kubectl get crds | grep infra.contrib
```
### Remove Flux
```
flux uninstall
kubectl get ns flux-system
```
### Remove Minikube
```
minikube delete
minikube status
```