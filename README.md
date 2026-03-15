
# Prometheus + Grafana Monitoring Setup (RHEL 9)

## Project Overview

In this project, I set up a complete monitoring system using **Prometheus and Grafana on Red Hat Enterprise Linux 9**.

The goal of this setup is to monitor system performance such as:
- CPU usage
- Memory usage
- Disk utilization
- Network activity

Tools used:

- **Prometheus** – Collects and stores monitoring metrics
- **Node Exporter** – Exposes system-level metrics
- **Grafana** – Visualizes metrics using dashboards

Architecture:

Client Servers (Node Exporter)
        |
        |  Metrics (Port 9100)
        |
Prometheus Server
        |
        |  Data Source
        |
Grafana Dashboard

---

# Prerequisites

- RHEL 9 Server
- Root or sudo privileges
- Internet connectivity
- Firewall access

Ports used in this setup:

| Service | Port |
|-------|------|
| Prometheus | 9090 |
| Grafana | 3000 |
| Node Exporter | 9100 |

---

# PART 1: Prometheus + Grafana Server Setup

## Step 0: Enable EPEL Repository

```
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y
```

---

## Step 1: Update System

```
dnf update -y
```

---

## Step 2: Create Required Users

For security reasons I created dedicated users for each service.

```
useradd -s /sbin/nologin prometheus
useradd -s /sbin/nologin node_exporter
```

---

## Step 3: Install Prometheus

Download Prometheus:

```
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.48.1/prometheus-2.48.1.linux-amd64.tar.gz
```

Extract files:

```
tar -xvf prometheus-2.48.1.linux-amd64.tar.gz
cd prometheus-2.48.1.linux-amd64
```

Create directories:

```
mkdir /etc/prometheus
mkdir /var/lib/prometheus
```

Copy binaries:

```
cp prometheus promtool /usr/local/bin/
cp -r consoles console_libraries /etc/prometheus/
```

Set permissions:

```
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

---

## Step 4: Configure Prometheus

Edit configuration file:

```
vim /etc/prometheus/prometheus.yml
```

Configuration:

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets:
          - "localhost:9090"

  - job_name: "node_exporter"
    static_configs:
      - targets:
          - "localhost:9100"
          - "CLIENT_IP:9100"
```

Replace `CLIENT_IP` with the client server IP.

---

## Step 5: Create Prometheus Service

```
vim /etc/systemd/system/prometheus.service
```

```
[Unit]
Description=Prometheus
After=network-online.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus   --config.file=/etc/prometheus/prometheus.yml   --storage.tsdb.path=/var/lib/prometheus

[Install]
WantedBy=multi-user.target
```

---

## Step 6: Start Prometheus

```
promtool check config /etc/prometheus/prometheus.yml

systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus
```

---

## Step 7: Install Node Exporter on Server

```
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz

tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz
cd node_exporter-1.7.0.linux-amd64/

cp node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

---

## Step 8: Create Node Exporter Service

```
vim /etc/systemd/system/node_exporter.service
```

```
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

---

## Step 9: Start Node Exporter

```
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter
```

---

## Step 10: Install Grafana

Add Grafana repository:

```
tee /etc/yum.repos.d/grafana.repo <<EOF
[grafana]
name=Grafana
baseurl=https://packages.grafana.com/oss/rpm
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
EOF
```

---

## Step 11: Install and Start Grafana

```
dnf install grafana -y

systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server
```

---

## Step 12: Configure Firewall

```
firewall-cmd --permanent --add-port=9090/tcp
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

---

# PART 2: Client Server Setup (Node Exporter)

## Install Node Exporter

```
useradd -s /sbin/nologin node_exporter

cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz

tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz
cd node_exporter-1.7.0.linux-amd64/

cp node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

---

## Create Service

```
vim /etc/systemd/system/node_exporter.service
```

```
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

---

## Start Service

```
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter
```

---

## Open Firewall Port

```
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

---

# Verification

## Verify Prometheus

Open browser:

```
http://PROMETHEUS_IP:9090
```

Navigate to:

```
Status → Targets
```

Node exporter should show **UP**.

---

## Verify Node Exporter

```
http://SERVER_IP:9100
```

You should see metrics output.

---

## Verify Grafana

```
http://GRAFANA_IP:3000
```

Default login:

```
Username: admin
Password: admin
```

---

## Add Prometheus Data Source

```
http://localhost:9090
```

---

## Import Dashboard

Dashboard ID:

```
1860
```

Dashboard Name:

```
Node Exporter Full
```

This dashboard provides monitoring for CPU, Memory, Disk, Network and System Load.

---

# Conclusion

In this project I successfully built a **complete monitoring stack using Prometheus and Grafana**.

The setup includes:

- Prometheus for metrics collection
- Node Exporter for system monitoring
- Grafana for visualization

This system helps monitor servers in real time and quickly detect performance issues.
