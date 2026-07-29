[Read this in Russian](README.ru.md)

# monitoring-server-configuration

Ansible playbooks and roles that deploy a small distributed monitoring stack:

- **Grafana** behind an nginx reverse proxy with Let's Encrypt TLS, with datasources and alert rules provisioned from static YAML files.
- **Prometheus** + **node_exporter**, on each monitored server.
- **blackbox_exporter**, on each monitored server, for external HTTP probes (e.g. "is this website up").
- **Loki** + **promtail**, on each monitored server, for log shipping.
- **nftables** firewall rules, deployed by every role.

## Architecture

The inventory has two groups, and they are not what the names might suggest at first glance:

- `observation_servers` — the server(s) where **Grafana** runs. This is where you *observe* dashboards and alerts.
- `monitoring_servers` — the server(s) that are *being monitored*. Each one runs its own local Prometheus, node_exporter, blackbox_exporter and Loki/promtail.

There is no central Prometheus/Loki server. Instead, Grafana (on `observation_servers`) is provisioned with one Prometheus datasource and one Loki datasource *per monitoring server*, pointed at that server's public IP (see `inventories/files/grafana/datasources/`). Each monitoring server is therefore self-contained: it scrapes its own `node_exporter`/`blackbox_exporter`, ships its own logs, and only Grafana talks to it remotely.

`playbooks/observation_servers.yml` targets `observation_servers` and runs the `grafana` role.
`playbooks/monitoring_servers.yml` targets `monitoring_servers` and runs `prometheus`, `loki`, then `blackbox_exporter`, in that order.

## Repository Layout

- `ansible.cfg` — sets `roles_path = ./roles` and disables SSH host key checking. No default inventory is configured, so `-i` must always be passed explicitly.
- `inventories/servers.yml` — the inventory: hosts, groups, SSH connection details.
- `inventories/group_vars/monitoring_servers/whitelist.yml` — firewall allow-lists (`prometheus_firewall_whitelist_ips`, `loki_firewall_whitelist_ips`, `blackbox_firewall_whitelist_ips`).
- `inventories/group_vars/observation_servers/vault.yml` — ansible-vault-encrypted secrets (`grafana_admin_password`).
- `inventories/host_vars/<host>.yml` — per-host variables.
- `inventories/files/<role>/<inventory_hostname>.yml` (with a `default.yml` fallback) — static, hand-edited config files: Prometheus scrape configs, blackbox_exporter probe modules, Grafana datasources and alert rules. These are copied to the target as-is; they are **not** Jinja templates and are **not** generated from role variables.
- `playbooks/observation_servers.yml`, `playbooks/monitoring_servers.yml` — the two playbooks.
- `roles/grafana`, `roles/prometheus`, `roles/loki`, `roles/blackbox_exporter` — the four roles.

## Quick Start

1. Fill in `inventories/servers.yml`, for example:

```yaml
all:
  vars:
    ssh_key_path: /home/<USER>/.ssh/<SSH_KEY>
  children:
    observation_servers:
      hosts:
        grafana01:
          ansible_host: <MONITORING_SERVER_IP>
          ansible_user: <ANSIBLE_USER>
          ansible_ssh_private_key_file: "{{ ssh_key_path }}"
    monitoring_servers:
      hosts:
        monitoring-server02:
          ansible_host: <OBSERVATION_SERVER_IP>
          ansible_user: <ANSIBLE_USER>
          ansible_ssh_private_key_file: "{{ ssh_key_path }}"
        monitoring-server01:
          ansible_host: <PROXY_SERVER_IP>
          ansible_user: <ANSIBLE_USER>
          ansible_ssh_private_key_file: "{{ ssh_key_path }}"
```

