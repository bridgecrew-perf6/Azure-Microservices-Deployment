1. Relay root CA

```
mkdir -p _output/pki
openssl req -new -newkey rsa:4096 -x509 -sha256 \
        -days 3650 -nodes -out _output/pki/relay-root-ca.crt -keyout _output/pki/relay-root-ca.key \
        -subj "/CN=relay-root-ca" \
        -addext "extendedKeyUsage = clientAuth, serverAuth"
```

2. Generate server certificate

```
cat > _output/pki/"enterprise-networking.conf" <<EOF
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
DNS = *.gloo-mesh
EOF

openssl genrsa -out _output/pki/"enterprise-networking.key" 2048

openssl req -new -key _output/pki/"enterprise-networking.key" -out _output/pki/enterprise-networking.csr -subj "/CN=enterprise-networking" -config _output/pki/"enterprise-networking.conf"

openssl x509 -req \
  -days 3650 \
  -CA _output/pki/relay-root-ca.crt -CAkey _output/pki/relay-root-ca.key \
  -set_serial 0 \
  -in _output/pki/enterprise-networking.csr -out _output/pki/enterprise-networking.crt \
  -extensions v3_req -extfile _output/pki/"enterprise-networking.conf"

kubectl --context ${MGMT_CONTEXT} create ns gloo-mesh

kubectl create secret generic relay-server-tls-secret \
  --from-file=tls.key=_output/pki/enterprise-networking.key \
  --from-file=tls.crt=_output/pki/enterprise-networking.crt \
  --from-file=ca.crt=_output/pki/relay-root-ca.crt \
  --context ${MGMT_CONTEXT} \
  --namespace gloo-mesh
```