# Beaker SRE
Documentation and tools for deploying a monitoring dashbaord for [Beaker](https://beaker-project.org/).

## Architecture
1. Beaker stores all relevant information in its MariaDB database
1. Prometheus exporter [scripts](./SLI/) compute metrics (SLIs) from the data and present them via included web server
1. [Prometheus](https://prometheus.io/) regularly scrapes the exporters endpoints and stores metrics in its database
2. [Grafana](https://grafana.com/) uses the prometheus as its datasource and visualize the metrics on a Dashboard 
