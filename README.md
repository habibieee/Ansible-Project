# Ansible Alloy - MSSQL Monitoring Automation

Ansible playbook untuk otomasi deployment dan konfigurasi **Grafana Alloy** pada node-node database MSSQL. Playbook ini mengotomatisasi pengumpulan metrik sistem operasi, metrik MSSQL, dan log error MSSQL, kemudian mengirimkannya ke sistem monitoring terpusat (Prometheus dan Loki).

## 📋 Daftar Isi

- [Fitur](#fitur)
- [Prasyarat](#prasyarat)
- [Struktur Proyek](#struktur-proyek)
- [Instalasi](#instalasi)
- [Konfigurasi](#konfigurasi)
- [Penggunaan](#penggunaan)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)

## ✨ Fitur

- **Otomasi Deployment Lengkap**: Instalasi Grafana Alloy secara otomatis di multiple MSSQL servers
- **Konfigurasi Dinamis**: Menggunakan Jinja2 template untuk konfigurasi yang fleksibel
- **Pengumpulan Metrik Multi-Source**:
  - OS System Metrics (CPU, Memory, Disk, Network)
  - MSSQL Database Metrics
  - MSSQL Error Logs
- **Integrasi Monitoring Terpusat**: 
  - Prometheus untuk metrik time-series
  - Loki untuk log aggregation
- **Idempotent**: Aman untuk dijalankan berkali-kali

## 🔧 Prasyarat

### Sistem yang Dibutuhkan

- **Ansible Control Node** (minimal Ansible 2.9+)
  ```bash
  pip install ansible>=2.9
  ```

- **Target Nodes** (MSSQL Servers)
  - Ubuntu/Debian-based Linux OS
  - SSH access dengan user yang memiliki sudo privileges
  - MSSQL Server 2016+ terinstall
  - Port 1433 terbuka untuk monitoring

- **Monitoring Infrastructure**
  - Prometheus Server running (untuk menerima metrik)
  - Loki Server running (untuk log aggregation)

### Network Requirements

- Akses SSH dari Control Node ke MSSQL Nodes
- Akses outbound dari MSSQL Nodes ke Prometheus dan Loki servers

## 📁 Struktur Proyek

```
ansible-alloy/
├── README.md                          # Dokumentasi proyek (file ini)
├── ansible.cfg                        # Konfigurasi Ansible
├── inventory.ini                      # Inventory hosts dan variables
├── diagram.excalidraw                 # Diagram arsitektur
├── playbook/
│   └── install-alloy.yml              # Main playbook untuk deployment Alloy
└── alloy-config/
    └── config.alloy.j2                # Jinja2 template untuk Alloy configuration
```

### Penjelasan File

| File | Deskripsi |
|------|-----------|
| `ansible.cfg` | Konfigurasi default Ansible (SSH args, inventory path, dll) |
| `inventory.ini` | Daftar target hosts dan variables (credentials, URLs) |
| `install-alloy.yml` | Playbook utama dengan 12 tasks untuk deployment & konfigurasi |
| `config.alloy.j2` | Template konfigurasi Alloy dengan variable injection |

## 🚀 Instalasi

### 1. Clone/Download Repository

```bash
cd /path/to/ansible-alloy
```

### 2. Verifikasi Python dan Ansible Terinstall

```bash
ansible --version
python3 --version
```

### 3. Install Dependencies (jika diperlukan)

```bash
# Install Ansible jika belum ada
pip install ansible>=2.9

# Install Ansible community modules
ansible-galaxy collection install community.general
```

## ⚙️ Konfigurasi

### 1. Update `inventory.ini`

Edit file [inventory.ini](inventory.ini) with target MSSQL servers dan monitoring infrastructure:

```ini
[mssql_nodes]
172.16.11.200 ansible_user=student
172.16.11.201 ansible_user=student
172.16.11.202 ansible_user=student

[mssql_nodes:vars]
# MSSQL Monitoring User Credentials
mssql_monitor_user=GrafanaMonitor
mssql_monitor_pass=admin123

# Central Monitoring Endpoints
prometheus_url=http://172.16.11.200:9090/api/v1/write
loki_url=http://172.16.11.200:3100/loki/api/v1/push
db_user=sa
db_pass=YourSAPassword123
```

**Variabel Penting:**
- `mssql_monitor_user` - User untuk connect ke MSSQL (gunakan least privilege)
- `db_user` / `db_pass` - SA credentials untuk monitoring
- `prometheus_url` - Prometheus remote write endpoint
- `loki_url` - Loki API endpoint

### 2. Konfigurasi SSH Access

Pastikan SSH keys sudah setup:

```bash
# Copy SSH public key ke target nodes
ssh-copy-id -i ~/.ssh/id_rsa.pub student@172.16.11.200
```

### 3. Test Connectivity

```bash
ansible all -i inventory.ini -m ping
```

Expected output:
```
172.16.11.200 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## 🎯 Penggunaan

### Menjalankan Playbook

```bash
# Run playbook dengan verbosity
ansible-playbook -i inventory.ini playbook/install-alloy.yml -v

# Run dengan extra variables
ansible-playbook -i inventory.ini playbook/install-alloy.yml \
  -e "prometheus_url=http://new-prometheus:9090/api/v1/write"

# Run hanya pada host tertentu
ansible-playbook -i inventory.ini playbook/install-alloy.yml \
  --limit 172.16.11.200
```

### Tasks yang Dijalankan

Playbook terdiri dari 12 tasks:

| # | Task | Deskripsi |
|---|------|-----------|
| 1 | Update APT Cache | Update package repository dan install prerequisites |
| 2 | Create APT Keyrings Dir | Buat directory untuk GPG keys |
| 3 | Download Grafana GPG Key | Download GPG key untuk verifikasi paket |
| 4 | Add Grafana APT Repo | Tambah Grafana repository ke APT |
| 5 | Install Grafana Alloy | Install Alloy package dari Grafana repo |
| 6 | Deploy Config Template | Deploy Alloy configuration dengan variable injection |
| 7 | Flush Handlers | Restart service immediately |
| 8 | Pause | Tunggu Alloy boot up (3 detik) |
| 9-10 | Check Status | Verifikasi service status |
| 11-12 | View Logs | Display service logs untuk debugging |

## 📊 Monitoring

### Metrics yang Dikumpulkan

**OS Metrics** (via Node Exporter):
- CPU utilization
- Memory usage
- Disk I/O
- Network interfaces
- System load

**MSSQL Metrics**:
- Database size
- Transaction log usage
- Connections count
- Query statistics
- Lock waits
- Page life expectancy

**MSSQL Logs**:
- Error logs dari `/var/opt/mssql/log/errorlog*`
- Startup messages
- Backup/restore events

### Mengakses Metrics di Grafana

1. Buka Grafana: `http://<prometheus-server>:3000`
2. Explore → Prometheus
3. Query example:
   ```promql
   # CPU Usage
   node_cpu_seconds_total{instance=~"172.16.11.*"}
   
   # MSSQL Connections
   mssql_connections{instance=~"172.16.11.*"}
   ```

### Mengakses Logs di Loki

1. Dashboard → Explore → Loki
2. Label filters:
   ```
   {job="mssql_errorlog", instance="172.16.11.200"}
   ```

## 🔍 Troubleshooting

### Issue: "Failed to connect to Alloy service"

```bash
# SSH ke target node
ssh student@172.16.11.200

# Check service status
sudo systemctl status alloy

# View logs
sudo journalctl -u alloy -n 50

# Restart service manually
sudo systemctl restart alloy
```

### Issue: "Connection refused to MSSQL"

```bash
# Verify MSSQL berjalan
systemctl status mssql-server

# Test connection string
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P '<password>'
```

### Issue: Metrics tidak muncul di Prometheus

1. Verifikasi Alloy config: `cat /etc/alloy/config.alloy`
2. Check Prometheus remote write endpoint: `curl -v http://prometheus:9090/api/v1/write`
3. Inspect Alloy logs: `journalctl -u alloy -f`

### Reset Configuration (jika diperlukan)

```bash
# SSH ke target node
ssh student@172.16.11.200

# Stop service
sudo systemctl stop alloy

# Restore default config
sudo cp /etc/alloy/config.alloy /etc/alloy/config.alloy.backup
sudo systemctl start alloy

# Re-run playbook
ansible-playbook -i inventory.ini playbook/install-alloy.yml
```

## 📝 Customization

### Modify Alloy Configuration

Edit [config.alloy.j2](alloy-config/config.alloy.j2) untuk menambah collectors atau change forwarding targets:

```alloy
// Contoh: Tambah custom exporter
prometheus.exporter.mysql "local_mysql" {
  data_source_name = "root:password@localhost:3306/"
}

prometheus.scrape "scrape_mysql" {
  targets    = prometheus.exporter.mysql.local_mysql.targets
  forward_to = [prometheus.remote_write.central_prom.receiver]
}
```

### Change Monitoring Interval

Tambah scrape interval ke inventory:

```ini
[mssql_nodes:vars]
scrape_interval=30s  # Default 15s
```

## 📚 Referensi

- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/)
- [Grafana Alloy GitHub](https://github.com/grafana/alloy)
- [Ansible Documentation](https://docs.ansible.com/)
- [Prometheus Remote Write API](https://prometheus.io/docs/prometheus/latest/storage/#remote_storage_integrations)
- [Loki API Documentation](https://grafana.com/docs/loki/latest/api/)


---

**Last Updated**: April 2026  
**Ansible Version**: 2.9+  
**Target OS**: Ubuntu 20.04+, Debian 10+