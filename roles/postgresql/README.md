Postgresql
=========

Installs PostgreSQL on Rocky Linux 9/10 or Ubuntu 24.04/26.04, creates a
database and one or more users with full privileges on it, applies some
basic hardening (`pg_hba.conf` access rule, `listen_addresses`, `postgres`
superuser password), and schedules automatic `pg_dump` backups.

The tasks are split into five tags you can run selectively:
- `install` — install PostgreSQL and start the service
- `createdb` — create the database
- `createusr` — create the users and grant them privileges on the database
- `hardening` — apply the `pg_hba.conf`/`postgresql.conf` and password changes
- `backup` — create the backup directory and schedule the backup cron job

Requirements
------------

- A Rocky Linux 9/10 or Ubuntu 24.04/26.04 host, reachable with `become: true`.
- `python3-psycopg2` gets installed by the role itself, so no manual
  Python prerequisites are needed on the target.

### Collections

This role needs the following collections on the control node (already
declared in the collection's own `requirements.yml`/`galaxy.yml`, and in
this role's own `requirements.yml`):
- `community.general` (`postgresql_db`, `postgresql_user`)
- `community.postgresql` (`postgresql_privs`)
- `ansible.posix` (`firewalld`, RedHat only, skipped inside containers)

Install them with:
```shell
ansible-galaxy collection install -r requirements.yml
```

Role Variables
--------------

- `postgresql_version` (default `"17"`) — PostgreSQL major version to install.
- `postgresql_conf_dir` — directory holding `postgresql.conf`/`pg_hba.conf`.
  Computed automatically from `postgresql_version` and the OS family
  (`/var/lib/pgsql/<version>/data` on RedHat, `/etc/postgresql/<version>/main`
  on Debian); override only if your layout differs.
- `postgresql_hba` — full path to `pg_hba.conf`, derived from `postgresql_conf_dir`.
- `postgresql_conf` — full path to `postgresql.conf`, derived from `postgresql_conf_dir`.
- `postgresql_port` (default `"5432"`) — reserved for the PostgreSQL listen
  port; not currently wired into any task in this role.
- `postgresql_sslmode` (default `"Disable"`) — reserved for SSL configuration;
  not currently wired into any task in this role.
- `postgresql_root_password` (default `"dbrootpass"`) — password set for the
  `postgres` superuser during hardening.
- `postgresql_users` (default: one entry, `name: dbuser`/`password: dbuser`) —
  list of `{name, password}` application database users to create, each
  granted full privileges on `postgresql_database_name`. Add more entries
  to create several users on the same database.
- `postgresql_database_name` (default `"awx"`) — name of the database to create.
- `postgresql_allow_subnet` (default `"0.0.0.0/0"`) — CIDR allowed to connect
  with `md5` auth in `pg_hba.conf`.
- `postgresql_backup_dir` (default `"/backup"`) — directory `pg_dump`
  backups are written to; created automatically (owned by `postgres`,
  mode `0700`) if it doesn't exist.
- `postgresql_backup_minute`, `postgresql_backup_hour`, `postgresql_backup_day`,
  `postgresql_backup_month`, `postgresql_backup_weekday` — standard cron
  schedule fields for the backup job (defaults: `"0"`, `"2"`, `"*"`, `"*"`,
  `"0"` — i.e. weekly, Sunday at 02:00).

Example Playbook
----------------

```yaml
---
- name: Deploy PostgreSQL
  hosts: all
  become: true

  vars:
    postgresql_version: "17"
    postgresql_users:
      - name: "awx-user"
        password: "do7iZNapRTVJXrrnFw"
      - name: "awx-readonly"
        password: "another-password"
    postgresql_database_name: "awx"
    postgresql_root_password: "password"
    postgresql_allow_subnet: "10.0.0.0/24"
    postgresql_backup_dir: "/backup"

  tasks:
    - name: PostgreSQL
      ansible.builtin.include_role:
        name: postgresql
      tags:
        - install
        - createdb
        - createusr
        - hardening
        - backup
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
