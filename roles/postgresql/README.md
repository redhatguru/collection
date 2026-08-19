Postgresql
=========

This playbook is created to install an Postgres database on an Redhat 9 Based machine,
You can also create users and databases

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

- postgresql_version: "17"
- postgresql_hba: "/var/lib/pgsql/17/data/pg_hba.conf"
- postgresql_conf: "/var/lib/pgsql/17/data/postgresql.conf"
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
        postgresql_hba: "/var/lib/pgsql/{{ postgresql_version }}/data/pg_hba.conf"
        postgresql_conf: "/var/lib/pgsql/{{ postgresql_version }}/data/postgresql.conf"

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

License
-------
license (BSD, MIT)

Author Information
------------------

for questions, Contact guido-_@live.nl
