# Example Centralized Logging Infrastructure

Пример Централизованной системы сбора, хранения, визуализации и алертинга логов на базе **Grafana Loki**, **Grafana Alloy** и **Grafana** с автоматизацией развёртывания агента через **Ansible**.

## Возможности

* Централизованный сбор логов с Linux-серверов;
* Автоматическое развёртывание агентов Grafana Alloy через Ansible;
* Преобразование системных логов в JSON-формат(rsyslog) + logrotate;
* Передача логов в Grafana Loki;
* Визуализация поиск логов и алертинг в Grafana;
* LDAP-аутентификация Grafana;
* Поддержка ротации логов;
* Возможность масштабирования инфраструктуры.

---

## Архитектура
![Grafana Platform](docs/Grafana-Loki-schema.drawio.svg)
## Структура проекта

```
.
├── alloy/                     # Ansible роль для установки Alloy
│   ├── defaults/
│   ├── handlers/
│   ├── meta/
│   ├── molecule/
│   ├── tasks/
│   ├── templates/
│   └── vars/
│
├── loki/
│   ├── docker-compose.yaml
│   └── loki-config.yaml
│
├── prometheus-grafana/
│   ├── docker-compose.yml
│   ├── prometheus.yml
│   └── ldap.toml
│
└── README.md
```

---

## Компоненты системы

### Grafana Alloy

Grafana Alloy используется как агент сбора логов.

Основные задачи:

* чтение локальных логов;
* добавление служебных меток;
* отправка логов в Loki.

Конфигурация Alloy предусматривает сбор:

* `/var/log/auth.json`;
* `/var/log/syslog.json`.

Шаблон конфигурации описан в:

```
alloy/templates/config.alloy.j2
```

Логи передаются в Loki посредством блока `loki.write`.

---

### Grafana Loki

Loki отвечает за хранение и обработку логов.

Основные характеристики:

* режим Single Binary;
* TSDB-хранилище;
* файловое хранение чанков;
* retention-политика — 30 дней.

Конфигурация ретенции:

```yaml
table_manager:
  retention_deletes_enabled: true
  retention_period: 720h
```

---

### Grafana

Grafana используется для:

* поиска логов через Explore;
* создания дашбордов;
![Grafana Platform](docs/Drilldown-logs.drawio.svg)
* настройки оповещений;
* управления доступом через LDAP.

Datasource Loki создаётся автоматически при старте контейнера.

---

## Развёртывание Loki

```bash
cd loki
docker compose up -d
```

Проверка доступности:

```bash
curl http://localhost:3100/ready
```

---

## Развёртывание Grafana

```bash
cd prometheus-grafana
docker compose up -d
```

После запуска Grafana будет доступна по адресу:

```
http://<HOST>:3000
```

---

## Развёртывание Alloy

### Требования

* Ansible ≥ 2.13;
* Debian / Ubuntu;
* SSH-доступ к серверам.

---

### Установка

```yaml
- hosts: linux
  become: true

  roles:
    - role: alloy
```

---

### Запуск

```bash
ansible-playbook playbook.yml
```

---

### Удаление Alloy

```bash
ansible-playbook playbook.yml -e "alloy_uninstall=true"
```

---

## Используемые метки

Alloy автоматически добавляет следующие labels:

job          - Тип логов    
host         - Имя сервера 
distribution - Дистрибутив ОС

Метки добавляются на этапе обработки логов.

---

## Примеры алертов

### SSH Authentication Alert

Срабатывает при обнаружении неудачных SSH-подключений.

```
sum by (user, ip, host) (
  count_over_time(
    {job="auth-logs"}
    | json
    | program = "sshd"
    | message =~ ".*Connection closed by authenticating.*"
    | line_format "{{trim .message}}"
    | pattern `<_>Connection closed by authenticating user <user> <ip> port <_> [preauth]<_>`
    [20m]
  )
) >= 1
```
![Grafana Platform](docs/auth-alerting.drawio.svg)

---

### System Error Alert

Срабатывает при появлении сообщений об ошибках.

```
sum by (service, host, message) (
  count_over_time(
    {job="system-logs"}
    | json
    | message =~ ".*Failed.*|.*Error.*|.*Critical.*"
    | line_format "{{trim .message}}"
    | pattern `<service>: <_>`
    | service != ""
    [20m]
  )
) >= 1
```
![Grafana Platform](docs/systemd-alerting.drawio.svg)

---

## Проверка работы

### Проверить статус Alloy

```bash
systemctl status alloy
```

---

### Проверить готовность Alloy

```bash
curl http://127.0.0.1:12345/-/ready
```

---

### Проверить поступление логов в Loki

```bash
curl http://loki:3100/ready
```

---

### Проверить наличие логов

Grafana → Explore → Loki

Пример запроса:

```logql
{job="auth-logs"}
```

---


## Используемые технологии

* Grafana
* Grafana Loki
* Grafana Alloy
* Ansible
* Docker Compose
* LogQL
* rsyslog
* LDAP

