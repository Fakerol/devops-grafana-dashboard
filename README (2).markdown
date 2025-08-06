# Grafana Dashboard Setup for System Monitoring

This guide provides a comprehensive, step-by-step process to install and configure **Grafana** and **Prometheus** on a Linux server for system monitoring. It includes setting up Prometheus for data collection, Node Exporter for server metrics, and Grafana for visualization, along with optional alerting.

---

## 📌 Prerequisites

- A Linux-based server (e.g., Ubuntu, Rocky Linux, CentOS, RHEL).
- `firewalld` enabled (if using a firewall).
- Internet access to download packages.
- Root or sudo access.
- Monitoring server IP: `192.168.1.10`.
- Server to monitor metrics IP: `192.168.1.11`.

---

## 🧱 Step 1: Install Prometheus

Prometheus is installed on the monitoring server (`192.168.1.10`) to collect and store metrics.

### 1.1 Create Prometheus User

Create a system user for Prometheus:

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```

### 1.2 Download and Install Prometheus

Download and extract the Prometheus binary:

```bash
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
tar xvf prometheus-2.52.0.linux-amd64.tar.gz
cd prometheus-2.52.0.linux-amd64
```

Move binaries and configuration files:

```bash
sudo mv prometheus /usr/local/bin/
sudo mv promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo mv consoles/ console_libraries/ prometheus.yml /etc/prometheus/
```

### 1.3 Set Permissions

Assign ownership to the Prometheus user:

```bash
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

### 1.4 Create Systemd Service for Prometheus

Create a systemd service file:

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Add the following content:

```
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

### 1.5 Start Prometheus

Reload systemd and start Prometheus:

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

### 1.6 Configure Firewall

Allow Prometheus web interface (port 9090):

```bash
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload
```

### 1.7 Verify Prometheus

Open a browser and navigate to:

```
http://192.168.1.10:9090
```

You should see the Prometheus web interface.

---

## 🧱 Step 2: Install Node Exporter (For Server Metrics)

Node Exporter collects system metrics (e.g., CPU, RAM, disk) on the server to be monitored (`192.168.1.11`).

### 2.1 Download and Extract Node Exporter

```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.1.linux-amd64.tar.gz
cd node_exporter-1.8.1.linux-amd64
```

Move the binary:

```bash
sudo mv node_exporter /usr/local/bin/
```

### 2.2 Create Systemd Service for Node Exporter

Create a systemd service file:

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Add the following content:

```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

### 2.3 Start Node Exporter

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

### 2.4 Verify Node Exporter

Check metrics in a browser:

```
http://192.168.1.11:9100/metrics
```

### 2.5 Configure Prometheus to Scrape Node Exporter

On the monitoring server (`192.168.1.10`), edit the Prometheus configuration:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add the following under `scrape_configs`:

```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['192.168.1.11:9100']  # Add all server IPs to monitor
```

### 2.6 Restart Prometheus

Apply the changes:

```bash
sudo systemctl restart prometheus
```

---

## 🧱 Step 3: Install Grafana

Grafana is installed on the monitoring server (`192.168.1.10`) for visualization.

### 3.1 Install Grafana

For Ubuntu/Debian-based systems:

```bash
sudo apt-get install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update
sudo apt-get install grafana
```

For Rocky Linux/CentOS/RHEL:

```bash
sudo yum install -y https://dl.grafana.com/oss/release/grafana-11.2.0-1.x86_64.rpm
```

### 3.2 Start Grafana

```bash
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

### 3.3 Configure Firewall

Allow Grafana web interface (port 3000):

```bash
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

### 3.4 Access Grafana

Open a browser and navigate to:

```
http://192.168.1.10:3000
```

- Default credentials: `admin` / `admin` (change the password on first login).

### 3.5 Connect Grafana to Prometheus

1. In Grafana, go to **Configuration** → **Data Sources** → **Add data source**.
2. Select **Prometheus**.
3. Set the URL to: `http://192.168.1.10:9090`.
4. Click **Save & Test**.

### 3.6 Import a Dashboard

1. In Grafana, click **+** → **Import**.
2. Use a community dashboard, e.g., **Node Exporter Full** (ID: `1860`).
3. Select the Prometheus data source and import.
4. The dashboard will display CPU, RAM, disk, network, and other metrics.

---

## 🧱 Step 4: Set Alerts (Optional)

Configure alerts in Grafana for proactive monitoring.

1. Go to a dashboard panel (e.g., CPU usage).
2. Click **Edit** → **Alert** → **Create Alert Rule**.
3. Example: Alert when CPU usage &gt; 80% for 5 minutes.
4. Configure notifications in **Alerting** → **Contact Points** (e.g., email, Slack).

---

## 📝 Notes

- Replace `192.168.1.10` and `192.168.1.11` with your actual server IPs.
- Ensure firewalls allow traffic on ports `9090` (Prometheus) and `3000` (Grafana).
- For production, secure Grafana with strong credentials and SSL.
- To monitor additional servers, install Node Exporter on each and add their IPs to `prometheus.yml`.
- Regularly check Prometheus and Grafana logs for issues:

  ```bash
  sudo journalctl -u prometheus
  sudo journalctl -u grafana-server
  ```

---

**Prepared by**: Fakhrul Najmi