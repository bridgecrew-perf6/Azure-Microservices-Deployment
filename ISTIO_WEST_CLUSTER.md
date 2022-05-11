Create namespaces and inject istio-operator config for control plane
```
kubectl config use-context $WEST_CONTEXT

kubectl create namespace istio-system 
kubectl create namespace istio-ingress
kubectl create namespace istio-eastwest
kubectl create namespace istio-config

istioctl operator init --watchedNamespaces=istio-system,istio-ingress,istio-eastwest

CLUSTER_NAME=$WEST_CLUSTER_NAME

cat <<EOF | istioctl --context ${WEST_CONTEXT} install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: demo-istio
  namespace: istio-system
spec:
  profile: minimal
  # This value is required for Gloo Mesh Istio
  hub: ${REPO}
  # This value can be any Gloo Mesh Istio tag
  tag: ${ISTIO_VERSION}
  revision: ${REVISION}

  # You may override parts of meshconfig by uncommenting the following lines.
  meshConfig:
    # enable access logging. Empty value disables access logging.
    accessLogFile: /dev/stdout
    # Encoding for the proxy access log (TEXT or JSON). Default value is TEXT.
    accessLogEncoding: JSON
    accessLogFormat: '{
      "start_time": "%START_TIME%",
      "authority": "%REQ(:AUTHORITY)%",
      "bytes_received": "%BYTES_RECEIVED%",
      "bytes_sent": "%BYTES_SENT%",
      "downstream_local_address": "%DOWNSTREAM_LOCAL_ADDRESS%",
      "downstream_remote_address": "%DOWNSTREAM_REMOTE_ADDRESS%",
      "duration": "%DURATION%",
      "method": "%REQ(:METHOD)%",
      "path": "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%",
      "protocol": "%PROTOCOL%",
      "requested_server_name": "%REQUESTED_SERVER_NAME%",
      "response_code": "%RESPONSE_CODE%",
      "response_flags": "%RESPONSE_FLAGS%",
      "route_name": "%ROUTE_NAME%",
      "upstream_cluster": "%UPSTREAM_CLUSTER%",
      "upstream_host": "%UPSTREAM_HOST%",
      "upstream_local_address": "%UPSTREAM_LOCAL_ADDRESS%",
      "upstream_service_time": "%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%",
      "upstream_transport_failure_reason": "%UPSTREAM_TRANSPORT_FAILURE_REASON%",
      "user_agent": "%REQ(USER-AGENT)%",
      "x_forwarded_for": "%REQ(X-FORWARDED-FOR)%",
      "x_user_id": "%REQ(X-USER-ID)%", 
      "x_tenant_id": "%REQ(X-TENANT-ID)%", 
      "baggage": "%REQ(BAGGAGE)%"
    }'

    enableTracing: true

    extensionProviders:
      - name: "opencensus-tracing"
        opencensus:
          context: ["W3C_TRACE_CONTEXT"]
          service: "oc-opentelemetry-collector.telemetry.svc"
          port: 55678
    defaultProviders:
      tracing: ["opencensus-tracing"]

    defaultConfig:
      # wait for the istio-proxy to start before application pods
      holdApplicationUntilProxyStarts: true
      # location of istiod service
      # discoveryAddress: istiod-${REVISION}.istio-system.svc:15012
      # enable GlooMesh metrics service
      envoyMetricsService:
        address: enterprise-agent.gloo-mesh:9977
      # enable GlooMesh accesslog service
      envoyAccessLogService:
        address: enterprise-agent.gloo-mesh:9977
      # Tracing with opencensus. 
      # TODO: This is a temp solution until opentelemetry is introduced in Istio/Envoy
      # tracing:
        # sample 100% of traffic
        # sampling: 100.0
        # openCensusAgent:
        #  address: "dns:oc-opentelemetry-collector.telemetry.svc:55678"
        #  context: [W3C_TRACE_CONTEXT]
      proxyMetadata:
        # Enable Istio agent to handle DNS requests for known hosts
        # Unknown hosts will automatically be resolved using upstream dns servers in resolv.conf
        ISTIO_META_DNS_CAPTURE: "true"
        # Enable automatic address allocation, optional
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
        # Used for gloo mesh metrics aggregation
        # should match trustDomain
        GLOO_MESH_CLUSTER_NAME: ${CLUSTER_NAME}
    
    # Specify if http1.1 connections should be upgraded to http2 by default. 
    # Can be overridden using DestinationRule
    # h2UpgradePolicy: UPGRADE

    # Set the default behavior of the sidecar for handling outbound traffic from the application.
    outboundTrafficPolicy:
      mode: ALLOW_ANY
    # The trust domain corresponds to the trust root of a system. For Gloo Mesh this should be the name of the cluster that cooresponds with the CA certificate CommonName identity
    trustDomain: ${CLUSTER_NAME}.solo.io
    # The namespace to treat as the administrative root namespace for Istio configuration.
    rootNamespace: istio-config

  # Traffic management feature
  components:
    base:
      enabled: true
    pilot:
      enabled: true
      k8s:
        replicaCount: 1
        resources:
          requests:
            cpu: 200m
            memory: 200Mi
        env:
        # Allow multiple trust domains (Required for Gloo Mesh east/west routing)
        - name: PILOT_SKIP_VALIDATE_TRUST_DOMAIN
          value: "true"

    # Disable Istio CNI feature
    cni:
      enabled: false

    # Istio Gateway featureß
    # Disable gateways deployments because they will be in separate IstioOperator configs
    ingressGateways:
    - name: istio-ingressgateway
      enabled: false
    - name: istio-eastwestgateway
      enabled: false
    egressGateways:
    - name: istio-egressgateway
      enabled: false

    # Helm values overrides
  values:
    # https://istio.io/v1.5/docs/reference/config/installation-options/#global-options
    global:
      # needed for connecting VirtualMachines to the mesh
      network: ${CLUSTER_NAME}-network
      # needed for annotating istio metrics with cluster
      multiCluster:
        clusterName: ${CLUSTER_NAME}
      # proxy:
      #  tracer: openCensusAgent 
EOF
```

