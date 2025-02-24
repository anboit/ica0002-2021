# Lab 6

In this lab we will install some monitoring. It's not a complete solution, we will make it better on next labs.

## Task 1: Install Node Exporters

Install Prometheus node exporters on all your VMs. No need to configure them, default settings are good enough.  

## Task 2: Install and configure Prometheus

Install Prometheus on VM that doesn't have bind9 running. For example, if your DNS server from previous lab is vm2, then install Prometheus on vm1 and vice-versa.

Configure Prometheus job "linux" with static_configs. Include **all** your VMs there, even future ones.

Hint: use Ansible variable "groups['all']".

It's not allowed to use IP addresses in Prometheus config, only names are allowed.

## Task 3: Make Prometheus and Node Exporters available from outside

Install Nginx and configure reverse proxy:

    vm:80/node-metrics -> localhost:9100/metrics  # Node exporters
    vm:80/prometheus -> localhost:9090  # Prometheus

VM with Prometheus should have at least 2 "proxy_pass" statements. VM with only Node Exporter should forward only /node-metrics.

## Task 4: Adjust Prometheus service

To make Prometheus reachable from outside, run it with

    --web.external-url=http://<your_public_http_endpoint>/prometheus

Put that argument in /etc/default/prometheus in ARGS variable.

## Task 5: Write some Prometheus queries

Using [docs](https://prometheus.io/docs/prometheus/latest/querying/basics/)
write a query for memory consumption and average CPU load for each VM.

Save queries to prom_queries.txt, you will use them during next lab.

## Expected result

Your repository contains these files and directories:

    ansible.cfg
    group_vars/all.yaml
    hosts
    infra.yaml
    prom_queries.txt
    roles/bind/tasks/main.yaml
    roles/node_exporter/tasks/main.yaml
    roles/prometheus/tasks/main.yaml

Your repository also contains all the required files from the previous labs.

Your repository **does not contain** Ansible Vault master password.

Prometheus and Node Exporters are installed and configured with this command:

	ansible-playbook infra.yaml

Running the same command again does not make any changes to any of the managed
hosts.

After playbook execution you should be able to:

1. Check node metrics by using \<your_VM_http_link\>/node-metrics.

2. Query historical data from Prometheus web interface \<your_VM_http_link\>/prometheus.
