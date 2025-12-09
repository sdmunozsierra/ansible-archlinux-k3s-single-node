# K3s Single-Node Deployment with Ansible

This repository contains an Ansible playbook and roles to deploy a **single-node [K3s](https://k3s.io/)** cluster on an Arch-based host, with automatic **sqlite database backup** functionality.

K3s is a lightweight Kubernetes distribution designed for edge computing, IoT, CI/CD, and development environments. This playbook simplifies the deployment of a single-node K3s cluster, making it ideal for development, testing, or small production workloads.

---

## Table of Contents

- [Features](#-features)
- [Repository Structure](#-repository-structure)
- [Prerequisites](#-prerequisites)
- [Installation](#-installation)
- [Configuration](#-configuration)
- [Usage](#-usage)
- [Verification](#-verification)
- [Backup and Restore](#-backup-and-restore)
- [Advanced Usage](#-advanced-usage)
- [Troubleshooting](#-troubleshooting)
- [Customization](#-customization)
- [Teardown](#-teardown)

---

## 🔧 Features

- **Single-node K3s deployment** - Deploys a complete Kubernetes cluster on a single host
- **Arch Linux support** - Optimized for Arch-based distributions
- **AUR package installation** - Installs `k3s-bin` from the Arch User Repository via `yay`
- **Automatic dependency management** - Ensures required packages (`iptables`) are installed
- **SQLite database backup** - Automated backup of the K3s state database
- **Idempotent operations** - Safe to run multiple times without side effects
- **Systemd integration** - Properly enables and starts the K3s service

---

## 📂 Repository Structure

```
.
├── ansible.cfg              # Ansible configuration
├── group_vars
│   └── all.yml              # Global variables and settings
├── inventory
│   └── hosts.yml            # Target host definitions
├── playbook.yml             # Main deployment playbook
└── roles
    ├── k3s-backup           # Backup role for sqlite database
    │   └── tasks
    │       └── main.yml
    └── k3s-single           # Single-node K3s installation role
        └── tasks
            └── main.yml
```

### File Descriptions

- **`ansible.cfg`** - Configures inventory path, disables host key checking, and sets role paths
- **`inventory/hosts.yml`** - Defines target hosts with connection details (IP, user, sudo)
- **`group_vars/all.yml`** - Centralized configuration variables (backup paths, service names, packages)
- **`playbook.yml`** - Main entry point that orchestrates K3s deployment and backup
- **`roles/k3s-single/`** - Handles K3s installation, dependency management, and service activation
- **`roles/k3s-backup/`** - Manages backup of the K3s sqlite database to a configurable location

---

## ✅ Prerequisites

### Control Node (Where you run Ansible)

- **Ansible** 2.9 or later installed
  ```bash
  # On Arch Linux
  sudo pacman -S ansible
  
  # On Ubuntu/Debian
  sudo apt install ansible
  
  # On macOS
  brew install ansible
  
  # Or via pip
  pip install ansible
  ```
- **SSH client** for connecting to remote hosts
- **Python 3** (usually comes with Ansible)
- **SSH key** configured for passwordless access to target host (recommended)

### Target Node (The K3s host)

- **Arch Linux** or Arch-based distribution (Manjaro, EndeavourOS, etc.)
- **`yay`** AUR helper installed
  ```bash
  # Install yay if not present
  cd /tmp
  git clone https://aur.archlinux.org/yay.git
  cd yay
  makepkg -si
  ```
- **Sudo access** for the Ansible user (or root access)
- **Network connectivity** for downloading packages and container images
- **At least 512MB RAM** (1GB+ recommended)
- **Disk space** - Minimum 1GB free (more for workloads)

### Optional Prerequisites

- **Python virtual environment** for isolating Ansible dependencies
  ```bash
  python3 -m venv venv
  source venv/bin/activate
  pip install ansible
  ```

---

## 📥 Installation

1. **Clone or download this repository:**
   ```bash
   git clone <repository-url> k3s-single-node-ansible
   cd k3s-single-node-ansible
   ```

2. **Verify Ansible is installed:**
   ```bash
   ansible --version
   ```

3. **Test connectivity to your target host:**
   ```bash
   ansible all -i inventory/hosts.yml -m ping
   ```

---

## ⚙️ Configuration

### Step 1: Configure Inventory

Edit **`inventory/hosts.yml`** to match your target host:

#### Remote Host Configuration

```yaml
---
all:
  children:
    k3s:
      hosts:
        k3s-node:
          ansible_host: 192.0.2.10   # Your node's IP address or FQDN
          ansible_user: arch         # SSH username
          ansible_become: true       # Enable sudo/privilege escalation
          # Optional: specify SSH key
          # ansible_ssh_private_key_file: ~/.ssh/id_rsa
          # Optional: SSH port if non-standard
          # ansible_port: 2222
```

#### Localhost Configuration

If deploying to the same machine where you run Ansible:

```yaml
---
all:
  children:
    k3s:
      hosts:
        localhost:
          ansible_connection: local
          ansible_become: true
```

#### Multiple Hosts (Future Expansion)

You can add multiple hosts if needed:

```yaml
---
all:
  children:
    k3s:
      hosts:
        k3s-node-1:
          ansible_host: 192.0.2.10
          ansible_user: arch
          ansible_become: true
        k3s-node-2:
          ansible_host: 192.0.2.11
          ansible_user: arch
          ansible_become: true
```

### Step 2: Configure Variables

Edit **`group_vars/all.yml`** to customize deployment:

```yaml
---
# Backup Configuration
backup_target: /home/arch/k3s-backups  # Where to store backups
use_etcd: false                         # Set true only if using external etcd

# K3s Service Configuration
k3s_service_name: k3s                   # Systemd service name

# Package Dependencies
k3s_packages:
  - iptables                             # Required for K3s networking
  # Add other packages as needed:
  # - curl
  # - wget
```

#### Variable Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `backup_target` | `/home/arch/k3s-backups` | Base directory for storing backups |
| `use_etcd` | `false` | Enable/disable sqlite backup (set `true` if using external etcd) |
| `k3s_service_name` | `k3s` | Systemd service name for K3s |
| `k3s_packages` | `['iptables']` | List of packages to install before K3s |

### Step 3: SSH Key Setup (Recommended)

For passwordless SSH access:

```bash
# Generate SSH key if you don't have one
ssh-keygen -t ed25519 -C "ansible@control-node"

# Copy key to target host
ssh-copy-id arch@192.0.2.10

# Test connection
ssh arch@192.0.2.10
```

---

## 🚀 Usage

### Basic Deployment

Run the playbook from the repository root:

```bash
ansible-playbook playbook.yml
```

This will:
1. Install required dependencies (`iptables`)
2. Install `k3s-bin` from AUR via `yay`
3. Enable and start the K3s systemd service
4. Create a backup of the sqlite database (if `use_etcd: false`)

### Using Custom Ansible Config

If you have a global Ansible configuration that might conflict:

```bash
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbook.yml
```

### Verbose Output

For detailed output during execution:

```bash
# Verbose mode (-v, -vv, -vvv for increasing verbosity)
ansible-playbook playbook.yml -v
ansible-playbook playbook.yml -vvv  # Maximum verbosity
```

### Running Specific Roles

You can run individual roles using tags (if added) or by creating separate playbooks:

```bash
# Example: Only run the k3s-single role (requires modifying playbook.yml with tags)
ansible-playbook playbook.yml --tags k3s-single
```

### Dry Run (Check Mode)

Test what would change without making actual modifications:

```bash
ansible-playbook playbook.yml --check
```

### Limiting to Specific Hosts

If your inventory has multiple hosts:

```bash
ansible-playbook playbook.yml --limit k3s-node
```

---

## ✅ Verification

### Check K3s Service Status

SSH into your target node and verify:

```bash
# Check service status
sudo systemctl status k3s

# Check if K3s is running
sudo systemctl is-active k3s

# View service logs
sudo journalctl -u k3s -f
```

### Verify K3s Installation

```bash
# Check K3s version
sudo k3s --version

# Check Kubernetes version
sudo k3s kubectl version

# Get cluster info
sudo k3s kubectl cluster-info

# List nodes
sudo k3s kubectl get nodes

# List all pods
sudo k3s kubectl get pods --all-namespaces
```

### Test Kubernetes Functionality

```bash
# Deploy a test pod
sudo k3s kubectl run test-nginx --image=nginx --port=80

# Check pod status
sudo k3s kubectl get pods

# Get pod details
sudo k3s kubectl describe pod test-nginx

# Delete test pod
sudo k3s kubectl delete pod test-nginx
```

### Access Kubeconfig Locally

To use `kubectl` from your local machine:

```bash
# On the target node, copy kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml

# On your local machine, save it
mkdir -p ~/.kube
# Paste the content into ~/.kube/config
# Update the server URL from 127.0.0.1 to your node's IP

# Test from local machine
kubectl get nodes
```

---

## 💾 Backup and Restore

### Automatic Backup

The playbook automatically backs up the K3s sqlite database to `{{ backup_target }}/sqlite/state.db` when `use_etcd: false`.

### Manual Backup

You can manually trigger a backup by running the backup role:

```bash
# Create a backup-only playbook or run the role directly
ansible-playbook playbook.yml --tags backup
```

Or manually on the node:

```bash
# Create backup directory
mkdir -p /home/arch/k3s-backups/sqlite

# Copy database
sudo cp /var/lib/rancher/k3s/server/db/state.db \
       /home/arch/k3s-backups/sqlite/state.db.$(date +%Y%m%d_%H%M%S)
```

### Scheduled Backups

Create a systemd timer for automated backups:

**`/etc/systemd/system/k3s-backup.service`**:
```ini
[Unit]
Description=Backup K3s SQLite Database
After=k3s.service

[Service]
Type=oneshot
ExecStart=/usr/bin/cp /var/lib/rancher/k3s/server/db/state.db /home/arch/k3s-backups/sqlite/state.db.$(date +%%Y%%m%%d_%%H%%M%%S)
```

**`/etc/systemd/system/k3s-backup.timer`**:
```ini
[Unit]
Description=Daily K3s Backup Timer

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

Enable the timer:
```bash
sudo systemctl enable --now k3s-backup.timer
```

### Restore from Backup

⚠️ **Warning**: Restoring will replace the current cluster state.

```bash
# Stop K3s
sudo systemctl stop k3s

# Backup current state (just in case)
sudo cp /var/lib/rancher/k3s/server/db/state.db \
       /var/lib/rancher/k3s/server/db/state.db.backup

# Restore from backup
sudo cp /home/arch/k3s-backups/sqlite/state.db \
       /var/lib/rancher/k3s/server/db/state.db

# Restore ownership
sudo chown root:root /var/lib/rancher/k3s/server/db/state.db

# Start K3s
sudo systemctl start k3s

# Verify
sudo k3s kubectl get nodes
```

---

## 🔧 Advanced Usage

### Custom K3s Configuration

Create a K3s config file on the target node:

**`/etc/rancher/k3s/config.yaml`**:
```yaml
# Disable Traefik ingress controller
disable:
  - traefik

# Set node IP
node-ip: 192.0.2.10

# Custom cluster DNS
cluster-dns: 10.43.0.10

# Additional arguments
kube-apiserver-arg:
  - "feature-gates=EphemeralContainers=true"
```

You can automate this with Ansible by adding a task:

```yaml
- name: Create K3s config directory
  file:
    path: /etc/rancher/k3s
    state: directory
    mode: '0755'

- name: Configure K3s
  copy:
    content: |
      disable:
        - traefik
    dest: /etc/rancher/k3s/config.yaml
    mode: '0644'
  notify: restart k3s
```

### Using AUR Collection Instead of Shell

If you prefer using the `kewlfft.aur.aur` collection:

1. Install the collection:
   ```bash
   ansible-galaxy collection install kewlfft.aur
   ```

2. Update `roles/k3s-single/tasks/main.yml`:
   ```yaml
   - name: Install k3s-bin via kewlfft.aur.aur
     kewlfft.aur.aur:
       name: k3s-bin
       state: present
       use: yay
     become: yes
     become_user: aur_builder  # Or your AUR builder user
   ```

### Adding Tags for Selective Execution

Modify `playbook.yml` to add tags:

```yaml
---
- name: Deploy single-node K3s
  hosts: k3s
  become: true
  gather_facts: true

  roles:
    - role: k3s-single
      tags: ['deploy', 'install']
    - role: k3s-backup
      when: not use_etcd
      tags: ['backup']
```

Then run specific parts:
```bash
ansible-playbook playbook.yml --tags deploy
ansible-playbook playbook.yml --tags backup
```

### Environment-Specific Configurations

Create environment-specific variable files:

**`group_vars/production.yml`**:
```yaml
backup_target: /mnt/backups/k3s
k3s_packages:
  - iptables
  - curl
```

**`group_vars/development.yml`**:
```yaml
backup_target: /tmp/k3s-backups
k3s_packages:
  - iptables
```

Use with inventory groups or `-e @group_vars/production.yml`.

---

## 🐛 Troubleshooting

### Common Issues

#### Issue: Ansible can't connect to host

**Symptoms**: `UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host..."}`

**Solutions**:
- Verify SSH connectivity: `ssh user@host`
- Check firewall rules on target host
- Verify `ansible_host` and `ansible_user` in inventory
- Ensure SSH key is properly configured

#### Issue: Permission denied (sudo)

**Symptoms**: `Permission denied` or `sudo: a password is required`

**Solutions**:
- Ensure `ansible_become: true` in inventory
- Configure passwordless sudo for the Ansible user:
  ```bash
  echo "arch ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/arch
  ```
- Or use `--ask-become-pass`:
  ```bash
  ansible-playbook playbook.yml --ask-become-pass
  ```

#### Issue: yay not found

**Symptoms**: `yay: command not found`

**Solutions**:
- Install yay (see Prerequisites section)
- Or switch to `kewlfft.aur.aur` collection method

#### Issue: K3s service fails to start

**Symptoms**: `systemctl status k3s` shows failed state

**Solutions**:
- Check logs: `sudo journalctl -u k3s -n 50`
- Verify iptables is installed: `pacman -Q iptables`
- Check disk space: `df -h`
- Verify network connectivity
- Check for port conflicts: `sudo netstat -tulpn | grep 6443`

#### Issue: Backup role fails

**Symptoms**: `SQLite DB not found at /var/lib/rancher/k3s/server/db/state.db`

**Solutions**:
- Ensure K3s has started successfully first
- Wait a few seconds after K3s starts before running backup
- Check if using etcd (set `use_etcd: true` to skip sqlite backup)
- Verify K3s data directory exists: `sudo ls -la /var/lib/rancher/k3s/server/db/`

#### Issue: Package installation fails

**Symptoms**: `pacman` or `yay` errors during package installation

**Solutions**:
- Update package database: `sudo pacman -Sy`
- Check AUR package availability: `yay -Ss k3s-bin`
- Verify internet connectivity on target host
- Check disk space: `df -h`

### Debugging Tips

1. **Enable verbose output**:
   ```bash
   ansible-playbook playbook.yml -vvv
   ```

2. **Test individual tasks**:
   ```bash
   ansible k3s -i inventory/hosts.yml -m shell -a "systemctl status k3s"
   ```

3. **Check facts**:
   ```bash
   ansible k3s -i inventory/hosts.yml -m setup
   ```

4. **Validate playbook syntax**:
   ```bash
   ansible-playbook playbook.yml --syntax-check
   ```

---

## 🎨 Customization

### Disable Default Addons

Create `/etc/rancher/k3s/config.yaml` on the target node:

```yaml
disable:
  - traefik
  - servicelb
  - local-storage
  - metrics-server
```

### Custom Node Labels and Taints

Add to K3s config:

```yaml
node-label:
  - "environment=production"
  - "node-type=compute"
node-taint:
  - "dedicated=compute:NoSchedule"
```

### Change Default Data Directory

```yaml
data-dir: /opt/k3s-data
```

### Add Custom Registries

```yaml
private-registry: /etc/rancher/k3s/registries.yaml
```

### Network Configuration

```yaml
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
cluster-dns: "10.43.0.10"
```

### Extend the Playbook

Add custom tasks by creating additional roles or modifying existing ones:

**`roles/k3s-single/tasks/custom.yml`**:
```yaml
---
- name: Pull custom images
  shell: k3s ctr images pull docker.io/library/nginx:latest
```

Include in `roles/k3s-single/tasks/main.yml`:
```yaml
- include_tasks: custom.yml
```

---

## 🧹 Teardown

### Manual Teardown

To completely remove K3s from a node:

```bash
# Stop and disable service
sudo systemctl disable --now k3s

# Remove K3s binary
sudo yay -Rns k3s-bin

# Remove K3s data (⚠️ DESTRUCTIVE - deletes all cluster state)
sudo rm -rf /var/lib/rancher/k3s
sudo rm -rf /etc/rancher/k3s

# Remove iptables rules (optional, may affect other services)
sudo iptables -F
sudo iptables -X
```

### Automated Teardown Playbook

Create **`teardown.yml`**:

```yaml
---
- name: Remove K3s
  hosts: k3s
  become: true
  gather_facts: true

  tasks:
    - name: Stop and disable K3s service
      systemd:
        name: "{{ k3s_service_name }}"
        enabled: false
        state: stopped

    - name: Remove k3s-bin package
      shell: yay -Rns --noconfirm k3s-bin
      ignore_errors: true

    - name: Remove K3s data directories
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /var/lib/rancher/k3s
        - /etc/rancher/k3s
      when: teardown_remove_data | default(false)
```

Run with:
```bash
ansible-playbook teardown.yml -e teardown_remove_data=true
```

---

## 📚 Additional Resources

- [K3s Official Documentation](https://docs.k3s.io/)
- [K3s GitHub Repository](https://github.com/k3s-io/k3s)
- [Ansible Documentation](https://docs.ansible.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

## 📝 License

Add your preferred license here (MIT, Apache 2.0, etc.).

---

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

## ⚠️ Notes

- This playbook is designed for **single-node** deployments. For multi-node clusters, consider using the full K3s HA setup with external etcd.
- The backup role only works with **sqlite** backend. If you switch to etcd, you'll need a different backup strategy.
- K3s uses **iptables** for networking. Ensure iptables is available and not conflicting with other firewall solutions.
- The default K3s installation includes **Traefik** as the ingress controller. You can disable it via configuration if needed.
