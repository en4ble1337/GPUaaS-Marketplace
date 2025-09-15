# Prometheus + Grafana GPU Monitoring Setup for Proxmox LXC

A complete guide to set up Prometheus and Grafana monitoring in an LXC container on Proxmox with NVIDIA GPU monitoring capabilities.

## Prerequisites

- Proxmox VE 9.x with NVIDIA drivers already installed on the host
- LXC container running Ubuntu 22.04 with GPU passthrough configured
- GPU accessible in the container (`nvidia-smi` working)
- Root or sudo access in the container

## Architecture Overview

This setup creates a monitoring stack that collects:
- System metrics via Node Exporter
- GPU metrics via custom NVIDIA exporter
- Data visualization through Grafana dashboards

## Part 1: Initial Container Setup

Updates the system and installs essential packages needed for the monitoring stack.

### 1.1 System Updates and Essential Packages

```bash
# Update system packages
apt update && apt upgrade -y

# Install essential packages for monitoring setup
apt install -y curl wget gnupg2 software-properties-common apt-transport-https ca-certificates
```

### 1.2 Create Monitoring Users

Creates dedicated system users for each service for better security isolation.

```bash
# Create service users with no login shell for security
useradd --no-create-home --shell /bin/false prometheus
useradd --no-create-home --shell /bin/false grafana
useradd --no-create-home --shell /bin/false node_exporter
```

## Part 2: Install Prometheus

Prometheus is the time-series database that collects and stores metrics from various exporters.

### 2.1 Download and Install Prometheus

```bash
# Create directories for Prometheus
mkdir -p /etc/prometheus /var/lib/prometheus

# Download Prometheus (check https://github.com/prometheus/prometheus/releases for latest)
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.47.2/prometheus-2.47.2.linux-amd64.tar.gz

# Extract and install binaries
tar xzf prometheus-2.47.2.linux-amd64.tar.gz
cd prometheus-2.47.2.linux-amd64

# Copy binaries to system path
cp prometheus promtool /usr/local/bin/

# Copy web assets
cp -r consoles console_libraries /etc/prometheus/

# Set proper ownership
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

### 2.2 Configure Prometheus

Creates the main configuration file that defines what metrics to collect and from where.

```bash
# Create Prometheus configuration
cat > /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'nvidia-gpu'
    static_configs:
      - targets: ['localhost:9835']
EOF

# Set proper ownership
chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

### 2.3 Create Prometheus Systemd Service

Sets up Prometheus as a system service that starts automatically on boot.

```bash
# Create systemd service file
cat > /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address=0.0.0.0:9090 \
    --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF

# Enable and start Prometheus
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus
```

## Part 3: Install Node Exporter

Node Exporter collects system-level metrics like CPU, memory, disk, and network usage.

### 3.1 Download and Install Node Exporter

```bash
# Download Node Exporter
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz

# Extract and install
tar xzf node_exporter-1.6.1.linux-amd64.tar.gz
cp node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/

# Set ownership
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### 3.2 Create Node Exporter Service

```bash
# Create systemd service for Node Exporter
cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

# Enable and start Node Exporter
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter
```

## Part 4: Install NVIDIA GPU Exporter

Custom Python-based exporter that collects GPU metrics using NVIDIA's management library.

### 4.1 Install Dependencies and Create GPU Exporter

```bash
# Install Python dependencies
apt install -y python3 python3-pip
pip3 install nvidia-ml-py3 prometheus-client

# Create directory for GPU exporter
mkdir -p /opt/gpu-exporter
```

### 4.2 Create GPU Exporter Script

This Python script uses NVIDIA's ML library to collect GPU metrics and expose them in Prometheus format.

```bash
# Create the GPU exporter Python script
cat > /opt/gpu-exporter/nvidia_gpu_exporter.py << 'EOF'
#!/usr/bin/env python3

import time
import pynvml
from prometheus_client import start_http_server, Gauge

# Initialize NVIDIA Management Library
pynvml.nvmlInit()

