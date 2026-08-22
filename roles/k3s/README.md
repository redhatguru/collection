k3s
=========

Installs a single-node [K3s](https://k3s.io) cluster (without the bundled
Traefik ingress controller) plus `kubectl`, on Rocky Linux 9/10 or Ubuntu
24.04/26.04. Sets up the upstream Kubernetes package repository (`dnf`
repo on RedHat, `apt` repo on Debian) to install `kubectl`, and writes a
working `~/.kube/config` for the connecting user.

Requirements
------------

- A Rocky Linux 9/10 or Ubuntu 24.04/26.04 host, reachable with `become: true`
  and outbound internet access (to `get.k3s.io` and `pkgs.k8s.io`).
- `firewalld` is stopped/disabled on RedHat hosts as part of the role; it is
  skipped on Debian family hosts, which don't ship it by default.

### Collections

This role only uses modules built into `ansible-core` (`ansible.builtin.*`),
so no extra collections need to be installed.

Role Variables
--------------

Below you can find the default vars used by this role:
- `k3s_version` (default `v1.28.6+k3s2`) — K3s release to install, passed as
  `INSTALL_K3S_VERSION` to the upstream install script.
- `k3s_kubernetes_version` (default `v1.30`) — Kubernetes minor version used
  to select the `pkgs.k8s.io` repository `kubectl` is installed from.

Example Playbook
----------------

Example playbook:
```yaml
---
- name: Install K3s
  hosts: all
  become: true
  vars:
    k3s_version: v1.28.6+k3s2
    k3s_kubernetes_version: v1.30

  tasks:
    - name: K3s
      ansible.builtin.include_role:
        name: k3s
```

Testing
------------

This role has a Molecule scenario that runs the role in VirtualBox VMs (via
Vagrant) on Rocky Linux 9, Rocky Linux 10, Ubuntu 24.04 and Ubuntu 26.04,
closely mirroring real servers:

```shell
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml

# Works around molecule-plugins not registering its custom vagrant module
# with newer molecule releases.
export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

molecule test
```

Requires Vagrant and VirtualBox installed locally.

License
-------

license (BSD, MIT)

Author Information
------------------

for questions, Contact guido-_@live.nl
