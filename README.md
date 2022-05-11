# Gloo Mesh 1.2.x PoC on Azure (WSL)

PoC for Steadfast.

This provisions Gloo Mesh 1.2.11 on 3 different Azure clusters using Istio 1.12.2.
Deploys various services to support running [demo microservices](https://github.com/pseudonator/Azure-Microservices).

<img src="https://user-images.githubusercontent.com/2648624/167958044-3d332c64-18f5-4fa2-9a29-5b8bac46fa79.png" width=50% height=50%>

## Prerequisites

After `az login` you can use the following command to get `subscription-id` and `tenant-id`.

```
az account list \
  --query "[].{Name:name,TenantID:tenantId,SubscriptionID:id}" \
  -o table

# Following assumes the default subscription
AZ_SUBSCRIPTION_ID=$(az account show --query id --output tsv)
AZ_TENANT_ID=$(az account show --query tenantId --output tsv)
AZ_REGION=australiaeast
```

1. Create a WSL environment (Optional)
This first step is optional. Only required to prove this works in WSL.
Use VMware Fusion for enabling Windows virtualization on Mac OS X.

    ```
    wsl --install -d Ubuntu-20.04
    ```
    - Add `<username>` to sudoers
    - Add `HOME` variable pointing to `/home/<username>`

2. Install Azure CLI on WSL VM and configure (Optional)
    ```
    # Login with Azure credentials
    az login
    # Set subscription
    az account set --subscription $AZ_SUBSCRIPTION_ID
    # Install Kubectl
    sudo az aks install-cli
    ```

3. Install the following tools;
    * Helm v3
    * `istioctl` 1.12
    * `meshctl` 1.2.x

4. Create multi clusters on Azure
Note that these are created on public subnets and not using best practices for prod clusters

    ```
    az group create --name kasunt-sf-west-cluster-rg --location $AZ_REGION
    az group create --name kasunt-sf-east-cluster-rg --location $AZ_REGION
    az group create --name kasunt-sf-mgmt-cluster-rg --location $AZ_REGION

    az aks create --resource-group kasunt-sf-west-cluster-rg --name kasunt-sf-west-cluster --node-count 3
    az aks create --resource-group kasunt-sf-east-cluster-rg --name kasunt-sf-east-cluster --node-count 3
    az aks create --resource-group kasunt-sf-mgmt-cluster-rg --name kasunt-sf-mgmt-cluster --node-count 3

    az aks get-credentials --resource-group kasunt-sf-west-cluster-rg --name kasunt-sf-west-cluster
    az aks get-credentials --resource-group kasunt-sf-east-cluster-rg --name kasunt-sf-east-cluster
    az aks get-credentials --resource-group kasunt-sf-mgmt-cluster-rg --name kasunt-sf-mgmt-cluster
    ```

5. Attach ACR to AKS for docker registry

    ```
    Didnt work
    #az aks update -n kasunt-sf-west-cluster -g kasunt-sf-west-cluster-rg --attach-acr kasuntimagesacr

    ACR_NAME=kasuntimagesacr
    SERVICE_PRINCIPAL_NAME=kasunt-sf-acr-service-principal

    # Obtain the full registry ID for subsequent command args
    ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query "id" --output tsv)

    PASSWORD=$(az ad sp create-for-rbac --name $SERVICE_PRINCIPAL_NAME --scopes $ACR_REGISTRY_ID --role acrpull --query "password" --output tsv)
    USER_NAME=$(az ad sp list --display-name $SERVICE_PRINCIPAL_NAME --query "[].appId" --output tsv)

    kubectl --context=${WEST_CONTEXT} -n apps create secret docker-registry acr-reg-pull-secret --docker-server=kasuntimagesacr.azurecr.io --docker-username=$USER_NAME --docker-password=$PASSWORD
    ```

6. Set up environment variables

    ```
    export GLOO_MESH_LICENSE_KEY=<license>
    export EAST_CONTEXT="kasunt-sf-east-cluster"
    export WEST_CONTEXT="kasunt-sf-west-cluster"
    export MGMT_CONTEXT="kasunt-sf-mgmt-cluster"
    export EAST_CLUSTER_NAME="east-cluster"
    export WEST_CLUSTER_NAME="west-cluster"
    export MGMT_CLUSTER_NAME="mgmt-cluster"

    export GLOO_MESH_VERSION=v1.2.11
    export REPO=<private repo url>
    export ISTIO_VERSION=1.12.2-solo
    export REVISION=1-12-2
    ```

7. Create dirs for generated files

    ```
    mkdir -p _output/pki
    mkdir -p _output/helm-values
    ```

## Instructions

### Setting up Istio

[West Cluster Setup](ISTIO_WEST_CLUSTER.md)

[East Cluster Setup](ISTIO_EAST_CLUSTER.md)

### Gloo Mesh relay server

[Setting up relay certificate](RELAY_SERVER_CA.md)

```
kubectl apply --context $MGMT_CONTEXT -f- <<EOF
apiVersion: v1
data:
  token: NVVCUktXNWRoWnh5aGgwSw==
kind: Secret
metadata:
  name: relay-identity-token-secret
  namespace: gloo-mesh
type: Opaque
EOF
kubectl apply --context $MGMT_CONTEXT -f- <<EOF
apiVersion: v1
data:
  token: Yk13bjhYaGg1cHk3a2ZDMg==
kind: Secret
metadata:
  name: relay-forwarding-identity-token-secret
  namespace: gloo-mesh
type: Opaque
EOF
```

### Deploy Gloo Management plane

```
helm install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise --namespace gloo-mesh \
  --set licenseKey=${GLOO_MESH_LICENSE_KEY} \
  --kube-context=${MGMT_CONTEXT} \
  --create-namespace \
  --values gloo-mesh-mgmt-plane.yaml
```

### Gloo Mesh relay agent

#### West cluster

[Setting up relay agent](GLOO_MESH_WEST_CLUSTER_RELAY_AGENT.md)

#### East cluster

[Setting up relay agent](GLOO_MESH_EAST_CLUSTER_RELAY_AGENT.md)

### Register the clusters with Virtual Mesh

```
kubectl apply --context $MGMT_CONTEXT -f - << EOF
apiVersion: networking.mesh.gloo.solo.io/v1
kind: VirtualMesh
metadata:
  name: virtual-mesh
  namespace: gloo-mesh
spec:
  mtlsConfig:
    autoRestartPods: true
    shared:
      rootCertificateAuthority:
        generated: {}
  federation:
    ingressGatewaySelectors:
      - portName: tls
        destinationSelectors:
          - kubeServiceMatcher:
              clusters:
                - ${WEST_CLUSTER_NAME}
                - ${EAST_CLUSTER_NAME}
              labels:
                istio: eastwestgateway-lb
              namespaces:
                - istio-eastwest
  meshes:
  - name: istiod-${REVISION}-istio-system-${WEST_CLUSTER_NAME}
    namespace: gloo-mesh
  - name: istiod-${REVISION}-istio-system-${EAST_CLUSTER_NAME}
    namespace: gloo-mesh
EOF
```

## Deploy GM & App Configuration

Currently all the artifacts are in `apps/configuration` dir.