# Create Prometheus metrics
gpu_temperature = Gauge('nvidia_gpu_temperature_celsius', 'GPU Temperature in Celsius', ['gpu'])
gpu_power_usage = Gauge('nvidia_gpu_power_usage_watts', 'GPU Power Usage in Watts', ['gpu'])
gpu_memory_used = Gauge('nvidia_gpu_memory_used_bytes', 'GPU Memory Used in Bytes', ['gpu'])
gpu_memory_total = Gauge('nvidia_gpu_memory_total_bytes', 'GPU Memory Total in Bytes', ['gpu'])
gpu_utilization = Gauge('nvidia_gpu_utilization_percent', 'GPU Utilization Percentage', ['gpu'])
gpu_memory_utilization = Gauge('nvidia_gpu_memory_utilization_percent', 'GPU Memory Utilization Percentage', ['gpu'])

def collect_gpu_metrics():
    device_count = pynvml.nvmlDeviceGetCount()
    
    for i in range(device_count):
        handle = pynvml.nvmlDeviceGetHandleByIndex(i)
        gpu_name = pynvml.nvmlDeviceGetName(handle).decode('utf-8')
        
        # Temperature
        temp = pynvml.nvmlDeviceGetTemperature(handle, pynvml.NVML_TEMPERATURE_GPU)
        gpu_temperature.labels(gpu=gpu_name).set(temp)
        
        # Power usage
        try:
            power = pynvml.nvmlDeviceGetPowerUsage(handle) / 1000.0  # Convert to watts
            gpu_power_usage.labels(gpu=gpu_name).set(power)
        except pynvml.NVMLError:
            pass
        
        # Memory info
        mem_info = pynvml.nvmlDeviceGetMemoryInfo(handle)
        gpu_memory_used.labels(gpu=gpu_name).set(mem_info.used)
        gpu_memory_total.labels(gpu=gpu_name).set(mem_info.total)
        
        # Utilization
        util = pynvml.nvmlDeviceGetUtilizationRates(handle)
        gpu_utilization.labels(gpu=gpu_name).set(util.gpu)
        gpu_memory_utilization.labels(gpu=gpu_name).set(util.memory)

if __name__ == '__main__':
    # Start HTTP server
    start_http_server(9835)
    print("NVIDIA GPU Exporter started on port 9835")
    
    while True:
        collect_gpu_metrics()
        time.sleep(10)
EOF

# Set permissions
chmod +x /opt/gpu-exporter/nvidia_gpu_exporter.py
chown -R prometheus:prometheus /opt/gpu-exporter
```

### 4.3 Create GPU Exporter Service

```bash
# Create systemd service for GPU exporter
cat > /etc/systemd/system/nvidia-gpu-exporter.service << 'EOF'
[Unit]
Description=NVIDIA GPU Prometheus Exporter
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/bin/python3 /opt/gpu-exporter/nvidia_gpu_exporter.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Enable and start the service
systemctl daemon-reload
systemctl enable nvidia-gpu-exporter
systemctl start nvidia-gpu-exporter
systemctl status nvidia-gpu-exporter
```

## Part 5: Install Grafana

Grafana provides the web-based dashboard interface for visualizing the collected metrics.

### 5.1 Install Grafana

```bash
# Add Grafana GPG key and repository
wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | tee -a /etc/apt/sources.list.d/grafana.list

# Install Grafana
apt update
apt install -y grafana

# Enable and start Grafana
systemctl daemon-reload
systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server
```

## Part 6: Configure Firewall

Sets up UFW firewall rules to allow access to the monitoring services while maintaining security.

### 6.1 Install and Configure UFW

```bash
# Install UFW if not already installed
apt install -y ufw

# Allow SSH access (adjust port if using non-standard)
ufw allow ssh

# Allow Grafana web interface
ufw allow 3000

# Allow Prometheus (optional, for direct access)
ufw allow 9090

# Enable firewall
ufw enable

