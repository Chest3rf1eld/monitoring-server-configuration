[Read this in English](README.md)

# monitoring-server-configuration

Ansible-плейбуки и роли для развёртывания небольшого распределённого стека мониторинга:

- **Grafana** за nginx reverse proxy с TLS от Let's Encrypt, с datasources и правилами алертинга, которые провижинятся из статических YAML-файлов.
- **Prometheus** + **node_exporter** — на каждом мониторируемом сервере.
- **blackbox_exporter** — на каждом мониторируемом сервере, для внешних HTTP-проверок (например, «доступен ли сайт»).
- **Loki** + **promtail** — на каждом мониторируемом сервере, для сбора логов.
- **nftables** — правила файрвола, которые разворачивает каждая роль самостоятельно.

## Архитектура

В инвентори две группы, и их названия могут ввести в заблуждение при первом взгляде:

- `observation_servers` — сервер(а), где живёт **Grafana**. Отсюда вы *наблюдаете* за дашбордами и алертами.
- `monitoring_servers` — сервер(а), которые *мониторятся*. На каждом из них крутится собственный локальный Prometheus, node_exporter, blackbox_exporter и Loki/promtail.

Отдельного центрального Prometheus или Loki здесь нет. Вместо этого в Grafana (на `observation_servers`) провижинится по одному Prometheus- и Loki-datasource *на каждый* мониторинг-сервер, указывающие на его публичный IP (см. `inventories/files/grafana/datasources/`). То есть каждый мониторинг-сервер самодостаточен: сам скрейпит свой `node_exporter`/`blackbox_exporter`, сам отправляет свои логи, а по сети к нему обращается только Grafana.

`playbooks/observation_servers.yml` нацелен на группу `observation_servers` и разворачивает роль `grafana`.
`playbooks/monitoring_servers.yml` нацелен на группу `monitoring_servers` и разворачивает роли `prometheus`, `loki`, затем `blackbox_exporter` — именно в этом порядке.

## Структура репозитория

- `ansible.cfg` — задаёт `roles_path = ./roles` и отключает проверку SSH host key. Инвентори по умолчанию не задан, поэтому `-i` всегда нужно указывать явно.
- `inventories/servers.yml` — инвентори: хосты, группы, параметры SSH-подключения.
- `inventories/group_vars/monitoring_servers/whitelist.yml` — белые списки IP для файрвола (`prometheus_firewall_whitelist_ips`, `loki_firewall_whitelist_ips`, `blackbox_firewall_whitelist_ips`).
- `inventories/group_vars/observation_servers/vault.yml` — секреты, зашифрованные ansible-vault (`grafana_admin_password`).
- `inventories/host_vars/<host>.yml` — переменные конкретных хостов.
- `inventories/files/<роль>/<inventory_hostname>.yml` (с фолбэком на `default.yml`) — статические, редактируемые вручную конфиги: scrape-конфигурация Prometheus, probe-модули blackbox_exporter, datasources и правила алертинга Grafana. Эти файлы копируются на сервер как есть — это **не** Jinja-шаблоны и они **не** генерируются из переменных ролей.
- `playbooks/observation_servers.yml`, `playbooks/monitoring_servers.yml` — два плейбука.
- `roles/grafana`, `roles/prometheus`, `roles/loki`, `roles/blackbox_exporter` — четыре роли.

## Быстрый старт

1. Заполни `inventories/servers.yml`, например:

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

