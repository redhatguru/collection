patching
=========

Role to update packages on Rocky Linux 9/10 and Ubuntu 24.04/26.04 hosts,
and reboot only when the update actually requires it (e.g. a kernel or
glibc update) and rebooting has been explicitly allowed.

# Requirements
------------

- A Rocky Linux 9/10 or Ubuntu 24.04/26.04 host, reachable with `become: true`.
- Reboot detection relies on `dnf-utils` (RedHat, provides `needs-restarting`)
  and `update-notifier-common` (Debian, maintains `/var/run/reboot-required`);
  the role installs both itself if missing.

### Collections

This role only uses modules built into `ansible-core` (`ansible.builtin.*`),
so no extra collections need to be installed.

# Role Variables
--------------

Below you can find the default vars used by this role:
- `patching_reboot` (default `false`) — allow the role to reboot the host.
  Even when `true`, a reboot only happens if one is actually required
  (checked via `needs-restarting -r` on RHEL family hosts, and
  `/var/run/reboot-required` on Debian family hosts). With the default
  `false`, packages are updated but the host is never rebooted, even if
  one is required.
- `patching_reboot_timeout` (default `600`) — seconds to wait for the host
  to come back up after a reboot, passed to `ansible.builtin.reboot`.

# Dependencies
------------

No dependencies

# Example Playbook
----------------

Example playbook:
```yaml
---
- name: Patch servers
  hosts: all
  become: true
  vars:
    patching_reboot: true

  tasks:
    - name: Patching
      ansible.builtin.include_role:
        name: patching
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
