# Ansible from zero to hero

A small demo repo showing what Ansible can do:

- an **inventory** with three hosts: `localhost` (local connection), `vm-35` (`10.150.47.35`, over SSH) and a placeholder `vm-random`
- **variables** at multiple levels (`group_vars`, `host_vars`, play `vars`, role `defaults`, `-e` on the CLI)
- **facts** and **`set_fact`** to process host information
- **Jinja2 templates** (`template`) to generate a text report and Kubernetes manifests
- **roles**, **tasks**, **handlers**, **tags**, **loops**, **register**
- creation of a **Pod on Kubernetes** (local cluster) with the `kubernetes.core` collection

## Layout

```
.
├── ansible.cfg                 # configuration (inventory, roles_path, ...)
├── site.yml                    # main playbook (2 plays)
├── requirements.txt            # Python dependencies (ansible-core, kubernetes)
├── requirements.yml            # Ansible collections (kubernetes.core)
├── inventory/
│   ├── hosts.yml               # flat list: localhost, vm-35, vm-random
│   ├── group_vars/all.yml
│   └── host_vars/{localhost,vm-35}.yml
└── roles/
    ├── common/                 # facts, set_fact, report template, handler
    └── k8s_pod/                # namespace + ConfigMap + Pod from templates
```

## What `site.yml` does

1. **Play 1** (`hosts: all`): gathers facts, reshapes them with `set_fact`
   (host summary, present/missing commands) and renders a report from a template into
   `report_dir` (`/tmp/ansible-from-zero-to-hero/` by default,
   `/home/lbassi/ansible-from-zero-to-hero/` on `vm-35`). It then reads the report back and prints it.
2. **Play 2** (`hosts: localhost`): checks `kubectl cluster-info` and uses the `k8s_pod` role to create,
   in the `ansible-demo` namespace, a ConfigMap (HTML rendered from a template) and an `nginx` Pod serving it.

## Prerequisites

- Python 3 and `kubectl` configured for your cluster (`kubectl get nodes` must work)
- For the VMs: SSH access as `lbassi` (`vm-35` is `10.150.47.35`; `vm-random` is a placeholder to replace)

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml   # installs into ./collections
```

### SSH access to the VMs

```bash
ssh-copy-id lbassi@10.150.47.35         # if your key is not authorized yet
ssh lbassi@10.150.47.35 true            # accept the host key the first time
```

If you need a specific key, add `ansible_ssh_private_key_file: ~/.ssh/<key>` to the host in
[inventory/hosts.yml](inventory/hosts.yml).

## Usage

```bash
# Inspect the inventory
ansible-inventory --graph
ansible-inventory --host vm-35           # resolved variables of the host
ansible all -m ping                      # connectivity test

# Run everything (vm-random is a placeholder: it will show as unreachable)
ansible-playbook site.yml

# Localhost only (pod + local report), without touching the VMs
ansible-playbook site.yml --limit localhost

# A single VM
ansible-playbook site.yml --limit vm-35

# Several hosts
ansible-playbook site.yml --limit localhost,vm-35

# Run only part of it, using tags
ansible-playbook site.yml --tags facts
ansible-playbook site.yml --tags report
ansible-playbook site.yml --tags k8s

# Dry run and diff of generated files
ansible-playbook site.yml --check --diff --limit vm-35

# Override variables from the CLI (highest precedence)
ansible-playbook site.yml --tags k8s -e k8s_pod_message="Hi!" -e k8s_pod_image=nginx:alpine

# Verbosity
ansible-playbook site.yml -v
```

If `sudo` is needed on the VM, add `--ask-become-pass` (the current demo does not need it).

### Verify the pod

```bash
kubectl -n ansible-demo get pod,cm
kubectl -n ansible-demo port-forward pod/hello-ansible 8080:80
curl http://localhost:8080
```

### Cleanup

```bash
ansible-playbook site.yml --limit localhost --tags k8s -e k8s_state=absent   # deletes the namespace
rm -rf /tmp/ansible-from-zero-to-hero                                        # local report
```

## Variable precedence (example in this repo)

`environment_name` is defined in several places: `demo` in `group_vars/all.yml`, but the host
`vm-35` overrides it with `vm-lab` in `host_vars/vm-35.yml`. Check it with:

```bash
ansible-inventory --host vm-35 | grep environment_name
```

## Notes

- The `k8s_pod` role defaults live in `roles/k8s_pod/defaults/main.yml`; inventory-specific
  values are in `inventory/host_vars/localhost.yml`.
- Runs are idempotent: re-running the playbook without changes reports every task as `ok`
  and the report handler does not fire.
- Optional lint: `pip install ansible-lint && ansible-lint site.yml roles`.
