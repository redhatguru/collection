k3s
=========

Role to install k3s on a single node. Supports Rocky Linux 9/10 and
Ubuntu 24.04/26.04.

# Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

# Role Variables
--------------

Below you can find the default vars used by this role:
- k3s_version: v1.28.6+k3s2
- k3s_kubernetes_version: v1.30


# Dependencies
------------

No dependencies

# Example Playbook
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
    - name: K3S
      ansible.builtin.include_role:
        name: K3S
      when: "'awx-prod' in group_names"

```

# Testing
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

