# 🧱 ARCHITECTURE (Simple samjho)

```
CLIENT SERVER (Node Exporter :9100)
        ↓
PROMETHEUS SERVER (9090)
        ↓
GRAFANA (3000)
```
---

# 🟢 PART 1: PROMETHEUS + GRAFANA SERVER (RHEL 9)

## 🔹 Step 0: EPEL Repository Setup on RHEL

```
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y
```

---

## 🔹 Step 1: System update

```bash
dnf update -y
```

---

## 🔹 Step 2: Required users banao

```bash
useradd --no-create-home --shell /bin/false prometheus
useradd --no-create-home --shell /bin/false node_exporter
```

---

## 🔹 Step 3: Prometheus install karo

```bash
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.48.1/prometheus-2.48.1.linux-amd64.tar.gz
tar -xvf prometheus-2.48.1.linux-amd64.tar.gz
cd prometheus-2.48.1.linux-amd64
```

### Files copy

```bash
mkdir /etc/prometheus
mkdir /var/lib/prometheus

cp prometheus promtool /usr/local/bin/
cp -r consoles console_libraries /etc/prometheus/
```

### Permission

```bash
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

---

## 🔹 Step 4: Prometheus config (⚠️ YAHI CLIENT IP ADD HOTA HAI)

```bash
vi /etc/prometheus/prometheus.yml
```

### ✅ FINAL WORKING CONFIG

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

👉 `CLIENT_IP` ko **actual client IP** se replace karo
⚠️ Tabs use mat karna, sirf spaces

---

## 🔹 Step 5: Prometheus service

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

## 🔹 Step 6: Validate + Start Prometheus

```bash
promtool check config /etc/prometheus/prometheus.yml
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus
```

Expected:

```
Active: active (running)
```

---

## 🔹 Step 7: Node Exporter (SERVER pe bhi)

```bash
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz
cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### Service

```bash
vi /etc/systemd/system/node_exporter.service
```

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

```bash
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
```

---

## 🔹 Step 8: Grafana install

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

```
yum clean all
yum repolist all
```


```bash
dnf install grafana -y
systemctl enable grafana-server
systemctl start grafana-server
```

---

## 🔹 Step 9: Firewall (SERVER)

```bash
firewall-cmd --permanent --add-port=9090/tcp
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

---

# 🟡 PART 2: CLIENT SERVER (Node Exporter only)

## 🔹 Step 1: Node Exporter install

```bash
useradd --no-create-home --shell /bin/false node_exporter
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz
cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### Service

```bash
vi /etc/systemd/system/node_exporter.service
```

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

```bash
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
```

---

## 🔹 Step 2: Client firewall

```bash
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

---

# 🔵 PART 3: VERIFICATION (MOST IMPORTANT)

## ✅ Prometheus check

```
http://PROMETHEUS_IP:9090
→ Status → Targets
→ node_exporter = UP
```

---

## ✅ Grafana setup

### Login

```
http://GRAFANA_IP:3000
```

### Add Data Source

```
Connections → Data sources → Add
Type: Prometheus
URL: http://localhost:9090
Save & Test (should be SUCCESS)
```

---

### Import Dashboard

```
Dashboards → Import
Dashboard ID: 1860
Select data source: Prometheus
Import
```

---

### Dashboard use

* Dashboards → Browse → **Node Exporter Full**
* **Top-left dropdown** → client IP select karo
* **Top-right** → Last 15 minutes

---

# 🎉 FINAL RESULT

- ✔️ Prometheus running
- ✔️ Client IP added
- ✔️ Node Exporter working
- ✔️ Grafana dashboard showing data
- ✔️ Firewall enabled

---

## 🧠 Yaad rakhne ka ONE-LINE RULE

```
Client add = prometheus.yml edit + restart prometheus
```