# Istio + Observability Stack Integration for Gogs

This guide demonstrates how to set up a production-grade Istio service mesh integrated with Prometheus, Jaeger, Loki, and Grafana for observability of the Gogs application.

## 1. Install Istio

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH="$PATH:/root/istio-1.27.3/bin"
istioctl install --set profile=demo -y
```

## 2. Enable Istio Sidecar Injection in Gogs Namespace

```bash
kubectl label namespace gogs istio-injection=enabled
```

## 3. Apply Telemetry Resource

```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: gogs
  namespace: gogs
spec:
  accessLogging:
  - providers:
    - name: envoy
  metrics:
  - providers:
    - name: prometheus
  tracing:
  - providers:
    - name: jaeger
```

## 4. Install Observability Stack

### Jaeger (all-in-one)

```bash
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.46.0/jaeger-operator.yaml
kubectl apply -n istio-system -f jaeger.yaml
```

### Prometheus

Deployed by default via Istio `demo` profile.

### Loki

> ⚠️ **Note:** You must create 3 unique S3 buckets for Loki's storage (`chunks`, `ruler`, and `admin`). You can either use AWS access keys or IAM roles with service accounts. Below example uses AWS access keys.

**values-loki.yaml**

```yaml
loki:
  deploymentMode: Monolithic
  image:
    tag: "3.5.2"
  schemaConfig:
    configs:
      - from: "2025-01-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: index_
          period: 24h
  storage:
    type: s3
    bucketNames:
      chunks: loki-chunks-gog
      ruler: loki-ruler-gog
      admin: loki-admin-gog
    s3:
      endpoint: s3.amazonaws.com
      region: us-east-1
      accessKeyId: "<your-access-key>"
      secretAccessKey: "<your-secret-key>"
      s3ForcePathStyle: false
  persistence:
    enabled: true
    storageClass: gp2
    size: 50Gi
    accessModes:
      - ReadWriteOnce 
```

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki grafana/loki --namespace observability -f values-loki.yaml --create-namespace
```

### Promtail

**values-promtail.yaml**

```yaml
config:
  positions:
    filename: /run/promtail/positions.yaml
  clients:
    - url: http://loki-gateway.observability.svc.cluster.local/loki/api/v1/push
      tenant_id: "prod"
scrape_configs:
  - job_name: kubernetes-pods
    pipeline_stages:
      - docker: {}
      - labeldrop:
          - container_name
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: ""
        action: drop
extraVolumes:
  - name: positions
    emptyDir: {}
extraVolumeMounts:
  - name: positions
    mountPath: /run/promtail
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
  requests:
    cpu: "100m"
    memory: "128Mi"
daemonSet:
  enabled: true
rbac:
  create: true
serviceAccount:
  create: true
```

```bash
helm install promtail grafana/promtail --namespace observability -f values-promtail.yaml
```

## 5. Configure Grafana

Grafana should be deployed in the `istio-system` namespace.

**Grafana ConfigMap**

```yaml
data:
  dashboardproviders.yaml: |
    apiVersion: 1
    providers:
    - name: istio
      folder: istio
      orgId: 1
      type: file
      disableDeletion: false
      options:
        path: /var/lib/grafana/dashboards/istio
    - name: istio-services
      folder: istio
      orgId: 1
      type: file
      disableDeletion: false
      options:
        path: /var/lib/grafana/dashboards/istio-services
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      orgId: 1
      isDefault: true
      jsonData:
        timeInterval: 15s
    - name: Loki
      type: loki
      access: proxy
      jsonData:
        timeInterval: 600s
        maxLines: 1000
        httpHeaderName1: X-Scope-OrgID
      secureJsonData:
        httpHeaderValue1: "prod"
    - name: Jaeger
      type: jaeger
      access: proxy
      orgId: 1
      uid: PC9A941E8F2E49454 //random
      url: http://tracing.istio-system.svc.cluster.local/jaeger
  grafana.ini: |
    [analytics]
    check_for_updates = true
    [grafana_net]
    url = https://grafana.net
    [log]
    mode = console
    [paths]
    data = /var/lib/grafana/
    logs = /var/log/grafana
    plugins = /var/lib/grafana/plugins
    provisioning = /etc/grafana/provisioning
    [server]
    domain = ''
```

Once deployed, Grafana will display:

- Metrics from Prometheus
- Logs from Loki
- Traces from Jaeger

All correlated through the Istio Telemetry configuration in the `gogs` namespace.
