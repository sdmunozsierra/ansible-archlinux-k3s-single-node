Here’s a `README.md` you can drop straight into the repo:

````md
# K3s Single-Node Deployment with Ansible

This repository contains an Ansible playbook and roles to deploy a **single-node [K3s](https://k3s.io/)** cluster on an Arch-based host, plus a simple **sqlite database backup** role.

It’s a stripped-down variant of a multi-node/cluster setup: no worker tokens, no extra coordination — just a single `k3s` server node using the default sqlite backend.

---

## 🔧 Features

- Deploys **K3s server** on a single node
- Installs `k3s-bin` from the AUR (via `yay` or `kewlfft.aur.aur`)
- Ensures basic dependencies (`iptables`) are present
- Provides a **backup role** for the K3s sqlite database:
  - Backs up `/var/lib/rancher/k3s/server/db/state.db`
  - Stores it under a configurable `backup_target` path

---

## 📂 Repository Layout

```text
.
├── ansible.cfg
├── group_vars
│   └── all.yml
├── inventory
│   └── hosts.yml
├── playbook.yml
└── roles
    ├── k3s-backup
    │   └── tasks
    │       └── main.yml
    └── k3s-single
        └── tasks
            └── main.yml
````

### Key Pieces

* **`ansible.cfg`** – Local Ansible configuration (inventory path, etc.).
* **`inventory/hosts.yml`** – Hosts and connection details for the k3s node.
* **`group_vars/all.yml`** – Global variables (backup path, etc.).
* **`playbook.yml`** – Entry point to deploy K3s and run backups.
* **`roles/k3s-single`** – Role to install and start the K3s server (single node).
* **`roles/k3s-backup`** – Role to back up the K3s sqlite database.

---

## ✅ Prerequisites

On the machine where you run Ansible (your control node):

* **Ansible** installed
* SSH access to the target host (unless using `localhost` mode)
* Optional but recommended: Python 3 and virtualenv for isolating Ansible

On the **target node** (the K3s node):

* Arch Linux (or Arch-based) system
* `yay` installed (if using the `shell: yay -S k3s-bin` approach)
* Ability to `sudo` (become root) from the Ansible user

---

## ⚙️ Configuration

### 1. Inventory

Edit **`inventory/hosts.yml`** and set your host details.

Example: remote Arch host

```yml
---
all:
  children:
    k3s:
      hosts:
        k3s-node:
          ansible_host: 192.0.2.10   # <-- change to your node IP or hostname
          ansible_user: arch         # <-- ssh user
          ansible_become: true
```

If you want to run everything directly on the local machine instead:

```yml
---
all:
  children:
    k3s:
      hosts:
        localhost:
          ansible_connection: local
          ansible_become: true
```

### 2. Global Variables

Edit **`group_vars/all.yml`** as needed:

```yml
---
backup_target: /home/arch/k3s-backups

# Single-node default: k3s uses sqlite
use_etcd: false  # set true only if you later switch to external etcd

k3s_service_name: k3s
k3s_packages:
  - iptables
```

* **`backup_target`** – Where K3s sqlite DB backups will be stored.
* **`use_etcd`** – Currently only used as a switch to gate the backup role; for a simple single-node setup, keep this `false`.

If you change the service name from the default `k3s`, update `k3s_service_name` accordingly.

---

## 🚀 Usage

From the root of the repo:

```bash
ansible-playbook playbook.yml
```

Ansible will:

1. Install dependencies (`iptables`, etc.).
2. Install `k3s-bin` from the AUR.
3. Enable and start the `k3s` systemd service.
4. If `use_etcd` is `false`, run the `k3s-backup` role to back up the sqlite DB.

> If you have a global Ansible config that might conflict, you can force this repo’s config:
>
> ```bash
> ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbook.yml
> ```

---

## 💾 Backup Details

The **`k3s-backup`** role:

* Verifies that `state.db` exists at:

  * `/var/lib/rancher/k3s/server/db/state.db`
* Creates a directory under:

  * `{{ backup_target }}/sqlite`
* Copies the DB file to:

  * `{{ backup_target }}/sqlite/state.db`

This role runs as part of `playbook.yml` when `use_etcd` is `false`, but you can also call it separately in your own playbooks if you want to schedule backups via cron/systemd timers around Ansible.

---

## 🔄 Customization Ideas

Things you might want to extend later:

* Disable or tweak default K3s addons (e.g., Traefik) via `/etc/rancher/k3s/config.yaml`.
* Pull `kubeconfig` from the node to your local machine automatically.
* Add tags (e.g., `deploy` / `backup`) to split deployment and backup runs.
* Add support for a different distro (swap out the `pacman` task).

---

## 🧹 Teardown (Manual)

This repo doesn’t provide a teardown playbook yet. On the node, you can manually:

* Stop and disable the service:

  ```bash
  sudo systemctl disable --now k3s
  ```
* Remove K3s data (⚠️ this deletes your cluster state):

  ```bash
  sudo rm -rf /var/lib/rancher/k3s /etc/rancher/k3s
  ```

Add an Ansible role for teardown if you want this automated.

---

## 📝 License

Add your preferred license here (MIT, Apache 2.0, etc.).

```

If you tell me the repo name and license you want to use, I can tweak the title and the last section to match exactly.
```

