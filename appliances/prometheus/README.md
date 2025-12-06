# Prometheus Appliance

A Prometheus monitoring system and time series database appliance for collecting and querying metrics.

## Quick Start

```bash
# Launch the appliance
incus launch appliance:prometheus my-prometheus

# Check status
incus exec my-prometheus -- systemctl status prometheus

# Access Web UI
echo "http://$(incus list my-prometheus -c4 --format csv | cut -d' ' -f1):9090"
```

## Configuration

### Default Setup

- Web UI on port 9090
- 15-second scrape interval
- 15-day data retention
- Self-monitoring enabled
- Lifecycle API enabled (for reloads)

### Adding Scrape Targets

Edit `/etc/prometheus/prometheus.yml` to add targets:

#### Static Targets

```yaml
scrape_configs:
  - job_name: 'node-exporters'
    static_configs:
      - targets:
        - '10.0.0.10:9100'
        - '10.0.0.11:9100'
        - '10.0.0.12:9100'
```

#### File-based Service Discovery

```yaml
scrape_configs:
  - job_name: 'file_sd'
    file_sd_configs:
      - files:
        - /etc/prometheus/targets/*.yml
        refresh_interval: 30s
```

Then create target files in `/etc/prometheus/targets/`:

```yaml
# /etc/prometheus/targets/webservers.yml
- targets:
  - 'web1:9100'
  - 'web2:9100'
  labels:
    role: webserver
```

### Connecting to Alertmanager

Edit `/etc/prometheus/prometheus.yml`:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - /etc/prometheus/rules/*.yml
```

### Creating Alert Rules

Create rule files in `/etc/prometheus/rules/`:

```yaml
# /etc/prometheus/rules/alerts.yml
groups:
  - name: node
    rules:
      - alert: NodeDown
        expr: up{job="node"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} is down"
```

### Using cloud-init

Automate configuration at launch:

```yaml
#cloud-config
write_files:
  - path: /etc/prometheus/prometheus.yml
    content: |
      global:
        scrape_interval: 30s
      scrape_configs:
        - job_name: 'prometheus'
          static_configs:
            - targets: ['localhost:9090']
        - job_name: 'nodes'
          static_configs:
            - targets: ['node1:9100', 'node2:9100']
runcmd:
  - systemctl restart prometheus
```

Apply with:

```bash
incus launch appliance:prometheus my-prometheus --config cloud-init.user-data="$(cat cloud-config.yaml)"
```

## Persistence

For production, attach a storage volume:

```bash
# Create a storage volume
incus storage volume create default prometheus-data

# Launch with the volume attached
incus launch appliance:prometheus my-prometheus \
  --device data,source=prometheus-data,path=/var/lib/prometheus/metrics2
```

## API Examples

### Query Metrics

```bash
# Instant query
curl -s 'http://localhost:9090/api/v1/query?query=up'

# Range query (last hour)
curl -s 'http://localhost:9090/api/v1/query_range?query=up&start='$(date -d '1 hour ago' +%s)'&end='$(date +%s)'&step=60'
```

### List Targets

```bash
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, instance: .labels.instance, health: .health}'
```

### Check Configuration

```bash
curl -s http://localhost:9090/api/v1/status/config | jq -r '.data.yaml'
```

### Reload Configuration

```bash
curl -X POST http://localhost:9090/-/reload
```

### Health Check

```bash
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/-/ready
```

## Federation

Set up Prometheus federation to aggregate metrics:

```yaml
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 30s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="node"}'
        - '{__name__=~"job:.*"}'
    static_configs:
      - targets:
        - 'prometheus-dc1:9090'
        - 'prometheus-dc2:9090'
```

## Remote Write

Send metrics to remote storage:

```yaml
remote_write:
  - url: "http://remote-storage:9201/write"
    queue_config:
      max_samples_per_send: 1000
      batch_send_deadline: 5s
```

## Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 9090 | TCP | Web UI and API |

## Volumes

| Path | Description |
|------|-------------|
| `/var/lib/prometheus/metrics2` | Time series database |
| `/etc/prometheus` | Configuration files |

## Health Check

```bash
incus exec my-prometheus -- curl -sf localhost:9090/-/healthy
```

## Common Operations

### Reload Configuration

```bash
# Via API (requires --web.enable-lifecycle)
curl -X POST http://localhost:9090/-/reload

# Or via systemctl
incus exec my-prometheus -- systemctl reload prometheus
```

### Change Retention

Edit `/etc/default/prometheus` and modify `--storage.tsdb.retention.time`:

```bash
ARGS="... --storage.tsdb.retention.time=30d ..."
```

### Check Storage Usage

```bash
incus exec my-prometheus -- du -sh /var/lib/prometheus/metrics2
```

### View Logs

```bash
incus exec my-prometheus -- journalctl -u prometheus -f
```

## Resource Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 1 core | 2+ cores |
| Memory | 256MB | 1GB+ |
| Disk | 1GB | 10GB+ |

Memory and disk requirements scale with the number of time series and retention period.

## Tags

`monitoring`, `metrics`, `time-series`, `observability`, `prometheus`
