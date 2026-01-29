# Week 4 – Ansible Monitoring Assignment

## Overview
This assignment provisions a complete monitoring stack and a small web workload using Ansible:

- A **Prometheus server** on the monitoring host.
- A **Grafana dashboard** (Debian only) for visualizing metrics.
- **Node Exporter** on one or more client hosts.
- **Docker + Nginx container** on clients.
- **WordPress** deployment on a client.
- Automatic **target registration** so Prometheus scrapes all clients.

The repository is organized into roles for server/client services and includes OS-specific update tasks for both Debian and RedHat families.

## Repository layout

| Path | Purpose |
| --- | --- |
| [site.yml](site.yml) | Main playbook wiring server/client roles |
| [hosts.yml](hosts.yml) | Inventory groups and hostnames |
| [ansible.cfg](ansible.cfg) | Ansible defaults (inventory, SSH settings) |
| [run](run) | Convenience wrapper for `ansible-playbook` with Vault prompt |
| [group_vars/all/vars.yml](group_vars/all/vars.yml) | Shared Ansible variables (user, key, become) |
| [group_vars/all/vault.yml](group_vars/all/vault.yml) | Encrypted secrets (passwords) |
| [host_vars/monitor-01.yml](host_vars/monitor-01.yml) | Prometheus/Grafana paths + IP for server |
| [host_vars/client-01.yml](host_vars/client-01.yml) | IP for client-01 |
| [host_vars/client-02.yml](host_vars/client-02.yml) | IP for client-02 |
| [host_vars/client-03.yml](host_vars/client-03.yml) | IP + WordPress settings |
| [roles/update](roles/update) | OS updates and base packages |
| [roles/prometheus_server](roles/prometheus_server) | Prometheus server role |
| [roles/grafana_server](roles/grafana_server) | Grafana visualization role |
| [roles/prometheus_client](roles/prometheus_client) | Node exporter role |
| [roles/docker](roles/docker) | Docker + Nginx container role |
| [roles/wordpress](roles/wordpress) | WordPress role |
| [vars/Debian.yml](vars/Debian.yml) | OS packages for Debian |
| [vars/RedHat.yml](vars/RedHat.yml) | OS packages for RedHat |

## What the project does

1. **Updates hosts** and installs common tooling (curl, git, python, etc.).
2. **Installs Prometheus** on the monitoring server.
3. **Installs Grafana** on the monitoring server for dashboard visualization (Debian only).
4. **Installs Node Exporter** on each client.
5. **Installs Docker** and runs an **Nginx** container on clients.
6. **Installs WordPress** on the designated client.
7. **Registers clients** in Prometheus by generating `targets.json` from inventory.

## Ansible controller setup

1. **Install Ansible** on the controller (your local machine or a VM).
2. **Ensure SSH access** to all managed hosts.
3. **Vault password** is required to decrypt secrets.

The controller settings are defined in [ansible.cfg](ansible.cfg):

- Inventory defaults to [hosts.yml](hosts.yml)
- Host key checking is disabled for convenience

### SSH and authentication settings

Shared connection settings live in [group_vars/all/vars.yml](group_vars/all/vars.yml):

- `ansible_user`: Ansible SSH user
- `ansible_password`: from Vault (`vault_ansible_password`)
- `ansible_ssh_private_key_file`: SSH key path
- `ansible_become_password`: from Vault

> The actual password is stored in the encrypted file [group_vars/all/vault.yml](group_vars/all/vault.yml).

## Where to find IPs, hostnames, users, passwords

| Data | Location |
| --- | --- |
| Inventory groups and hostnames | [hosts.yml](hosts.yml) |
| Host IP addresses | [host_vars/monitor-01.yml](host_vars/monitor-01.yml), [host_vars/client-01.yml](host_vars/client-01.yml), [host_vars/client-02.yml](host_vars/client-02.yml), [host_vars/client-03.yml](host_vars/client-03.yml) |
| SSH user, key, become settings | [group_vars/all/vars.yml](group_vars/all/vars.yml) |
| Encrypted passwords | [group_vars/all/vault.yml](group_vars/all/vault.yml) |

## What site.yml does

[site.yml](site.yml) defines **three plays**:

1. **Update play** (group `monitoring`)
	 - Applies role `update` with tag `update` to all monitoring machines.
2. **Monitoring servers setup** (group `monitoring_servers`)
	 - Applies roles `prometheus_server` and `grafana_server` with tags `prometheus` and `grafana`.
3. **Monitoring clients setup** (group `monitoring_clients`)
	 - Applies roles `prometheus_client`, `docker`, and `wordpress` with tags `node_exporter`, `docker`, and `wordpress`.

This layout lets you run updates across all monitoring hosts, or target a specific role by tag.

## Tags and how to run tasks

Tags are defined inside each role’s `tasks/main.yml`:

### Update role tags

- `update` – OS updates and base packages

### Prometheus server role tags

- `prometheus` – installs and configures Prometheus
- `register` – generates `targets.json` and restarts Prometheus

### Grafana server role tags

- `grafana` – installs and configures Grafana (Debian only)

### Prometheus client role tags

- `node_exporter` – installs and starts Node Exporter

### Docker role tags

- `docker` – installs Docker and configures the service
- `nginx` – pulls and runs the Nginx container

### WordPress role tags

- `wordpress` – installs and configures WordPress

### Examples

- Run updates on all monitoring machines:
	- `./run -t update`
- Only install Prometheus on server:
	- `./run -l monitor-01 -t prometheus`
- Only install Grafana on server:
	- `./run -l monitor-01 -t grafana`
- Only register clients in Prometheus:
	- `./run -l monitor-01 -t register`
- Only install Node Exporter on clients:
	- `./run -l client-01 -t node_exporter`
- Only install Docker on a client:
	- `./run -l client-02 -t docker`
- Only run the Nginx container on a client:
	- `./run -l client-02 -t nginx`
- Only install WordPress on the WordPress client:
	- `./run -l client-03 -t wordpress`

## Notes and important details

- The Prometheus server uses **file-based service discovery** with `targets.json`.
	- Config is written to `{{ prometheus_config }}` (see [host_vars/monitor-01.yml](host_vars/monitor-01.yml)).
- The `register` tag rebuilds targets from the `monitoring_clients` inventory group.
- **Grafana** is only deployed on Debian-based systems. See [roles/grafana_server/tasks/main.yml](roles/grafana_server/tasks/main.yml).
- WordPress settings are defined per-host in [host_vars/client-03.yml](host_vars/client-03.yml). Store database passwords in Vault if required.
- OS-specific updates are implemented in:
	- [roles/prometheus_server/tasks/update_debian.yml](roles/prometheus_server/tasks/update_debian.yml)
	- [roles/prometheus_server/tasks/update_redhat.yml](roles/prometheus_server/tasks/update_redhat.yml)
	- [roles/grafana_server/tasks/update_debian.yml](roles/grafana_server/tasks/update_debian.yml)
	- [roles/grafana_server/tasks/update_redhat.yml](roles/grafana_server/tasks/update_redhat.yml)
	- [roles/update/tasks/update_debian.yml](roles/update/tasks/update_debian.yml)
	- [roles/update/tasks/update_redhat.yml](roles/update/tasks/update_redhat.yml)
	- [roles/prometheus_client/tasks/update_debian.yml](roles/prometheus_client/tasks/update_debian.yml)
	- [roles/prometheus_client/tasks/update_redhat.yml](roles/prometheus_client/tasks/update_redhat.yml)


---

**Course:** School assignment – Week 4 <br>
**Authors:** Maid Hrustic & Joshua de Graaf
