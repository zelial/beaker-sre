# Beaker SRE
Documentation and tools for deploying a monitoring dashbaord for [Beaker](https://beaker-project.org/).

## Architecture
1. Beaker stores all relevant information in its MariaDB database
1. Prometheus exporter [scripts](./SLI/) compute metrics (SLIs) from the data and present them via included web server
1. [Prometheus](https://prometheus.io/) regularly scrapes the exporters endpoints and stores metrics in its database
2. [Grafana](https://grafana.com/) uses the prometheus as its datasource and visualize the metrics on a Dashboard 

## Deployment
All pieces are available as container images in public registries so using docker/podman would be the preferable method of deplyment. Described below is the manual deployment directly to host system I used on an older system without container support.

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
### Blackbox exporter
1. create user beaker-sre
    1. `useradd beaker-sre` 
1. as beaker-sre user in its homedir:
    1. download latest binary release from `https://github.com/prometheus/blackbox_exporter/releases`
    1. untar and symlink to `blackbox_exporter` directory
1. as root
   1. deploy [service file](./blackbox_exporter/blackbox_exporter.service) to systemd
      1. `cp blackbox_exporter.service /etc/systemd/system/blackbox_exporter.service`
      1. `systemctl enable blackbox_exporter.service`
   1. start blackbox_exporter
      1. `systemctl start blackbox_exporter.service`
1. Verify blackbox_exporter functionality
    1. `systemctl status blackbox_exporter`
    1. go to `http://ip.of.prometheus.server:3000`, Status -> Targets
       1. blackbox should be listed in blue color and as "(X/X up)"
       1. on the graph page, try to execute query using the blackbox_exporter's data, e.g.`probe_http_duration_seconds`.
### Grafana
1. create user grafana
   1. `useradd grafana`
1. as grafana user in its homedir:
   1. download latest standalone binary release from https://grafana.com/grafana/download?plcmt=top-nav&cta=downloads
   1. untar and symlink to `grafana` directory
   1. install aditional plugins 
      1. `./grafana-cli --pluginsDir /home/grafana/grafana/data/plugins/ plugins install fzakaria-simple-annotations-datasource`
      1. `./grafana-cli --pluginsDir /home/grafana/grafana/data/plugins/ plugins install simpod-json-datasource`
1. as root
   1. deploy [service file](./grafana/grafana.service) to systemd
      1. `cp grafana.service /etc/systemd/system/grafana.service`
      1. `systemctl enable grafana.service`
   1. start grafana
      1. `systemctl start grafana.service`
   1. allow access from outside
       1. `firewall-cmd --add-port=3000/tcp --permanent`
       1. `firewall-cmd --reload`
1. Verify Grafana functionality
    1. `systemctl status grafana`
    1. go to `http://ip.of.prometheus.server:3000`
1. Log in as admin/admin, change password to something secure
1. Connect Grafana to Prometheus
    1. in WebUI navigate to Configuration -> Data sources -> Add data source -> Prometheus -> Select
    1. Use `http://localhost:9090` as the suggested URL, keep the rest on default values. Confirm.
1. Setup Alerting as you see fit
