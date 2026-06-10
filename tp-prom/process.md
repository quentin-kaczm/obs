# TP Observabilité — Procédures

> Commandes pour Docker. Tout est conteneurisé.

---

## Module 1 — Prometheus

### Exercice 1 : Installer Prometheus et accéder à l'interface web

```bash
docker pull prom/prometheus:latest

docker run -d --name prometheus -p "192.168.1.182:9090:9090" prom/prometheus:latest
```

L'hôte dispose de plusieurs interfaces réseau — le bind sur `192.168.1.182` permet de n'exposer Prometheus que sur cette interface précise plutôt que sur toutes (`0.0.0.0`).

Ouvrir [http://192.168.1.182:9090](http://192.168.1.182:9090), puis **Status > Targets** et vérifier que `prometheus` est **UP**.

```bash
docker logs prometheus  # chercher la ligne annonçant le répertoire de stockage
```

```bash
time=2026-04-27T09:32:51.711Z level=INFO source=main.go:1678 msg="updated GOGC" old=100 new=75
time=2026-04-27T09:32:51.712Z level=INFO source=main.go:744 msg="Leaving GOMAXPROCS=16: CPU quota undefined" component=automaxprocs
time=2026-04-27T09:32:51.712Z level=INFO source=memlimit.go:198 msg="GOMEMLIMIT is updated" component=automemlimit package=github.com/KimMachineGun/automemlimit/memlimit GOMEMLIMIT=29740643942 previous=9223372036854775807
time=2026-04-27T09:32:51.712Z level=INFO source=main.go:851 msg="Starting Prometheus Server" mode=server version="(version=3.11.2, branch=HEAD, revision=f0f0fdd679dcd6df320b0558b20919f7cd44c407)"
time=2026-04-27T09:32:51.712Z level=INFO source=main.go:856 msg="operational information" build_context="(go=go1.26.2, platform=linux/amd64, user=root@d2684485c347, date=20260413-11:53:40, tags=netgo,builtinassets)" host_details="(Linux 6.12.33-production+truenas #1 SMP PREEMPT_DYNAMIC Mon Apr 13 19:09:57 UTC 2026 x86_64 de2eebacf28c (none))" fd_limits="(soft=1048576, hard=1048576)" vm_limits="(soft=unlimited, hard=unlimited)"
time=2026-04-27T09:32:51.794Z level=INFO source=web.go:710 msg="Start listening for connections" component=web address=0.0.0.0:9090
time=2026-04-27T09:32:51.794Z level=INFO source=main.go:1410 msg="Starting TSDB ..."
time=2026-04-27T09:32:51.795Z level=INFO source=tls_config.go:354 msg="Listening on" component=web address=[::]:9090
time=2026-04-27T09:32:51.795Z level=INFO source=tls_config.go:357 msg="TLS is disabled." component=web http2=false address=[::]:9090
time=2026-04-27T09:32:51.796Z level=INFO source=head.go:698 msg="Replaying on-disk memory mappable chunks if any" component=tsdb
time=2026-04-27T09:32:51.796Z level=INFO source=head.go:784 msg="On-disk memory mappable chunks replay completed" component=tsdb duration=771ns
time=2026-04-27T09:32:51.796Z level=INFO source=head.go:792 msg="Replaying WAL, this may take a while" component=tsdb
time=2026-04-27T09:32:51.798Z level=INFO source=head.go:865 msg="WAL segment loaded" component=tsdb segment=0 maxSegment=0 duration=1.389033ms
time=2026-04-27T09:32:51.798Z level=INFO source=head.go:902 msg="WAL replay completed" component=tsdb checkpoint_replay_duration=35.687µs wal_replay_duration=1.406265ms wbl_replay_duration=130ns chunk_snapshot_load_duration=0s mmap_chunk_replay_duration=771ns total_replay_duration=1.458673ms
time=2026-04-27T09:32:51.799Z level=INFO source=main.go:1431 msg="filesystem information" fs_type=2fc12fc1
time=2026-04-27T09:32:51.799Z level=INFO source=main.go:1434 msg="TSDB started"
time=2026-04-27T09:32:51.799Z level=INFO source=main.go:1632 msg="Loading configuration file" filename=/etc/prometheus/prometheus.yml
time=2026-04-27T09:32:51.799Z level=INFO source=main.go:1048 msg="TSDB retention updated" duration=15d size=0B percentage=0
time=2026-04-27T09:32:51.799Z level=INFO source=main.go:1671 msg="Completed loading of configuration file" db_storage=18.655µs remote_storage=1.382µs web_handler=340ns query_engine=451ns scrape=263.506µs scrape_sd=14.007µs notify=61.947µs notify_sd=5.63µs rules=1.122µs tracing=2.866µs filename=/etc/prometheus/prometheus.yml totalDuration=548.812µs
time=2026-04-27T09:32:51.799Z level=INFO source=main.go:1395 msg="Server is ready to receive web requests."
time=2026-04-27T09:32:51.799Z level=INFO source=manager.go:209 msg="Starting rule manager..." component="rule manager"
```
L'emplacement du fichier de configuration est disponible(msg="Loading configuration file" filename=/etc/prometheus/prometheus.yml), mais pas le celui du stockage,on peut cependant le trouver avec la commande suivante:
```bash
docker inspect prometheus --format '{{ json .Mounts }}'
```

---

### Exercice 2 : Écrire son premier prometheus.yml

**Objectif :** Remplacer la config par défaut par un `prometheus.yml` personnalisé — intervalle de scrape de 10s, external label `environment=lab` — et recharger Prometheus sans le redémarrer.

```bash
# Supprimer le conteneur précédent
docker rm -f prometheus
```

Créer un fichier `prometheus.yml` sur l'hôte :

```yaml
global:
  scrape_interval: 10s
  external_labels:
    environment: lab

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

Relancer le conteneur en montant le fichier et en activant l'endpoint de rechargement :

```bash
docker run -d --name prometheus \
  -p "192.168.1.182:9090:9090" \
  -v /root/docker/ynov/obs/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
```

Status/configuration:
```yaml
global:
  scrape_interval: 10s
  scrape_timeout: 10s
  evaluation_interval: 1m
  external_labels:
    environment: lab
  metric_name_validation_scheme: utf8
  scrape_native_histograms: false
  extra_scrape_metrics: false
runtime:
  gogc: 75
scrape_configs:
- job_name: prometheus
  honor_timestamps: true
  track_timestamps_staleness: false
  scrape_interval: 10s
  scrape_timeout: 10s
  scrape_protocols:
  - OpenMetricsText1.0.0
  - OpenMetricsText0.0.1
  - PrometheusText1.0.0
  - PrometheusText0.0.4
  scrape_native_histograms: false
  always_scrape_classic_histograms: false
  convert_classic_histograms_to_nhcb: false
  metrics_path: /metrics
  scheme: http
  enable_compression: true
  metric_name_validation_scheme: utf8
  metric_name_escaping_scheme: allow-utf-8
  extra_scrape_metrics: false
  follow_redirects: true
  enable_http2: true
  static_configs:
  - targets:
    - localhost:9090
storage:
  tsdb:
    outofordertimewindow: 0
    retention:
      time: 15d
otlp:
  translation_strategy: UnderscoreEscapingWithSuffixes
  label_name_underscore_sanitization: true
  label_name_preserve_multiple_underscores: true
```

On change le tag 'lab' par 'lab-test' dans le fichier de configuration.

Pour recharger la config sans redémarrer le conteneur :

```bash
curl -X POST http://192.168.1.182:9090/-/reload
```

```yaml
global:
  scrape_interval: 10s
  scrape_timeout: 10s
  evaluation_interval: 1m
  external_labels:
    environment: lab-test
  metric_name_validation_scheme: utf8
  scrape_native_histograms: false
  extra_scrape_metrics: false
runtime:
  gogc: 75
scrape_configs:
- job_name: prometheus
  honor_timestamps: true
  track_timestamps_staleness: false
  scrape_interval: 10s
  scrape_timeout: 10s
  scrape_protocols:
  - OpenMetricsText1.0.0
  - OpenMetricsText0.0.1
  - PrometheusText1.0.0
  - PrometheusText0.0.4
  scrape_native_histograms: false
  always_scrape_classic_histograms: false
  convert_classic_histograms_to_nhcb: false
  metrics_path: /metrics
  scheme: http
  enable_compression: true
  metric_name_validation_scheme: utf8
  metric_name_escaping_scheme: allow-utf-8
  extra_scrape_metrics: false
  follow_redirects: true
  enable_http2: true
  static_configs:
  - targets:
    - localhost:9090
storage:
  tsdb:
    outofordertimewindow: 0
    retention:
      time: 15d
otlp:
  translation_strategy: UnderscoreEscapingWithSuffixes
  label_name_underscore_sanitization: true
  label_name_preserve_multiple_underscores: true
```

On peut voir que le tag a bien changé.

Vérifier la prise en compte dans **Status > Configuration**.

---

### Exercice 3 : Ajouter node_exporter et scraper les métriques système

**Objectif :** Lancer node_exporter et configurer Prometheus pour le scraper. Vérifier que `node_cpu_seconds_total` apparaît dans l'expression browser.

```bash
docker run -d --name node-exporter -p "192.168.1.182:9100:9100" prom/node-exporter:latest
```

Ajouter le job dans `prometheus.yml` :

```yaml
  - job_name: 'node'
    static_configs:
      - targets: ['192.168.1.182:9100']
```

Recharger la config :

```bash
curl -X POST http://192.168.1.182:9090/-/reload
```

Vérifier dans **Status > Targets** que la cible `node` est **UP**, puis exécuter dans l'expression browser :

```
node_cpu_seconds_total
```

Résultat: 
```bash
node_cpu_seconds_total{cpu="0", instance="192.168.1.182:9100", job="node", mode="idle"}	1026996.09
node_cpu_seconds_total{cpu="0", instance="192.168.1.182:9100", job="node", mode="iowait"}	2053.41
node_cpu_seconds_total{cpu="0", instance="192.168.1.182:9100", job="node", mode="irq"}	0
node_cpu_seconds_total{cpu="0", instance="192.168.1.182:9100", job="node", mode="nice"}	390.6
node_cpu_seconds_total{cpu="0", instance="192.168.1.182:9100", job="node", mode="softirq"}	4237.63
node_cpu_seconds_total{cpu="0", instance="192.168.1.182:9100", job="node", mode="steal"}	0
node_cpu_seconds_total{cpu="0", instance="192.168.1.182:9100", job="node", mode="system"}	1708.93
node_cpu_seconds_total{cpu="0", instance="192.168.1.182:9100", job="node", mode="user"}	2035.3
node_cpu_seconds_total{cpu="1", instance="192.168.1.182:9100", job="node", mode="idle"}	1033751.17
node_cpu_seconds_total{cpu="1", instance="192.168.1.182:9100", job="node", mode="iowait"}	1247.46
node_cpu_seconds_total{cpu="1", instance="192.168.1.182:9100", job="node", mode="irq"}	0
node_cpu_seconds_total{cpu="1", instance="192.168.1.182:9100", job="node", mode="nice"}	388.6
node_cpu_seconds_total{cpu="1", instance="192.168.1.182:9100", job="node", mode="softirq"}	575.24
node_cpu_seconds_total{cpu="1", instance="192.168.1.182:9100", job="node", mode="steal"}	0
node_cpu_seconds_total{cpu="1", instance="192.168.1.182:9100", job="node", mode="system"}
```
---

### Exercice 4 : Découverte de service par fichier (file_sd)

**Objectif :** Remplacer les `static_configs` par une découverte dynamique via un fichier JSON. Ajouter ou retirer une cible sans recharger Prometheus.

Créer un unique fichier `sd/targets.json` avec un label `scrape_job` pour distinguer les cibles :

```json
[
  {
    "targets": ["192.168.1.182:9090"],
    "labels": { "scrape_job": "prometheus" }
  },
  {
    "targets": ["192.168.1.182:9100"],
    "labels": { "scrape_job": "node" }
  }
]
```

Mettre à jour `prometheus.yml` — chaque job filtre via `relabel_configs` :

```yaml
  - job_name: 'prometheus'
    file_sd_configs:
      - files: ['/etc/prometheus/sd/targets.json']
        refresh_interval: 5s
    relabel_configs:
      - source_labels: [scrape_job]
        action: keep
        regex: prometheus

  - job_name: 'node'
    file_sd_configs:
      - files: ['/etc/prometheus/sd/targets.json']
        refresh_interval: 5s
    relabel_configs:
      - source_labels: [scrape_job]
        action: keep
        regex: node
```

Relancer le conteneur avec le volume supplémentaire :

```bash
docker rm -f prometheus

docker run -d --name prometheus \
  -p "192.168.1.182:9090:9090" \
  -v /root/docker/ynov/obs/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /root/docker/ynov/obs/sd:/etc/prometheus/sd \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
```

Pour tester la découverte dynamique, modifier `targets.json` (ajouter ou supprimer une entrée) et attendre 5s — Prometheus prend en compte le changement sans rechargement.

Modification du fichier targets.json:
```yaml
[
  {
    "targets": ["192.168.1.182:9090"],
    "labels": { "scrape_job": "prometheus" }
  },
  {
    "targets": ["192.168.1.182:9100"],
    "labels": { "scrape_job": "node" }
  },
  {
   "targets": ["192.168.1.182:9090"],
   "labels": { "scrape_job": "node" }
  }
]
```

```table
node

1 / 2 up

Endpoint	                        Labels	                                                    Last scrape	      State
http://192.168.1.182:9100/metrics instance="192.168.1.182:9100" job="node"  scrape_job="node" 6.887s ago  66ms  up

http://192.168.1.182:9090/metrics instance="192.168.1.182:9090" job="node"  scrape_job="node" 2.144s ago  4ms   unknown
```

---

### Exercice 5 : Règles d'enregistrement (recording rules)

**Objectif :** Pré-calculer une requête PromQL coûteuse et l'enregistrer sous un nouveau nom de métrique, évaluée toutes les 30s.


Copier le fichier zip de de demo-app vers l'hôte et le dezipper :
```bash
scp demo-api.zip root@192.168.1.180:/root/docker/ynov/obs/

unzip /root/docker/ynov/obs/demo-api.zip
```

Création du docker-compose dans le dossier /app:
```yaml
services:
  demo-api:
    build: .
    container_name: demo-api
    ports:
      - "192.168.1.182:8000:8000"
```

Lancement de l'image dans le dossier /app:
```bash
docker compose up -d --build
```

Ajouter un job dans le fichier prometheus.yml:
```yaml
  - job_name: 'demo-api'
    static_configs:
      - targets: ['192.168.1.182:8000']
```

Créer le fichier `rules/api_rules.yml` :

```yaml
groups:
  - name: api
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: rate(demo_http_requests_total[5m])
```

Ajouter `rule_files` dans `prometheus.yml` et monter le répertoire :

```yaml
rule_files:
  - '/etc/prometheus/rules/*.yml'
```

```bash
docker rm -f prometheus

docker run -d --name prometheus \
  -p "192.168.1.182:9090:9090" \
  -v /root/docker/ynov/obs/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /root/docker/ynov/obs/sd:/etc/prometheus/sd \
  -v /root/docker/ynov/obs/rules:/etc/prometheus/rules \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
```

Vérifier dans **Status > Rules** que le groupe `api` est bien chargé, puis interroger la métrique dans l'expression browser :
| Endpoint                                                               | Labels                                        | Last scrape      | State |
| ---------------------------------------------------------------------- | --------------------------------------------- | ---------------- | ----- |
| [http://192.168.1.182:8000/metrics](http://192.168.1.182:8000/metrics) | instance="192.168.1.182:8000", job="demo-api" | 3.195s ago (1ms) | up    |




```
job:http_requests:rate5m
```

```
job:http_requests:rate5m{endpoint="/", instance="192.168.1.182:8000", job="demo-api", method="GET", status="200"}	0.010277105263157893
```

> Une recording rule suit la convention de nommage `<niveau>:<métrique>:<opération>`. **Status > Rules** confirme la fréquence d'évaluation et les éventuelles erreurs.

---

### Exercice 6 : Règles d'alerte et Alertmanager

**Objectif :** Déclencher une alerte quand le taux d'erreur de demo-api dépasse 5% pendant 2 minutes, et la router vers Alertmanager.

Lancer Alertmanager :

```bash
docker run -d --name alertmanager \
  -p "192.168.1.182:9093:9093" \
  prom/alertmanager:latest
```

Créer `alerts/api_alerts.yml` :

```yaml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(demo_http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(demo_http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Taux d'erreur demo-api > 5%"
```

Ajouter le bloc `alerting` dans `prometheus.yml` :

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['192.168.1.182:9093']
```

Recharger Prometheus :

```bash
docker rm -f prometheus

docker run -d --name prometheus \
  -p "192.168.1.182:9090:9090" \
  -v /root/docker/ynov/obs/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /root/docker/ynov/obs/sd:/etc/prometheus/sd \
  -v /root/docker/ynov/obs/rules:/etc/prometheus/rules \
  -v /root/docker/ynov/obs/alerts:/etc/prometheus/alerts \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
```

Pour déclencher l'alerte, générer des erreurs sur demo-api :

```bash
while true; do curl -s http://192.168.1.182:8000/api/orders > /dev/null; done
```

Règle dans Prometheus:
| Field        | Value                                                                                                     |
| ------------ | --------------------------------------------------------------------------------------------------------- |
| Alert        | HighErrorRate                                                                                             |
| Status       | firing (1)                                                                                                |
| Expression   | `sum(rate(demo_http_requests_total{status=~"5.."}[5m])) / sum(rate(demo_http_requests_total[5m])) > 0.05` |
| For          | 2m                                                                                                        |
| Severity     | warning                                                                                                   |
| Summary      | Taux d'erreur demo-api > 5%                                                                               |
| Labels       | alertname="HighErrorRate", severity="warning"                                                             |
| State        | firing                                                                                                    |
| Active Since | 2m 6.217s                                                                                                 |
| Value        | 0.08928571428571429                                                                                       |


AlertManager:
alertname="HighErrorRate"
1 alert
2026-04-27T12:38:19.735Z

Suivre l'alerte dans **Status > Alerts** (état `PENDING` puis `FIRING` après 2 minutes), et l'interface Alertmanager sur [http://192.168.1.182:9093](http://192.168.1.182:9093).

> `for: 2m` = la condition doit rester vraie 2 minutes avant que l'alerte passe en `FIRING`. Le receiver vide dans la config Alertmanager par défaut suffit pour le TP.

---

### Exercice 7 : PromQL — vecteurs instantanés, vecteurs de plage, scalaires

**Objectif :** Comprendre les trois types de résultats PromQL à travers des requêtes sur demo-api.

Ouvrir l'expression browser sur [http://192.168.1.182:9090](http://192.168.1.182:9090) et exécuter les requêtes suivantes :

| Requête | Type de résultat | Ce que ça renvoie |
|---|---|---|
| `demo_http_requests_total` | Vecteur instantané | Un échantillon par série au moment de l'évaluation |
| `demo_http_requests_total[1m]` | Vecteur de plage | Tous les échantillons de la dernière minute par série — non traçable directement |
| `rate(demo_http_requests_total[1m])` | Vecteur instantané | Taux de requêtes par seconde calculé sur 1 minute, un résultat par combinaison de labels |
| `scalar(sum(demo_http_requests_total))` | Scalaire | Somme totale de toutes les requêtes, sans labels |

> `rate()` prend un vecteur de plage en entrée et retourne un vecteur instantané — c'est pourquoi il est traçable dans un graphe.

---

### Exercice 8 : PromQL — agrégations et jointures

**Objectif :** Calculer trois requêtes sur demo-api à partir du trafic généré.

Générer du trafic en parallèle sur les deux endpoints :

```bash
while true; do curl -s http://192.168.1.182:8000/api/users > /dev/null; curl -s http://192.168.1.182:8000/api/orders > /dev/null; done
```

**a) Taux de requêtes total par endpoint**

```
sum by (endpoint) (rate(demo_http_requests_total[5m]))
```

| endpoint    | value                |
| ----------- | -------------------- |
| /           | 0.013793103448275864 |
| /api/orders | 0.9310344827586208   |
| /api/users  | 0.9344827586206897   |


**b) Ratio d'erreurs par endpoint**

```
sum by (endpoint) (rate(demo_http_requests_total{status=~"5.."}[5m]))
/
sum by (endpoint) (rate(demo_http_requests_total[5m]))
```

| endpoint                 | value                |
| ------------------------ | -------------------- |
| {endpoint="/api/orders"} | 0.09893048128342245  |

**c) Taux de requêtes par endpoint, top 3**

```
topk(3, sum by (endpoint) (rate(demo_http_requests_total[5m])))
```

| endpoint    | value                |
| ----------- | -------------------- |
| /api/users  | 2.017241379310345    |
| /api/orders | 2.013793103448276    |
| /           | 0.020689655172413796 |


> Toujours appliquer `rate()` avant `sum()`. `sum by (label)` ne conserve que les labels listés — tous les autres sont supprimés du résultat.

---

### Exercice 9 : PromQL avancé — histogrammes et predict_linear

**Objectif :** Calculer la latence p95 de `/api/orders` et estimer le volume de requêtes dans une heure.

**a) Latence p95 sur les 5 dernières minutes**

