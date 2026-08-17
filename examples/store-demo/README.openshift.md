# OBI Store Demo — OpenShift

This example runs the Google Cloud Online Boutique application on OpenShift
(CRC) and instruments it with OBI through the OpenTelemetry eBPF
Instrumentation Helm chart.

The vendored application comes from `GoogleCloudPlatform/microservices-demo`
v0.10.5. The OBI-specific files live in [`k8s`](./k8s) and provide:

- the `obi-store-demo` namespace and Online Boutique service manifests
- cluster-wide operator installs for Tempo, OTel, and Cluster Observability
- an OTel Collector that bridges OTLP from OBI to Prometheus and Tempo
- Helm values for the `opentelemetry-ebpf-instrumentation` chart

Telemetry is visible in the OpenShift web console under **Observe > Metrics**
(user-workload Prometheus) and **Observe > Traces** (Tempo via the Distributed
Tracing UI plugin).

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the service topology. It is
generated from the manifests by [fix_architecture.py](./fix_architecture.py);
after changing a manifest, run
`python3 examples/store-demo/fix_architecture.py` to regenerate it (CI checks
it with `--check`).

## Prerequisites

- `podman`
- `crc`
- `oc` / `kubectl`
- Helm

## Create The Cluster

```bash
crc start
oc new-project obi-store-demo
```

## Build And Load Images

```bash
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

podman login --tls-verify=false -u $(oc whoami) -p $(oc whoami -t) \
  default-route-openshift-image-registry.apps-crc.testing

for service_context in "${services[@]}"; do
  service="${service_context%%:*}"
  context="${service_context#*:}"
  image="default-route-openshift-image-registry.apps-crc.testing/obi-store-demo/obi-store-demo-${service}:local"
  podman build -t "${image}" "${context}"
  podman push --tls-verify=false "${image}"
done
```

## Install Cluster-Wide Operators

These resources span multiple namespaces and must be applied directly — not
through `oc apply -k`, which would stamp the `obi-store-demo` namespace
override onto them.

```bash
oc apply -f examples/store-demo/k8s/02-operators.yaml
```

Wait for the operator CRDs to be registered before continuing:

```bash
oc wait --for=condition=Established crd/opentelemetrycollectors.opentelemetry.io --timeout=180s
oc wait --for=condition=Established crd/tempomonolithics.tempo.grafana.com --timeout=180s
oc wait --for=condition=Established crd/uiplugins.observability.openshift.io --timeout=180s
```

## Deploy Observability Instances And The Store

Apply the observability instances (OTel Collector, Tempo, UIPlugin) and then
the app workloads:

```bash
oc apply -f examples/store-demo/k8s/03-instances.yaml
oc apply -k examples/store-demo/k8s
oc -n obi-store-demo wait --for=condition=Available deploy --all --timeout=5m
```

## Generate Traffic

The `loadgenerator` deployment starts sending traffic to the frontend
automatically. You can also port-forward the frontend and send manual requests:

```bash
kubectl -n obi-store-demo port-forward svc/frontend 8080:80
```

```bash
curl http://127.0.0.1:8080/
curl http://127.0.0.1:8080/product/OLJCESPC7Z
curl http://127.0.0.1:8080/cart
```

## Explore Telemetry In The OpenShift Console

Log in to the OpenShift web console:

```bash
crc console --credentials   # shows the kubeadmin password
crc console                  # opens the browser
```

**Metrics** — navigate to **Observe > Metrics** and query a metric produced by
OBI, for example:

```
http_server_request_duration_seconds_count{k8s_namespace_name="obi-store-demo"}
```

**Traces** — navigate to **Observe > Traces**. Select the `dev` tenant and
search by service name (e.g. `frontend`) to inspect distributed traces across
the store services.

## Expected OBI Visibility And Current Gaps

- Service discovery should find the store deployments listed in
  [`03-obi-values.yaml`](./k8s/03-obi-values.yaml) and attach Kubernetes
  workload metadata to their telemetry.
- Frontend HTTP traffic should produce traces and metrics for requests such as
  `/`, `/product/:product_id`, `/cart`, and `/cart/checkout`.
- Metrics should appear in user-workload Prometheus with HTTP metrics grouped
  by route, status code, and Kubernetes workload metadata.
- Expect partial backend gRPC visibility. OBI can report supported gRPC spans
  and metrics but not every backend RPC is guaranteed to appear with a complete
  method name or matching client/server pair.
- Current gRPC propagation does not guarantee a fully stitched checkout trace.
  Separate backend gRPC traces or incomplete service-graph edges are expected
  OBI visibility gaps, not store-demo failures.

## Cleanup

```bash
oc delete -k examples/store-demo/k8s
oc delete -f examples/store-demo/k8s/03-instances.yaml
oc delete -f examples/store-demo/k8s/02-operators.yaml
```
