1. Generate agent certificate

```
cat > _output/pki/"enterprise-agent-${CLUSTER_NAME}.conf" <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, serverAuth
subjectAltName = @alt_names
[alt_names]
DNS = ${CLUSTER_NAME}
EOF

openssl genrsa -out _output/pki/"enterprise-agent-${CLUSTER_NAME}.key" 2048

openssl req -new -key _output/pki/"enterprise-agent-${CLUSTER_NAME}.key" -out _output/pki/"enterprise-agent-${CLUSTER_NAME}.csr" -subj "/CN=enterprise-networking-ca" -config _output/pki/"enterprise-agent-${CLUSTER_NAME}.conf"

openssl x509 -req \
  -days 3650 \
  -CA _output/pki/relay-root-ca.crt -CAkey _output/pki/relay-root-ca.key \
  -set_serial 0 \
  -in _output/pki/"enterprise-agent-${CLUSTER_NAME}.csr" -out _output/pki/"enterprise-agent-${CLUSTER_NAME}.crt" \
  -extensions v3_req -extfile _output/pki/"enterprise-agent-${CLUSTER_NAME}.conf"

kubectl --context ${CLUSTER_CONTEXT} create ns gloo-mesh

kubectl create secret generic "enterprise-agent-${CLUSTER_NAME}-tls-cert" \
  --from-file=tls.key=_output/pki/"enterprise-agent-${CLUSTER_NAME}.key" \
  --from-file=tls.crt=_output/pki/"enterprise-agent-${CLUSTER_NAME}.crt" \
  --from-file=ca.crt=_output/pki/relay-root-ca.crt \
  --context $CLUSTER_CONTEXT \
  --namespace gloo-mesh
```