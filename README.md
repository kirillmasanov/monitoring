# Deploy Prometheus and Grafana

Below is a guide for installing *Prometheus* and *Grafana* using *Docker Compose*. As well as installing *node-exporter* and *mysqld_exporter* and connecting them to Prometheus. Adding the corresponding dashboards to Grafana.
```
Versions used:
- Ubuntu 22.04.4 LTS
- Grafana v11.2.0
- Prometheus 2.54.1
- node_exporter 1.8.2
- mysqld_exporter 0.15.1
- blackbox_exporter 0.25.0
```
## Deploy with Dockercompose
([Docker docs](https://docs.docker.com/engine/install/ubuntu/))
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
```bash
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
```bash
# Install the latest version
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-compose 
```
```bash
# Verify
sudo docker run hello-world
``` 
```bash
# Create a new directory for your Prometheus and Grafana configuration files:
mkdir /opt/monitoring
cd /opt/monitoring
```
```bash
# Create a new Docker Compose file:
vi docker-compose.yml
```
```yaml
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml
    - ./prometheus-data:/prometheus
    command:
    - --config.file=/etc/prometheus/prometheus.yml
    - --storage.tsdb.retention.time=30d
    - --storage.tsdb.retention.size=50GB
    ports:
    - "9090:9090"
    networks:
    - monitoring
    restart: always

  grafana:
    image: grafana/grafana
    container_name: grafana
    volumes:
    - ./grafana-data:/var/lib/grafana
    ports:
    - "3000:3000"
    networks:
    - monitoring
    restart: always

volumes:
  grafana-data:
  prometheus-data:

networks:
  monitoring:
```
```bash
# Create local folders for volumes
mkdir prometheus-data
chmod 777 prometheus-data
    
mkdir grafana-data
chmod 777 grafana-data
```
```bash
# Create a new Prometheus configuration file
vi prometheus.yml
```
```yaml
global:
  scrape_interval: 15s

scrape_configs:
- job_name: 'prometheus'
  scrape_interval: 5s
  static_configs:
  - targets: ['localhost:9090']
  ```
  ```bash
  # Start the Docker containers
  docker compose up -d
  ```
  Grafana is available at `<ip>:3000`. 
  
  Default credential: *login:admin, password:admin*

  Grafana setting: *Connections > Data sources > Add data source > Prometheus >  http://prometheus:9090/ > Save & test*

Docker compose using:
```bash
# Stop and remove containers, networks
docker compose down

# Remove named volumes declared in the "volumes" section of the Compose file and anonymous volumes attached to containers
docker compose down -v

# Create and start containers
docker compose up

# Detached mode: Run containers in the background
docker compose up -d

# View output from containers
docker compose logs
docker compose logs -f
docker compose logs -f [service_name]

# List containers
docker compose ps

# Execute a command in a running container
docker compose exec [service name] [command]

# List images used by the created containers
docker compose images
```
## node_exporter

[Github releases](https://github.com/prometheus/node_exporter/releases)
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvf node_exporter-1.8.2.linux-amd64.tar.gz
cd node_exporter-1.8.2.linux-amd64
cp node_exporter /usr/local/bin
useradd --no-create-home --shell /bin/false node_exporter
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```
```bash
# Check version of node_exporter
node_exporter --version
```
```bash
# Create systemd unit file
vim /etc/systemd/system/node_exporter.service
```
```yaml
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
```
```bash
systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter  #dtart service on-boot
systemctl status node_exporter  # check service
```
```bash
# check to retrieve metrics
curl localhost:9100/metrics
```
```yaml
# Edit prometheus.yml
scrape_configs:
- job_name: 'node_exporter'
  scrape_interval: 15s
  scrape_timeout: 5s
  sample_limit: 1000
  static_configs:
  - targets: ['TARGET1_IP:9100']
```
```bash
docker compose restart prometheus
```
```bash
# Check target in prometheus UI
http://<node_ip>:9090/targets
Status > Targets
or
Graph > 'up' > Execute
```
```bash
# Load Grafana dashboard
Dashboards > New > Import > write ID > Load
# Node Exporter Full (ID: 1860)
```
## mysqld_exporter

[Github releases](https://github.com/prometheus/mysqld_exporter/releases)

```bash
# Add Prometheus system user and group
sudo groupadd --system prometheus
sudo useradd -s /sbin/nologin --system -g prometheus prometheus
```
```bash
# Download and install Prometheus MySQL Exporter
curl -s https://api.github.com/repos/prometheus/mysqld_exporter/releases/latest   | grep browser_download_url   | grep linux-amd64 | cut -d '"' -f 4   | wget -qi -
tar xvf mysqld_exporter*.tar.gz
sudo mv  mysqld_exporter-*.linux-amd64/mysqld_exporter /usr/local/bin/
chown prometheus:prometheus /usr/local/bin/mysqld_exporter
rm -rf ./mysqld*
```
```bash
# Check version of mysqld_exporter
mysqld_exporter --version
<node_ip>:9104
```
```bash
# Create Prometheus exporter database user
mysql -u root -p # default password root
```
```sql
CREATE USER 'mysqld_exporter'@'localhost' IDENTIFIED BY 'VeryStrongPassword' WITH MAX_USER_CONNECTIONS 2;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqld_exporter'@'localhost';
FLUSH PRIVILEGES;
EXIT
```
```bash
# Configure database credentials
sudo vim /etc/.mysqld_exporter.cnf
```
```yaml
[client]
user=mysqld_exporter
password=VeryStrongPassword
```
```bash
sudo chown root:prometheus /etc/.mysqld_exporter.cnf
```
```bash
# Create systemd unit file
sudo vim /etc/systemd/system/mysql_exporter.service
```
```yaml
[Unit]
Description=Prometheus MySQL Exporter
After=network.target
User=prometheus
Group=prometheus

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/.mysqld_exporter.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.innodb_metrics \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=0.0.0.0:9104

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable mysql_exporter
sudo systemctl start mysql_exporter
sudo systemctl status mysql_exporter
```
```yaml
# Configure MySQL endpoint to be scraped by Prometheus Server
scrape_configs:
- job_name: 'mysqld_exporter'
  scrape_interval: 5s
  static_configs:
  - targets: ['TARGET1_IP:9104']
    labels:
      alias: db1
```
```bash
docker compose restart prometheus
# Check target
http://<node_ip>:9090/targets  # Status > Targets
```
Import dashboard to Grafana: `MySQL Overview ID 7362`

> [!NOTE]
Fix the chart `Pool Size of Total RAM graph` in the dashboard ID 7362:
The problem is that the ports of the exporters (node_exporter and mysqld_exporter) are different (since they are on the same node), so the `$host` variable cannot be used.We should add an additional label to `prometheus.yml` (for example, alias=[db1, db2, ..]) to each job.\
In Grafana create the `alias` variable:
`label_value(node_uname_info,alias)`.\
And we receive a request: \
`(mysql_global_variables_innodb_buffer_pool_size{alias="$alias"} * 100) / on (alias) node_memory_MemTotal_bytes{alias="$alias"}`
instead of `(mysql_global_variables_innodb_buffer_pool_size{instance="$host"} * 100) / on (instance) node_memory_MemTotal_bytes{instance="$host"}`

## blackbox_exporter

[Github releases](https://github.com/prometheus/blackbox_exporter/releases)
```bash
# Download and install blackbox_exporter
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
tar xvfz blackbox_exporter-0.25.0.linux-amd64.tar.gz
cp blackbox_exporter-0.25.0.linux-amd64/blackbox_exporter /usr/local/bin
```
```bash
# Create configuration folders for the blackbox_exporter
mkdir -p /etc/blackbox
cp blackbox_exporter-0.25.0.linux-amd64/blackbox.yml /etc/blackbox
rm -rf ./black*
```
```bash
# create a user account for the blackbox_exporter
useradd -rs /bin/false blackbox
chown blackbox:blackbox /usr/local/bin/blackbox_exporter
chown -R blackbox:blackbox /etc/blackbox/*
```
```bash
# Check version of the blackbox_exporter
blackbox_exporter --version
```
```bash
# Create systemd unit file
vim /etc/systemd/system/blackbox.service
```
```yaml
[Unit]
Description=Blackbox Exporter Service
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=blackbox
Group=blackbox
ExecStart=/usr/local/bin/blackbox_exporter \
  --config.file=/etc/blackbox/blackbox.yml \
  --web.listen-address=":9115"

Restart=always

[Install]
WantedBy=multi-user.target
```
```bash
systemctl daemon-reload
systemctl start blackbox
systemctl enable blackbox
systemctl status blackbox
```
```bash
# Check the gathering metrics
curl http://<ip>:9115/metrics
# In browser: 
<ip>:9115
```
```bash
# Binding the blackbox_exporter with Prometheus
vi prometheus.yml
```
```yaml
- job_name: blackbox-ssl
    metrics_path: /probe
    params:
      module:
      - http_2xx
    relabel_configs:
    - source_labels:
      - __address__
      target_label: __param_target
    - source_labels:
      - __param_target
      target_label: instance
    - replacement: <ip>:9115
      target_label: __address__
    static_configs:
    - targets:
    - <domain_name>
```
```bash
docker compose restart prometheus
```
```bash
# Check target
http://<node_ip>:9090/targets  # Status > Targets
```
Import dashboard to Grafana: `BlackBox Exporter. ID 18538`