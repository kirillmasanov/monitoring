# Deploy Prometheus and Grafana
Install Docker:

[Docker docs](https://docs.docker.com/engine/install/ubuntu/)
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

Create a new directory for your Prometheus and Grafana configuration files: 
```bash
mkdir /opt/monitoring
cd /opt/monitoring
```
Create a new Docker Compose file:
```bash
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