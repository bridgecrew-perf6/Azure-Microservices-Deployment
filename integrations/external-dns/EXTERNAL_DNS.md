# DNS

## Prerequisites

```
DOMAIN_NAME=steadfast.kasunt.fe.gl00.net
AZ_DNS_GROUP=kasunt-sf-externaldns-rg
AZ_SP_NAME=kasunt-sf-externaldns-sp

DNS_RG_ID=$(az group create -n $AZ_DNS_GROUP \
  -l $AZ_REGION \
  --query id -o tsv
)

DNS_ID=$(az network dns zone create -n $DOMAIN_NAME \
  -g $AZ_DNS_GROUP \
  --query id -o tsv
)
```

At this point add the NS servers to Route53 (where the kasunt.fe.gl00.net is hosted - Route53)

```
az network dns zone show -n $DOMAIN_NAME \
  -g $AZ_DNS_GROUP \
  --query nameServers -o tsv
```

Create a new Service Principal

```
SP_CFG=$(az ad sp create-for-rbac -n $AZ_SP_NAME -o json)
SP_CLIENT_ID=$(echo $SP_CFG | jq -e -r 'select(.appId != null) | .appId')
SP_CLIENT_SECRET=$(echo $SP_CFG | jq -e -r 'select(.password != null) | .password')
SP_SUBSCRIPTIONID=$(az account show --query id -o tsv)
SP_TENANTID=$(echo $SP_CFG | jq -e -r 'select(.tenant != null) | .tenant')
```

Create DNS Zone Contributor role assignment

```
az role assignment create --assignee $SP_CLIENT_ID \
  --role "DNS Zone Contributor" \
  --scope $DNS_ID
```

Create Reader role assignment

```
az role assignment create --assignee $SP_CLIENT_ID \
  --role "Reader" \
  --scope $DNS_RG_ID
```

Inject the Azure authorization for External DNS

```
mkdir -p _output/integrations/external-dns
kubectl --context $MGMT_CONTEXT create ns external-dns
envsubst < integrations/external-dns/azure.json > _output/integrations/external-dns/azure.json
kubectl --context $MGMT_CONTEXT -n external-dns create secret generic azure-config-file --from-file=integrations/external-dns/azure.json
```


## Installation 

### On Management Cluster

Deploy with Helm

```
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update

helm install --kube-context=${MGMT_CONTEXT} \
  external-dns external-dns/external-dns \
  -n external-dns --create-namespace \
  -f integrations/external-dns/external-dns-helm-values.yaml
```

### On West cluster

```
helm install --kube-context=${WEST_CONTEXT} \
  external-dns external-dns/external-dns \
  -n external-dns --create-namespace \
  -f integrations/external-dns/external-dns-helm-values.yaml
```