# Ansible Collection - redhatguru.general

General purpose Ansible roles and modules maintained by redhatguru.

## Requirements

- ansible-core >= 2.15
- The following collections (installed automatically as dependencies, or via `requirements.yml`):
  - `community.general` >= 8.0.0
  - `community.postgresql` >= 3.0.0
  - `ansible.posix` >= 1.5.0

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
| [`k3s`](roles/k3s) | Install a single-node K3s cluster without Traefik, plus `kubectl`. | Rocky Linux 9/10, Ubuntu 24.04/26.04 |
| [`postgresql`](roles/postgresql) | Install and configure PostgreSQL, including database and user creation and `pg_hba.conf`/`postgresql.conf` hardening. | Rocky Linux 9/10, Ubuntu 24.04/26.04 |
| [`patching`](roles/patching) | Update all packages via `dnf`/`apt`, rebooting only when a reboot is actually required and explicitly allowed (`patching_reboot: true`). | Rocky Linux 9/10, Ubuntu 24.04/26.04 |
| [`kubernetes`](roles/kubernetes) | Install Kubernetes via `kubeadm` (containerd, Calico), as a single node or a `masters`/`workers` cluster, plus the Dashboard, opt-in hardening, and an opt-in rolling upgrade. | Rocky Linux 9/10, Ubuntu 24.04/26.04 |
| [`awx`](roles/awx) | Deploy AWX on an existing Kubernetes cluster via the AWX Operator, using an external PostgreSQL database. | Any host with the `kubernetes` role already applied |

See each role's README for its full variable list and an example playbook.

## Contents

- `roles/` — reusable Ansible roles (see table above)
- `playbooks/` — example/utility playbooks
- `plugins/` — custom modules, filters, and other plugins

## Testing

Every role has at least a `roles/<role>/molecule/default` scenario that runs it
in VirtualBox VMs via Vagrant, typically across Rocky Linux 9, Rocky Linux 10,
Ubuntu 24.04 and Ubuntu 26.04 (see each role's README for its exact scenarios —
`kubernetes` also has multi-node cluster scenarios, and `awx` a two-VM scenario
on Ubuntu 26.04). From inside a role directory:

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
