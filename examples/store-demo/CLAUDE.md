# Store Demo — OpenShift CRC Deployment Notes

## Collector Image

Do **not** set an explicit `image:` on the `OpenTelemetryCollector` CR in `03-instances.yaml`.
The OBI receiver ships in the RHOSDT 3.11 Red Hat OTel build; let the operator pick its default image.

## Deployment Order

```bash
# 1. Log in (CRC kubeadmin password from `crc console --credentials`)
oc login -u kubeadmin -p <pw> https://api.crc.testing:6443

# 2. Create project
oc new-project obi-store-demo

# 3. Cluster-wide operators (SCC, COO, Tempo, OTel)
oc apply -f examples/store-demo/k8s/02-operators.yaml

# 4. Wait for CRDs (required before instances can be applied)
oc wait --for=condition=Established crd/opentelemetrycollectors.opentelemetry.io --timeout=180s
oc wait --for=condition=Established crd/tempomonolithics.tempo.grafana.com --timeout=180s
oc wait --for=condition=Established crd/uiplugins.observability.openshift.io --timeout=180s

# 5. Log in to the internal registry and build+push all app images
podman login --tls-verify=false -u $(oc whoami) -p $(oc whoami -t) \
  default-route-openshift-image-registry.apps-crc.testing

services=(
  "adservice:examples/store-demo/app/src/adservice"
  "cartservice:examples/store-demo/app/src/cartservice/src"
  "checkoutservice:examples/store-demo/app/src/checkoutservice"
  "currencyservice:examples/store-demo/app/src/currencyservice"
  "emailservice:examples/store-demo/app/src/emailservice"
  "frontend:examples/store-demo/app/src/frontend"
  "loadgenerator:examples/store-demo/app/src/loadgenerator"
  "paymentservice:examples/store-demo/app/src/paymentservice"
  "productcatalogservice:examples/store-demo/app/src/productcatalogservice"
  "recommendationservice:examples/store-demo/app/src/recommendationservice"
  "shippingservice:examples/store-demo/app/src/shippingservice"
)
for service_context in "${services[@]}"; do
  service="${service_context%%:*}"
  context="${service_context#*:}"
  image="default-route-openshift-image-registry.apps-crc.testing/obi-store-demo/obi-store-demo-${service}:local"
  podman build -t "${image}" "${context}"
  podman push --tls-verify=false "${image}"
done

# 6. Apply observability instances + app workloads
oc apply -f examples/store-demo/k8s/03-instances.yaml
oc apply -k examples/store-demo/k8s
oc -n obi-store-demo wait --for=condition=Available deploy --all --timeout=5m
```

All commands are run from the repo root (`opentelemetry-ebpf-instrumentation/`).

## Cleanup

```bash
oc delete -k examples/store-demo/k8s
oc delete -f examples/store-demo/k8s/03-instances.yaml
oc delete -f examples/store-demo/k8s/02-operators.yaml
```

## Access

- Frontend: `https://frontend-obi-store-demo.apps-crc.testing`
- Console: `crc console` (kubeadmin credentials from `crc console --credentials`)
- Traces: **Observe > Traces**, tenant `dev`
- Metrics: **Observe > Metrics**, e.g. `http_server_request_duration_seconds_count{k8s_namespace_name="obi-store-demo"}`
