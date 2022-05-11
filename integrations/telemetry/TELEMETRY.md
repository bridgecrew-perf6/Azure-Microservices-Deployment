# Install OTel Collector and Jaeger

```
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

helm install --kube-context=${WEST_CONTEXT} jaeger jaegertracing/jaeger -n telemetry --create-namespace -f integrations/telemetry/jaeger-helm-values.yaml

helm install --kube-context=${WEST_CONTEXT} oc open-telemetry/opentelemetry-collector -n telemetry --create-namespace -f integrations/telemetry/otel-collector-helm-values.yaml
```