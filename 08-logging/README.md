# Lab 8

In this lab we will setup centralized logging.

## Task 1: Install InfluxDB

Follow the official guide: https://docs.influxdata.com/influxdb/v1.8/introduction/install/#installing-influxdb-oss

## Task 2: Create pinger service on one of VMs

Find bash script "pinger.sh" in 08-files.

Place this script to /usr/local/bin/pinger.

Create a service "pinger" that runs from user "pinger". Check previous 08-files for systemd service unit example. Place it into /etc/systemd/system/.

Don't forget to execute on systemd config change:

    systemctl daemon-reload

Pinger script requires config file:

    /etc/pinger/pinger.conf

Example can be found in 08-files as well.

## Task 3: Add latency monitoring to main Grafana dashboard

Add new Grafana datasource: influxdb:latency. It should be provisoned automaticaly via Grafana provisioning.

Use this datasource for new panel in dashboard from previous lab.

No ip addresses allowed.

## Task 4: Setup Telegraf

Install Telegraf on the same VM where InfluxDB is located. Docs: https://docs.influxdata.com/telegraf/v1.20/introduction/installation/#installation

Repo should be already added in task 1.

Installation steps can be done in "influxdb" role.

Configure Telegraph for only syslog input and only influxdb output. Hint:

    telegraf --help

UDP transport is more preferable.

## Task 5: Setup rsyslog

Configure rsyslog on all VMs to send all logs to Telegraf. Docs: https://github.com/influxdata/telegraf/blob/master/plugins/inputs/syslog/README.md

UDP transport is more preferable.

## Task 6: Create logging dashboard in Grafana

Add one more datasource: influxdb:telegraf

Import Grafana dashboard for Syslog: https://grafana.com/grafana/dashboards/12433

Add new datasource and dashboard to Grafana provisioning.

## Task 7: Add InfluxDB monitoring

Install InfluxDB stats exporter: https://github.com/carlpett/influxdb_stats_exporter

Download binary from latest [release](https://github.com/carlpett/influxdb_stats_exporter/releases/tag/v0.1.1) to /usr/local/bin/.

Create new systemd service. Run it with user `prometheus` as all other exporter do. Describe the service in /etc/systemd/system/prometheus-influxdb-stats-exporter.service.

Add couple more panels to your `Main` Grafana dashboard:

- InfluxDB health (influxdb_exporter_stats_query_success)
- InfluxDB write rate (influxdb_write_write_ok)

## Task 8: Supress InfluxDB request logging

By default InfluxDB logs every request, which floods the logs. Disable logging of HTTP requests. Use `influxdb.conf` from 08-files. 

## Expected result

Your repository contains these files and directories:

    ansible.cfg
    group_vars/all.yaml
    hosts
    infra.yaml
    roles/influxdb/tasks/main.yaml
    roles/influxdb_exporter/tasks/main.yaml
    roles/pinger/tasks/main.yaml
    roles/rsyslog/tasks/main.yaml

Your repository also contains all the required files from the previous labs.

Your repository **does not contain** Ansible Vault master password.

Everything is installed and configured with this command:

	ansible-playbook infra.yaml

Running the same command again does not make any changes to any of the managed
hosts.

After playbook execution you should be able to see all logs in one Grafana dashboard.
