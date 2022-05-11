# Storage Services

## MongoDB
```
helm install --kube-context=${WEST_CONTEXT} mongodb bitnami/mongodb -n storage --create-namespace -f integrations/services/mongodb/mongodb-helm-values.yaml
```

## RabbitMQ
```
helm install --kube-context=${WEST_CONTEXT} rabbitmq bitnami/rabbitmq -n storage --create-namespace -f integrations/services/rabbitmq/rabbitmq-helm-values.yaml
```

## Redis
```
helm install --kube-context=${WEST_CONTEXT} redis bitnami/redis -n storage --create-namespace -f integrations/services/redis/redis-helm-values.yaml
```

## MSSql Server
```
helm install --kube-context=${WEST_CONTEXT} mssql-service integrations/services/mssql-service -n storage
```