# Cert Manager

## Prerequisites

```
AZ_CERT_MANAGER_SP_NAME=kasunt-sf-certmanager-sp
AZ_DNS_ZONE_RESOURCE_GROUP=kasunt-sf-externaldns-rg
AZ_DNS_ZONE=steadfast.kasunt.fe.gl00.net

DNS_SP=$(az ad sp create-for-rbac --name $AZ_CERT_MANAGER_SP_NAME --output json)
AZ_CERT_MANAGER_SP_APP_ID=$(echo $DNS_SP | jq -r '.appId')
AZ_CERT_MANAGER_SP_PASSWORD=$(echo $DNS_SP | jq -r '.password')
AZ_TENANT_ID=$(echo $DNS_SP | jq -r '.tenant')
AZ_SUBSCRIPTION_ID=$(az account show --output json | jq -r '.id')

az role assignment delete --assignee $AZ_CERT_MANAGER_SP_APP_ID --role Contributor

DNS_ID=$(az network dns zone show --name $AZ_DNS_ZONE --resource-group $AZ_DNS_ZONE_RESOURCE_GROUP --query "id" --output tsv)
az role assignment create --assignee $AZ_CERT_MANAGER_SP_APP_ID --role "DNS Zone Contributor" --scope $DNS_ID

kubectl --context=${WEST_CONTEXT} create secret -n cert-manager generic az-cert-manager-config --from-literal=client-secret=$AZ_CERT_MANAGER_SP_PASSWORD
kubectl --context=${EAST_CONTEXT} create secret -n cert-manager generic az-cert-manager-config --from-literal=client-secret=$AZ_CERT_MANAGER_SP_PASSWORD
```

## Installation

```
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install --kube-context=${WEST_CONTEXT} \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.7.1 \
  -f integrations/cert-manager/cert-manager-helm-values.yaml
```

### Issuer Configuration in both clusters (Cluster wide)

```
kubectl --context=${EAST_CONTEXT} apply -n cert-manager -f- <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: global-cert-issuer
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    preferredChain: "ISRG Root X1"
    email: kasun.talwatta@solo.io
    privateKeySecretRef:
      name: global-cert-issuer
    solvers:
    - dns01:
        azureDNS:
          clientID: $AZ_CERT_MANAGER_SP_APP_ID
          clientSecretSecretRef:
            name: az-cert-manager-config
            key: client-secret
          subscriptionID: $AZ_SUBSCRIPTION_ID
          tenantID: $AZ_TENANT_ID
          resourceGroupName: $AZ_DNS_ZONE_RESOURCE_GROUP
          hostedZoneName: $AZ_DNS_ZONE
          # Azure Cloud Environment, default to AzurePublicCloud
          environment: AzurePublicCloud
EOF

kubectl --context=${WEST_CONTEXT} apply -n cert-manager -f- <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: global-cert-issuer
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    preferredChain: "ISRG Root X1"
    email: kasun.talwatta@solo.io
    privateKeySecretRef:
      name: global-cert-issuer
    solvers:
    - dns01:
        azureDNS:
          clientID: $AZ_CERT_MANAGER_SP_APP_ID
          clientSecretSecretRef:
            name: az-cert-manager-config
            key: client-secret
          subscriptionID: $AZ_SUBSCRIPTION_ID
          tenantID: $AZ_TENANT_ID
          resourceGroupName: $AZ_DNS_ZONE_RESOURCE_GROUP
          hostedZoneName: $AZ_DNS_ZONE
          # Azure Cloud Environment, default to AzurePublicCloud
          environment: AzurePublicCloud
EOF
```