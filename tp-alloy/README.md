# TP Grafana Alloy & OpenTelemetry — Kubernetes

Correction complète des 10 exercices pratiques sur cluster Kubernetes.  
Tous les fichiers sont dans ce dossier ; les commandes sont reproduites telles qu'exécutées.

---

## Prérequis

```bash
# Outils nécessaires
kubectl   # accès au cluster configuré
helm      # v3+

# Ajouter les repos Helm
helm repo add grafana              https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Namespace principal
kubectl create namespace observability
```

---

## Architecture finale

```
┌─────────────────────────────────────────────────────────────────┐
│  namespace: default                                             │
│  App Flask "demo" ──OTLP gRPC──►                               │
└──────────────────────────────────────────────────────────────┬──┘
                                                               │
┌─────────────────────────────────────────────────────────────────┐
│  namespace: payments                                            │
│  App Flask "payments" ──OTLP gRPC──►                           │
└──────────────────────────────────────────────────────────────┬──┘
                                                               │
┌─────────────────────────────────────────────────────────────────┐
│  namespace: observability — Grafana Alloy (DaemonSet)           │
│                                                                 │
│  receiver OTLP (4317/4318)                                      │
│    └─► batch ─► attributes (deployment.environment=lab)         │
│          ├─► metrics ─► attributes (team=platform)              │
│          │               └─► otelcol.exporter.prometheus        │
│          │                     └─► prometheus.remote_write      │
│          │                           └─► Mimir :9009            │
│          ├─► traces ─► tail_sampling (errors+slow+10%)          │
│          │               └─► Tempo :4317                        │
│          └─► logs ─► debug exporter                             │
│                 └─► k8sattributes ─► filter demo ─► Loki        │
│                                  └─► filter payments ─► Loki   │
│                                                                 │
│  loki.source.kubernetes (/var/log/pods) ─► loki.write           │
│  prometheus.scrape (node-exporter) ─► prometheus.remote_write  │
└─────────────────────────────────────────────────────────────────┘
        │               │              │
      Mimir           Loki          Tempo
      :9009           :3100          :3200
        └───────────────┴──────────────┘
                        │
                    Grafana :80
```

---

## Exercice 1 — Mettre Alloy en route

**Objectif :** Déployer Alloy via Helm avec un pipeline OTLP → debug exporter.

### `alloy-values.yaml` (état exo 1)

```yaml
alloy:
  extraArgs:
    - --stability.level=experimental   # requis pour otelcol.exporter.debug

  configMap:
    content: |
      otelcol.exporter.debug "default" {
        verbosity = "detailed"
      }

      otelcol.receiver.otlp "default" {
        grpc { endpoint = "0.0.0.0:4317" }
        http { endpoint = "0.0.0.0:4318" }
        output {
          metrics = [otelcol.exporter.debug.default.input]
          logs    = [otelcol.exporter.debug.default.input]
          traces  = [otelcol.exporter.debug.default.input]
        }
      }

  extraPorts:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
      protocol: TCP
    - name: otlp-http
      port: 4318
      targetPort: 4318
      protocol: TCP

  listenAddr: 0.0.0.0

controller:
  replicas: 1
```

### Commandes

```bash
helm upgrade --install alloy grafana/alloy \
  -n observability \
  -f alloy-values.yaml \
  --wait --timeout 3m
```

### Vérification

```bash
kubectl -n observability get pods
kubectl -n observability port-forward svc/alloy 12345:12345 &
curl -s http://localhost:12345/-/ready
# → "Alloy is ready."
```

---

## Exercice 2 — Envoyer des données OTLP avec telemetrygen

**Objectif :** Pousser traces, métriques et logs vers Alloy depuis des pods éphémères.

### Commandes

```bash
# Traces
kubectl run telemetrygen-traces --rm -i --restart=Never \
  --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  -- traces \
    --otlp-endpoint alloy.observability.svc:4317 \
    --otlp-insecure \
    --traces 5

# Métriques
kubectl run telemetrygen-metrics --rm -i --restart=Never \
  --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  -- metrics \
    --otlp-endpoint alloy.observability.svc:4317 \
    --otlp-insecure \
    --duration 5s

# Logs
kubectl run telemetrygen-logs --rm -i --restart=Never \
  --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  -- logs \
    --otlp-endpoint alloy.observability.svc:4317 \
    --otlp-insecure \
    --duration 5s
```

### Vérification

