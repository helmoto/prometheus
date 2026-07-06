# prometheus-agent Helm Chart

Deploys [Prometheus](https://prometheus.io) in **Agent mode** — a lightweight, forward-only scraping mode that ships metrics to a remote_write endpoint (e.g. Thanos, Mimir, Grafana Cloud, VictoriaMetrics).

## Prerequisites

- Kubernetes 1.22+
- Helm 3.8+
- A reachable `remote_write` endpoint

## Install

```bash
# Add chart (if published to a registry)
helm install prometheus-agent ./prometheus-agent \
  --namespace monitoring \
  --create-namespace \
  --set config.remoteWrite[0].url=https://your-remote-write-endpoint/api/v1/write
```

## Key differences from full Prometheus

| Feature | Agent mode | Full Prometheus |
|---|---|---|
| Local query (PromQL) | ❌ | ✅ |
| Alerting rules | ❌ | ✅ |
| Recording rules | ❌ | ✅ |
| Remote write | ✅ | ✅ |
| Low memory footprint | ✅ | ❌ |
| WAL-only storage | ✅ | ❌ |

## Configuration

| Parameter | Description | Default |
|---|---|---|
| `replicaCount` | Number of agent replicas | `1` |
| `image.repository` | Image repository | `prom/prometheus` |
| `image.tag` | Image tag (defaults to chart appVersion) | `""` |
| `config.remoteWrite` | Remote write targets (**required**) | example endpoint |
| `config.scrapeConfigs` | Scrape job definitions | k8s SD configs |
| `persistence.enabled` | Persist WAL to PVC | `true` |
| `persistence.size` | WAL PVC size | `10Gi` |
| `resources` | CPU/memory requests and limits | see values.yaml |
| `rbac.create` | Create ClusterRole for k8s SD | `true` |
| `podDisruptionBudget.enabled` | Enable PDB | `false` |
| `autoscaling.enabled` | Enable HPA | `false` |
| `serviceMonitor.enabled` | Create ServiceMonitor CRD | `false` |
| `networkPolicy.enabled` | Enable NetworkPolicy | `false` |

## High-availability setup

For HA, run 2+ replicas with topology spread constraints:

```yaml
replicaCount: 2

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: prometheus-agent

podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

## Remote write with authentication

```yaml
config:
  remoteWrite:
    - url: https://mimir.example.com/api/v1/push
      basicAuth:
        username:
          name: remote-write-secret
          key: username
        password:
          name: remote-write-secret
          key: password
```

Create the secret separately:

```bash
kubectl create secret generic remote-write-secret \
  --from-literal=username=myuser \
  --from-literal=password=mypassword \
  -n monitoring
```
