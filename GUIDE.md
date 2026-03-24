# Deployment Guide

This guide walks through a full deployment from scratch — DNS setup, installing Ansible, configuring the inventory, running the playbooks, and accessing the monitoring stack.

## What you need

- 2 servers minimum (panel + node), 3 if deploying monitoring
- A domain with access to DNS settings
- A Linux machine or WSL to run Ansible from

---

## 1. DNS setup

Create an A record for each domain in your DNS provider's dashboard:

| Type | Name | Value |
|------|------|-------|
| A | `panel.example.com` | `192.168.1.1` |
| A | `node.example.com` | `192.168.0.1` |

Each domain must point to its own server — they cannot share an IP. DNS propagation can take a few minutes. You can verify with:

```bash
dig panel.example.com
```

The monitoring server does not need a domain.

---

## 2. Install Ansible

On Ubuntu/Debian (or WSL):

```bash
sudo apt update
sudo apt install ansible sshpass -y
```

`sshpass` is only needed if connecting with a password instead of an SSH key.

Verify:

```bash
ansible --version
```

You need Ansible 2.12 or newer.

---

## 3. Clone the repo

```bash
git clone https://github.com/kyoug3n/remnawave-monitoring-ansible.git
cd remnawave-monitoring-ansible
```

---

## 4. Configure inventory

Open `inventory/hosts.yml` and fill in your values. The file is annotated — everything marked with a comment is something you should set. Everything inside `# ↓ do not change ↓` blocks should be left as-is.

At minimum you need to set:
- `ansible_host` for each server
- `remnawave_panel_domain` under `panel.vars`
- `node_name`, `node_port`, `node_domain` for each node
- `ssh_key` or `ssh_password` under `all.vars`

If your SSH key has a passphrase, load it into the agent before running:

```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
```

---

## 5. (Optional) Create a vault file

By default, all credentials are auto-generated. If you want to supply your own, create a vault file:

```bash
EDITOR=nano ansible-vault create vault.yml
```

Add any of the following:

```yaml
panel_superadmin_username: "admin"
panel_superadmin_password: "yourpassword"
remnawave_secret_header: "yoursecretheader"
grafana_admin_user: "admin"
grafana_admin_password: "yourpassword"
```

Password requirements — the playbook will error if these aren't met:
- `panel_superadmin_password`: min 24 chars, at least one uppercase, one lowercase, one number
- `grafana_admin_password`: min 4 chars, alphanumeric only — special characters will break the Docker `.env` file

Save the vault password to a file for convenience:

```bash
echo "yourvaultpassword" > .vault_pass
chmod 600 .vault_pass
```

---

## 6. Run the playbooks

**Deploy everything at once:**

```bash
ansible-playbook playbook.yml
```

With vault:

```bash
ansible-playbook playbook.yml --vault-password-file .vault_pass --extra-vars "@vault.yml"
```

**Or deploy individually:**

```bash
ansible-playbook playbook-panel.yml
ansible-playbook playbook-nodes.yml
ansible-playbook playbook-monitoring.yml
```

At the end of each run, credentials are printed to the terminal and saved on the server:
- Panel admin: `/opt/remnawave/admin_credentials.txt`
- Secret header: `/opt/remnawave/secret_header.txt`
- Grafana: `/opt/monitoring/grafana_credentials.txt`

All playbooks are idempotent and safe to re-run. Existing credentials and configuration are preserved.

---

## 7. Access Grafana

Grafana runs on `127.0.0.1:3000` on the monitoring server. Use an SSH tunnel to reach it:

```bash
ssh -L 3000:127.0.0.1:3000 root@<monitoring-server-ip>
```

Then open `http://localhost:3000` in your browser and log in with the Grafana credentials.

To access other services on the same server, tunnel their ports the same way:

| Service    | Port |
|------------|------|
| Prometheus | 9090 |
| Grafana    | 3000 |
| Whitebox   | 9116 |
| Blackbox   | 9115 |

---

## Troubleshooting

### SSH connection refused

- Make sure the server is reachable and SSH is running
- Check that `ansible_port` in `hosts.yml` matches the server's SSH port
- If using a key, verify the path in `ssh_key` is correct and the key is loaded in `ssh-agent` if it has a passphrase

### TLS certificate fails to issue

- Make sure the domain's A record is pointing to the correct server IP and has propagated
- Port 80 must be open on the server — acme.sh uses it for the HTTP challenge
- Check acme.sh logs on the server: `~/.acme.sh/acme.sh.log`

### API authentication fails

- The panel must be fully up before nodes are deployed — run `playbook-panel.yml` first if deploying separately
- If using vault credentials, make sure `panel_superadmin_username` and `panel_superadmin_password` match what was registered on the panel
- Check that the panel domain resolves correctly from the node server

### Docker network error on panel

If you see `network remnawave-network has active endpoints` during a re-run, nginx is still attached to the network. Bring it down manually on the panel server:

```bash
cd /opt/remnawave/nginx && docker compose down
cd /opt/remnawave && docker compose down && docker compose up -d
cd /opt/remnawave/nginx && docker compose up -d
```

### Whitebox has no targets

Whitebox targets are populated from the `WHITEBOX_USER` connection keys fetched during `playbook-monitoring.yml`. If the user has no enabled keys, it means either:
- The panel has no squads configured — create at least one squad in the panel UI, then re-run `playbook-monitoring.yml`
- The user was created but not added to any squads — same fix

### Grafana password rejected

If Grafana rejects the password on first login, check that `grafana_admin_password` in your vault contains only alphanumeric characters. Special characters can get mangled by Docker's `.env` interpolation.

### Grafana can't connect to Prometheus

- Make sure both containers are on the same Docker network (`monitoring-network`)
- Check container status: `docker ps` on the monitoring server
- Prometheus is referenced as `http://prometheus:9090` inside the network — verify the container name matches