Copy the configmap across other Istio namespaces
```
CM_DATA=$(kubectl get configmap istio-$REVISION -n istio-system -o jsonpath={.data})
cat <<EOF | kubectl apply -f -
{
    "apiVersion": "v1",
    "kind": "ConfigMap",
    "metadata": {
        "name": "istio-${REVISION}",
        "namespace": "istio-ingress",
        "labels": {
            "istio.io/rev": "${REVISION}"
        }
    },
    "data": $CM_DATA
}
EOF

cat <<EOF | kubectl apply -f -
{
    "apiVersion": "v1",
    "kind": "ConfigMap",
    "metadata": {
        "name": "istio-${REVISION}",
        "namespace": "istio-eastwest",
        "labels": {
            "istio.io/rev": "${REVISION}"
        }
    },
    "data": $CM_DATA
}
EOF
```

Deploy ingress gateway
```
cat << EOF | kubectl --context=${WEST_CONTEXT} apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: istio-ingressgateway
  namespace: istio-ingress
---
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-ingress
  labels:
    istio: ingressgateway-lb
  annotations:
    external-dns.alpha.kubernetes.io/hostname: apps.steadfast.kasunt.fe.gl00.net
spec:
  type: LoadBalancer
  selector:
    istio: ingressgateway
    version: ${REVISION}
  ports:
  # Health check port. For AWS ELBs, this port must be listed first.
  - name: status-port
    port: 15021
    targetPort: 15021
  - name: http2
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: tcp
    port: 31400
    targetPort: 31400
EOF

cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: ingress-gateway-${REVISION}
  namespace: istio-ingress
spec:
  profile: empty
  # This value is required for Gloo Mesh Istio
  hub: ${REPO}
  # This value can be any Gloo Mesh Istio tag
  tag: ${ISTIO_VERSION}
  revision: ${REVISION}
  components:
    ingressGateways:
      - name: istio-ingressgateway-${REVISION}
        namespace: istio-ingress
        enabled: true
        label:
          istio: ingressgateway
          version: ${REVISION}
          app: istio-ingressgateway
          # matches spec.values.global.network in istiod deployment
          topology.istio.io/network: ${CLUSTER_NAME}-network
        k8s:
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 100m
              memory: 40Mi
          service:
            # Since we created our own LoadBalanced service, tell istio to create a ClusterIP service for this gateway
            type: ClusterIP
            # match the LoadBalanced Service
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: http2
                port: 80
                targetPort: 8080
              - name: https
                port: 443
                targetPort: 8443
              - name: tcp
                port: 31400
                targetPort: 31400
          overlays:
            # Update role binding to use shared service account
            - apiVersion: rbac.authorization.k8s.io/v1
              kind: RoleBinding
              name: istio-ingressgateway-$REVISION-sds
              patches:
                - path: subjects.[0]
                  value:
                    name: "istio-ingressgateway"
                    kind: ServiceAccount
            - apiVersion: v1
              kind: Deployment
              name: istio-ingressgateway-$REVISION
              patches:
              # Update the deployment to use shared service account
              - path: spec.template.spec.serviceAccountName
                value: "istio-ingressgateway"
              - path: spec.template.spec.serviceAccount
                value: "istio-ingressgateway"
              # Sleep 25s on pod shutdown to allow connections to drain
              - path: spec.template.spec.containers.[name:istio-proxy].lifecycle
                value:
                  preStop:
                    exec:
                      command:
                      - sleep
                      - "25"
              # Schedule pods on separate nodes if possible
              - path: spec.template.spec.affinity
                value:
                  podAntiAffinity:
                    preferredDuringSchedulingIgnoredDuringExecution:
                    - podAffinityTerm:
                        labelSelector:
                          matchExpressions:
                          - key: app
                            operator: In
                            values:
                            - istio-ingressgateway-$REVISION
                        topologyKey: kubernetes.io/hostname
                      weight: 100

  # Helm values overrides
  values:
    global:
      # needed for connecting VirtualMachines to the mesh
      network: ${CLUSTER_NAME}-network
      # needed for annotating istio metrics with cluster
      multiCluster:
        clusterName: ${CLUSTER_NAME}
EOF
```

