# Ansible from zero to hero

A small demo repo showing what Ansible can do:

- an **inventory** with three hosts: `localhost` (local connection), `vm-35` (`10.150.47.35`) and `vm-34` (`10.150.47.34`), both over SSH
- **variables** at multiple levels (`group_vars`, `host_vars`, play `vars`, role `defaults`, `-e` on the CLI)
- **facts** and **`set_fact`** to process host information
- **Jinja2 templates** (`template`) to generate a text report and Kubernetes manifests
- **roles**, **tasks**, **handlers**, **tags**, **loops**, **register**, **`delegate_to`**
- the **kubeconfig of each VM is fetched over SSH** and stored locally, so one run creates a **Deployment (or StatefulSet) on every host's own Kubernetes cluster** with the `kubernetes.core` collection

## Layout

```
.
├── ansible.cfg                 # configuration (inventory, roles_path, ...)
├── site.yml                    # main playbook (2 plays)
├── requirements.txt            # Python dependencies (ansible-core, kubernetes)
├── requirements.yml            # Ansible collections (kubernetes.core)
├── inventory/
│   ├── hosts.yml               # flat list: localhost, vm-35, vm-34
│   ├── group_vars/all.yml
│   └── host_vars/{vm-35,vm-34}.yml
├── kubeconfigs/                # created at runtime: <host>.yml (git-ignored, contains credentials)
└── roles/
    ├── common/                 # facts, set_fact, report template, handler
    ├── kubeconfig/             # fetches the kubeconfig of each VM, selects it with set_fact
    └── k8s_app/                # namespace + ConfigMap + Service + Deployment/StatefulSet from templates
```

## What `site.yml` does

1. **Play 1** (`hosts: all`): gathers facts, reshapes them with `set_fact`
   (host summary, present/missing commands) and renders a report from a template into
   `report_dir` (`/tmp/ansible-from-zero-to-hero/` by default,
   `/home/lbassi/ansible-from-zero-to-hero/` on `vm-35`, `/root/ansible-from-zero-to-hero/` on `vm-34`). It then reads the report back and prints it.
2. **Play 2** (`hosts: all`), for every host:
   - role `kubeconfig`: for a VM it reads `~/.kube/config` over SSH and saves it as
     `kubeconfigs/<host>.yml` (mode `0600`), rewriting `server: https://127.0.0.1...` to the VM address
     if needed; for `localhost` it uses the local `~/.kube/config`. It then checks the cluster is reachable
     and stores the chosen path in `k8s_kubeconfig` (`set_fact`).
   - role `k8s_app`: runs on the controller (`delegate_to: localhost`) with that kubeconfig and creates,
     in the `ansible-demo` namespace of that cluster, a ConfigMap (HTML rendered from a template), a
     Service and an `nginx` **Deployment** `hello-<host>` (2 replicas) serving it. With
     `-e k8s_app_kind=StatefulSet` it creates a **StatefulSet** instead (pods `hello-<host>-0`, `-1`,
     plus a headless Service); switching kind removes the previous workload. The pods are restarted
     automatically when the ConfigMap content changes (checksum annotation).

## Prerequisites

- Python 3 and `kubectl` configured for your local cluster (`kubectl get nodes` must work)
- For each VM: SSH access (`lbassi` on `vm-35`, `root` on `vm-34`) and a kubeconfig readable at `~/.kube/config` on the VM
  (the Kubernetes API must be reachable from this machine on the address in the kubeconfig)

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
ssh-copy-id root@10.150.47.34
ssh lbassi@10.150.47.35 true            # accept the host key the first time (same for root@10.150.47.34)
```

If you need a specific key, add `ansible_ssh_private_key_file: ~/.ssh/<key>` to the host in
[inventory/hosts.yml](inventory/hosts.yml).

### Kubeconfig on the VMs

By default the role reads `~/.kube/config` of the SSH user. To change that, override these
variables (in `host_vars/<vm>.yml` or with `-e`):

| Variable | Default | Purpose |
|---|---|---|
| `kubeconfig_remote_src` | `~/.kube/config` on the VM | file to read |
| `kubeconfig_remote_become` | `false` | read it with sudo (e.g. k0s `/var/lib/k0s/pki/admin.conf`) |
| `kubeconfig_local_dir` | `./kubeconfigs` | where the copies are saved |

On a k0s VM you can create the file with `sudo k0s kubeconfig admin > ~/.kube/config`.

## Usage

```bash
# Inspect the inventory
ansible-inventory --graph
ansible-inventory --host vm-35           # resolved variables of the host
ansible all -m ping                      # connectivity test