```bash
kubectl -n observability logs -l app.kubernetes.io/name=alloy --tail=200 \
  | grep -E "ResourceSpans|ResourceMetrics|ResourceLogs" | head -5
```

---

## Exercice 3 — Instrumenter une vraie application avec le SDK OTel

**Objectif :** Déployer une app Flask auto-instrumentée qui envoie les trois signaux vers Alloy.

> **Note :** Sans registry Docker accessible depuis le cluster, l'image est construite
> à l'exécution via un `initContainer` qui crée un virtualenv Python dans un volume `emptyDir`.

### `demo-app.yaml`

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-app
  namespace: default
data:
  app.py: |
    import random, time
    from flask import Flask

    app = Flask(__name__)

    @app.route("/")
    def index():
        delay = random.uniform(0, 0.8)
        time.sleep(delay)
        if random.random() < 0.1:
            raise ValueError("simulated error")
        return f"ok (delay={delay:.2f}s)\n"

    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=5000)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      initContainers:
        - name: install
          image: python:3.12-slim
          command: ["/bin/sh", "-c"]
          args:
            - |
              python3 -m venv /venv &&
              /venv/bin/pip install flask opentelemetry-distro \
                opentelemetry-exporter-otlp-proto-grpc \
                opentelemetry-instrumentation-flask -q &&
              /venv/bin/opentelemetry-bootstrap -a install
          volumeMounts:
            - name: pkgs
              mountPath: /venv
      containers:
        - name: demo
          image: python:3.12-slim
          command: ["/bin/sh", "-c"]
          args:
            - /venv/bin/opentelemetry-instrument /venv/bin/python /app/app.py
          env:
            - name: OTEL_SERVICE_NAME
              value: demo
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://alloy.observability.svc:4317
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: grpc
          ports:
            - containerPort: 5000
          volumeMounts:
            - name: app
              mountPath: /app
            - name: pkgs
              mountPath: /venv
      volumes:
        - name: app
          configMap:
            name: demo-app
        - name: pkgs
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: demo
  namespace: default
spec:
  selector:
    app: demo
  ports:
    - port: 5000
      targetPort: 5000