```
histogram_quantile(0.95,
  sum by (le, endpoint) (rate(demo_http_request_duration_seconds_bucket[5m]))
)
```

| endpoint    | value              |
| ----------- | ------------------ |
| /api/orders | 0.8423684210526317 |
| /api/users  | 0.4300233644859813 |


**b) Prédiction du nombre de requêtes dans 1 heure**

```
predict_linear(demo_http_requests_total[1h], 3600)
```
| endpoint    | instance           | job      | method | status | value              |
| ----------- | ------------------ | -------- | ------ | ------ | ------------------ |
| /           | 192.168.1.182:8000 | demo-api | GET    | 200    | 16.69300614003364  |
| /api/orders | 192.168.1.182:8000 | demo-api | GET    | 500    | 334.30611912597897 |
| /api/orders | 192.168.1.182:8000 | demo-api | GET    | 200    | 3116.9896297338273 |
| /api/users  | 192.168.1.182:8000 | demo-api | GET    | 200    | 3193.7263890964205 |


> `histogram_quantile()` travaille sur les séries `_bucket` — ne pas oublier de conserver le label `le` dans le `sum by`, sans quoi le calcul est faux.
> `predict_linear(v[d], t)` extrapole la tendance sur la fenêtre `d` et renvoie la valeur estimée à `+t` secondes.

