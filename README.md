# Ansible Collection - redhatguru.general

General purpose Ansible roles and modules maintained by redhatguru.

## Requirements

- ansible-core >= 2.15
- The following collections (installed automatically as dependencies, or via `requirements.yml`):
  - `community.general` >= 8.0.0
  - `community.postgresql` >= 3.0.0
  - `ansible.posix` >= 1.5.0
  - `kubernetes.core` >= 3.0.0

## Installation

```bash
ansible-galaxy collection install redhatguru.general
```

Or, from source:

```bash
ansible-galaxy collection build
ansible-galaxy collection install redhatguru-general-*.tar.gz
```

Either way, also install the collection's own dependencies:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Roles

| Role | Description | Supported platforms |
| --- | --- | --- |
| [`postgresql`](roles/postgresql) | Install and configure PostgreSQL, including database and user creation and `pg_hba.conf`/`postgresql.conf` hardening. | Rocky Linux 9/10, Ubuntu 24.04/26.04 |
| [`patching`](roles/patching) | Update all packages via `dnf`/`apt`, rebooting only when a reboot is actually required and explicitly allowed (`patching_reboot: true`). | Rocky Linux 9/10, Ubuntu 24.04/26.04 |
| [`k3s`](roles/k3s) | Install, configure and harden a K3s cluster — single node, single control plane with workers, or an HA embedded-etcd control plane, all derived from the inventory. Includes OS preparation, a node-role aware firewall that stays enabled, CIS-style hardening, MetalLB, Traefik (HA), cert-manager, etcd snapshots, post-install validation and a rolling upgrade. | RHEL/Rocky/AlmaLinux 9+, Ubuntu 22.04+, Debian 12+ (x86_64, ARM64) |
| [`k3s_app`](roles/k3s_app) | Deploy applications onto an existing cluster from a declarative definition — namespace, workload, Service, Ingress, ConfigMaps, Secrets, PVCs and NetworkPolicies — plus raw manifests and Helm releases. Restricted Pod Security Standards and a default-deny NetworkPolicy by default. | Any host with a kubeconfig for the cluster |
| [`awx`](roles/awx) | Deploy AWX on an existing Kubernetes cluster via the AWX Operator, using an external PostgreSQL database. | Any host with a `kubeadm` Kubernetes cluster already applied |
| [`pxeboot`](roles/pxeboot) | Set up a PXE network boot server (proxy-DHCP + TFTP via `dnsmasq`), with Memtest86+ as the first BIOS/UEFI boot menu entry. | Rocky Linux 9, Ubuntu 24.04/26.04 |

See each role's README for its full variable list and an example playbook.

## Contents

- `roles/` — reusable Ansible roles (see table above)
- `playbooks/` — `site.yml`, `k3s.yml`, `applications.yml` and
  `upgrade.yml` for the K3s roles, plus other utility playbooks
- `inventories/` — worked inventory examples (`lab-single-node`, `lab-ha`)
- `plugins/` — custom modules, filters, and other plugins

## Testing

Every role has at least one Molecule scenario that runs it in VirtualBox VMs
via Vagrant, typically across Rocky Linux 9, Rocky Linux 10, Ubuntu 24.04 and
Ubuntu 26.04 (see each role's README for its exact scenarios — `k3s` has
`default`, `rockylinux` and a four-VM `cluster-ha` scenario, and `awx` a two-VM
scenario on both Ubuntu 26.04 and Rocky Linux 9). From inside a role
directory:

```bash
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml

# Works around molecule-plugins not registering its custom vagrant module
# with newer molecule releases.
export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

molecule test
```

Requires Vagrant and VirtualBox installed locally.

Linting (`ansible-lint` and `yamllint`) runs on every pull request via
`.github/workflows/ansible-lint.yml`. To run it locally:

```bash
pip install ansible-lint yamllint
yamllint .
ansible-lint -c ansible-lint.cfg .
```

## License

GPL-2.0-or-later
