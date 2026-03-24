# monitoring server configuration

Этот репозиторий содержит роли Ansible для развёртывания:
- Grafana (с nginx reverse proxy, Let's Encrypt, provisioning data sources)
- Prometheus (опционально node_exporter, web config)
- Loki (с promtail)
- nftables (минимальные правила)

## Структура

- `roles/grafana` — установка Grafana + nginx + TLS + provisioning
- `roles/prometheus` — установка Prometheus + node_exporter + firewall
- `roles/loki` — установка Loki + promtail + firewall
- `inventories/` — инвентори и group_vars
- `playbooks/` — плейбуки окружений

## Быстрый старт

1. Заполни инвентори

`inventories/servers.yml`:
```yaml
all:
  vars:
    ssh_key_path: /home/<USER>/.ssh/<SSH_KEY>
  children:
    observation_servers:
      hosts:
        <MONITORING_SERVER_IP>:
          ansible_user: <ANSIBLE_USER>
          ansible_ssh_private_key_file: "{{ ssh_key_path }}"
    monitoring_servers:
      hosts:
        <OBSERVATION_SERVER_IP>:
          ansible_user: <ANSIBLE_USER>
          ansible_ssh_private_key_file: "{{ ssh_key_path }}"
```

2. Запусти плейбуки

Grafana (monitoring сервер):
```bash
ansible-playbook -i inventories/servers.yml playbooks/monitoring_server.yml --ask-vault-pass
```

Prometheus + Loki (observation сервер):
```bash
ansible-playbook -i inventories/servers.yml playbooks/monitoring_servers.yml
```

## Роль Grafana

### Что делает

- Устанавливает Grafana из APT зеркала
- Конфигурирует nginx reverse proxy и TLS (Let's Encrypt)
- Настраивает provisioning data sources (Prometheus, Loki)
- Опционально включает firewall
- Сбрасывает админ-пароль по флагу

### Основные переменные

`roles/grafana/defaults/main.yml`:

```yaml
grafana_domain: <GRAFANA_DOMAIN>
grafana_enable_https: true
grafana_letsencrypt_email: "admin@{{ grafana_domain }}"
grafana_letsencrypt_webroot: /var/www/letsencrypt

grafana_firewall_tcp_ports:
  - 22
  - 80
  - 8080
  - 443
  - 9090
  - 3100

grafana_admin_password: ""
grafana_admin_password_file: /var/lib/grafana/.admin_password_set

grafana_prometheus_url: ""
grafana_prometheus_datasource_name: "Prometheus"

grafana_loki_url: ""
grafana_loki_datasource_name: "Loki"

# Feature toggles: enable/disable optional Grafana tasks
grafana_install_nginx: true
grafana_enable_firewall: true
grafana_admin_force_reset: false
```

### Provisioning datasource

Datasource файлы появляются на сервере тут:
- `/etc/grafana/provisioning/datasources/prometheus.yml`
- `/etc/grafana/provisioning/datasources/loki.yml`

По умолчанию IP берётся из группы `monitoring_servers`.

## Роль Prometheus

### Что делает

- Устанавливает Prometheus (binaries)
- Создаёт пользователя/группу
- Разворачивает конфиг и systemd unit
- Включает node_exporter
- Проверяет readiness `/ready`
- Настраивает firewall (nftables)

### Основные переменные

`roles/prometheus/defaults/main.yml`:

```yaml
prometheus_version: "2.49.1"
prometheus_arch: "linux-amd64"
prometheus_download_url: "https://github.com/prometheus/prometheus/releases/download/v{{ prometheus_version }}/prometheus-{{ prometheus_version }}.{{ prometheus_arch }}.tar.gz"

prometheus_web_listen_address: "0.0.0.0:9090"
prometheus_enable_web_config: true
prometheus_web_config_file: "{{ prometheus_config_dir }}/web.yml"
prometheus_basic_auth_users: {}

prometheus_enable_node_exporter: true
prometheus_prometheus_targets:
  - "127.0.0.1:9090"
prometheus_node_exporter_targets:
  - "127.0.0.1:9100"
prometheus_extra_scrape_jobs: []

prometheus_manage_firewall: true
prometheus_firewall_tcp_ports:
  - 22
  - 9090
prometheus_firewall_allow_9090_from_monitoring: true
prometheus_firewall_allowed_ips: []
prometheus_firewall_whitelist_ips: []
```

### Разные targets для разных хостов

Задай переменные в `inventories/host_vars/<ip>.yml`, например:

```yaml
prometheus_prometheus_targets:
  - "127.0.0.1:9090"
  - "10.10.10.11:9090"

prometheus_node_exporter_targets:
  - "127.0.0.1:9100"
  - "10.10.10.11:9100"
  - "10.10.10.12:9100"

prometheus_extra_scrape_jobs:
  - job_name: nginx_exporter
    static_configs:
      - targets:
          - "10.10.10.11:9113"
  - job_name: alloy
    static_configs:
      - targets:
          - "10.10.10.11:12345"
```

### Белый список IP для Prometheus

Создай файл:
`group_vars/monitoring_servers/whitelist.yml`

```yaml
prometheus_firewall_whitelist_ips:
  - <MONITORING_SERVER_IP>
```

Если список пустой, используется группа `observation_servers`.

## Роль Loki

### Что делает

- Устанавливает Loki и promtail (binaries)
- Создаёт пользователя/группу
- Разворачивает конфиг и systemd unit
- Проверяет readiness `/ready`
- Настраивает firewall (nftables)

### Основные переменные

`roles/loki/defaults/main.yml`:

```yaml
loki_version: "2.9.6"
loki_arch: "linux-amd64"
loki_download_url: "https://github.com/grafana/loki/releases/download/v{{ loki_version }}/loki-{{ loki_arch }}.zip"

loki_listen_address: "0.0.0.0:3100"

promtail_version: "2.9.6"
promtail_arch: "linux-amd64"
promtail_download_url: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-{{ promtail_arch }}.zip"

loki_firewall_tcp_ports:
  - 22
  - 3100
loki_firewall_allow_3100_from_monitoring: true
loki_firewall_allowed_ips: []
loki_firewall_whitelist_ips: []
```

### Белый список IP для Loki

Файл:
`group_vars/monitoring_servers/whitelist.yml`

```yaml
loki_firewall_whitelist_ips:
  - <MONITORING_SERVER_IP>
```

Если список пустой, используется группа `observation_servers`.

## Vault

Админ-пароль Grafana хранится в vault:
`inventories/group_vars/observation_servers/vault.yml`

Пример содержимого:
```yaml
grafana_admin_password: "YourSecretPassword"
```

Запуск с вводом пароля:
```bash
ansible-playbook -i inventories/servers.yml playbooks/monitoring_server.yml --ask-vault-pass
```

## Плейбуки

`playbooks/monitoring_server.yml` — Grafana

`playbooks/monitoring_servers.yml` — Prometheus + Loki

## Примечания

- Grafana устанавливается через зеркало `mirrors.cernet.edu.cn` из-за блокировок прямого доступа.
- Проверь DNS для домена Grafana и доступность 80/443, иначе Let's Encrypt не выдаст сертификат.
- Для firewall убедись, что нужные IP указаны, иначе потеряешь доступ.
