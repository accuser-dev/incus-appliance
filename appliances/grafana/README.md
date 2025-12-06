# Grafana Appliance

A Grafana observability and visualization platform for creating dashboards and exploring metrics, logs, and traces.

## Quick Start

```bash
# Launch the appliance
incus launch appliance:grafana my-grafana

# Check status
incus exec my-grafana -- systemctl status grafana-server

# Access Web UI
echo "http://$(incus list my-grafana -c4 --format csv | cut -d' ' -f1):3000"
```

**Default credentials:** `admin` / `admin` (you'll be prompted to change on first login)

## Configuration

### Default Setup

- Web UI on port 3000
- SQLite database
- File-based session storage
- Anonymous access disabled
- Provisioning directories configured

### Changing Admin Password

Via the UI on first login, or via CLI:

```bash
incus exec my-grafana -- grafana-cli admin reset-admin-password newpassword
```

### Adding Datasources

#### Via Provisioning (Recommended)

Edit `/etc/grafana/provisioning/datasources/datasources.yaml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

Then restart Grafana:

```bash
incus exec my-grafana -- systemctl restart grafana-server
```

#### Via API

```bash
curl -X POST http://admin:admin@localhost:3000/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true
  }'
```

### Adding Dashboards

#### Via Provisioning

1. Place JSON dashboard files in `/var/lib/grafana/dashboards/`
2. Grafana automatically loads them (30-second scan interval)

Example dashboard file:

```bash
# Download a community dashboard
curl -o /var/lib/grafana/dashboards/node-exporter.json \
  https://grafana.com/api/dashboards/1860/revisions/latest/download

# Set ownership
chown grafana:grafana /var/lib/grafana/dashboards/node-exporter.json
```

#### Via API

```bash
curl -X POST http://admin:admin@localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d '{
    "dashboard": {
      "title": "My Dashboard",
      "panels": []
    },
    "overwrite": false
  }'
```

### Using cloud-init

Automate configuration at launch:

```yaml
#cloud-config
write_files:
  - path: /etc/grafana/provisioning/datasources/datasources.yaml
    content: |
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://prometheus:9090
          isDefault: true
        - name: Loki
          type: loki
          url: http://loki:3100
  - path: /etc/grafana/grafana.ini
    append: true
    content: |
      [security]
      admin_password = mysecretpassword
runcmd:
  - systemctl restart grafana-server
```

Apply with:

```bash
incus launch appliance:grafana my-grafana --config cloud-init.user-data="$(cat cloud-config.yaml)"
```

## Persistence

For production, attach a storage volume:

```bash
# Create a storage volume
incus storage volume create default grafana-data

# Launch with the volume attached
incus launch appliance:grafana my-grafana \
  --device data,source=grafana-data,path=/var/lib/grafana
```

## Complete Observability Stack

Deploy Grafana with Prometheus and Loki:

```bash
# Deploy the stack
incus launch appliance:prometheus prometheus
incus launch appliance:loki loki
incus launch appliance:grafana grafana

# Configure Grafana datasources
incus exec grafana -- bash -c 'cat > /etc/grafana/provisioning/datasources/stack.yaml << EOF
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus.incusbr0:9090
    isDefault: true
  - name: Loki
    type: loki
    url: http://loki.incusbr0:3100
EOF'

incus exec grafana -- systemctl restart grafana-server
```

## Installing Plugins

```bash
# Install a plugin
incus exec my-grafana -- grafana-cli plugins install grafana-piechart-panel

# Restart to load plugin
incus exec my-grafana -- systemctl restart grafana-server

# List installed plugins
incus exec my-grafana -- grafana-cli plugins ls
```

Popular plugins:
- `grafana-clock-panel` - Clock panel
- `grafana-piechart-panel` - Pie chart
- `grafana-worldmap-panel` - World map
- `grafana-polystat-panel` - Polystat

## API Examples

### Health Check

```bash
curl -s http://localhost:3000/api/health
```

### Get Current User

```bash
curl -s http://admin:admin@localhost:3000/api/user
```

### List Datasources

```bash
curl -s http://admin:admin@localhost:3000/api/datasources
```

### List Dashboards

```bash
curl -s http://admin:admin@localhost:3000/api/search
```

### Create API Key

```bash
curl -X POST http://admin:admin@localhost:3000/api/auth/keys \
  -H "Content-Type: application/json" \
  -d '{"name": "mykey", "role": "Admin"}'
```

## Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 3000 | TCP | Web UI and API |

## Volumes

| Path | Description |
|------|-------------|
| `/var/lib/grafana` | Dashboards, plugins, database |
| `/etc/grafana` | Configuration files |
| `/var/log/grafana` | Log files |

## Health Check

```bash
incus exec my-grafana -- curl -sf localhost:3000/api/health
```

## Common Operations

### Reload Configuration

```bash
# Restart to apply config changes
incus exec my-grafana -- systemctl restart grafana-server
```

### Reset Admin Password

```bash
incus exec my-grafana -- grafana-cli admin reset-admin-password newpassword
```

### Backup Database

```bash
incus exec my-grafana -- sqlite3 /var/lib/grafana/grafana.db ".backup '/tmp/grafana-backup.db'"
incus file pull my-grafana/tmp/grafana-backup.db ./
```

### View Logs

```bash
incus exec my-grafana -- journalctl -u grafana-server -f
# Or
incus exec my-grafana -- tail -f /var/log/grafana/grafana.log
```

### Check Version

```bash
incus exec my-grafana -- grafana-server -v
```

## Resource Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 1 core | 2+ cores |
| Memory | 256MB | 512MB+ |
| Disk | 1GB | 5GB+ |

Requirements scale with number of dashboards, users, and query complexity.

## Tags

`monitoring`, `visualization`, `dashboards`, `observability`, `grafana`
