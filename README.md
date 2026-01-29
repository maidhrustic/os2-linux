# Week 4 – Ansible Monitoring Assignment

## Overview
This assignment provisions a complete monitoring and visualization stack using Ansible:

- A **Prometheus server** on the monitoring host.
- A **Grafana dashboard** (Debian only) for visualizing metrics.
- **Node Exporter** on one or more client hosts.
- Automatic **target registration** so Prometheus scrapes all clients.

The repository is organized into roles for server/client and includes OS-specific update tasks for both Debian and RedHat families.

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
| [host_vars/client-01.yml](host_vars/client-01.yml) | IP for client |
| [roles/prometheus_server](roles/prometheus_server) | Prometheus server role |
| [roles/grafana_server](roles/grafana_server) | Grafana visualization role |
| [roles/prometheus_client](roles/prometheus_client) | Node exporter role |
| [vars/Debian.yml](vars/Debian.yml) | OS packages for Debian |
| [vars/RedHat.yml](vars/RedHat.yml) | OS packages for RedHat |

## What the project does

1. **Updates hosts** and installs common tooling (curl, git, python, etc.).
2. **Installs Prometheus** on the monitoring server.
3. **Installs Grafana** on the monitoring server for dashboard visualization (Debian only).
4. **Installs Node Exporter** on each client.
5. **Registers clients** in Prometheus by generating `targets.json` from inventory.

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
| Host IP addresses | [host_vars/monitor-01.yml](host_vars/monitor-01.yml), [host_vars/client-01.yml](host_vars/client-01.yml) |
| SSH user, key, become settings | [group_vars/all/vars.yml](group_vars/all/vars.yml) |
| Encrypted passwords | [group_vars/all/vault.yml](group_vars/all/vault.yml) |

## What site.yml does

[site.yml](site.yml) defines **three plays**:

1. **Prometheus server play** (group `monitoring_servers`)
	 - Applies role `prometheus_server` with tag `all`.
2. **Monitoring clients play** (group `monitoring_clients`)
	 - Applies role `prometheus_client` with tag `all`.
3. **Grafana server play** (group `monitoring_servers`)
	 - Applies role `grafana_server` with tag `all`.

This layout makes it easy to run everything or target a specific role.

## Tags and how to run tasks

Tags are defined inside each role’s `tasks/main.yml`:

### Prometheus server role tags

- `update` – OS updates and base packages
- `prometheus` – installs and configures Prometheus
- `register` – generates `targets.json` and restarts Prometheus

### Grafana server role tags

- `update` – OS updates and base packages
- `grafana` – installs and configures Grafana (Debian only)

### Prometheus client role tags

- `update` – OS updates and base packages
- `node_exporter` – installs and starts Node Exporter

### Common tag

- `all` – applied at the role level in [site.yml](site.yml)

### Examples

- Run everything (Prometheus + Grafana + Node Exporters):
	- Use [run](run) with no extra tags
- Only update packages everywhere:
	- `./run -t update`
- Only install Prometheus on server:
	- `./run -l monitor-01 -t prometheus`
- Only install Grafana on server:
	- `./run -l monitor-01 -t grafana`
- Only register clients in Prometheus:
	- `./run -l monitor-01 -t register`
- Only install Node Exporter on clients:
	- `./run -l client-01 -t node_exporter`

## Notes and important details

- The Prometheus server uses **file-based service discovery** with `targets.json`.
	- Config is written to `{{ prometheus_config }}` (see [host_vars/monitor-01.yml](host_vars/monitor-01.yml)).
- The `register` tag rebuilds targets from the `monitoring_clients` inventory group.
- **Grafana** is only deployed on Debian-based systems. See [roles/grafana_server/tasks/main.yml](roles/grafana_server/tasks/main.yml).
- OS-specific updates are implemented in:
	- [roles/prometheus_server/tasks/update_debian.yml](roles/prometheus_server/tasks/update_debian.yml)
	- [roles/prometheus_server/tasks/update_redhat.yml](roles/prometheus_server/tasks/update_redhat.yml)
	- [roles/grafana_server/tasks/update_debian.yml](roles/grafana_server/tasks/update_debian.yml)
	- [roles/grafana_server/tasks/update_redhat.yml](roles/grafana_server/tasks/update_redhat.yml)
	- [roles/prometheus_client/tasks/update_debian.yml](roles/prometheus_client/tasks/update_debian.yml)
	- [roles/prometheus_client/tasks/update_redhat.yml](roles/prometheus_client/tasks/update_redhat.yml)


---

**Course:** School assignment – Week 4 <br>
**Authors:** Maid Hrustic & Joshua de Graaf