# Check firewall status
ufw status
```

## Part 7: Verify Installation

### 7.1 Check All Services

Verifies that all components are running correctly.

```bash
# Check service status
systemctl status prometheus node_exporter nvidia-gpu-exporter grafana-server

# Test metric endpoints
curl http://localhost:9090/metrics  # Prometheus
curl http://localhost:9100/metrics  # Node Exporter
curl http://localhost:9835/metrics  # GPU Exporter
```

### 7.2 Verify GPU Metrics

```bash
# Test GPU access
nvidia-smi

# Test Python modules
python3 -c "import pynvml; print('pynvml imported successfully')"
python3 -c "from prometheus_client import start_http_server, Gauge; print('prometheus_client imported successfully')"

# Check GPU metrics are being collected
curl http://localhost:9835/metrics | grep nvidia_gpu
```

## Part 8: Configure Grafana

### 8.1 Initial Grafana Setup

1. Access Grafana at `http://your-container-ip:3000`
2. Default login: username `admin`, password `admin`
3. Change the default password when prompted

### 8.2 Add Prometheus Data Source

1. Go to **Configuration > Data Sources**
2. Click **Add data source**
3. Select **Prometheus**
4. Set URL to `http://localhost:9090`
5. Click **Save & Test**

### 8.3 Create Custom GPU Dashboard

Since the pre-built DCGM dashboards expect different metric names, create a custom dashboard:

1. Go to **Dashboards > New Dashboard**
2. Add panels with these queries:

**GPU Temperature Panel:**
- Query: `nvidia_gpu_temperature_celsius`
- Visualization: Gauge or Stat
- Unit: Celsius (°C)

**GPU Utilization Panel:**
- Query: `nvidia_gpu_utilization_percent`
- Visualization: Gauge
- Unit: Percent (%)

**Memory Usage Panel:**
- Query: `nvidia_gpu_memory_used_bytes / nvidia_gpu_memory_total_bytes * 100`
- Visualization: Gauge
- Unit: Percent (%)

**Power Usage Panel:**
- Query: `nvidia_gpu_power_usage_watts`
- Visualization: Stat
- Unit: Watts (W)

**Memory Usage Over Time:**
- Query: `nvidia_gpu_memory_used_bytes`
- Visualization: Time series
- Unit: Bytes

## Troubleshooting

### Common Issues

**GPU Exporter Not Starting:**
```bash
# Check if script has correct permissions
ls -la /opt/gpu-exporter/nvidia_gpu_exporter.py

# Test script manually
sudo -u prometheus python3 /opt/gpu-exporter/nvidia_gpu_exporter.py

# Check service logs
journalctl -u nvidia-gpu-exporter -f
```

**No GPU Data in Grafana:**
```bash
# Verify GPU metrics are available
curl http://localhost:9835/metrics | grep nvidia

# Check Prometheus is scraping GPU target
# Go to Prometheus UI: http://your-ip:9090
# Status > Targets > check nvidia-gpu job status
```

**Services Not Starting:**
```bash
# Check individual service logs
journalctl -u prometheus -f
journalctl -u node_exporter -f
journalctl -u grafana-server -f

# Verify configuration files
promtool check config /etc/prometheus/prometheus.yml
```

## Access URLs

- **Grafana Dashboard:** `http://your-container-ip:3000`
- **Prometheus UI:** `http://your-container-ip:9090`
- **Node Exporter Metrics:** `http://your-container-ip:9100/metrics`
- **GPU Exporter Metrics:** `http://your-container-ip:9835/metrics`

## Security Notes

- All services run under dedicated non-privileged users
- Firewall rules limit access to necessary ports only
- Consider using reverse proxy with SSL for production deployments
- Change default Grafana admin password immediately

## Performance Monitoring

The setup provides comprehensive monitoring of:
- System resources (CPU, memory, disk, network)
- GPU performance (temperature, utilization, memory, power)
- Service health and availability
- Historical trend analysis through Grafana dashboards

This monitoring stack is particularly useful for:
- AI/ML workload monitoring
- GPU performance optimization
- System health monitoring
- Resource usage tracking
- Performance troubleshooting
