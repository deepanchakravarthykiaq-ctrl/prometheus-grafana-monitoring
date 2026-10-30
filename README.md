# Prometheus + Node Exporter + Grafana + Alertmanager

A Linux server monitoring lab for Ubuntu. Prometheus scrapes its own metrics and Node Exporter host metrics; Grafana visualizes them; Prometheus alert rules send notifications to Alertmanager.

## Architecture and ports

| Service | Port | URL |
|---|---:|---|
| Prometheus | 9090 | http://SERVER_IP:9090 |
| Node Exporter | 9100 | http://SERVER_IP:9100/metrics |
| Grafana | 3000 | http://SERVER_IP:3000 |
| Alertmanager | 9093 | http://SERVER_IP:9093 |

## Requirements

- Ubuntu 22.04/24.04 (or compatible Debian-based Linux)
- sudo access and internet connectivity
- At least 2 GB RAM recommended for this practice lab

## Install

Clone this repository, then run:

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/prometheus-grafana-monitoring.git
cd prometheus-grafana-monitoring
sudo bash scripts/install-ubuntu.sh
```

The script installs Prometheus, Node Exporter, Alertmanager and Grafana, copies the repository configurations into their system locations, enables services and starts them.

## Verify services

```bash
sudo systemctl status prometheus --no-pager
sudo systemctl status prometheus-node-exporter --no-pager
sudo systemctl status prometheus-alertmanager --no-pager
sudo systemctl status grafana-server --no-pager
```

Check endpoints:

```bash
curl -s http://localhost:9090/-/ready
curl -s http://localhost:9090/api/v1/targets
curl -s http://localhost:9100/metrics | head
curl -s http://localhost:9093/-/ready
curl -s http://localhost:3000/api/health
```

In Prometheus, open **Status → Targets**. Both `prometheus` and `node-exporter` should show **UP**.

## Grafana

1. Open `http://SERVER_IP:3000`.
2. Sign in with `admin` / `admin` on a fresh package installation and change the password when prompted.
3. The Prometheus datasource is provisioned automatically.
4. Open **Dashboards → Linux Monitoring → Linux Node Monitoring**.
5. Select an instance from the dashboard's **Instance** variable.

The dashboard includes target status, CPU, memory, root disk usage, network receive rate, load average and available memory.

## PromQL

See [docs/promql-queries.md](docs/promql-queries.md) for eight queries and explanations.

## Alerting

Prometheus loads `prometheus/rules/node-alerts.yml` and sends alerts to Alertmanager on port 9093. Included rules:

- `InstanceDown`: Node Exporter target unavailable for 1 minute.
- `HighCPUUsage`: CPU above 80% for 2 minutes.
- `HighMemoryUsage`: memory above 85% for 2 minutes.
- `LowDiskSpace`: root filesystem free space below 15% for 2 minutes.

### Verify a firing alert safely

On a lab machine only, stop Node Exporter temporarily:

```bash
sudo systemctl stop prometheus-node-exporter
```

Wait at least 75–90 seconds, then open:

- Prometheus: `http://SERVER_IP:9090/alerts`
- Alertmanager: `http://SERVER_IP:9093`

`InstanceDown` should become **FIRING** and appear in Alertmanager. Restore monitoring immediately:

```bash
sudo systemctl start prometheus-node-exporter
```

Wait for the next scrape; the alert should resolve. Do not stop the exporter on a production host.

### Configure email delivery

The committed `alertmanager/alertmanager.yml` intentionally contains example SMTP values. Before expecting email delivery, edit `/etc/alertmanager/alertmanager.yml` with your SMTP host, sender, recipient and a secret/app password. Do not commit real passwords or tokens to Git. Then run:

```bash
sudo systemctl restart prometheus-alertmanager
sudo systemctl status prometheus-alertmanager --no-pager
```

## Remote EC2 access and security

If running on EC2, allow inbound TCP ports only from your own public IP or trusted network:
- 3000 (Grafana)
- 9090 (Prometheus)
- 9093 (Alertmanager)
- 9100 (Node Exporter)

For a real environment, do not expose these monitoring interfaces publicly. Prefer VPN, SSH tunnel, private subnets or a bastion, and restrict Node Exporter access to the Prometheus server. If Prometheus and Node Exporter are on different hosts, replace `localhost:9100` in `prometheus.yml` with the Node Exporter server's private IP and configure the security group/firewall accordingly.

## Useful troubleshooting

```bash
sudo journalctl -u prometheus -n 100 --no-pager
sudo journalctl -u prometheus-node-exporter -n 100 --no-pager
sudo journalctl -u prometheus-alertmanager -n 100 --no-pager
sudo journalctl -u grafana-server -n 100 --no-pager
sudo promtool check config /etc/prometheus/prometheus.yml
sudo promtool check rules /etc/prometheus/rules/node-alerts.yml
```

If a target is DOWN, verify the service is active, the target address/port is correct, and network rules allow the scrape connection.

## Repository structure

```text
prometheus-grafana-monitoring/
├── alertmanager/
│   └── alertmanager.yml
├── docs/
│   └── promql-queries.md
├── grafana/
│   ├── dashboards/
│   │   └── linux-node-monitoring.json
│   └── provisioning/
│       ├── dashboards/dashboards.yml
│       └── datasources/prometheus.yml
├── prometheus/
│   ├── prometheus.yml
│   └── rules/node-alerts.yml
├── scripts/
│   └── install-ubuntu.sh
└── README.md
```

## Submission evidence checklist

- [ ] Screenshot of Prometheus **Status → Targets** showing all targets UP.
- [ ] `prometheus/prometheus.yml` configuration file.
- [ ] Screenshot of the Grafana Linux Node Monitoring dashboard.
- [ ] `docs/promql-queries.md` with queries and explanations.
- [ ] Alert rules file and screenshots showing `InstanceDown` FIRING in Prometheus/Alertmanager.
- [ ] GitHub repository URL.
- [ ] This README documentation.