---

### Exercice 10 : Exporter personnalisé — demo-api

**Objectif :** Brancher demo-api comme scrape job et vérifier les quatre métriques `demo_*`.

Ajouter le job dans `sd/targets.json` :

```json
{
  "targets": ["192.168.1.182:8000"],
  "labels": { "scrape_job": "demo-api" }
}
```

Ajouter le job dans `prometheus.yml` :

```yaml
  - job_name: 'demo-api'
    file_sd_configs:
      - files: ['/etc/prometheus/sd/targets.json']
        refresh_interval: 5s
    relabel_configs:
      - source_labels: [scrape_job]
        action: keep
        regex: demo-api
```

Recharger Prometheus :

```bash
curl -X POST http://192.168.1.182:9090/-/reload
```

Générer du trafic pour alimenter les métriques :

```bash
while true; do
  curl -s http://192.168.1.182:8000/api/users > /dev/null
  curl -s http://192.168.1.182:8000/api/orders > /dev/null
done
```

Vérifier dans l'expression browser que les quatre métriques sont présentes :

demo_http_requests_total

| metric                   | endpoint    | instance           | job      | scrape_job | method | status | value |
| ------------------------ | ----------- | ------------------ | -------- | ---------- | ------ | ------ | ----- |
| demo_http_requests_total | /           | 192.168.1.182:8000 | demo-api | demo-api   | GET    | 200    | 13    |
| demo_http_requests_total | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | GET    | 500    | 563   |
| demo_http_requests_total | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | GET    | 200    | 4914  |
| demo_http_requests_total | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | GET    | 200    | 4850  |


