# Nginx Appliance

A production-ready Nginx reverse proxy and web server appliance based on Debian with cloud-init support for automated configuration.

## Quick Start

```bash
# Launch the appliance
incus launch appliance:nginx my-proxy

# Check status
incus exec my-proxy -- nginx -t

# View the default page
incus exec my-proxy -- curl -s localhost
```

## Features

- **Debian-based**: Stable Debian Bookworm with broad compatibility
- **cloud-init support**: Automated last-mile configuration
- **Production-ready**: Optimized configuration with gzip, security headers
- **Health endpoint**: Built-in `/health` endpoint for monitoring
- **Log rotation**: Automatic log rotation configured
- **Easy configuration**: Drop configs in `/etc/nginx/conf.d/`

## Cloud-Init Configuration

This appliance supports cloud-init for automated configuration at launch time.

### Basic Example

```bash
# Create instance without starting
incus init appliance:nginx my-proxy

# Configure via cloud-init
incus config set my-proxy cloud-init.user-data - << 'EOF'
#cloud-config
write_files:
  - path: /etc/nginx/conf.d/mysite.conf
    content: |
      server {
        listen 80;
        server_name example.com;
        location / {
          proxy_pass http://backend:8080;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
        }
      }

runcmd:
  - nginx -t
  - systemctl reload nginx
EOF

# Start the instance
incus start my-proxy
```

### Advanced Example with SSL

```bash
incus init appliance:nginx my-proxy

incus config set my-proxy cloud-init.user-data - << 'EOF'
#cloud-config
packages:
  - certbot
  - python3-certbot-nginx

write_files:
  - path: /etc/nginx/conf.d/mysite.conf
    content: |
      server {
        listen 80;
        server_name example.com;
        root /var/www/html;

        location /.well-known/acme-challenge/ {
          root /var/www/html;
        }

        location / {
          return 301 https://$host$request_uri;
        }
      }

runcmd:
  - mkdir -p /var/www/html
  - nginx -t && systemctl reload nginx
  # Uncomment after DNS is configured:
  # - certbot --nginx -d example.com --non-interactive --agree-tos -m admin@example.com
EOF

incus start my-proxy
```

### Network Configuration

```bash
incus config set my-proxy cloud-init.network-config - << 'EOF'
version: 2
ethernets:
  eth0:
    addresses:
      - 10.0.0.10/24
    gateway4: 10.0.0.1
    nameservers:
      addresses: [8.8.8.8, 8.8.4.4]
EOF
```

## Manual Configuration

### Adding a Site

Create a configuration file on your host:

```nginx
# mysite.conf
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Push it to the appliance:

```bash
incus file push mysite.conf my-proxy/etc/nginx/conf.d/
incus exec my-proxy -- nginx -t
incus exec my-proxy -- systemctl reload nginx
```

### Editing Configuration

```bash
# Edit config inside the container
incus exec my-proxy -- vi /etc/nginx/conf.d/default.conf

# Test and reload
incus exec my-proxy -- nginx -t && systemctl reload nginx
```

## Networking

Expose ports using Incus devices:

```bash
# Proxy port 80 from host to container
incus config device add my-proxy http proxy listen=tcp:0.0.0.0:80 connect=tcp:127.0.0.1:80

# Proxy port 443 for HTTPS
incus config device add my-proxy https proxy listen=tcp:0.0.0.0:443 connect=tcp:127.0.0.1:443
```

## Persistent Data

Important directories to persist (use disk devices or snapshots):

- `/etc/nginx/conf.d/` - Site configurations
- `/var/log/nginx/` - Log files
- `/usr/share/nginx/html/` - Static content

Example:

```bash
# Create a storage volume for configs
incus storage volume create default nginx-config
incus config device add my-proxy config disk source=nginx-config path=/etc/nginx/conf.d
```

## SSL/TLS Certificates

### Using Let's Encrypt with Certbot

```bash
# Install certbot inside the appliance
incus exec my-proxy -- apt-get update
incus exec my-proxy -- apt-get install -y certbot python3-certbot-nginx

# Obtain a certificate
incus exec my-proxy -- certbot --nginx -d example.com

# Auto-renewal is configured automatically
```

### Using External Certificates

```bash
# Create SSL directory
incus exec my-proxy -- mkdir -p /etc/nginx/ssl

# Push certificates to the appliance
incus file push cert.pem my-proxy/etc/nginx/ssl/cert.pem
incus file push key.pem my-proxy/etc/nginx/ssl/key.pem

# Set permissions
incus exec my-proxy -- chmod 600 /etc/nginx/ssl/key.pem
```

## Monitoring

### Health Check

```bash
# Check health endpoint
incus exec my-proxy -- curl -sf http://localhost/health
```

### Logs

```bash
# Access logs
incus exec my-proxy -- tail -f /var/log/nginx/access.log

# Error logs
incus exec my-proxy -- tail -f /var/log/nginx/error.log

# Both
incus exec my-proxy -- tail -f /var/log/nginx/*.log
```

### Service Status

```bash
incus exec my-proxy -- systemctl status nginx
```

## Common Tasks

### Reload Configuration

```bash
incus exec my-proxy -- systemctl reload nginx
```

### Test Configuration

```bash
incus exec my-proxy -- nginx -t
```

### Restart Nginx

```bash
incus exec my-proxy -- systemctl restart nginx
```

### View Version

```bash
incus exec my-proxy -- nginx -v
```

## Troubleshooting

### Configuration Errors

```bash
# Validate config syntax
incus exec my-proxy -- nginx -t

# Check error logs
incus exec my-proxy -- cat /var/log/nginx/error.log
```

### Service Not Starting

```bash
# Check service status
incus exec my-proxy -- systemctl status nginx

# View journal logs
incus exec my-proxy -- journalctl -u nginx
```

### Cloud-Init Issues

```bash
# Check cloud-init status
incus exec my-proxy -- cloud-init status

# View cloud-init logs
incus exec my-proxy -- cat /var/log/cloud-init-output.log
```

### Permission Issues

```bash
# Check www-data user exists
incus exec my-proxy -- id www-data

# Fix permissions
incus exec my-proxy -- chown -R www-data:www-data /var/log/nginx
incus exec my-proxy -- chown -R www-data:www-data /var/cache/nginx
```

## Resource Requirements

- **Minimum CPU**: 1 core
- **Minimum Memory**: 128MB
- **Minimum Disk**: 256MB
- **Recommended Memory**: 256MB
- **Recommended Disk**: 1GB

## See Also

- [Traefik Appliance](../traefik/) - Modern reverse proxy with automatic HTTPS
- [Caddy Appliance](../caddy/) - Web server with automatic HTTPS
