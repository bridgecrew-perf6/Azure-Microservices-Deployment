export CLUSTER_NAME=$EAST_CLUSTER_NAME
CLUSTER_CONTEXT=$EAST_CONTEXT

[Setting up CA for relay agent](RELAY_AGENT_CA.md)

```
kubectl apply --context $CLUSTER_CONTEXT -f- <<EOF
apiVersion: v1
data:
  token: NVVCUktXNWRoWnh5aGgwSw==
kind: Secret
metadata:
  name: relay-identity-token-secret
  namespace: gloo-mesh
type: Opaque
EOF
```
```
kubectl apply --context $CLUSTER_CONTEXT -f- <<EOF
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

## Setting up agent with Helm

```
MGMT_INGRESS_ADDRESS=$(kubectl get svc -n gloo-mesh enterprise-networking --context ${MGMT_CONTEXT} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
MGMT_INGRESS_PORT=$(kubectl -n gloo-mesh get service enterprise-networking --context ${MGMT_CONTEXT} -o jsonpath='{.spec.ports[?(@.name=="grpc")].port}')
export ENTERPRISE_NETWORKING_ADDRESS=${MGMT_INGRESS_ADDRESS}:${MGMT_INGRESS_PORT}

envsubst < gloo-mesh-agent.yaml > _output/helm-values/"gloo-mesh-${CLUSTER_NAME}-agent.yaml"

helm repo add gloo-mesh-enterprise-agent https://storage.googleapis.com/gloo-mesh-enterprise/enterprise-agent
helm repo update
helm install gloo-mesh-enterprise-agent gloo-mesh-enterprise-agent/enterprise-agent \
  --namespace gloo-mesh \
  --kube-context=${CLUSTER_CONTEXT} \
  --create-namespace \
  --values _output/helm-values/"gloo-mesh-${CLUSTER_NAME}-agent.yaml"

kubectl apply --context $MGMT_CONTEXT -f- <<EOF
apiVersion: multicluster.solo.io/v1alpha1
kind: KubernetesCluster
metadata:
  name: ${CLUSTER_NAME}
  namespace: gloo-mesh
spec:
  clusterDomain: cluster.local
EOF
```

## Setting up Gloo Mesh addons

```
helm install gloo-mesh-enterprise-agent-addons gloo-mesh-enterprise-agent/enterprise-agent \
   --namespace gloo-mesh-addons \
   --kube-context=${EAST_CONTEXT} \
   --create-namespace \
   --values gloo-mesh-agent-addons.yaml
```

## mTLS

```
kubectl apply --context $EAST_CONTEXT -f - << EOF
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "istio-config"
spec:
  mtls:
    mode: STRICT
EOF
```