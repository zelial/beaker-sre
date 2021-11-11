# Beaker SRE
Documentation and tools for deploying a monitoring dashbaord for [Beaker](https://beaker-project.org/).

## Architecture
1. Beaker stores all relevant information in its MariaDB database
2. [prometheus-sql(https://github.com/chop-dbhi/prometheus-sql) and [sql-agent](https://github.com/chop-dbhi/sql-agent) queries and exports the Beaker database in time-series data format
3. [Prometheus](https://prometheus.io/) regularly scrapes the exporters endpoints and stores metrics in its database
4. [Grafana](https://grafana.com/) uses the prometheus as its datasource and visualize the metrics on a Dashboard 

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

### sql-agent
1. Read [sql-agent](https://github.com/chop-dbhi/sql-agent)
   1. This is the project that we will use that accesses the Beaker MariaDB database
1. To run, start the docker image 
   1. `docker run --name sql_agent -d -p 5000:5000 dbhi/sql-agent`
1. Get the container IP address
   1. `docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' sql_agent`
3. Give access to the Beaker database
   1. `mysql`
   1. `GRANT SELECT ON beaker.* to <username>@'<ip_addr>' identified by '<password>';`
      1. Where:
         1. `<username>` and `<password>` are mysql credentials that the service will use to read the beaker database
         2. `<ip_addr>` is the sql-agent docker container's IP address
   3. `flush privileges;`
   4. `quit`
5. Verify sql-agent functionality
   1. Make `test_query.json` file:
      ```json
      {
         "driver":"mariadb",
         "connection": {
                 "host": "<beaker_db_ip>",
                 "user": "<username>",
             "database": "beaker",
             "password":"<password>"
         },
         "sql": "SELECT count(fqdn) as cnt from system"
      }   
      ```   
      
      1. Where:
         1. `<beaker_db_ip>` is the IP address of the machine hosting the Beaker mariaDB, likely the lab controller
         2. `<username>` and `<password>` are the mysql credentials given in the previous step
   1. `curl -v -H "Content-Type: application/json" -X POST -d @test_query.json http://localhost:5000`
      1. output should contiain `{"cnt":"<number>"}` where `<number>` is the total number of machines in the beaker network
### prometheus-sql
   1. Read [prometheus-sql](https://github.com/chop-dbhi/prometheus-sql) docs
   2. Install latest release
   3. `cd` into directory with `prometheus-sql` executable
   4. make `config.yml`
      ```yml
      defaults:
        data-source: beaker-ds
        query-interval: 1m
        query-timeout: 20s
        query-value-on-error: -1

      # Defined data sources
      data-sources:
        beaker-ds:
          driver: mysql
          properties:
            host: <beaker_db_ip>
            port: <port>
            user: <username>
            password: <password>
            database: beaker
         ```
         1. where:
            1. `<port>` is the port to use for exporting the returned data
            2. `<beaker_db_ip>` is the host that the service is running, likely the same as where the beaker db is
            3. `<username>` and `<password>` are mysql credentials that the service will use to read the beaker database
   5. make `queries.yml`
      ```yml
         - machines_total:
             sql: >
                 select count(fqdn) as cnt from system WHERE system.status != 'Removed';
             data-field: cnt
         - machines_available:
             sql: >
                 select count(fqdn) as cnt from system where scheduler_status='Idle' AND system.status != 'Removed';
             data-field: cnt

         - machines_reserved:
             sql: >
                 select count(fqdn) as cnt from system where scheduler_status='Reserved' AND system.status != 'Removed';
             data-field: cnt

         - machines_broken:
             sql: >
                 select count(fqdn) as cnt from system where status = 'Broken';
          data-field: cnt   
         ```   
         1. This file is filled with the sql queries that will be converted into time-series data that we can read with prometheus
         2. The above is an example and can be edited in anyway to suite the users needs.
         3. For more information checkout (prometheus-sql examples)[https://github.com/chop-dbhi/prometheus-sql/blob/master/examples/working_example/queries.yml] and (Beaker example qeueries)[https://github.com/beaker-project/beaker/tree/master/Server/bkr/server/reporting-queries]
   4. Run `./prometheus-sql -config config.yml -queries queries.yml -service http://localhost:5000` and ensure no errors appear
   5. Turn `prometheus-sql` into a service
      5. Make `prometheus-sql.service`
      ```
      [Unit]
      Description=Open endpoints from SQL to prometheus communication

      [Service]
      User=prometheus
      Group=prometheus
      WorkingDirectory=/home/prometheus-sql/linux-amd64
      ExecStart=/home/prometheus-sql/linux-amd64/prometheus-sql -config /home/prometheus-sql/linux-amd64/config.yml -queries /home/prometheus-sql/linux-amd64/queries.yml -service http://localhost:5000
      KillMode=process
      Restart=always

      [Install]
      WantedBy=multi-user.target
      ```
         1. Fix paths where necessary here
      2. move `prometheus-sql.service` to `/etc/systemd/system/`
      3. `systemctl daemon-reload`
      4. `systemctl enable prometheus-sql`
      5. `systemctl start prometheus-sql`
   6. Verify prometheus-sql functionality
      1. `systemctl status prometheus-sql`
      1. go to `http://ip.of.prometheus.server:3000`, Status -> Targets
       1. beaker_data should be listed in blue color and as "(X/X up)"
       1. on the graph page, try to execute query using the newly exported data, e.g.`query_result_machines_total`.


