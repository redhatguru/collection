k3s
=========

Role to install k3s on an single node

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

License
-------

license (BSD, MIT)

Author Information
------------------

for questions, Contact guido-_@live.nl

