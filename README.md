
# Prometheus + Grafana Monitoring Setup (RHEL 9)

## Project Overview

In this project, I set up a **complete monitoring system using Prometheus and Grafana on Red Hat Enterprise Linux 9**.

The main objective of this setup is to monitor system performance such as:

- CPU usage
- Memory usage
- Disk utilization
- Network activity

For this project I used the following tools:

- **Prometheus** – A monitoring and alerting toolkit that collects metrics.
- **Node Exporter** – An agent that exposes system metrics to Prometheus.
- **Grafana** – A visualization platform used to create dashboards.

### Architecture

```
Client Server(s)  
(Node Exporter running on servers)

        |
        |  Metrics (Port 9100)
        |
Prometheus Server
        |
        |  Data Source
        |
Grafana Dashboard
```




In this setup:

- **Prometheus collects metrics from servers**
- **Node Exporter provides system metrics**
- **Grafana visualizes the collected data using dashboards**

---

# System Architecture

### Monitoring Server
Installed Components:

- Prometheus
- Node Exporter
- Grafana

### Client Server
Installed Components:

- Node Exporter only

Prometheus pulls metrics from servers using **port 9100**.

---

# Prerequisites

Before starting the installation, I ensured the following:

- Operating System: **RHEL 9**
- Root or sudo privileges
- Internet connectivity
- Firewall access

### Ports Used

| Service | Port |
|------|------|
| Prometheus | 9090 |
| Grafana | 3000 |
| Node Exporter | 9100 |

---

# PART 1: Prometheus + Grafana Server Setup (RHEL 9)

In this section I installed **Prometheus, Node Exporter, and Grafana on the monitoring server**.

---

# Step 0: Enable EPEL Repository

First I enabled the **EPEL repository** which provides additional packages for Enterprise Linux.

```
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y
```

---

# Step 1: Update the System

Before installing any software, I updated the system packages.

```
dnf update -y
```

Updating ensures all packages are up to date and avoids dependency issues.

---

# Step 2: Create Required System Users

For security reasons I created **dedicated users** for Prometheus and Node Exporter.

```
useradd -s /sbin/nologin prometheus
useradd -s /sbin/nologin node_exporter
```

Explanation:

- `/sbin/nologin` prevents these users from logging into the system.
- These users will only run their respective services.

---

# Step 3: Install Prometheus

I downloaded the Prometheus binary from the official GitHub release.

```
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.48.1/prometheus-2.48.1.linux-amd64.tar.gz
```

Extract the archive:

```
tar -xvf prometheus-2.48.1.linux-amd64.tar.gz
cd prometheus-2.48.1.linux-amd64
```

### Create Required Directories

Prometheus requires directories for configuration and data storage.

```
mkdir /etc/prometheus
mkdir /var/lib/prometheus
```

### Copy Required Files

I copied the necessary binaries and console files.

```
cp prometheus promtool /usr/local/bin/
cp -r consoles console_libraries /etc/prometheus/
```

### Set Proper Permissions

```
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

This ensures Prometheus has permission to access its configuration and store metrics.

---

# Step 4: Configure Prometheus

Next I created the Prometheus configuration file.

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

Explanation:

- `scrape_interval: 15s` → Prometheus collects metrics every 15 seconds
- `localhost:9100` → monitoring the server itself
- `CLIENT_IP:9100` → monitoring the client machine

I replaced **CLIENT_IP** with the real IP address of the client server.

Save the file:

```
:wq!
```

---

# Step 5: Create Prometheus Systemd Service

To run Prometheus as a background service I created a **systemd service file**.

```
vim /etc/systemd/system/prometheus.service
```

Service configuration:

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

# Step 6: Validate and Start Prometheus

Before starting Prometheus I validated the configuration.

```
promtool check config /etc/prometheus/prometheus.yml
```

Then I started the service:

```
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus
```

---

# Step 7: Install Node Exporter on Server

Node Exporter collects system metrics such as CPU, memory, disk usage etc.

Download Node Exporter:

```
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
```

Extract it:

```
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz
cd node_exporter-1.7.0.linux-amd64/
```

Copy binary:

```
cp node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

---

# Step 8: Create Node Exporter Systemd Service

```
vim /etc/systemd/system/node_exporter.service
```

Service configuration:

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

# Step 9: Start Node Exporter

```
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter
```

---

# Step 10: Install Grafana

First I added the official Grafana repository.

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

# Step 11: Install and Start Grafana

Install Grafana:

```
dnf install grafana -y
```

Start Grafana service:

```
systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server
```

---

# Step 12: Configure Firewall

To allow external access I opened required ports.

```
firewall-cmd --permanent --add-port=9090/tcp
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

---

# PART 2: Client Server Setup (Node Exporter Only)

On the client server I installed **only Node Exporter** so Prometheus can collect metrics.

---

# Step 1: Install Node Exporter

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

# Step 2: Create Node Exporter Service

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

# Step 3: Start Node Exporter

```
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter
```

---

# Step 4: Configure Firewall

```
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

---

# PART 3: Verification

After completing installation I verified all services through the browser.

---

# Step 1: Verify Prometheus

Open browser:

```
http://PROMETHEUS_IP:9090
```

Navigate to:

```
Status → Targets
```

If configuration is correct you will see:

```
node_exporter = UP
```

---

# Step 2: Verify Node Exporter

Open:

```
http://SERVER_OR_CLIENT_IP:9100
```

If Node Exporter is working you will see **raw metrics data**.

---

# Step 3: Verify Grafana

Open:

```
http://GRAFANA_IP:3000
```

Default login:

```
Username: admin
Password: admin
```

---

# Configure Prometheus Data Source in Grafana

Add data source:

```
Prometheus
```

URL:

```
http://localhost:9090
```

or

```
http://SERVER_IP:9090
```

---

# Import Dashboard

To visualize metrics I imported a Grafana dashboard.

Dashboard ID:

```
1860
```

Dashboard Name:

```
Node Exporter Full
```

This dashboard provides monitoring for:

- CPU usage
- Memory usage
- Disk usage
- Network traffic
- System load

---

# Conclusion

In this project I successfully implemented a **complete monitoring stack using Prometheus and Grafana on RHEL 9**.

The setup includes:

- Prometheus for **metrics collection**
- Node Exporter for **system monitoring**
- Grafana for **data visualization**

This monitoring system helps administrators monitor servers in **real time** and quickly detect performance issues.
