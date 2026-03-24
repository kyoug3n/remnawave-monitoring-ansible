# Руководство по развёртыванию

Это руководство охватывает полное развёртывание с нуля — настройку DNS, установку Ansible, конфигурацию инвентаря, запуск плейбуков и доступ к стеку мониторинга.

## Что потребуется

- Минимум 2 сервера (панель + нода), 3 — если разворачивается мониторинг
- Домен с доступом к настройкам DNS
- Linux-машина или WSL для запуска Ansible

---

## 1. Настройка DNS

Создайте A-запись для каждого домена в панели вашего DNS-провайдера:

| Тип | Имя | Значение |
|-----|-----|----------|
| A | `panel.example.com` | `192.168.1.1` |
| A | `node.example.com` | `192.168.0.1` |

Каждый домен должен указывать на свой сервер — один IP для нескольких доменов не подойдёт. Распространение DNS может занять несколько минут. Проверить можно так:

```bash
dig panel.example.com
```

Серверу мониторинга домен не нужен.

---

## 2. Установка Ansible

На Ubuntu/Debian (или WSL):

```bash
sudo apt update
sudo apt install ansible sshpass -y
```

`sshpass` нужен только при подключении по паролю, а не по SSH-ключу.

Проверьте версию:

```bash
ansible --version
```

Требуется Ansible 2.12 или новее.

---

## 3. Клонирование репозитория

```bash
git clone https://github.com/kyoug3n/remnawave-monitoring-ansible.git
cd remnawave-monitoring-ansible
```

---

## 4. Настройка инвентаря

Откройте `inventory/hosts.yml` и заполните значения. Файл аннотирован — всё, что помечено комментарием, нужно задать. Блоки внутри `# ↓ do not change ↓` трогать не нужно.

Минимально необходимо указать:
- `ansible_host` для каждого сервера
- `remnawave_panel_domain` в `panel.vars`
- `node_name`, `node_port`, `node_domain` для каждой ноды
- `ssh_key` или `ssh_password` в `all.vars`

Если SSH-ключ защищён паролем, загрузите его в агент перед запуском:

```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
```

---

## 5. (Опционально) Создание vault-файла

По умолчанию все учётные данные генерируются автоматически. Если хотите задать свои — создайте vault-файл:

```bash
EDITOR=nano ansible-vault create vault.yml
```

Добавьте любые из следующих значений:

```yaml
panel_superadmin_username: "admin"
panel_superadmin_password: "yourpassword"
remnawave_secret_header: "yoursecretheader"
grafana_admin_user: "admin"
grafana_admin_password: "yourpassword"
```

Требования к паролям — плейбук завершится с ошибкой, если они не соблюдены:
- `panel_superadmin_password`: минимум 24 символа, хотя бы одна заглавная, одна строчная буква и одна цифра
- `grafana_admin_password`: минимум 4 символа, только буквы и цифры — спецсимволы сломают Docker `.env`

Сохраните пароль от vault в файл для удобства:

```bash
echo "yourvaultpassword" > .vault_pass
chmod 600 .vault_pass
```

---

## 6. Запуск плейбуков

**Развернуть всё сразу:**

```bash
ansible-playbook playbook.yml
```

С vault:

```bash
ansible-playbook playbook.yml --vault-password-file .vault_pass --extra-vars "@vault.yml"
```

**Или по отдельности:**

```bash
ansible-playbook playbook-panel.yml
ansible-playbook playbook-nodes.yml
ansible-playbook playbook-monitoring.yml
```

В конце каждого запуска учётные данные выводятся в терминал и сохраняются на сервере:
- Администратор панели: `/opt/remnawave/admin_credentials.txt`
- Секретный заголовок: `/opt/remnawave/secret_header.txt`
- Grafana: `/opt/monitoring/grafana_credentials.txt`

Все плейбуки идемпотентны и безопасны для повторного запуска. Существующие учётные данные и конфигурация сохраняются.

---

## 7. Доступ к Grafana

Grafana работает на `127.0.0.1:3000` на сервере мониторинга. Для доступа используйте SSH-туннель:

```bash
ssh -L 3000:127.0.0.1:3000 root@<ip-сервера-мониторинга>
```

Затем откройте `http://localhost:3000` в браузере и войдите с учётными данными Grafana.

Для доступа к другим сервисам на том же сервере пробросьте их порты аналогично:

| Сервис     | Порт |
|------------|------|
| Prometheus | 9090 |
| Grafana    | 3000 |
| Whitebox   | 9116 |
| Blackbox   | 9115 |

---

## Устранение неполадок

### SSH: отказано в соединении

- Убедитесь, что сервер доступен и SSH запущен
- Проверьте, что `ansible_port` в `hosts.yml` совпадает с SSH-портом сервера
- При использовании ключа убедитесь, что путь в `ssh_key` верный и ключ загружен в `ssh-agent`, если он защищён паролем

### TLS-сертификат не выдаётся

- Убедитесь, что A-запись домена указывает на правильный IP и успела распространиться
- Порт 80 должен быть открыт на сервере — acme.sh использует его для HTTP-challenge
- Проверьте логи acme.sh на сервере: `~/.acme.sh/acme.sh.log`

### Ошибка аутентификации API

- Панель должна быть полностью запущена до развёртывания нод — если разворачиваете по отдельности, сначала запустите `playbook-panel.yml`
- При использовании vault убедитесь, что `panel_superadmin_username` и `panel_superadmin_password` совпадают с тем, что было зарегистрировано на панели
- Проверьте, что домен панели корректно резолвится с сервера ноды

### Ошибка Docker-сети на панели

Если при повторном запуске появляется `network remnawave-network has active endpoints`, nginx всё ещё подключён к сети. Остановите его вручную на сервере панели:

```bash
cd /opt/remnawave/nginx && docker compose down
cd /opt/remnawave && docker compose down && docker compose up -d
cd /opt/remnawave/nginx && docker compose up -d
```

### Whitebox не имеет целей для проверки

Цели Whitebox берутся из ключей подключения `WHITEBOX_USER`, полученных во время запуска `playbook-monitoring.yml`. Если у пользователя нет активных ключей, возможные причины:
- На панели не настроены отряды — создайте хотя бы один отряд в интерфейсе панели, затем повторно запустите `playbook-monitoring.yml`
- Пользователь создан, но не добавлен ни в один отряд — то же решение

### Grafana не принимает пароль

Если Grafana отклоняет пароль при первом входе, убедитесь, что `grafana_admin_password` в vault содержит только буквы и цифры. Спецсимволы могут быть искажены при интерполяции Docker `.env`.

### Grafana не подключается к Prometheus

- Убедитесь, что оба контейнера находятся в одной Docker-сети (`monitoring-network`)
- Проверьте статус контейнеров: `docker ps` на сервере мониторинга
- Prometheus доступен внутри сети как `http://prometheus:9090` — убедитесь, что имя контейнера совпадает
