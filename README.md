# Beaker SRE
Documentation and tools for deploying a monitoring dashbaord for [Beaker](https://beaker-project.org/).

## Architecture
1. Beaker stores all relevant information in its MariaDB database
1. Prometheus exporter [scripts](./SLI/) compute metrics (SLIs) from the data and present them via included web server
1. [Prometheus](https://prometheus.io/) regularly scrapes the exporters endpoints and stores metrics in its database
2. [Grafana](https://grafana.com/) uses the prometheus as its datasource and visualize the metrics on a Dashboard 

## Deployment
### Prometheus
1. create user prometheus
   1. `useradd prometheus`
1. as prometheus user in its homedir:
   1. download latest release from https://prometheus.io/download/
   1. untar and symlink to `prometheus` directory
   1. deploy [config file](./prometheus/prometheus.yml) to the prometheus directory
1. as root
   1. deploy [service file](./prometheus/prometheus.service) to systemd
      1. `cp prometheus.service /etc/systemd/system/prometheus.service`
      1. `systemctl enable prometheus.service`
   1. start prometheus
      1. `systemctl start prometheus.service`
   1. allow access from outside (optional):
       1. `firewall-cmd --add-port=9090/tcp --permanent`
       1. `firewall-cmd --reload`
 1. Verify Prometheus functionality
    1. go to `http://ip.of.prometheus.server:9090`
### Grafana
