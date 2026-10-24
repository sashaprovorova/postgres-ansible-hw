# Ansible PostgreSQL

Проект для автоматизированной установки и настройки PostgreSQL на Ubuntu с помощью Ansible.

Реализована роль `postgres`, которая устанавливает PostgreSQL из стандартных репозиториев Ubuntu, настраивает сервер и аутентификацию, создаёт базы данных и пользователей, а также устанавливает PostgreSQL Exporter для мониторинга.

## Структура проекта

```text
postgres-ansible-hw/
├── README.md
├── ansible.cfg
├── inventories/
│   └── main/
│       ├── hosts.yaml
│       └── group_vars/
│           └── all/
│               └── vault.yml
├── postgres.play.yml
├── requirements.yml
└── roles/
    └── postgres/
        ├── defaults/
        ├── handlers/
        ├── tasks/
        ├── templates/
        ├── vars/
        └── README.md
```

## Запуск

Установить необходимые коллекции Ansible:

```bash
ansible-galaxy collection install -r requirements.yml
```

Запустить playbook:

```bash
ansible-playbook postgres.play.yml --ask-vault-pass --ask-become-pass
```

Для запуска необходим пароль Ansible Vault.

## Проверка

Проверить состояние сервисов:

```bash
systemctl is-active postgresql-custom prometheus-postgres-exporter
```

Проверить подключение к PostgreSQL:

```bash
sudo -u postgres psql -c "SELECT 1;"
```

Проверить метрики экспортера:

```bash
curl -s http://localhost:9187/metrics | grep '^pg_up'
```

При повторном запуске playbook должен выполняться идемпотентно (`changed=0`).

## Документация

Подробное описание роли, её переменных и настроек находится в [roles/postgres/README.md](roles/postgres/README.md).
