# Overview

## install kube prometheus stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack \
prometheus-community/kube-prometheus-stack \
--version 44.3.0 \
--namespace monitoring \
--create-namespace \
--values - <<EOF
prometheus:
  service:
    type: LoadBalancer
    port: 9090
grafana:
  service:
    type: LoadBalancer
    port: 3000
EOF
```

## install loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana/loki-stack \
--version 2.9.9 \
--namespace logging \
--create-namespace

helm upgrade --install kube-prometheus-stack \
prometheus-community/kube-prometheus-stack \
--version 44.3.1 \
--namespace monitoring \
--create-namespace \
--values - <<EOF
grafana:
  service:
    type: LoadBalancer
    port: 3000
  additionalDataSources:
  - name: Loki
    type: loki
    url: http://loki.logging.svc.cluster.local:3100
    jsonData:
      maxLines: 2000
EOF
```

## install tempo

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install tempo grafana/tempo \
--kube-context $MGMT \
--version 1.0.2 \
--namespace tracing \
--create-namespace \
--values - <<EOF
service:
  type: LoadBalancer
tempo:
  receivers:
    zipkin:
      endpoint: 0.0.0.0:9411
EOF

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack \
prometheus-community/kube-prometheus-stack \
--version 44.3.1 \
--namespace monitoring \
--create-namespace \
--values - <<EOF
grafana:
  service:
    type: LoadBalancer
    port: 3000
  additionalDataSources:
  - name: Loki
    type: loki
    url: http://loki.logging.svc.cluster.local:3100
    jsonData:
      maxLines: 2000
  - name: Tempo
    type: tempo
    url: http://tempo.tracing.svc.cluster.local:3101
EOF
```

## install istio with tracing extension providers

```bash
./istio-1.15.3/bin/istioctl install --context ${MGMT} --set profile=demo -y -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    accessLogFile: /dev/stdout
    extensionProviders:
      - name: zipkincustom
        zipkin:
          service: tracing.global
          port: 9411
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
EOF

kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-tracing
  namespace: istio-system
spec:
  hosts:
  - "tracing.global"
  addresses:
  - "192.168.0.1" # This is only internal. The proxy will resolve to the endpoint address
  location: MESH_EXTERNAL
  ports:
  - number: 9411
    name: tcp
    protocol: TCP
  resolution: STATIC
  endpoints:
  - address: ${TRACING_BACKEND_IP}
EOF

kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: default
  namespace: istio-system
spec:
  tracing:
  - providers:
      - name: zipkincustom
    randomSamplingPercentage: 100
    disableSpanReporting: false
EOF
```

## add istio scrape configs

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack \
prometheus-community/kube-prometheus-stack \
--version 44.3.1 \
--namespace monitoring \
--create-namespace \
--values - <<EOF
grafana:
  service:
    type: LoadBalancer
    port: 3000
  additionalDataSources:
  - name: Loki
    type: loki
    url: http://loki.logging.svc.cluster.local:3100
    jsonData:
      maxLines: 2000
  - name: Tempo
    type: tempo
    url: http://tempo.tracing.svc.cluster.local:3101
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: provider-site
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/provider-site
  dashboards:
    provider-site:
      node-exporter-full:
        url: https://grafana.com/api/dashboards/7630/revisions/159/download
        # gnetId: 7630
        datasource: Prometheus
prometheus:
  prometheusSpec:
    podMonitorSelectorNilUsesHelmValues: false
    additionalScrapeConfigs:
      - job_name: 'istiod'
        kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names:
            - istio-system
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: istiod;http-monitoring

      - job_name: 'envoy-stats'
        metrics_path: /stats/prometheus
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_container_port_name]
          action: keep
          regex: '.*-envoy-prom'
  service:
    type: LoadBalancer
    port: 9090
EOF
```