2. Set `grafana_admin_password` in the vault (see [Secrets / Vault](#secrets--vault) below).

3. Run the playbooks:

Grafana, on the observation server (needs the vault password because `grafana_admin_password` lives there):

```bash
ansible-playbook -i inventories/servers.yml playbooks/observation_servers.yml --ask-vault-pass
```

Prometheus + Loki + blackbox_exporter, on every monitoring server (no vault secrets involved):

```bash
ansible-playbook -i inventories/servers.yml playbooks/monitoring_servers.yml
```

## Role: `grafana`

### What it does

- Installs Grafana from the official APT repository, served through the `mirrors.cernet.edu.cn` mirror (worked around direct-access blocks to `apt.grafana.com`).
- Installs and configures nginx as a reverse proxy in front of Grafana, and obtains/renews a Let's Encrypt certificate via the HTTP-01 (webroot) challenge — always, there is no toggle to skip nginx or TLS.
- Provisions Grafana datasources and alert rules from static files.
- Sets the Grafana admin password once.
- Deploys an nftables ruleset that only allows 22, 80 and 443 — this is hardcoded in the template, not driven by a port-list variable.

### Variables (`roles/grafana/defaults/main.yml`)

```yaml
grafana_admin_password: ""
grafana_domain: <GRAFANA_DOMAIN>
grafana_enable_https: true
grafana_letsencrypt_email: "admin@<GRAFANA_DOMAIN>"
grafana_letsencrypt_webroot: /var/www/letsencrypt
grafana_manage_alerting_provisioning: true
```

`grafana_admin_password` must be set (normally via the vault) for the admin password to actually be reset; an empty value is a no-op.

There is no `grafana_firewall_tcp_ports`, `grafana_install_nginx`, `grafana_enable_firewall`, `grafana_prometheus_url`/`grafana_loki_url`, or `grafana_admin_force_reset` variable — nginx, TLS and the firewall always run, and datasource URLs come from the static files described below, not from role variables.

### Datasources and alerting

Grafana datasources and alert rules are provisioned from plain YAML copied verbatim to the target:

- `inventories/files/grafana/datasources/<inventory_hostname>.yml` (falls back to `default.yml`) → `/etc/grafana/provisioning/datasources/datasources.yml`
- `inventories/files/grafana/alerting/<inventory_hostname>.yml` (falls back to `default.yml`) → `/etc/grafana/provisioning/alerting/provision.yml` (only when `grafana_manage_alerting_provisioning` is true)

To add, change or remove a datasource or alert rule, edit these files directly — for example, `inventories/files/grafana/datasources/grafana01.yml` lists one Prometheus and one Loki datasource per monitoring server, pointed at that server's public IP and port 9090/3100.

### Admin password reset behavior

The admin password is set with `grafana-cli admin reset-admin-password` only once: after the first successful reset, a marker file is created at `/var/lib/grafana/.admin_password_set` on the target, and the role skips the reset on every subsequent run (even if `grafana_admin_password` changes). To force a new password, delete that marker file on the server and rerun the playbook.

## Role: `prometheus`

### What it does

- Downloads and installs the Prometheus and node_exporter binaries from GitHub releases (node_exporter is always installed — there is no toggle to disable it).
- Creates a dedicated `prometheus` system user/group, systemd units, and validates the config with `promtool` before starting.
- Waits for `/-/ready` to return 200.
- Deploys an nftables ruleset (always — there is no toggle to disable the firewall step either).

### Variables (`roles/prometheus/defaults/main.yml`)

```yaml
prometheus_version: "2.49.1"
node_exporter_version: "1.8.1"
prometheus_firewall_whitelist_ips: []
```

That's the entire set of role defaults. There is no `prometheus_prometheus_targets`, `prometheus_node_exporter_targets`, `prometheus_extra_scrape_jobs`, `prometheus_web_listen_address`, `prometheus_basic_auth_users`, `prometheus_manage_firewall`, or `prometheus_firewall_tcp_ports` variable in the role.

### Scrape configuration (targets and jobs)

`prometheus.yml` is **not** a template rendered from variables. It is copied as-is from:

`inventories/files/prometheus/<inventory_hostname>.yml` (falls back to `default.yml`)

To add a scrape target or job for a given host, edit that host's file directly, e.g. `inventories/files/prometheus/monitoring-server01.yml`:

```yaml
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - "127.0.0.1:9090"

  - job_name: node_exporter
    static_configs:
      - targets:
          - "127.0.0.1:9100"
          - "10.10.10.11:9100"   # another box you want scraped from this monitoring server
          - "10.10.10.12:9100"

  - job_name: nginx_exporter
    static_configs:
      - targets:
          - "10.10.10.11:9113"
```

(`10.10.10.11` / `10.10.10.12` above are illustrative private RFC1918 addresses, not real infrastructure — replace them with your own targets.)

Note: `inventories/host_vars/monitoring-server02.yml` and `inventories/host_vars/monitoring-server01.yml` still define `prometheus_prometheus_targets`, `prometheus_node_exporter_targets` and `prometheus_extra_scrape_jobs` — these look like they used to drive a templated config in an earlier version of the role, but current `roles/prometheus/tasks/main.yml` never reads them. They have no effect; the static files above are what actually matters.

Prometheus's own web config (`web.yml`, referenced via `--web.config.file`) currently only ever contains `basic_auth_users: {}` — the template has no variable in it, so the Prometheus web UI on port 9090 has no built-in authentication or TLS. Access is controlled purely by the firewall.

### Firewall / allow-list

Port 22 is always open. Port 9090 is opened to `prometheus_firewall_whitelist_ips` if that list is non-empty; otherwise it falls back to the `ansible_host` of every host in the `observation_servers` group.

## Role: `loki`

### What it does

- Downloads and installs the Loki and Promtail binaries, creates a `loki` system user/group, systemd units, and waits for Loki's `/ready` endpoint.
- Deploys its own nftables ruleset (always).

### Variables (`roles/loki/defaults/main.yml`)

```yaml
loki_version: "2.9.6"
promtail_version: "2.9.6"
```

Only the two version pins are configurable. Everything else — listen addresses (`0.0.0.0:3100` for Loki, `9080` for Promtail), storage paths, and the log paths Promtail scrapes (`/var/log/*log`) — is fixed in `roles/loki/templates/loki.yml.j2` and `promtail.yml.j2`.

### Firewall / allow-list

Loki's nftables template opens port 3100 to `loki_firewall_whitelist_ips` (falling back to the `observation_servers` group's IPs, same logic as Prometheus), and *also* reopens port 9090, using `prometheus_firewall_whitelist_ips` directly (with no group fallback of its own). This matters because every role in `playbooks/monitoring_servers.yml` fully rewrites `/etc/nftables.conf` — see the note under `blackbox_exporter` below for why this still ends up correct in practice.

## Role: `blackbox_exporter`

### What it does

- Downloads and installs the `blackbox_exporter` binary, creates a `blackbox` system user/group and a systemd unit listening on port 9115, and waits for `/metrics` to respond.
- Deploys the final nftables ruleset for the host (see below).

### Variables (`roles/blackbox_exporter/defaults/main.yml`)

```yaml
blackbox_exporter_version: "0.25.0"
blackbox_firewall_whitelist_ips: []
```

### Probe modules

`blackbox.yml` is copied as-is from `inventories/files/blackbox_exporter/<inventory_hostname>.yml` (falls back to `default.yml`). It defines named probe modules (e.g. `proverka_cheka_ru`) that Prometheus's own `blackbox_exporter` scrape job (defined in the matching `inventories/files/prometheus/<host>.yml`) refers to by name and target URL.

### Firewall — the role that has the last word

`playbooks/monitoring_servers.yml` runs `prometheus`, then `loki`, then `blackbox_exporter`, and **every one of them deploys a full `/etc/nftables.conf` (`flush ruleset` included) and reloads it**. Since each role's template completely replaces the previous ruleset, the actually-effective firewall on a monitoring server is whichever role's template ran *last* in the play — currently `blackbox_exporter`. Its template recomputes the allow-lists for all three services and writes one combined ruleset covering ports 22, 9090, 3100 and 9115. `blackbox_firewall_whitelist_ips` falls back to `prometheus_firewall_whitelist_ips` if empty (not to the `observation_servers` group).

In short: if you ever reorder the roles in `playbooks/monitoring_servers.yml`, double-check the firewall rules that end up in place — whichever role runs last determines the final ruleset.

## Firewall / IP Allow-Lists

`inventories/group_vars/monitoring_servers/whitelist.yml` sets the three allow-lists used above:

```yaml
prometheus_firewall_whitelist_ips:
  - <PROXY_IP_1> # example proxy IP
  - <MONITORING_SERVER_IP>

loki_firewall_whitelist_ips:
  - <PROXY_IP_1>
  - <MONITORING_SERVER_IP>
  - <PROXY_IP_2>
  - <PROXY_IP_3>

blackbox_firewall_whitelist_ips:
  - <PROXY_IP_1>
```

If a list is empty, the fallback described under each role above kicks in instead.

## Secrets / Vault

`inventories/group_vars/observation_servers/vault.yml` is an ansible-vault-encrypted file that holds:

```yaml
grafana_admin_password: "YourSecretPassword"
```

Only `playbooks/observation_servers.yml` needs it, since only the `grafana` role reads `grafana_admin_password`:

```bash
ansible-playbook -i inventories/servers.yml playbooks/observation_servers.yml --ask-vault-pass
```

`playbooks/monitoring_servers.yml` does not touch any vaulted variable and does not need `--ask-vault-pass`.

## Notes

- Grafana is installed through the `mirrors.cernet.edu.cn` APT mirror instead of `apt.grafana.com` directly, to work around access blocks.
- Make sure DNS for `grafana_domain` and ports 80/443 are reachable *before* running the `grafana` role, or the Let's Encrypt HTTP-01 challenge will fail.
- Every role deploys nftables by fully replacing `/etc/nftables.conf`. Double-check the IPs in your whitelists before applying — a mistake can lock you out of a host.