Deploy east-west gateway
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: istio-eastwestgateway
  namespace: istio-eastwest
---
apiVersion: v1
kind: Service
metadata:
  name: istio-eastwestgateway
  namespace: istio-eastwest
  labels:
    istio: eastwestgateway-lb
spec:
  type: LoadBalancer
  selector:
    istio: eastwestgateway
    version: ${REVISION}
  ports:
  # Health check port. For AWS ELBs, this port must be listed first.
  - name: status-port
    port: 15021
    targetPort: 15021
  # Port for multicluster mTLS passthrough
  # Dont change the name since Gloo Mesh looks for "tls"
  - name: tls
    port: 15443
    targetPort: 15443
EOF

cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: eastwest-gateway-${REVISION}
  namespace: istio-eastwest
spec:
  # No control plane components are installed
  profile: empty
  # This value is required for Gloo Mesh Istio
  hub: ${REPO}
  # The Solo.io Gloo Mesh Istio version
  tag: ${ISTIO_VERSION}
  # Istio revision to create resources with
  revision: ${REVISION}
  
  components:
    ingressGateways:
      # Enable the default east-west gateway
      - name: istio-eastwestgateway-${REVISION}
        namespace: istio-eastwest
        enabled: true
        label:
          istio: eastwestgateway
          version: ${REVISION}
          app: istio-eastwestgateway
          # Matches spec.values.global.network in the istiod deployment
          topology.istio.io/network: ${CLUSTER_NAME}
        k8s:
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 100m
              memory: 40Mi
          env:
            # Required by Gloo Mesh for east/west routing
            - name: ISTIO_META_ROUTER_MODE
              value: "sni-dnat"
          service:
            # Create a ClusterIP service for this gateway, because a LoadBalancer service is created separately
            type: ClusterIP
            # Match the LoadBalancer service
            ports:
              # Health check port. For AWS ELBs, this port must be listed first.
              - name: status-port
                port: 15021
                targetPort: 15021
              # Port for multicluster mTLS passthrough
              # Dont change the name since Gloo Mesh looks for "tls"
              - name: tls
                port: 15443
                targetPort: 15443
          overlays:
            # Update role binding to use shared service account
            - apiVersion: rbac.authorization.k8s.io/v1
              kind: RoleBinding
              name: istio-eastwestgateway-${REVISION}-sds
              patches:
                - path: subjects.[0]
                  value:
                    name: "istio-eastwestgateway"
                    kind: ServiceAccount
            - apiVersion: apps/v1
              kind: Deployment
              name: istio-eastwestgateway-${REVISION}
              patches:
                # Update the deployment to use shared service account
                - path: spec.template.spec.serviceAccountName
                  value: "istio-eastwestgateway"
                - path: spec.template.spec.serviceAccount
                  value: "istio-eastwestgateway"
                # Sleep 25s on pod shutdown to allow connections to drain
                - path: spec.template.spec.containers.[name:istio-proxy].lifecycle
                  value:
                    preStop:
                      exec:
                        command:
                        - sleep
                        - "25"
                # Schedule pods on separate nodes if possible
                - path: spec.template.spec.affinity
                  value:
                    podAntiAffinity:
                      preferredDuringSchedulingIgnoredDuringExecution:
                      - podAffinityTerm:
                          labelSelector:
                            matchExpressions:
                            - key: app
                              operator: In
                              values:
                              - istio-eastwestgateway-${REVISION}
                          topologyKey: kubernetes.io/hostname
                        weight: 100

  # Helm values overrides
  values:
    global:
      # needed for connecting VirtualMachines to the mesh
      network: ${CLUSTER_NAME}-network
      # needed for annotating istio metrics with cluster
      multiCluster:
        clusterName: ${CLUSTER_NAME}
EOF
```