demo_http_request_duration_seconds_bucket

| metric                                    | endpoint    | instance           | job      | scrape_job | le    | value |
| ----------------------------------------- | ----------- | ------------------ | -------- | ---------- | ----- | ----- |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 0.005 | 0     |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 0.01  | 0     |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 0.025 | 42    |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 0.05  | 263   |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 0.1   | 749   |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 0.25  | 2188  |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 0.5   | 4562  |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 1.0   | 5561  |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 2.5   | 5561  |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | 5.0   | 5561  |
| demo_http_request_duration_seconds_bucket | /api/orders | 192.168.1.182:8000 | demo-api | demo-api   | +Inf  | 5561  |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 0.005 | 0     |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 0.01  | 0     |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 0.025 | 256   |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 0.05  | 671   |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 0.1   | 1556  |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 0.25  | 4077  |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 0.5   | 4934  |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 1.0   | 4934  |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 2.5   | 4934  |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | 5.0   | 4934  |
| demo_http_request_duration_seconds_bucket | /api/users  | 192.168.1.182:8000 | demo-api | demo-api   | +Inf  | 4934  |


demo_http_requests_in_flight

| metric                       | instance           | job      | scrape_job | value |
| ---------------------------- | ------------------ | -------- | ---------- | ----- |
| demo_http_requests_in_flight | 192.168.1.182:8000 | demo-api | demo-api   | 1     |


demo_active_users

| metric            | instance           | job      | scrape_job | value |
| ----------------- | ------------------ | -------- | ---------- | ----- |
| demo_active_users | 192.168.1.182:8000 | demo-api | demo-api   | 104   |



---

