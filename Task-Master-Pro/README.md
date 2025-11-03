# Training-Prometheus---Grafana

## 1. Launch two instance:
- for Application
- for Monitoring the Application

## 2. Then install some tool on Both Instance:

- In Application
  - install node-exporter:
    https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
    
- In Monitoring
  - install prometheus: https://github.com/prometheus/prometheus/releases/download/v3.7.3/prometheus-3.7.3.linux-amd64.tar.gz
  - install alert-manager: https://github.com/prometheus/alertmanager/releases/download/v0.29.0-rc.1/alertmanager-0.29.0-rc.1.linux-amd64.tar.gz
  - install blackbox-exporter: https://github.com/prometheus/blackbox_exporter/releases/download/v0.27.0/blackbox_exporter-0.27.0.linux-amd64.tar.gz
  - install grafana:
    - sudo apt-get install -y adduser libfontconfig1 musl
    - wget https://dl.grafana.com/grafana-enterprise/release/12.2.1/grafana-enterprise_12.2.1_18655849634_linux_amd64.deb
    - sudo dpkg -i grafana-enterprise_12.2.1_18655849634_linux_amd64.deb  

## 3. After installation of tool
## 4. Then Add Alert_Rule in Prometheus
alert_rule.yml
```
groups:
- name: alert_rules
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: Endpoint {{ $labels.instance }} down
      description: "{{ $labels.instance }} of job {{ $labels.jobs }} has been down for more than 1 minute."
  - alert: WebsiteDown
    expr: prob_success == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      description: The website at {{ $labels.instance }} is down.
      summary: Website down
  - alert: HostOutOfMemory
    expr: node_memory_MemAvailable / node_memory_MemTotal * 100 <25
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host out of memory (instance {{ $labels.instance }})
      description: |-
       Node memory is filling up (< 25% left)
         VALUE = {{ $value }}
         LABELS: {{ $labels }}
  - alert: HostOutOfDisk
    expr: (node_filesystem_avail{mountpoint="/"} * 100) /
     node_filesystem_size{mountpoint="/"} < 50
    for: 1s
    labels:
      severity: warning
    annotations:
      summary: Host out of disk space (instance {{ $labels.instance }})
      description: |-
        Disk is almost full (< 50% left )
          VALUE = {{ $value }}
          LABELS: {{ $labels }}
  - alert: HostHighMemoryUsage
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      description: |
        Memory usage on {{ $labels.instance }} is above 90% for more than 5 minutes.
        Current usage: {{ printf "%.2f" $value }}%
```
## 5. Then go to the prometheus.yml and add the alert.yml in rule_files section
```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 3.109.47.94:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
   - "alert_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
       # The label name is added as a label `label_name=<label_value>` to any timeseries scraped from this config.
        labels:
          app: "prometheus"

  - job_name: "node_exporter"

    static_configs:
      - targets: ["13.235.114.105:9100"]

  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - https://github.com/singsachin1709/Training-Prometheus---Grafana.git    # Target to probe with http.
        - http://13.235.114.105:8080 # Target to probe with http on port 8080.
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 3.109.47.94:9115 # The blackbox exporter's real hostname:port.
```
then restart the alert-manager & prometheus
