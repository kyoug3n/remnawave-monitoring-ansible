# remnawave-monitoring-ansible

Ansible playbooks for deploying and managing a [Remnawave](https://github.com/remnawave/panel) VPN panel, nodes, and monitoring stack.

## What it deploys

**Panel** — Remnawave backend + Nginx reverse proxy with TLS, secret header protection, and auto-generated admin credentials.

**Nodes** — Remnanode instances with Nginx, TLS, and automatic registration to the panel via API.

**Monitoring** — Prometheus + Grafana + Whitebox + Blackbox on a separate server. Panel metrics are scraped over TLS, VPN connections are probed via Whitebox, and node HTTPS availability is checked via Blackbox. The hemera dashboard comes pre-loaded in Grafana.

## Requirements

- [Ansible 2.12+](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) running on Linux or WSL (Windows Subsystem for Linux)
- Debian-based target servers (Debian, Ubuntu, etc.)
- SSH access to all servers (key or password); if your key has a passphrase, add it to `ssh-agent` before running (`eval $(ssh-agent) && ssh-add /path/to/key`)
- `sshpass` if connecting with a password (`apt install sshpass`)
- Separate DNS A records for the panel and each node, each pointing to their respective server IP

## Setup

**1. Clone the repo**

```bash
git clone https://github.com/kyoug3n/remnawave-monitoring-ansible.git
cd remnawave-monitoring-ansible
```

**2. Configure inventory**

Edit `inventory/hosts.yml` — all configurable values are annotated inline.

**3. (Optional) Vault for sensitive values**

By default, credentials are auto-generated and saved to the server. If you'd rather supply your own, create a vault file:

```bash
EDITOR=nano ansible-vault create vault.yml
```

```yaml
panel_superadmin_username: "admin"
panel_superadmin_password: "yourpassword"
remnawave_secret_header: "yoursecretheader"
grafana_admin_user: "admin"
grafana_admin_password: "yourpassword"
```

Password requirements — the playbook will error if these aren't met:
- `panel_superadmin_password`: min 24 chars, at least one uppercase, one lowercase, one number
- `grafana_admin_password`: min 4 chars; alphanumeric only — special characters will break the Docker `.env` file

Pass it at runtime with `--extra-vars "@vault.yml" --ask-vault-pass`.

## Running

**Deploy everything**

```bash
ansible-playbook playbook.yml
```

**Deploy individually**

```bash
ansible-playbook playbook-panel.yml
ansible-playbook playbook-nodes.yml
ansible-playbook playbook-monitoring.yml
```

Credentials are printed at the end of each run and saved on the respective server:
- Panel admin: `/opt/remnawave/admin_credentials.txt`
- Secret header: `/opt/remnawave/secret_header.txt`
- Grafana: `/opt/monitoring/grafana_credentials.txt`

If using a vault file, append `--extra-vars "@vault.yml" --ask-vault-pass` (or `--vault-password-file .vault_pass`) to any command.

## Adding a node

1. Add a new entry under `nodes.hosts` in `inventory/hosts.yml`:

```yaml
node_ams:
  ansible_host: "192.168.0.2"
  node_name: "ams"
  node_port: 3333
  node_domain: "ams.example.com"
```

2. Point the domain at the server, then run:

```bash
ansible-playbook playbook-nodes.yml
```

If using a vault file, append `--extra-vars "@vault.yml" --ask-vault-pass` (or `--vault-password-file .vault_pass`) to any playbook command.

The node is automatically registered to the panel and added to all existing squads.

## Monitoring stack

| Service    | Default port | Description                        |
|------------|--------------|------------------------------------|
| Prometheus | 9090         | Metrics collection                 |
| Grafana    | 3000         | Dashboards (hemera pre-loaded)     |
| Whitebox   | 9116         | VPN connection probing             |
| Blackbox   | 9115         | Node HTTPS availability probing    |

All ports are bound to `127.0.0.1`. Use SSH tunneling or a reverse proxy to access them.

Whitebox probes are automatically configured using a dedicated `WHITEBOX_USER` created on the panel during the monitoring playbook run.

## Accessing Grafana

Since all monitoring ports are bound to `127.0.0.1`, use an SSH tunnel to reach them from your machine:

```bash
ssh -L 3000:127.0.0.1:3000 root@192.168.2.1
```

If using an SSH key:

```bash
ssh -L 3000:127.0.0.1:3000 -i /path/to/key root@192.168.2.1
```

Then open `http://localhost:3000`. Swap `3000` for `9090`, `9116`, or `9115` to reach the other services.

## Re-running playbooks

All playbooks are idempotent and safe to re-run. Existing credentials and configuration are preserved — only actual changes show up as `changed`.

## Vault password file

To avoid typing the vault password on every run, save it to a file and reference it directly:

```bash
echo "yourpassword" > .vault_pass
chmod 600 .vault_pass
ansible-playbook playbook.yml --vault-password-file .vault_pass --extra-vars "@vault.yml"
```

Make sure `.vault_pass` is in `.gitignore`.

## Security

- SSH hardened (configurable root login, password auth, max retries, fail2ban)
- UFW firewall on all servers
- ufw-docker on the panel to prevent Docker from bypassing UFW
- Panel hidden behind a secret header
- Metrics endpoint proxied through nginx with TLS, restricted to the monitoring server IP
- All generated secrets are alphanumeric (safe for Docker `.env` files)

## Acknowledgements

- [Remnawave](https://github.com/remnawave/backend) — Proxy management panel
- [Remnanode](https://github.com/remnawave/node) — Xray node
- [Prometheus](https://github.com/prometheus/prometheus) — metrics collection
- [Grafana](https://github.com/grafana/grafana) — dashboards
- [Whitebox](https://github.com/quyxishi/whitebox) — Proxy connection probing and hemera dashboard
- [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter) — HTTPS availability probing
- [ufw-docker](https://github.com/chaifeng/ufw-docker) — UFW + Docker integration
- [acme.sh](https://github.com/acmesh-official/acme.sh) — TLS certificate management
