# remnawave-monitoring-ansible

<div align="center"><a href="README.md">🇬🇧 English</a><br><br></div>

Ansible-плейбуки для развёртывания и управления прокси-панелью [Remnawave](https://github.com/remnawave/panel), нодами и стеком мониторинга.

Подробное руководство — в [GUIDE_RU.md](GUIDE_RU.md).

## Что разворачивается

**Панель** — бэкенд Remnawave + Nginx с TLS, защитой через секретный заголовок и автоматически сгенерированными учётными данными администратора.

**Ноды** — экземпляры Remnanode с Nginx, TLS и автоматической регистрацией на панели через API.

**Мониторинг** — Prometheus + Grafana + Whitebox + Blackbox на отдельном сервере. Метрики панели собираются по TLS, VPN-соединения проверяются через Whitebox, доступность нод по HTTPS — через Blackbox. Дашборд hemera предустановлен в Grafana.

## Требования

- [Ansible 2.12+](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) на Linux или WSL (Windows Subsystem for Linux)
- Целевые серверы на базе Debian (Debian, Ubuntu и т.д.)
- SSH-доступ ко всем серверам (ключ или пароль); если ключ защищён паролем, добавьте его в `ssh-agent` перед запуском (`eval $(ssh-agent) && ssh-add /path/to/key`)
- `sshpass` при подключении по паролю (`apt install sshpass`)
- Отдельные DNS A-записи для панели и каждой ноды, указывающие на соответствующие IP-адреса серверов

## Установка

**1. Клонировать репозиторий**

```bash
git clone https://github.com/kyoug3n/remnawave-monitoring-ansible.git
cd remnawave-monitoring-ansible
```

**2. Настроить инвентарь**

Отредактируйте `inventory/hosts.yml` — все настраиваемые значения аннотированы прямо в файле.

**3. Запустить**

Подробное пошаговое руководство — в [GUIDE_RU.md](GUIDE_RU.md).

```bash
ansible-playbook playbook.yml
```

Или по отдельности:

```bash
ansible-playbook playbook-panel.yml
ansible-playbook playbook-nodes.yml
ansible-playbook playbook-monitoring.yml
```

Учётные данные выводятся в конце каждого запуска и сохраняются на соответствующем сервере:
- Администратор панели: `/opt/remnawave/admin_credentials.txt`
- Секретный заголовок: `/opt/remnawave/secret_header.txt`
- Grafana: `/opt/monitoring/grafana_credentials.txt`

## Добавление ноды

1. Добавьте новую запись в `nodes.hosts` в `inventory/hosts.yml`:

```yaml
node_ams:
  ansible_host: "192.168.0.2"
  node_name: "ams"
  node_port: 3333
  node_domain: "ams.example.com"
```

2. Направьте домен на сервер, затем запустите:

```bash
ansible-playbook playbook-nodes.yml
```

Нода автоматически регистрируется на панели и добавляется во все существующие отряды.

## Стек мониторинга

| Сервис     | Порт по умолчанию | Описание                                  |
|------------|-------------------|-------------------------------------------|
| Prometheus | 9090              | Сбор метрик                               |
| Grafana    | 3000              | Дашборды (hemera предустановлен)          |
| Whitebox   | 9116              | Проверка VPN-соединений                   |
| Blackbox   | 9115              | Проверка доступности нод по HTTPS         |

Все порты привязаны к `127.0.0.1`. Для доступа используйте SSH-туннель:

```bash
ssh -L 3000:127.0.0.1:3000 root@192.168.2.1
```

## Безопасность

- SSH усилен (настраиваемый вход root, аутентификация по паролю, максимальное число попыток, fail2ban)
- UFW-файрвол на всех серверах
- ufw-docker на панели для предотвращения обхода UFW через Docker
- Панель скрыта за секретным заголовком
- Эндпоинт метрик проксируется через nginx с TLS, доступ ограничен IP-адресом сервера мониторинга
- Все генерируемые секреты — буквенно-цифровые (безопасны для Docker `.env`)

## Благодарности

- [Remnawave](https://github.com/remnawave/backend) — панель управления прокси
- [Remnanode](https://github.com/remnawave/node) — Xray-нода
- [Prometheus](https://github.com/prometheus/prometheus) — сбор метрик
- [Grafana](https://github.com/grafana/grafana) — дашборды
- [Whitebox](https://github.com/quyxishi/whitebox) — проверка VPN-соединений и дашборд hemera
- [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter) — проверка доступности по HTTPS
- [ufw-docker](https://github.com/chaifeng/ufw-docker) — интеграция UFW и Docker
- [acme.sh](https://github.com/acmesh-official/acme.sh) — управление TLS-сертификатами

## Лицензия

[MIT](LICENSE)
