# Host provisioning (Ansible)

Turns a bare Ubuntu Server install into a hardened host running k3s — the
layer below GitOps. Together with Flux this makes the whole homelab
reproducible in two commands:

```
1. ansible-playbook playbook.yaml -k     # bare OS → hardened host + k3s
2. flux bootstrap github ...             # empty cluster → full homelab from Git
```

## What it does

| Role | Purpose |
|---|---|
| `ssh` | Installs the workstation public key, then disables password auth and root login (drop-in in `sshd_config.d/`). Order matters: key first, so a fresh host can't lock us out. |
| `firewall` | UFW: allow OpenSSH (before enabling!), k3s API `6443/tcp`, pod network `10.42.0.0/16`, service network `10.43.0.0/16`. |
| `headless` | Laptop-as-server: logind drop-in so closing the lid doesn't suspend the machine. |
| `k3s` | Downloads and runs the official installer (idempotent — skipped if `/usr/local/bin/k3s` exists), waits for the node to be Ready. Version pin and extra TLS SANs via role vars. |

## Usage

```bash
# one-time: install required collections
ansible-galaxy collection install -r requirements.yaml

# edit inventory.yaml: server IP and user

# first run against a fresh install (password auth still on, needs sshpass)
ansible-playbook playbook.yaml -k --ask-become-pass

# after that keys are in place
ansible-playbook playbook.yaml --ask-become-pass
```

Not managed here: DHCP reservation (router-side), WiFi/netplan (contains the
network password — secrets don't belong in a public repo; see
`cheatsheets/Ubuntu Server Setup.md`).

Quirk: sudo-rs (Ubuntu's default since 25.10) breaks Ansible's privilege
escalation, so the inventory pins `ansible_become_exe` to classic sudo
(`/usr/bin/sudo.ws`); see
`journal/2026-07-14-sudo-rs-breaks-ansible-become.md`.
