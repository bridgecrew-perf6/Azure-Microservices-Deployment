# Azure Gateway Controller

## Prerequisites

```
AZ_REGION=australiaeast
AZ_MGMT_CLUSTER_NAME=kasunt-sf-mgmt-cluster
AZ_MGMT_CLUSTER_RG=kasunt-sf-mgmt-cluster-rg
AZ_GW_CONTROLLER_NAME=kasunt-sf-mgmt-gw-controller
AZ_GW_CONTROLLER_SP=kasunt-sf-mgmt-gw-controller-sp

az network public-ip create -n $AZ_GW_CONTROLLER_NAME -g $AZ_MGMT_CLUSTER_RG --allocation-method Static --sku Standard
az network vnet create -n $AZ_GW_CONTROLLER_NAME -g $AZ_MGMT_CLUSTER_RG --address-prefix 11.0.0.0/8 --subnet-name $AZ_GW_CONTROLLER_NAME --subnet-prefix 11.1.0.0/16 
az network application-gateway create -n $AZ_GW_CONTROLLER_NAME -l $AZ_REGION -g $AZ_MGMT_CLUSTER_RG --sku Standard_v2 --public-ip-address $AZ_GW_CONTROLLER_NAME --vnet-name $AZ_GW_CONTROLLER_NAME --subnet $AZ_GW_CONTROLLER_NAME

AZ_APP_GW_ID=$(az network application-gateway show -n $AZ_GW_CONTROLLER_NAME -g $AZ_MGMT_CLUSTER_RG -o tsv --query "id") 
az aks enable-addons -n $AZ_MGMT_CLUSTER_NAME -g $AZ_MGMT_CLUSTER_RG -a ingress-appgw --appgw-id $AZ_APP_GW_ID

AZ_NODE_RESOURCE_GROUP=$(az aks show -n $AZ_MGMT_CLUSTER_NAME -g $AZ_MGMT_CLUSTER_RG -o tsv --query "nodeResourceGroup")
AZ_RT_TABLE_ID=$(az network route-table list -g $AZ_NODE_RESOURCE_GROUP --query "[].id | [0]" -o tsv)

# Get the application gateway's subnet
AZ_APP_GW_SUBNET_ID=$(az network application-gateway show -n $AZ_GW_CONTROLLER_NAME -g $AZ_MGMT_CLUSTER_RG -o tsv --query "gatewayIpConfigurations[0].subnet.id")

# Associate the route table to Application Gateway's subnet
az network vnet subnet update \
--ids $AZ_APP_GW_SUBNET_ID \
--route-table $AZ_RT_TABLE_ID

AZ_VNET_PEERING_GW_TO_AKS=kasunt-sf-mgmt-vnet-peering-gw-aks
AZ_VNET_PEERING_AKS_TO_GW=kasunt-sf-mgmt-vnet-peering-aks-gw

AZ_VNET_NAME=$(az network vnet list -g $AZ_NODE_RESOURCE_GROUP -o tsv --query "[0].name")
AZ_VNET_ID=$(az network vnet show -n $AZ_VNET_NAME -g $AZ_NODE_RESOURCE_GROUP -o tsv --query "id")
az network vnet peering create -n $AZ_VNET_PEERING_GW_TO_AKS -g $AZ_MGMT_CLUSTER_RG --vnet-name $AZ_GW_CONTROLLER_NAME --remote-vnet $AZ_VNET_ID --allow-vnet-access

AZ_GW_VNET_ID=$(az network vnet show -n $AZ_GW_CONTROLLER_NAME -g $AZ_MGMT_CLUSTER_RG -o tsv --query "id")
az network vnet peering create -n $AZ_VNET_PEERING_AKS_TO_GW -g $AZ_NODE_RESOURCE_GROUP --vnet-name $AZ_VNET_NAME --remote-vnet $AZ_GW_VNET_ID --allow-vnet-access

SP_CFG=$(az ad sp create-for-rbac -n $AZ_GW_CONTROLLER_SP -o json)
SP_CLIENT_ID=$(echo $SP_CFG | jq -e -r 'select(.appId != null) | .appId')
SP_CLIENT_SECRET=$(echo $SP_CFG | jq -e -r 'select(.password != null) | .password')
SP_OBJECT_ID=$(az ad sp show --id $SP_CLIENT_ID --query "objectId" -o tsv)
SP_SUBSCRIPTIONID=$(az account show --query id -o tsv)
SP_TENANTID=$(echo $SP_CFG | jq -e -r 'select(.tenant != null) | .tenant')

mkdir -p _output/integrations/gw-controller
cat <<EOF > _output/integrations/gw-controller/parameters.json
{
  "clientId": "$SP_CLIENT_ID",
  "clientSecret": "$SP_CLIENT_SECRET",
  "subscriptionId": "$SP_SUBSCRIPTIONID",
  "tenantId": "$SP_TENANTID",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
EOF

BASE64_ENCODED_GW_PARAMETERS=(cat _output/integrations/gw-controller/parameters.json | base64 -w0)

AZ_RG_ID=$(az group show -n $AZ_MGMT_CLUSTER_RG \
  --query id -o tsv
)

az role assignment create --role Reader --scope $AZ_RG_ID --assignee $SP_CLIENT_ID

az role assignment create --role Contributor --scope $AZ_APP_GW_ID --assignee $SP_CLIENT_ID
```

## Helm install

```
helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update

envsubst < integrations/azure-gateway-controller/azure-gateway-helm-values.yaml > _output/integrations/gw-controller/azure-gateway-helm-values.yaml

helm install --kube-context=${MGMT_CONTEXT} ingress-azure application-gateway-kubernetes-ingress/ingress-azure -n azure-gw --create-namespace -f _output/integrations/gw-controller/azure-gateway-helm-values.yaml
```