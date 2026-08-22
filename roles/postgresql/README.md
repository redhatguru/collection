Postgresql
=========

This role installs and configures a PostgreSQL database on Rocky Linux 9/10 or
Ubuntu 24.04/26.04. You can also create users and databases.

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

- postgresql_version: "17"
- postgresql_hba: computed from postgresql_conf_dir, e.g. "/var/lib/pgsql/17/data/pg_hba.conf" (RedHat) or "/etc/postgresql/17/main/pg_hba.conf" (Debian)
- postgresql_conf: computed from postgresql_conf_dir, same pattern as postgresql_hba
- postgresql_port: "5432"
- postgresql_sslmode: "Disable"
- postgresql_root_password: "dbrootpass"
- postgresql_user: "dbuser"
- postgresql_user_password: "dbuser"
- postgresql_database_name: "awx"
- postgresql_allow_subnet: "0.0.0.0/0"


Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:
```YAML
    ---
    - name: Deploy Postgress
      hosts: all
      become: true

      vars:
        postgresql_version: "17"
        postgresql_user: "awx-user"
        postgresql_user_password: "do7iZNapRTVJXrrnFw"
        postgresql_database_name: "awx"
        postgresql_root_password: password
        postgres_host: "10.0.0.x"
        postgresql_allow_subnet: "0.0.0.0/0"
        

    tasks:

      - name: Postgress
        ansible.builtin.include_role:
          name: postgres
        tags:
          - prepare
          - install
          - createdb
          - createusr
          - hardening
```

Testing
-------

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