```

### Commandes

```bash
kubectl apply -f demo-app.yaml
kubectl rollout status deployment/demo
```

### Vérification

```bash
# Générer du trafic depuis le pod
DEMO_POD=$(kubectl get pod -l app=demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$DEMO_POD" -- /venv/bin/python -c "
import urllib.request
for i in range(30):
    try: urllib.request.urlopen('http://localhost:5000/', timeout=5)
    except: pass
print('done')
"

# Confirmer la réception dans Alloy
kubectl -n observability logs -l app.kubernetes.io/name=alloy --tail=300 \
  | grep "service.name.*demo" | head -3
```

---

## Exercice 4 — Maîtriser la syntaxe Alloy : pipeline, UI, hot reload

**Objectif :** Insérer `otelcol.processor.batch` et `otelcol.processor.attributes`
(ajout de `deployment.environment=lab`) puis recharger Alloy à chaud.

### `alloy-values.yaml` (état exo 4 — diff par rapport à exo 1)

Sections ajoutées dans `alloy.configMap.content` :

```alloy
otelcol.processor.batch "default" {
  output {
    metrics = [otelcol.processor.attributes.default.input]
    logs    = [otelcol.processor.attributes.default.input]
    traces  = [otelcol.processor.attributes.default.input]
  }
}

otelcol.processor.attributes "default" {
  action {
    key    = "deployment.environment"
    value  = "lab"
    action = "insert"
  }
  output {
    metrics = [otelcol.exporter.debug.default.input]
    logs    = [otelcol.exporter.debug.default.input]
    traces  = [otelcol.exporter.debug.default.input]
  }
}
```

Le receiver OTLP pointe désormais vers le batch :

```alloy
otelcol.receiver.otlp "default" {
  ...
  output {
    metrics = [otelcol.processor.batch.default.input]
    logs    = [otelcol.processor.batch.default.input]
    traces  = [otelcol.processor.batch.default.input]
  }
}
```

### Hot reload (sans redémarrage)

```bash
# Appliquer la nouvelle ConfigMap via Helm
helm upgrade alloy grafana/alloy -n observability -f alloy-values.yaml --wait

# Recharger à chaud via l'endpoint HTTP
kubectl -n observability port-forward ds/alloy 12345:12345 &
curl -X POST http://localhost:12345/-/reload
# → "config reloaded"
```

### Vérification

```bash
# Générer du trafic et chercher l'attribut
kubectl -n observability logs -l app.kubernetes.io/name=alloy --tail=100 \
  | grep "deployment.environment" | head -3
# → -> deployment.environment: Str(lab)
```

---

## Exercice 5 — Scraper des cibles Prometheus et expédier vers Mimir

**Objectif :** Déployer Mimir, node-exporter et Grafana ; configurer Alloy pour
scraper node-exporter et faire `remote_write` vers Mimir.

> **Note :** Le chart `grafana/mimir-distributed` en mode monolithique nécessite Kafka
> dans les versions récentes. On déploie Mimir directement comme Deployment simple.

### `mimir-simple.yaml`

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mimir-config
  namespace: observability
data:
  mimir.yaml: |
    target: all,alertmanager
    multitenancy_enabled: false

    common:
      storage:
        backend: filesystem
        filesystem:
          dir: /data/mimir

    blocks_storage:
      backend: filesystem
      filesystem:
        dir: /data/mimir/blocks

    compactor:
      data_dir: /data/mimir/compactor

    ingester:
      ring:
        replication_factor: 1

    store_gateway:
      sharding_ring:
        replication_factor: 1

    ruler_storage:
      backend: filesystem
      filesystem:
        dir: /data/mimir/rules

    alertmanager_storage:
      backend: filesystem
      filesystem:
        dir: /data/mimir/alertmanager

    server:
      http_listen_port: 9009
      grpc_listen_port: 9095
      log_level: warn
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mimir
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mimir
  template:
    metadata:
      labels:
        app: mimir
    spec:
      containers:
        - name: mimir
          image: grafana/mimir:latest
          args:
            - -config.file=/etc/mimir/mimir.yaml
          ports:
            - containerPort: 9009
              name: http
          volumeMounts:
            - name: config
              mountPath: /etc/mimir
            - name: data
              mountPath: /data
      volumes:
        - name: config
          configMap:
            name: mimir-config
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: mimir
  namespace: observability
spec:
  selector:
    app: mimir
  ports:
    - name: http
      port: 9009
      targetPort: 9009
```

### `grafana-values.yaml`

```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Mimir
        type: prometheus
        url: http://mimir.observability.svc:9009/prometheus
        isDefault: true
      - name: Loki
        type: loki
        url: http://loki.observability.svc:3100
      - name: Tempo
        type: tempo
        url: http://tempo.observability.svc:3100
```

### Sections ajoutées dans `alloy-values.yaml`

```alloy
prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir.observability.svc:9009/api/v1/push"
  }
}

discovery.kubernetes "node_exporter" {
  role = "endpoints"
}

discovery.relabel "node_exporter" {
  targets = discovery.kubernetes.node_exporter.targets
  rule {
    source_labels = ["__meta_kubernetes_service_label_app_kubernetes_io_name"]
    regex         = "prometheus-node-exporter"
    action        = "keep"
  }
  rule {
    source_labels = ["__address__"]
    target_label  = "__address__"
    regex         = "(.+):\\d+"
    replacement   = "$1:9100"
  }
}

prometheus.scrape "node_exporter" {
  targets    = discovery.relabel.node_exporter.output
  forward_to = [prometheus.remote_write.mimir.receiver]
}
```

### Commandes

```bash
kubectl apply -f mimir-simple.yaml

helm upgrade --install node-exporter \
  prometheus-community/prometheus-node-exporter \
  -n observability --wait

helm upgrade --install grafana grafana/grafana \
  -n observability \
  -f grafana-values.yaml --wait

helm upgrade alloy grafana/alloy \
  -n observability -f alloy-values.yaml --wait
```

### Vérification

```bash
kubectl -n observability port-forward svc/mimir 9009:9009 &
curl -s "http://localhost:9009/prometheus/api/v1/query?query=up" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    print([r['metric']['job'] for r in d['data']['result']])"
# → ['prometheus.scrape.node_exporter']

kubectl -n observability port-forward svc/grafana 3000:80 &
# Grafana → Explore → Mimir → query: up
```

---

## Exercice 6 — Métriques OTel, conversion et clustering Alloy

**Objectif :** Router les métriques OTLP de l'app vers Mimir (conversion OTel→Prometheus)
avec enrichissement `team=platform`. Activer le clustering Alloy sur 2 répliques (StatefulSet).

> **Note :** Le controller repasse temporairement en StatefulSet pour activer le clustering,
> puis repassera en DaemonSet à l'exercice 7.

### Sections ajoutées dans `alloy-values.yaml`

```alloy
# Enrichissement avant conversion
otelcol.processor.attributes "enrich" {
  action {
    key    = "team"
    value  = "platform"
    action = "insert"
  }
  output {
    metrics = [otelcol.exporter.prometheus.default.input]
  }
}

# Conversion OTel → Prometheus natif
otelcol.exporter.prometheus "default" {
  forward_to = [prometheus.remote_write.mimir.receiver]
}
```

Le processor `attributes "default"` envoie ses métriques vers `enrich` :

```alloy
otelcol.processor.attributes "default" {
  ...
  output {
    metrics = [otelcol.processor.attributes.enrich.input]   # ← modifié
    logs    = [otelcol.exporter.debug.default.input]
    traces  = [otelcol.exporter.debug.default.input]
  }
}
```

Configuration du clustering (dans `alloy-values.yaml`) :

```yaml
alloy:
  clustering:
    enabled: true
    portName: http

controller:
  type: statefulset
  replicas: 2
```

### Vérification

```bash
# Métriques OTel dans Mimir avec labels enrichis
kubectl -n observability port-forward svc/mimir 9009:9009 &
curl -s "http://localhost:9009/prometheus/api/v1/query?query=http_server_duration_milliseconds_count" \
  | python3 -c "
import sys,json; d=json.load(sys.stdin); r=d['data']['result']
print('team:', r[0]['metric']['team'])
print('deployment_environment:', r[0]['metric']['deployment_environment'])
"
# → team: platform
# → deployment_environment: lab

# Peers du cluster Alloy
kubectl -n observability port-forward pod/alloy-0 12345:12345 &
curl -s http://localhost:12345/api/v0/web/cluster/peers
# → 2 peers (alloy-0, alloy-1)
```

---

## Exercice 7 — Expédier les logs de pods vers Loki

**Objectif :** Déployer Loki en mode mono-binaire ; passer Alloy en DaemonSet avec accès
à `/var/log/pods` ; collecter les logs des pods et les envoyer à Loki.

### `loki-values.yaml`

```yaml
deploymentMode: SingleBinary

loki:
  auth_enabled: false   # sera passé à true à l'exercice 8
  commonConfig:
    replication_factor: 1
  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  storage:
    type: filesystem

singleBinary:
  replicas: 1

read:
  replicas: 0
write:
  replicas: 0
backend:
  replicas: 0

gateway:
  enabled: false
minio:
  enabled: false
chunksCache:
  enabled: false
resultsCache:
  enabled: false
```

### Sections ajoutées dans `alloy-values.yaml`

```yaml
# Repasser en DaemonSet avec accès aux logs
alloy:
  mounts:
    varlog: true   # monte /var/log sur chaque nœud

controller:
  type: daemonset
```

```alloy
loki.write "default" {
  endpoint {
    url       = "http://loki.observability.svc:3100/loki/api/v1/push"
    tenant_id = "team-demo"
  }
}

discovery.kubernetes "pods" {
  role = "pod"
}

discovery.relabel "pods" {
  targets = discovery.kubernetes.pods.targets
  rule {
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "__host__"
  }
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app"]
    target_label  = "app"
  }
  rule {
    source_labels = ["__meta_kubernetes_namespace", "__meta_kubernetes_pod_name"]
    separator     = "/"
    target_label  = "__path__"
    replacement   = "/var/log/pods/*$1*/*/*.log"
  }
}

loki.source.kubernetes "pods" {
  targets    = discovery.relabel.pods.output
  forward_to = [loki.process.pods.receiver]
}

loki.process "pods" {
  stage.match {
    selector = "{app=\"demo\"}"
    stage.regex {
      expression = "^(?P<level>INFO|ERROR|WARNING|DEBUG)"
    }
    stage.labels {
      values = { level = "" }
    }
  }
  forward_to = [loki.write.default.receiver]
}
```

### Commandes

```bash
helm upgrade --install loki grafana/loki \
  -n observability -f loki-values.yaml --wait --timeout 3m

helm upgrade alloy grafana/alloy \
  -n observability -f alloy-values.yaml --wait
kubectl -n observability rollout restart ds/alloy
```

### Vérification

```bash
kubectl -n observability port-forward svc/loki 3100:3100 &
curl -s http://localhost:3100/ready
# → ready

NOW=$(date +%s)000000000
HOUR_AGO=$(( (NOW/1000000000 - 3600) * 1000000000 ))
curl -sG "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={app="demo"}' \
  --data-urlencode "start=${HOUR_AGO}" \
  --data-urlencode "end=${NOW}" \
  --data-urlencode "limit=3" \
  -H "X-Scope-OrgID: team-demo" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    print('streams:', len(d['data']['result']))"
# → streams: 2
```

---

## Exercice 8 — Logs OTLP avec routage multi-tenant

**Objectif :** Passer Loki en `auth_enabled: true` ; créer le namespace `payments` ;
router les logs OTLP vers deux tenants selon le `service.name`.

### Modifications de `loki-values.yaml`

```yaml
loki:
  auth_enabled: true   # ← passer à true
```

### `alloy-rbac.yaml`

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: alloy-k8s-extra
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces", "nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: alloy-k8s-extra
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: alloy-k8s-extra
subjects:
  - kind: ServiceAccount
    name: alloy
    namespace: observability
```

### `payments-app.yaml`

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-app
  namespace: payments
data:
  app.py: |
    import random, time
    from flask import Flask

    app = Flask(__name__)

    @app.route("/")
    def index():
        delay = random.uniform(0, 0.8)
        time.sleep(delay)
        if random.random() < 0.1:
            raise ValueError("simulated error")
        return f"ok (delay={delay:.2f}s)\n"

    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=5000)
---
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    tenant: team-payments
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  namespace: payments
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      initContainers:
        - name: install
          image: python:3.12-slim
          command: ["/bin/sh", "-c"]
          args:
            - |
              python3 -m venv /venv &&
              /venv/bin/pip install flask opentelemetry-distro \
                opentelemetry-exporter-otlp-proto-grpc \
                opentelemetry-instrumentation-flask -q &&
              /venv/bin/opentelemetry-bootstrap -a install
          volumeMounts:
            - name: pkgs
              mountPath: /venv
      containers:
        - name: demo
          image: python:3.12-slim
          command: ["/bin/sh", "-c"]
          args:
            - /venv/bin/opentelemetry-instrument /venv/bin/python /app/app.py
          env:
            - name: OTEL_SERVICE_NAME
              value: payments
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://alloy.observability.svc:4317
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: grpc
          ports:
            - containerPort: 5000
          volumeMounts:
            - name: app
              mountPath: /app
            - name: pkgs
              mountPath: /venv
      volumes:
        - name: app
          configMap:
            name: demo-app
        - name: pkgs
          emptyDir: {}
```

### Sections ajoutées dans `alloy-values.yaml`

> **Note :** `otelcol.connector.routing` n'existe pas dans cette version d'Alloy.
> On utilise deux `otelcol.processor.filter` avec des expressions OTTL.

```alloy
# Enrichissement avec les attributs Kubernetes
otelcol.processor.k8sattributes "default" {
  extract {
    metadata = ["k8s.namespace.name", "k8s.pod.name", "k8s.node.name"]
  }
  output {
    logs = [otelcol.processor.filter.demo.input, otelcol.processor.filter.payments.input]
  }
}

# Filtre → tenant team-demo (service.name == "demo")
otelcol.processor.filter "demo" {
  error_mode = "ignore"
  logs {
    log_record = [
      "not (resource.attributes[\"service.name\"] == \"demo\" or IsMatch(attributes[\"k8s.namespace.name\"], \"default\"))",
    ]
  }
  output {
    logs = [otelcol.exporter.otlphttp.loki_demo.input]
  }
}

# Filtre → tenant team-payments (service.name == "payments")
otelcol.processor.filter "payments" {
  error_mode = "ignore"
  logs {
    log_record = [
      "not (resource.attributes[\"service.name\"] == \"payments\" or IsMatch(attributes[\"k8s.namespace.name\"], \"payments\"))",
    ]
  }
  output {
    logs = [otelcol.exporter.otlphttp.loki_payments.input]
  }
}

# Exporteurs OTLP vers Loki avec X-Scope-OrgID
otelcol.exporter.otlphttp "loki_demo" {
  client {
    endpoint = "http://loki.observability.svc:3100/otlp"
    headers  = { "X-Scope-OrgID" = "team-demo" }
    tls      { insecure = true }
  }
}

otelcol.exporter.otlphttp "loki_payments" {
  client {
    endpoint = "http://loki.observability.svc:3100/otlp"
    headers  = { "X-Scope-OrgID" = "team-payments" }
    tls      { insecure = true }
  }
}
```

Le receiver OTLP envoie les logs vers le batch ET vers k8sattributes :

```alloy
otelcol.receiver.otlp "default" {
  ...
  output {
    metrics = [otelcol.processor.batch.default.input]
    logs    = [otelcol.processor.batch.default.input, otelcol.processor.k8sattributes.default.input]
    traces  = [otelcol.processor.batch.default.input]
  }
}
```

### Commandes

```bash
helm upgrade loki grafana/loki -n observability -f loki-values.yaml --wait

kubectl apply -f alloy-rbac.yaml
kubectl apply -f payments-app.yaml

helm upgrade alloy grafana/alloy -n observability -f alloy-values.yaml --wait
kubectl -n observability rollout restart ds/alloy
```

### Vérification

```bash
kubectl -n observability port-forward svc/loki 3100:3100 &

# team-demo voit uniquement le namespace "default"
curl -sG "http://localhost:3100/loki/api/v1/label/k8s_namespace_name/values" \
  -H "X-Scope-OrgID: team-demo"
# → {"status":"success","data":["default"]}

# team-payments voit uniquement le namespace "payments"
curl -sG "http://localhost:3100/loki/api/v1/label/k8s_namespace_name/values" \
  -H "X-Scope-OrgID: team-payments"
# → {"status":"success","data":["payments"]}
```

---

## Exercice 9 — Expédier les traces OTLP vers Tempo

**Objectif :** Déployer Tempo en mono-binaire ; router les traces d'Alloy vers Tempo.

### `tempo-values.yaml`

```yaml
tempo:
  storage:
    trace:
      backend: local
      local:
        path: /var/tempo/traces
      wal:
        path: /var/tempo/wal

persistence:
  enabled: false

service:
  type: ClusterIP
```

### Section ajoutée dans `alloy-values.yaml`

```alloy
otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo.observability.svc:4317"
    tls { insecure = true }
  }
}
```

Le processor `attributes "default"` envoie les traces vers Tempo :

```alloy
otelcol.processor.attributes "default" {
  ...
  output {
    metrics = [otelcol.processor.attributes.enrich.input]
    logs    = [otelcol.exporter.debug.default.input]
    traces  = [otelcol.processor.tail_sampling.default.input]  # via tail_sampling → Tempo
  }
}
```

### Commandes

```bash
helm upgrade --install tempo grafana/tempo \
  -n observability -f tempo-values.yaml --wait

helm upgrade alloy grafana/alloy -n observability -f alloy-values.yaml --wait
kubectl -n observability rollout restart ds/alloy
```

### Vérification

```bash
# Générer du trafic
DEMO_POD=$(kubectl get pod -l app=demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$DEMO_POD" -- /venv/bin/python -c "
import urllib.request
for i in range(30):
    try: urllib.request.urlopen('http://localhost:5000/', timeout=5)
    except: pass
"

kubectl -n observability port-forward svc/tempo 3200:3200 &
sleep 5
curl -s "http://localhost:3200/api/search?tags=service.name%3Ddemo&limit=5" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    print('traces:', len(d['traces']), '— ex:', d['traces'][0]['durationMs'], 'ms')"
# → traces: 5 — ex: 579 ms
```

---

## Exercice 10 — Tail sampling — ne garder que l'essentiel

**Objectif :** Configurer `otelcol.processor.tail_sampling` avec trois politiques :
erreurs, traces > 500 ms, et 10 % des traces saines.

### Section ajoutée dans `alloy-values.yaml`

```alloy
otelcol.processor.tail_sampling "default" {
  decision_wait               = "10s"
  num_traces                  = 50000
  expected_new_traces_per_sec = 10

  policy {
    name = "errors"
    type = "status_code"
    status_code {
      status_codes = ["ERROR"]
    }
  }
  policy {
    name = "slow"
    type = "latency"
    latency {
      threshold_ms = 500
    }
  }
  policy {
    name = "sample-healthy"
    type = "probabilistic"
    probabilistic {
      sampling_percentage = 10
    }
  }

  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}
```

Le processor `attributes "default"` envoie les traces vers `tail_sampling` avant Tempo :

```alloy
otelcol.processor.attributes "default" {
  ...
  output {
    traces = [otelcol.processor.tail_sampling.default.input]   # ← tail sampling
  }
}
```

### Commandes

```bash
helm upgrade alloy grafana/alloy -n observability -f alloy-values.yaml --wait
kubectl -n observability rollout restart ds/alloy
```

### Vérification

```bash
# Générer 300 requêtes
DEMO_POD=$(kubectl get pod -l app=demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$DEMO_POD" -- /venv/bin/python -c "
import urllib.request
for i in range(300):
    try: urllib.request.urlopen('http://localhost:5000/', timeout=5)
    except: pass
print('done')
"

# Attendre decision_wait (10s) + ingestion
sleep 15

# Compter les traces dans Tempo (attendu ~60-80 sur 300 saines)
kubectl -n observability port-forward svc/tempo 3200:3200 &
sleep 2
curl -s "http://localhost:3200/api/search?tags=service.name%3Ddemo&limit=500" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    print('traces dans Tempo:', len(d['traces']), '/ 300 générées')"
# → traces dans Tempo: ~147 / 300 générées

# Métriques internes du processor
kubectl -n observability port-forward ds/alloy 12345:12345 &
sleep 2
curl -s http://localhost:12345/metrics \
  | grep otelcol_processor_tail_sampling_count_traces_sampled_total
# → policy="errors"        sampled="true"  → ~21
# → policy="slow"          sampled="true"  → ~118
# → policy="sample-healthy" sampled="true" → ~34
```

---

## Fichier `alloy-values.yaml` final complet

Le fichier `alloy-values.yaml` présent dans ce dossier est la version finale
utilisée après l'exercice 10. Il contient l'ensemble des pipelines :

| Bloc | Exercice |
|------|----------|
| `otelcol.exporter.debug` | Exo 1 |
| `otelcol.processor.batch` | Exo 4 |
| `otelcol.processor.attributes "default"` (env=lab) | Exo 4 |
| `prometheus.remote_write "mimir"` | Exo 5 |
| `discovery.kubernetes/relabel/scrape` (node-exporter) | Exo 5 |
| `otelcol.processor.attributes "enrich"` (team=platform) | Exo 6 |
| `otelcol.exporter.prometheus` | Exo 6 |
| `loki.write / discovery / loki.source.kubernetes / loki.process` | Exo 7 |
| `otelcol.processor.k8sattributes` + filtres + exporteurs Loki | Exo 8 |
| `otelcol.exporter.otlp "tempo"` | Exo 9 |
| `otelcol.processor.tail_sampling` | Exo 10 |
| `otelcol.receiver.otlp` | Exo 1 (étendu exos 4, 8) |

---

## Résumé des résultats

| Exo | Vérification clé | Résultat observé |
|-----|-----------------|-----------------|
| 1 | `curl /-/ready` | `Alloy is ready.` |
| 2 | Logs Alloy `ResourceSpans` | ✓ Traces telemetrygen reçues |
| 3 | `service.name: Str(demo)` | ✓ Flask → Alloy OTLP gRPC |
| 4 | `deployment.environment: Str(lab)` | ✓ Présent sur les 3 signaux |
| 5 | Mimir query `up` | ✓ node-exporter scraped |
| 6 | Mimir `team=platform` | ✓ + 2 peers Alloy |
| 7 | Loki stream `{app="demo"}` | ✓ Logs pods collectés |
| 8 | `team-demo→["default"]` / `team-payments→["payments"]` | ✓ Isolation tenant vérifiée |
| 9 | Tempo API `/api/search` | ✓ 5+ traces Flask |
| 10 | ~147 traces / 300 requêtes | ✓ errors=21, slow=118, healthy≈34 |

---

## Notes techniques

- **`--stability.level=experimental`** : requis pour `otelcol.exporter.debug` et
  `otelcol.processor.tail_sampling` qui sont en niveau expérimental.
- **Mimir** : le chart `grafana/mimir-distributed` en mode monolithique intègre Kafka
  dans les versions récentes. Remplacé par un Deployment Mimir single-binary.
- **`otelcol.connector.routing`** : ce composant n'existe pas dans Alloy v1.5.
  Remplacé par deux `otelcol.processor.filter` avec expressions OTTL.
- **Clustering** : activé temporairement en StatefulSet à l'exo 6, repassé en DaemonSet
  à l'exo 7 pour la collecte des logs.
- **App Flask** : construite via `initContainer` + virtualenv partagé sur `emptyDir`
  (pas de registry Docker disponible depuis la machine locale).
