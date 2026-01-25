# 📊 Prometheus & Grafana Monitoring Setup (RHEL 9)

This repository documents how **I designed and implemented a complete monitoring stack** using **Prometheus, Node Exporter, and Grafana** on RHEL 9 systems.

The goal of this project was to build a **clean, production-style monitoring setup** that is easy to understand, easy to scale, and suitable for real-world Linux environments. Everything here is written in **first-person**, exactly the way I implemented it.

---

## 🧱 Architecture Overview

I followed a very simple and practical architecture:

```
CLIENT SERVER (Node Exporter :9100)
        ↓
PROMETHEUS SERVER (9090)
        ↓
GRAFANA (3000)
```

* Each client exposes system metrics using **Node Exporter**
* The **Prometheus server** scrapes metrics from all clients
* **Grafana** is used for visualization and dashboards

---

# 🟢 PART 1: Prometheus + Grafana Server (RHEL 9)

### 🔹 Step 0: Enable EPEL Repository

First, I enabled the EPEL repository on my RHEL 9 server:

```bash
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y
```

---

### 🔹 Step 1: System Update

Before installing anything, I updated the system to avoid dependency issues:

```bash
dnf update -y
```

---

### 🔹 Step 2: Create Required System Users

For security and best practices, I created dedicated system users:

```bash
useradd --no-create-home --shell /bin/false prometheus
useradd --no-create-home --shell /bin/false node_exporter
```

---

### 🔹 Step 3: Install Prometheus

I downloaded and extracted the Prometheus binary manually:

```bash
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.48.1/prometheus-2.48.1.linux-amd64.tar.gz
tar -xvf prometheus-2.48.1.linux-amd64.tar.gz
cd prometheus-2.48.1.linux-amd64
```

Then I created required directories and copied files:

```bash
mkdir /etc/prometheus
mkdir /var/lib/prometheus

cp prometheus promtool /usr/local/bin/
cp -r consoles console_libraries /etc/prometheus/
```

Permissions were set properly:

```bash
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

---

### 🔹 Step 4: Configure Prometheus

This is the most important step. Here I define **which targets Prometheus will monitor**.

```bash
vi /etc/prometheus/prometheus.yml
```

#### ✅ Final Working Configuration

```yaml
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

> 🔹 I replace `CLIENT_IP` with the actual client server IP
> 🔹 Only spaces are used (no tabs)

---

### 🔹 Step 5: Create Prometheus Systemd Service

```bash
vi /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
After=network-online.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus

[Install]
WantedBy=multi-user.target
```

---

### 🔹 Step 6: Validate and Start Prometheus

```bash
promtool check config /etc/prometheus/prometheus.yml
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus
```

Expected result:

```
Active: active (running)
```

---

### 🔹 Step 7: Install Node Exporter on Server

Even the Prometheus server monitors itself using Node Exporter:

```bash
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz
cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Systemd service:

```ini
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

### 🔹 Step 8: Install Grafana

```bash
tee /etc/yum.repos.d/grafana.repo <<EOF
[grafana]
name=Grafana
baseurl=https://packages.grafana.com/oss/rpm
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
EOF
```

```bash
dnf install grafana -y
systemctl enable grafana-server
systemctl start grafana-server
```

---

### 🔹 Step 9: Configure Firewall

```bash
firewall-cmd --permanent --add-port=9090/tcp
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

---

# 🟡 PART 2: Client Server (Node Exporter Only)

On each client server, I installed only Node Exporter:

```bash
useradd --no-create-home --shell /bin/false node_exporter
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz
cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Firewall:

```bash
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

---

# 🔵 PART 3: Verification

### ✅ Prometheus

```
http://PROMETHEUS_IP:9090
→ Status → Targets
→ node_exporter = UP
```

### ✅ Grafana

* URL: `http://GRAFANA_IP:3000`
* Default login: `admin / admin`
* Data source: Prometheus (`http://localhost:9090`)
* Dashboard ID used: **1860 (Node Exporter Full)**

---

# 🎉 Final Outcome

* Prometheus successfully scraping all nodes
* Node Exporter running on server and clients
* Grafana dashboards displaying real-time metrics
* Firewall and services configured properly

---

## 🧠 Key Learning (One-Line Rule)

```
Adding a new client = update prometheus.yml + restart Prometheus
```

---

### 📌 Conclusion

This project gave me practical experience in setting up a Linux monitoring system using Prometheus, Node Exporter, and Grafana.
I configured services, firewall rules, and monitoring targets in a way that reflects real-world system administration practices. The setup is simple, reliable, and easy to manage.
The architecture allows new servers to be added easily by updating the Prometheus configuration, making it suitable for scalable and enterprise environments.
Overall, this project helped me strengthen my understanding of monitoring, Linux administration, and production-style service management.

---

## ✍️ Author

**Vishwaraj Kumar**  
🔗 [GitHub Profile](https://github.com/vishwaraj-kumar)  
🔗 [LinkedIn Profile](https://www.linkedin.com/in/vishwaraj-kumar/)