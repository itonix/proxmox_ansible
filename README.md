# Proxmox Infrastructure Automation with Ansible

This repository contains a **multi-role Ansible project** for provisioning, configuring, and maintaining virtual machines running on **Proxmox VE**.

lab architecture:

┌─────────────────────────────────────────────┐
│               Windows Laptop                │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │          VMware Workstation            │  │
│  │                                       │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │     Ansible Control Node         │  │  │
│  │  │     (Linux VM)                   │  │  │
│  │  │                                 │  │  │
│  │  │  ansible-playbook                │  │  │
│  │  │  SSH / HTTPS (8006)              │  │  │
│  │  └───────────────┬─────────────────┘  │  │
│  │                  │                    │  │
│  │                  ▼                    │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │         Proxmox VE               │  │  │
│  │  │         (Virtualized)            │  │  │
│  │  │                                 │  │  │
│  │  │  Proxmox API                     │  │  │
│  │  │  VM Scheduler                   │  │  │
│  │  └───────────────┬─────────────────┘  │  │
│  │                  │                    │  │
│  │        ┌─────────┴─────────┐          │  │
│  │        │                   │          │  │
│  │        ▼                   ▼          │  │
│  │  ┌─────────────┐   ┌─────────────┐    │  │
│  │  │ Ubuntu VM   │   │ Arch VM     │    │  │
│  │  │ Web Server  │   │ App / Test  │    │  │
│  │  └─────────────┘   └─────────────┘    │  │
│  │                                       │  │
│  └───────────────────────────────────────┘  │
│                                             │
└─────────────────────────────────────────────┘




It is structured the way real infrastructure code is structured: roles are small and focused, inventories are explicit, and playbooks describe intent rather than implementation.

---

## Repository Structure

```
ansible/
├── ansible.cfg
├── inventories_proxmox/
│   ├── hosts
│   ├── proxmox.yml
│   └── group_vars/
│       ├── _ubuntu.yml
│       ├── _arch.yaml
│       └── _webservers.yml
│
├── playbooks/
│   ├── provision_vm_playbook.yaml
│   ├── configuration_vm_playbook.yaml
│   └── patch_webservers_playbook.yaml
│
├── proxmox_vm/          # VM lifecycle (Proxmox API)
│   ├── tasks/
│   ├── defaults/
│   ├── vars/
│   └── README.md
│
├── configure_vm/        # Guest OS configuration
│   ├── tasks/
│   ├── defaults/
│   ├── vars/
│   └── README.md
│
├── patch_webservers/    # OS patching & maintenance
│   ├── tasks/
│   ├── defaults/
│   ├── vars/
│   └── README.md
│
└── vars/
    ├── all.yaml
    └── variables.yaml
```

---

## Roles Overview

### `proxmox_vm`

Handles **VM lifecycle operations** using the Proxmox API.

Responsibilities:

* Authenticate using Proxmox API tokens
* List QEMU virtual machines
* Clone VMs from templates
* Enforce idempotent VM creation

This role never logs into the guest OS. It talks only to Proxmox.

---

### `configure_vm`

Handles **post-provision configuration inside the VM**.

Responsibilities:

* Base OS configuration
* Package installation
* User and SSH setup
* Environment hardening

This role assumes the VM already exists and is reachable via SSH.

---

### `patch_webservers`

Handles **ongoing maintenance and patching** for web servers.

Responsibilities:

* OS updates
* Service restarts via handlers
* Controlled, repeatable patch runs

This role is safe to run repeatedly in production.

---

## Inventories and Group Variables

Inventory is split by responsibility:

* `inventories_proxmox/hosts` — host and group definitions
* `group_vars/_ubuntu.yml` — Ubuntu-specific settings
* `group_vars/_arch.yaml` — Arch Linux–specific settings
* `group_vars/_webservers.yml` — role-based configuration

This keeps OS logic, role logic, and environment logic cleanly separated.

---

## Authentication with Proxmox

The project uses **API token authentication** (no passwords).

Example variables:

```yaml
apihost: "192.168.88.133"
api_token_id: "root@pam!TOKEN_ID"
api_token_secret: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

Notes:

* The token ID must include `user@realm!tokenid`
* `api_token_id` and `api_token_secret` must always be provided together
* Secrets should be stored using **Ansible Vault**  - I skipped this part

---


## python environment setting : 

root@ansible:~/ansible# python3 -m venv ansible_venv
root@ansible:~/ansible# source ansible_venv/bin/activate
(ansible_venv) root@ansible:~/ansible#  pip install ansible proxmoxer requests


---

## Playbooks

### Provision a VM

```bash
provision vms:  ansible-playbook -i inventories_proxmox/hosts playbooks/provision_vm_playbook.yaml
```

Creates a VM from a Proxmox template.

---

### Configure a VM

```bash
 ansible-playbook -i inventories_proxmox/proxmox.yml playbooks/configuration_vm_playbook.yaml --limit _webservers               -------------- target all servers under group _webservers

 ansible-playbook -i inventories_proxmox/proxmox.yml playbooks/configuration_vm_playbook.yaml --limit '_webservers:&_dev'       -------------- target all servers under group _webservers AND _dev group (intersection)

 ansible-playbook -i inventories_proxmox/proxmox.yml playbooks/configuration_vm_playbook.yaml --limit _webservers:&_ubuntu              -------------- target all servers under group _webservers and _ubuntu group

```

Applies OS-level configuration inside the VM.

---

### Patch Web Servers

```bash
ansible-playbook -i inventories_proxmox/proxmox.yml playbooks/patch_webservers_playbook.yaml --limit '_webservers:&_arch'
ansible-playbook -i inventories_proxmox/proxmox.yml playbooks/patch_webservers_playbook.yaml --limit '_webservers:&_ubuntu'
 
```

Safely applies updates and restarts services when required.

---

## Design Principles

* **Separation of concerns** — Proxmox ≠ Guest OS ≠ App maintenance
* **Idempotency** — repeatable without surprises
* **Readable over clever** — infrastructure is a liability, not a puzzle

This repository is meant to be understood at 2 a.m. during an incident.

---

## Security Notes

* Never commit API tokens
* You may use Ansible Vault for secrets
* Use least-privilege roles in Proxmox

---
Proxmos   gui:
<img width="1915" height="882" alt="image" src="https://github.com/user-attachments/assets/76d28f94-a4fb-460c-84ff-8ba9fd40e64c" />

<img width="1905" height="885" alt="image" src="https://github.com/user-attachments/assets/8d34badb-10d2-4767-a0e4-6542493beee1" />

Dynamic inventory:

<img width="1083" height="763" alt="image" src="https://github.com/user-attachments/assets/4aaa3d0e-5558-4d51-a01b-94dae0e66b11" />


## License

MIT License
