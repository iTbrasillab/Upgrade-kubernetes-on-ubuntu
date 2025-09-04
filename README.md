# Kubernetes Node Upgrade (Ansible)

Upgrade Kubernetes **worker nodes** and **secondary control-plane nodes** (kubeadm-based) on Ubuntu/Debian hosts — safely, one node at a time, over SSH.

> ✅ This role runs `kubeadm upgrade node`, installs the exact `kubeadm / kubelet / kubectl` version you choose, switches the APT repo to `pkgs.k8s.io`, restarts `kubelet`, and re-applies package holds.  
> ❌ It does **not** drain/cordon. Do those manually for maximum control.

---

## Table of Contents

- [Features](#features)  
- [Requirements](#requirements)  
- [Project Layout](#project-layout)  
- [Inventory](#inventory)  
- [Variables](#variables)  
- [Quick Start](#quick-start)  
- [One-Node-at-a-Time](#one-node-at-a-time)  
- [Primary Control-Plane](#primary-control-plane)  
- [Tips & Troubleshooting](#tips--troubleshooting)  
- [Security Notes](#security-notes)  
- [License](#license)

---

## Features

- Uses the **new Kubernetes APT repo** (`pkgs.k8s.io`) for a specific minor (`v1.31`, `v1.32`, …)
- Installs **exact** package versions (e.g. `1.31.0-1.1`) for repeatable upgrades
- Runs `kubeadm upgrade node` on the host
- Restarts `kubelet` and **holds** packages (`apt-mark hold`)
- **Idempotent**: safe to re-run on the same node
- **No drain/uncordon** inside the play — you keep manual control

---

## Requirements

- Target nodes: Ubuntu/Debian with `apt`, `sudo`, and `systemd`
- Cluster bootstrapped with **kubeadm**
- SSH access per node (password or key)
- A Kubernetes version target that exists in the selected minor repo  
  (use `apt-cache madison kubeadm` on a node to see exact versions)

---

## Project Layout

ansible-k8s-upgrade/
├── ansible.cfg
├── inventory.ini
├── group_vars/
│ └── k8s_nodes.yml
├── playbooks/
│ └── upgrade-nodes.yml
└── roles/
└── k8s_node_upgrade/
├── defaults/
│ └── main.yml
├── handlers/
│ └── main.yml
├── tasks/
│ ├── main.yml
│ ├── repos.yml
│ ├── kubeadm.yml
│ ├── kubeadm_upgrade_node.yml
│ ├── kubelet_kubectl.yml
│ └── hold.yml
└── templates/
└── kubernetes-stable.list.j2



> Make sure `roles_path = ./roles` is set in `ansible.cfg`, or run from the repo root.

---

## Inventory

`inventory.ini`
```ini
[k8s_nodes]
cp-2 ansible_host=10.0.0.12 ansible_user=kubemaster
worker-1 ansible_host=10.0.0.21 ansible_user=kubemaster
worker-2 ansible_host=10.0.0.22 ansible_user=kubemaster
```

Use --ask-pass --ask-become-pass at runtime or store credentials via Ansible Vault.

You keep full control of which node to upgrade using --limit.

Variables
group_vars/k8s_nodes.yml


# Minor line used to select the APT repo on pkgs.k8s.io
k8s_minor: "v1.31"              # e.g., v1.31, v1.32

# Exact package version as shown by: apt-cache madison kubeadm
k8s_pkg_version: "1.31.0-1.1"    # match kubeadm/kubelet/kubectl

# Paths for repo/keyring (rarely changed)
k8s_repo_list: "/etc/apt/sources.list.d/kubernetes-stable-{{ k8s_minor }}.list"
k8s_keyring: "/etc/apt/keyrings/kubernetes-apt-keyring.gpg"
old_repo_list: "/etc/apt/sources.list.d/kubernetes.list"
Why two knobs?

k8s_minor selects the repo (.../stable:/v1.31/...).

k8s_pkg_version pins the exact packages (repeatable + safe).

Check available versions:

```bash
apt update
apt-cache madison kubeadm | head
Quick Start
From the repo root:
```

# Upgrade a single node (example: worker-1) to Kubernetes 1.31.0
```bash 
ansible-playbook playbooks/upgrade-nodes.yml \
  --limit worker-1 \
  -u kubemaster \
  --ask-pass --ask-become-pass \
  -e 'k8s_minor=v1.31 k8s_pkg_version=1.31.0-1.1'
```
What it does on that node:

Switches APT to pkgs.k8s.io for v1.31

Installs kubeadm=1.31.0-1.1

Runs kubeadm upgrade node

Installs kubelet and kubectl to the same version

Restarts kubelet

Reapplies package holds

One-Node-at-a-Time
This repo is designed for strict control:

The playbook uses serial: 1 by default.

You explicitly select the target with --limit <host>.

Examples:


# upgrade a single worker
```bash
ansible-playbook playbooks/upgrade-nodes.yml --limit worker-2 -u kubemaster --ask-pass --ask-become-pass \
  -e 'k8s_minor=v1.31 k8s_pkg_version=1.31.0-1.1'
```
# upgrade a secondary control-plane
```bash
ansible-playbook playbooks/upgrade-nodes.yml --limit cp-2 -u kubemaster --ask-pass --ask-become-pass \
  -e 'k8s_minor=v1.31 k8s_pkg_version=1.31.0-1.1'
```

If you prefer, drain/uncordon manually before/after each run.


Tips & Troubleshooting
Role not found: ensure ansible.cfg has roles_path=./roles and you run from the repo root.
Check: ansible-config dump | grep DEFAULT_ROLES_PATH

Package versions not found: confirm k8s_minor matches the repo (e.g., v1.31) and use a version that apt-cache madison kubeadm actually lists (e.g., 1.31.0-1.1).

Cilium/Hubble: after a minor bump, verify Cilium and Hubble health. If Hubble Relay TLS fails, re-issue certs for the current cluster.name.

Security Notes
Prefer --ask-pass --ask-become-pass or SSH keys.

If you need to store credentials, use Ansible Vault (don’t commit plaintext passwords).

Package holds are managed with dpkg_selections for idempotency.

icense

MIT — do your thing. PRs welcome!

## Author
<a href="https://github.com/alepertile28">
  <img src="https://github.com/alepertile28.png" width="48" height="48" style="border-radius:50%" alt="n2" />
</a>

::contentReference[oaicite:0]{index=0}