2. Задай `grafana_admin_password` в vault-файле (см. раздел [Секреты / Vault](#секреты--vault) ниже).

3. Запусти плейбуки:

Grafana, на observation-сервере (нужен пароль от vault, потому что там лежит `grafana_admin_password`):

```bash
ansible-playbook -i inventories/servers.yml playbooks/observation_servers.yml --ask-vault-pass
```

Prometheus + Loki + blackbox_exporter, на каждом мониторинг-сервере (секретов из vault здесь нет):

```bash
ansible-playbook -i inventories/servers.yml playbooks/monitoring_servers.yml
```

## Роль `grafana`

### Что делает

- Устанавливает Grafana из официального APT-репозитория, но через зеркало `mirrors.cernet.edu.cn` — это обход блокировок прямого доступа к `apt.grafana.com`.
- Устанавливает и настраивает nginx как reverse proxy перед Grafana, получает и продлевает сертификат Let's Encrypt через HTTP-01 (webroot) challenge — всегда, отключить nginx или TLS отдельным флагом нельзя.
- Провижинит datasources и правила алертинга Grafana из статических файлов.
- Один раз задаёт пароль администратора Grafana.
- Разворачивает nftables-правила, которые пропускают только 22, 80 и 443 — это зашито прямо в шаблон, никакой переменной со списком портов не существует.

### Переменные (`roles/grafana/defaults/main.yml`)

```yaml
grafana_admin_password: ""
grafana_domain: <GRAFANA_DOMAIN>
grafana_enable_https: true
grafana_letsencrypt_email: "admin@<GRAFANA_DOMAIN>"
grafana_letsencrypt_webroot: /var/www/letsencrypt
grafana_manage_alerting_provisioning: true
```

Чтобы пароль администратора реально поменялся, `grafana_admin_password` должен быть непустым (обычно он задаётся через vault) — с пустым значением задача просто ничего не делает.

Переменных `grafana_firewall_tcp_ports`, `grafana_install_nginx`, `grafana_enable_firewall`, `grafana_prometheus_url`/`grafana_loki_url` или `grafana_admin_force_reset` в роли нет: nginx, TLS и файрвол выполняются всегда, а адреса datasources берутся из статических файлов ниже, а не из переменных роли.

### Datasources и алертинг

Datasources и правила алертинга Grafana провижинятся из обычного YAML, который копируется на сервер без изменений:

- `inventories/files/grafana/datasources/<inventory_hostname>.yml` (фолбэк на `default.yml`) → `/etc/grafana/provisioning/datasources/datasources.yml`
- `inventories/files/grafana/alerting/<inventory_hostname>.yml` (фолбэк на `default.yml`) → `/etc/grafana/provisioning/alerting/provision.yml` (только если `grafana_manage_alerting_provisioning` включён)

Чтобы добавить, изменить или удалить datasource или правило алерта, редактируй эти файлы напрямую — например, `inventories/files/grafana/datasources/grafana01.yml` перечисляет по одному Prometheus- и Loki-datasource на каждый мониторинг-сервер, с адресом на публичный IP этого сервера и порт 9090/3100.

### Как на самом деле сбрасывается пароль администратора

Пароль администратора задаётся командой `grafana-cli admin reset-admin-password` только один раз: после первого успешного сброса на сервере создаётся файл-маркер `/var/lib/grafana/.admin_password_set`, и при всех последующих запусках роль пропускает сброс пароля — даже если значение `grafana_admin_password` поменялось. Чтобы принудительно задать новый пароль, нужно вручную удалить этот файл-маркер на сервере и повторно запустить плейбук.

## Роль `prometheus`

### Что делает

- Скачивает и устанавливает бинарники Prometheus и node_exporter из GitHub releases (node_exporter ставится всегда — отключить его отдельным флагом нельзя).
- Создаёт отдельного системного пользователя/группу `prometheus`, systemd-юниты, проверяет конфиг через `promtool` перед запуском.
- Ждёт, пока `/-/ready` не начнёт отвечать 200.
- Разворачивает nftables-правила (тоже всегда — отключить этот шаг тоже нельзя).

### Переменные (`roles/prometheus/defaults/main.yml`)

```yaml
prometheus_version: "2.49.1"
node_exporter_version: "1.8.1"
prometheus_firewall_whitelist_ips: []
```

Это весь набор переменных роли. Переменных `prometheus_prometheus_targets`, `prometheus_node_exporter_targets`, `prometheus_extra_scrape_jobs`, `prometheus_web_listen_address`, `prometheus_basic_auth_users`, `prometheus_manage_firewall` или `prometheus_firewall_tcp_ports` в роли нет.

### Scrape-конфигурация (targets и jobs)

`prometheus.yml` — это **не** шаблон, рендерящийся из переменных. Он копируется как есть из:

`inventories/files/prometheus/<inventory_hostname>.yml` (фолбэк на `default.yml`)

Чтобы добавить target или job для конкретного хоста, редактируй файл этого хоста напрямую, например `inventories/files/prometheus/monitoring-server01.yml`:

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
          - "10.10.10.11:9100"   # ещё один сервер, который нужно скрейпить с этого мониторинг-сервера
          - "10.10.10.12:9100"

  - job_name: nginx_exporter
    static_configs:
      - targets:
          - "10.10.10.11:9113"
```

(`10.10.10.11` / `10.10.10.12` выше — иллюстративные приватные адреса из диапазона RFC1918, а не реальная инфраструктура; замени их на свои targets.)

Обрати внимание: в `inventories/host_vars/monitoring-server02.yml` и `inventories/host_vars/monitoring-server01.yml` до сих пор объявлены переменные `prometheus_prometheus_targets`, `prometheus_node_exporter_targets` и `prometheus_extra_scrape_jobs` — похоже, раньше они действительно использовались для генерации конфига из шаблона в более старой версии роли, но текущий `roles/prometheus/tasks/main.yml` их нигде не читает. Никакого эффекта они не дают — реально работают только статические файлы, описанные выше.

Собственный веб-конфиг Prometheus (`web.yml`, который передаётся через `--web.config.file`) сейчас всегда содержит только `basic_auth_users: {}` — в шаблоне нет ни одной переменной, поэтому веб-интерфейс Prometheus на порту 9090 не имеет встроенной аутентификации или TLS. Доступ к нему регулируется исключительно файрволом.

### Файрвол / белый список

Порт 22 открыт всегда. Порт 9090 открывается для `prometheus_firewall_whitelist_ips`, если этот список непустой; иначе используется фолбэк — `ansible_host` всех хостов из группы `observation_servers`.

## Роль `loki`

### Что делает

- Скачивает и устанавливает бинарники Loki и Promtail, создаёт системного пользователя/группу `loki`, systemd-юниты, ждёт, пока Loki не ответит на `/ready`.
- Разворачивает свои собственные nftables-правила (тоже всегда).

### Переменные (`roles/loki/defaults/main.yml`)

```yaml
loki_version: "2.9.6"
promtail_version: "2.9.6"
```

Настраиваются только версии. Всё остальное — адреса, на которых слушают сервисы (`0.0.0.0:3100` у Loki, `9080` у Promtail), пути хранения данных и пути логов, которые скрейпит Promtail (`/var/log/*log`) — зашито в `roles/loki/templates/loki.yml.j2` и `promtail.yml.j2`.

### Файрвол / белый список

Шаблон nftables у Loki открывает порт 3100 для `loki_firewall_whitelist_ips` (с тем же фолбэком на группу `observation_servers`, что и у Prometheus), а *заодно* повторно открывает порт 9090 — но уже напрямую по `prometheus_firewall_whitelist_ips`, без собственного фолбэка на группу. Это важно, потому что каждая роль в `playbooks/monitoring_servers.yml` полностью перезаписывает `/etc/nftables.conf` — почему в итоге всё равно получается корректный результат, смотри примечание в разделе про `blackbox_exporter` ниже.

## Роль `blackbox_exporter`

### Что делает

- Скачивает и устанавливает бинарник `blackbox_exporter`, создаёт системного пользователя/группу `blackbox` и systemd-юнит, слушающий порт 9115, ждёт ответа от `/metrics`.
- Разворачивает итоговые nftables-правила для хоста (см. ниже).

### Переменные (`roles/blackbox_exporter/defaults/main.yml`)

```yaml
blackbox_exporter_version: "0.25.0"
blackbox_firewall_whitelist_ips: []
```

### Probe-модули

`blackbox.yml` копируется как есть из `inventories/files/blackbox_exporter/<inventory_hostname>.yml` (фолбэк на `default.yml`). В нём описаны именованные probe-модули (например, `proverka_cheka_ru`), на которые по имени и целевому URL ссылается собственная scrape-job `blackbox_exporter` в Prometheus (описанная в соответствующем `inventories/files/prometheus/<host>.yml`).

### Файрвол — последнее слово за этой ролью

`playbooks/monitoring_servers.yml` запускает роли `prometheus`, затем `loki`, затем `blackbox_exporter`, и **каждая из них разворачивает полный `/etc/nftables.conf` (с `flush ruleset`) и применяет его заново**. Поскольку шаблон каждой роли целиком заменяет предыдущий набор правил, реально действующий на мониторинг-сервере файрвол — это тот, чей шаблон отработал *последним* в плейбуке, а это сейчас `blackbox_exporter`. Его шаблон заново пересчитывает белые списки для всех трёх сервисов и пишет один общий набор правил, покрывающий порты 22, 9090, 3100 и 9115. `blackbox_firewall_whitelist_ips`, если пуст, откатывается на `prometheus_firewall_whitelist_ips` (а не на группу `observation_servers`).

Короче говоря: если когда-нибудь поменяешь порядок ролей в `playbooks/monitoring_servers.yml`, обязательно перепроверь, какие правила файрвола в итоге получаются — действующий набор правил определяет та роль, что выполняется последней.

## Файрвол / белые списки IP

`inventories/group_vars/monitoring_servers/whitelist.yml` задаёт три белых списка, которые используются выше:

```yaml
prometheus_firewall_whitelist_ips:
  - <PROXY_IP_1> # пример IP прокси
  - <MONITORING_SERVER_IP>

loki_firewall_whitelist_ips:
  - <PROXY_IP_1>
  - <MONITORING_SERVER_IP>
  - <PROXY_IP_2>
  - <PROXY_IP_3>

blackbox_firewall_whitelist_ips:
  - <PROXY_IP_1>
```

Если список пуст, вступает в силу фолбэк, описанный в разделе соответствующей роли выше.

## Секреты / Vault

`inventories/group_vars/observation_servers/vault.yml` — файл, зашифрованный ansible-vault, в котором хранится:

```yaml
grafana_admin_password: "YourSecretPassword"
```

Он нужен только для `playbooks/observation_servers.yml`, потому что `grafana_admin_password` читает только роль `grafana`:

```bash
ansible-playbook -i inventories/servers.yml playbooks/observation_servers.yml --ask-vault-pass
```

`playbooks/monitoring_servers.yml` не обращается ни к одной переменной из vault, поэтому `--ask-vault-pass` для него не нужен.

## Примечания

- Grafana устанавливается через зеркало `mirrors.cernet.edu.cn`, а не напрямую через `apt.grafana.com` — это обход блокировок доступа.
- Перед запуском роли `grafana` убедись, что DNS для `grafana_domain` настроен, а порты 80/443 доступны снаружи — иначе HTTP-01 challenge Let's Encrypt не пройдёт.
- Каждая роль разворачивает nftables, полностью заменяя `/etc/nftables.conf`. Перед применением обязательно проверь IP в белых списках — ошибка может лишить тебя доступа к серверу.
