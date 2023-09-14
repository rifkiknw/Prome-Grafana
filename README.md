# Install Prometheus and Grafana Using docker compose

## Setup host
This time I used centos 7 and docker ce

First, i will configure hostname and fqdn. I use "bongkar" as the hostname and added the IP Address in /etc/hosts
```
hostnamectl set-hostname monitoring
cat <<"EOF" >> /etc/hosts
192.168.30.14 monitoring
192.168.30.201 bongkar
EOF
```

Then I installed chronyd to set the time and customize the timedatectl
```
yum install chrony -y
systemctl enable chronyd --now
timedatectl set-timezone Asia/Jakarta
```

To save you trouble, I disabled the firewall and changed Selinux to Permissive mode.
```
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
```

If necessary, turn off selinux when booting by changing the configuration in /etc/selinux/config from "enforcing" to "disabled"
```
From
SELINUX=enforcing

To
SELINUX=disabled
```


## Installing Docker, and Docker Compose
Saya biasa menggunakan script ini yang didapat dari docker.com and other recommendation
```
sudo yum check-update
curl -fsSL https://get.docker.com/ | sh
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker bongkar
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```


## Define General Component
There are several key components required such as: 
* docker-compose.yaml
* prometheus.yaml

First we create a directory and docker compose file with the command
```
mkdir -p compose && cd compose
touch docker-compose.yaml
```
I created a docker compose with the following main components:

##### Network:
* monitoring: bridge
##### Volume:
* prometheus_data
* grafana_data
##### Service:
* prometheus
* grafana

The results are as follows
```
version: '3'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data: {}
  grafana_data: {}

services:
#
#  node-exporter:
#    image: prom/node-exporter:latest
#    container_name: node-exporter
#    restart: unless-stopped
#    volumes:
#    - /proc:/host/proc:ro
#    - /sys:/host/sys:ro
#    - /:/rootfs:ro
#    command:
#    - '--path.procfs=/host/proc'
#    - '--path.rootfs=/rootfs'
#    - '--path.sysfs=/host/sys'
#    - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
#    ports:
#    - 9100:9100
#    networks:
#    - monitoring
#
#  alertmanager:
#    container_name: alertmanager
#    hostname: alertmanager
#    image: prom/alertmanager
#    volumes:
#    - ./alertmanager.conf:/etc/alertmanager/alertmanager.conf
#    command:
#    - '--config.file=/etc/alertmanager/alertmanager.conf'
#    ports:
#    - 9093:9093
#    networks:
#    - monitoring
#
  prometheus:
    container_name: prometheus
    hostname: prometheus
    image: prom/prometheus
    volumes:
    - ./prometheus.yaml:/etc/prometheus/prometheus.yaml
    - ./alert_rules.yaml:/etc/prometheus/alert_rules.yaml
    - prometheus_data:/prometheus
    command:
    - '--config.file=/etc/prometheus/prometheus.yaml'
    links:
    - alertmanager:alertmanager
    ports:
    - 9090:9090
    networks:
    - monitoring

  grafana:
    container_name: grafana
    hostname: grafana
    image: grafana/grafana
    volumes:
    - ./grafana_datasources.yaml:/etc/grafana/provisioning/datasources/all.yaml
    - ./grafana_config.ini:/etc/grafana/config.ini
    - grafana_data:/var/lib/grafana
    ports:
    - 3000:3000
    networks:
    - monitoring

```

And second, we create the configuration for prometheus with prometheus.yaml
```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
  - static_configs:
    - targets: ["alertmanager:9093"]

rule_files:
- /etc/prometheus/alert_rules.yaml

scrape_configs:
- job_name: 'prometheus'
  scrape_interval: 1m
  static_configs:
  - targets: ['localhost:9090']

- job_name: 'host'
  static_configs:
  - targets: ['localhost:9100']

#remote_write:
#  - url: '<Your Prometheus remote_write endpoint>'
#    basic_auth:
#      username: '<Your Grafana Username>'
#      password: '<Your Grafana API key>'

```
After that, we run the entire file with the command: 
```
docker-compose up -d
```

As a result we will have containers, networks and volumes for our Prometheus and Grafana

We can access prometheus with http://ip-server:9090 and graphana with http://ip-server:3000

To login grafana we can enter the user and password "admin"


#Source code from Grafana Labs, Prometheus, Medium, DEV Community, dzlab.github.io
