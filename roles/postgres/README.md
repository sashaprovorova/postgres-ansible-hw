# Ansible Role: PostgreSQL

Ansible-роль для автоматической установки и настройки PostgreSQL на Ubuntu.

Роль устанавливает PostgreSQL из стандартных репозиториев Ubuntu, инициализирует кластер в заданном каталоге, настраивает конфигурацию и аутентификацию, создаёт базы данных и пользователей, а также устанавливает PostgreSQL Exporter для мониторинга.

## Требования

- Ubuntu 20.04
- Ansible
- Python 3
- Коллекция `community.postgresql`
- Права sudo на целевом хосте

## Возможности

- Установка PostgreSQL из стандартных репозиториев Ubuntu
- Настраиваемый каталог данных (по умолчанию `/data/postgres/data`)
- Идемпотентная инициализация через `pg_ctl initdb`
- Настройка `postgresql.conf` через словарь
- Настройка `pg_hba.conf` через список словарей
- Управление PostgreSQL через systemd
- Создание пользователей и баз данных
- Установка PostgreSQL Exporter
- Создание пользователя мониторинга с правами `pg_monitor`
- Запуск и автозапуск PostgreSQL Exporter
- Перезапуск сервисов через handlers при изменении конфигурации

## Переменные роли

Все пользовательские параметры определены в `defaults/main.yml` и имеют префикс `postgres__`.

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `postgres__version` | `"12"` | Версия PostgreSQL |
| `postgres__data_dir` | `/data/postgres/data` | Каталог данных PostgreSQL |
| `postgres__opts` | Словарь | Параметры `postgresql.conf` |
| `postgres__hba_rules` | Список словарей | Правила аутентификации `pg_hba.conf` |
| `postgres__databases` | `[{name: app_db}]` | Список создаваемых баз данных |
| `postgres__users` | Список пользователей | Пользователи PostgreSQL |
| `postgres__exporter_package` | `prometheus-postgres-exporter` | Пакет экспортера |
| `postgres__exporter_port` | `9187` | Порт экспортера |
| `postgres__exporter_user` | `prometheus` | Пользователь мониторинга |

Внутренние переменные определены в `vars/main.yml` и имеют префикс `__`.

## Пример конфигурации

```yaml
postgres__version: "12"

postgres__data_dir: /data/postgres/data

postgres__opts:
  listen_addresses: "'localhost'"
  port: 5432
  max_connections: 100
  shared_buffers: 128MB

postgres__hba_rules:
  - type: local
    database: all
    user: postgres
    address: ""
    method: peer

  - type: local
    database: all
    user: all
    address: ""
    method: peer

  - type: host
    database: all
    user: all
    address: 127.0.0.1/32
    method: scram-sha-256

  - type: host
    database: all
    user: all
    address: "::1/128"
    method: scram-sha-256

postgres__databases:
  - name: app_db

postgres__users:
  - name: app_user
    password: "{{ vault_postgres_app_password }}"
    databases:
      - app_db

postgres__exporter_package: prometheus-postgres-exporter
postgres__exporter_port: 9187
postgres__exporter_user: prometheus
```

Пароли пользователей хранятся в зашифрованном файле Ansible Vault:

```text
inventories/main/group_vars/all/vault.yml
```

## Пример playbook

Файл `postgres.play.yml`:

```yaml
---
- name: Configure PostgreSQL on vm1
  hosts: vm1
  become: true

  roles:
    - postgres
```

## Запуск

Установить необходимые коллекции:

```bash
ansible-galaxy collection install -r requirements.yml
```

Проверить доступность целевого хоста:

```bash
ansible vm1 -m ping
```

Проверить синтаксис playbook:

```bash
ansible-playbook postgres.play.yml --syntax-check
```

Запустить playbook:

```bash
ansible-playbook postgres.play.yml --ask-vault-pass --ask-become-pass
```

## Проверка работоспособности

### PostgreSQL и PostgreSQL Exporter

Проверить состояние сервисов:

```bash
systemctl is-active postgresql-custom prometheus-postgres-exporter
```

Ожидаемый результат:

```text
active
active
```

### Проверка SQL-запроса

```bash
sudo -u postgres psql -c "SELECT 1;"
```

Ожидаемый результат:

```text
 ?column?
----------
        1
(1 row)
```

### Проверка баз данных

```bash
sudo -u postgres psql -c "\l"
```

В списке должна присутствовать база данных `app_db`.

### Проверка пользователей

```bash
sudo -u postgres psql -c "\du"
```

В списке должен присутствовать пользователь `app_user`.

### Проверка PostgreSQL Exporter

```bash
curl -s http://127.0.0.1:9187/metrics | grep -E '^pg_up|^pg_exporter_last_scrape_error'
```

Ожидаемый результат:

```text
pg_exporter_last_scrape_error 0
pg_up 1
```

Значение `pg_up 1` подтверждает успешное подключение экспортера к PostgreSQL.

## Идемпотентность

Роль разработана с учётом идемпотентности:

- Пакеты устанавливаются через `apt` со `state: present`.
- Каталоги создаются через `file` со `state: directory`.
- Инициализация PostgreSQL выполняется только при отсутствии файла `PG_VERSION`.
- Параметры `postgresql.conf` изменяются через `lineinfile`, без удаления остальных настроек.
- Файл `pg_hba.conf` формируется из списка словарей через Jinja2-шаблон.
- Перезапуск сервисов выполняется через handlers при изменении конфигурации.

Повторный запуск:

```bash
ansible-playbook postgres.play.yml --ask-vault-pass --ask-become-pass
```

Полученный результат:

```text
PLAY RECAP
vm1 : ok=19 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Значение `changed=0` подтверждает, что повторный запуск playbook не вносит изменений в уже настроенную систему.
