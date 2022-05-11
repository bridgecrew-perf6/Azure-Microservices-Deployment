## Installation

### Setting up ArgoCD

```
# Add the argocd helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install the CLI
brew install argocd
```

## Configuration

### Install in Management cluster

```
helm install --kube-context=${MGMT_CONTEXT} argocd argo/argo-cd --namespace gitops --create-namespace -f integrations/gitops/argocd-helm-values.yaml
```

### Register remote clusters

```
argocd cluster add kasunt-sf-west-cluster
argocd cluster add kasunt-sf-east-cluster
```

### Setup Helm OCI Registry

```
envsubst < <(cat apps/argocd/argocd-azure-oci-repo-credentials.yaml) | kubectl apply --context $MGMT_CONTEXT -f -
envsubst < <(cat apps/argocd/argocd-microservices-project.yaml) | kubectl apply --context $MGMT_CONTEXT -f -
envsubst < <(cat apps/argocd/argocd-application-template-ci.yaml) | kubectl apply --context $MGMT_CONTEXT -f -
```