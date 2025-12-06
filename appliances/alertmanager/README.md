# Prometheus Alertmanager Appliance

A Prometheus Alertmanager appliance for handling alerts, routing notifications, and managing silences.

## Quick Start

```bash
# Launch the appliance
incus launch appliance:alertmanager my-alertmanager

# Check status
incus exec my-alertmanager -- systemctl status prometheus-alertmanager

# Access Web UI
echo "http://$(incus list my-alertmanager -c4 --format csv | cut -d' ' -f1):9093"
```

## Configuration

### Default Setup

- Web UI on port 9093
- Cluster communication on port 9094
- Default receiver configured (no notifications sent)
- 5-minute resolve timeout

### Connecting to Prometheus

Add Alertmanager to your Prometheus configuration:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['<alertmanager-ip>:9093']

rule_files:
  - /etc/prometheus/rules/*.yml
```

### Configuring Receivers

Edit `/etc/prometheus/alertmanager.yml` to add notification receivers:

#### Email Notifications

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'your-email@gmail.com'
  smtp_auth_password: 'app-password'

receivers:
  - name: 'email-receiver'
    email_configs:
      - to: 'admin@example.com'
```

#### Slack Notifications

```yaml
receivers:
  - name: 'slack-receiver'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
        channel: '#alerts'
        title: '{{ .Status | toUpper }}: {{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
```

#### Webhook Notifications

```yaml
receivers:
  - name: 'webhook-receiver'
    webhook_configs:
      - url: 'http://webhook-handler:5001/alerts'
        send_resolved: true
```

### Routing Alerts

Configure routes to send alerts to different receivers:

```yaml
route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'severity']
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-receiver'
    - match:
        severity: warning
      receiver: 'slack-receiver'
    - match:
        team: database
      receiver: 'dba-email'
```

### Using cloud-init

Automate configuration at launch:

```yaml
#cloud-config
write_files:
  - path: /etc/prometheus/alertmanager.yml
    content: |
      global:
        resolve_timeout: 5m
      route:
        receiver: 'webhook'
        group_wait: 10s
      receivers:
        - name: 'webhook'
          webhook_configs:
            - url: 'http://handler:5001/alerts'
runcmd:
  - systemctl restart prometheus-alertmanager
```

Apply with:

```bash
incus launch appliance:alertmanager my-alertmanager --config cloud-init.user-data="$(cat cloud-config.yaml)"
```

## High Availability

Run multiple Alertmanager instances in a cluster:

```bash
# Launch first instance
incus launch appliance:alertmanager alertmanager-1

# Get first instance IP
AM1_IP=$(incus list alertmanager-1 -c4 --format csv | cut -d' ' -f1)

# Launch second instance with cluster peer
incus launch appliance:alertmanager alertmanager-2
incus exec alertmanager-2 -- bash -c "sed -i 's/ARGS=\"/ARGS=\"--cluster.peer=${AM1_IP}:9094 /' /etc/default/prometheus-alertmanager"
incus exec alertmanager-2 -- systemctl restart prometheus-alertmanager
```

## Persistence

For production, attach a storage volume:

```bash
# Create a storage volume
incus storage volume create default alertmanager-data

# Launch with the volume attached
incus launch appliance:alertmanager my-alertmanager \
  --device data,source=alertmanager-data,path=/var/lib/prometheus/alertmanager
```

## API Examples

### List Alerts

```bash
curl -s http://localhost:9093/api/v2/alerts | jq
```

### Create Silence

```bash
curl -X POST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [{"name": "alertname", "value": "HighCPU", "isRegex": false}],
    "startsAt": "2024-01-01T00:00:00Z",
    "endsAt": "2024-01-02T00:00:00Z",
    "createdBy": "admin",
    "comment": "Maintenance window"
  }'
```

### Check Health

```bash
curl -s http://localhost:9093/-/healthy
curl -s http://localhost:9093/-/ready
```

## Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 9093 | TCP | Web UI and API |
| 9094 | TCP | Cluster communication |

## Volumes

| Path | Description |
|------|-------------|
| `/var/lib/prometheus/alertmanager` | Silences and notification state |
| `/etc/prometheus` | Configuration files |

## Health Check

```bash
incus exec my-alertmanager -- curl -sf localhost:9093/-/healthy
```

## Common Operations

### Reload Configuration

```bash
# Via API
curl -X POST http://localhost:9093/-/reload

# Or via systemctl
incus exec my-alertmanager -- systemctl reload prometheus-alertmanager
```

### View Active Silences

```bash
curl -s http://localhost:9093/api/v2/silences | jq '.[] | select(.status.state == "active")'
```

### Delete a Silence

```bash
curl -X DELETE http://localhost:9093/api/v2/silence/<silence-id>
```

## Resource Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 1 core | 1+ cores |
| Memory | 64MB | 256MB |
| Disk | 128MB | 512MB |

## Tags

`monitoring`, `alerting`, `prometheus`, `observability`, `notifications`
