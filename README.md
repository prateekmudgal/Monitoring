
# Django Notes App with Observability using Prometheus and Grafana

This project sets up a **Django Notes Application** running in a Docker container along with monitoring using **Prometheus**, **cAdvisor**, and **Node Exporter** for capturing metrics.

## Clone the Repository

Start by cloning the repository using the following command:

```bash
git clone https://github.com/LondheShubham153/django-notes-app.git
```

## Step 1: Launch an EC2 Instance

Launch a `t2.medium` instance on AWS to handle our application and monitoring tools. Ensure the following ports are open: `8000`, `8080`, `9090`, and `9100`.

## Step 2: Connect to the Instance via SSH

Once the instance is launched, connect to it using SSH:

```bash
ssh -i <your-key>.pem ubuntu@<your-ec2-ip>
```

## Step 3: Install Docker

Once connected, update the instance and install Docker:

```bash
sudo apt-get update
sudo apt-get install docker.io
```

Add the user to the Docker group and update the session:

```bash
sudo usermod -aG docker $USER && newgrp docker
```

## Step 4: Set Up Project Directory

Create a directory named `observability` and navigate into it:

```bash
mkdir observability
cd observability
```

## Step 5: Create Docker Compose File

Create a `docker-compose.yml` file for the Django Notes app, Redis, cAdvisor, Prometheus, and Node Exporter.

```bash
vim docker-compose.yml
```

Paste the following content into the file:

```yaml
version: "3.8"

networks:
  monitoring:
    driver: bridge

services:
  notes-app:
    build:
      context: ./django-notes-app
    ports:
      - "8000:8000"
    container_name: "notes-app"
    networks:
      - monitoring
    depends_on:
      - redis

  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6380:6379"
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    networks:
      - monitoring
    depends_on:
      - cadvisor

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    networks:
      - monitoring
```

## Step 6: Configure Prometheus

Download the Prometheus configuration file:

```bash
wget https://raw.githubusercontent.com/prometheus/prometheus/main/documentation/examples/prometheus.yml
```

Edit `prometheus.yml` to scrape metrics from cAdvisor and Node Exporter:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "Docker Containers"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

Save the file.

## Step 7: Start the Docker Containers

Run the application and all monitoring tools using Docker Compose:

```bash
docker-compose up -d
```

This will start the Django Notes App, Redis, cAdvisor, Prometheus, and Node Exporter.

## Step 8: Access the Application and Metrics

- **Notes App**: Open `http://<your-ec2-ip>:8000` in your browser.
- **cAdvisor**: Monitor container metrics at `http://<your-ec2-ip>:8080`.
- **Prometheus**: View metrics at `http://<your-ec2-ip>:9090/metrics`.
- **Node Exporter**: View server metrics at `http://<your-ec2-ip>:9100/metrics`.

## Step 9: Update and Restart Prometheus

After editing the `prometheus.yml` file, restart the Prometheus container:

```bash
docker-compose restart prometheus
```

This will apply your changes and restart Prometheus to scrape the latest metrics.

## Step 10: Visualizing Metrics

To see container and node-level metrics, you can use Prometheus's query language (PromQL) at `http://<your-ec2-ip>:9090`.

---

### Main Prometheus Configuration File

The main Prometheus configuration file (`prometheus.yml`) looks like this:

```yaml
# my global config
global:
  scrape_interval: 15s  # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s  # Evaluate rules every 15 seconds. The default is every 1 minute.

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# Scrape configurations
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "Docker Containers"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

### Main Docker Compose File

The main `docker-compose.yml` file looks like this:

```yaml
version: "3.8"

networks:
  monitoring:
    driver: bridge

services:
  notes-app:
    build:
      context: ./django-notes-app
    ports:
      - "8000:8000"
    container_name: "notes-app"
    networks:
      - monitoring
    depends_on:
      - redis

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - 8080:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring

  redis:
    image: redis:latest
    container_name: redis
    ports:
      - 6380:6379
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - 9090:9090
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    networks:
      - monitoring
    depends_on:
      - cadvisor

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - 9100:9100
    networks:
      - monitoring
```