# Run everything on all hosts
ansible-playbook site.yml

# Localhost only (workload + local report), without touching the VMs
ansible-playbook site.yml --limit localhost

# A single VM
ansible-playbook site.yml --limit vm-35

# Several hosts: each one gets its own kubeconfig and its own workload
ansible-playbook site.yml --limit localhost,vm-35,vm-34

# Run only part of it, using tags
ansible-playbook site.yml --tags facts
ansible-playbook site.yml --tags report
ansible-playbook site.yml --tags kubeconfig    # only fetch the kubeconfigs and check the clusters
ansible-playbook site.yml --tags k8s           # kubeconfig + workload

# Dry run and diff of generated files
ansible-playbook site.yml --check --diff --limit vm-35

# Override variables from the CLI (highest precedence)
ansible-playbook site.yml --tags k8s -e k8s_app_message="Hi!" -e k8s_app_image=nginx:alpine
ansible-playbook site.yml --tags k8s -e k8s_app_kind=StatefulSet -e k8s_app_replicas=3

# Verbosity
ansible-playbook site.yml -v
```

If `sudo` is needed on a VM, add `--ask-become-pass`. An unreachable host is reported and skipped;
the other hosts keep running.

### Verify the workload

```bash
# localhost cluster
kubectl -n ansible-demo get deploy,sts,svc,pod,cm
kubectl -n ansible-demo port-forward svc/hello-localhost 8080:80
curl http://localhost:8080

# a VM cluster: use its fetched kubeconfig
kubectl --kubeconfig kubeconfigs/vm-34.yml -n ansible-demo get deploy,sts,svc,pod,cm
kubectl --kubeconfig kubeconfigs/vm-34.yml -n ansible-demo port-forward svc/hello-vm-34 8080:80
```

### Cleanup

```bash
ansible-playbook site.yml --limit localhost,vm-35,vm-34 --tags k8s -e k8s_state=absent   # deletes the namespace
rm -rf /tmp/ansible-from-zero-to-hero kubeconfigs                                       # local report and kubeconfigs
```

## Variable precedence (example in this repo)

`environment_name` is defined in several places: `demo` in `group_vars/all.yml`, but the host
`vm-35` and `vm-34` override it with `vm-lab` in `host_vars/vm-35.yml` and `host_vars/vm-34.yml`. Check it with:

```bash
ansible-inventory --host vm-35 | grep environment_name
```

## Notes

- The `k8s_app` role defaults live in `roles/k8s_app/defaults/main.yml`; the message is
  overridden for all hosts in `inventory/group_vars/all.yml`.
- `kubeconfigs/` holds cluster credentials: it is git-ignored and the tasks that handle the
  kubeconfig content use `no_log`.
- If two hosts share the same cluster (e.g. `localhost` and `vm-35` on the same machine) they do not
  collide: each one creates its own `hello-<host>` workload. The cleanup deletes the whole
  `ansible-demo` namespace, so it removes the workloads of every host on that cluster.
- The StatefulSet has no volume claims: the cluster has no StorageClass, so it demonstrates stable
  pod names and the headless Service only.
- Runs are idempotent: re-running the playbook without changes reports every task as `ok`
  and the report handler does not fire.
- Optional lint: `pip install ansible-lint && ansible-lint site.yml roles`.
