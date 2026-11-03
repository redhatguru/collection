Postgresql
=========

Installs PostgreSQL on Rocky Linux 9/10 or Ubuntu 24.04/26.04 at a pinned
release and holds it there, configures the server from variables, and creates
databases with their users and extensions. Installing and creating databases
are separate entry points, each with a playbook of its own:

| Playbook | Does |
| --- | --- |
| [`playbooks/postgresql-install.yml`](../../playbooks/postgresql-install.yml) | Installs or updates PostgreSQL and configures the server |
| [`playbooks/postgresql-databases.yml`](../../playbooks/postgresql-databases.yml) | Creates databases, users and extensions, and schedules backups |
| [`playbooks/postgresql.yml`](../../playbooks/postgresql.yml) | Both, in that order |
| [`playbooks/postgresql-major-upgrade.yml`](../../playbooks/postgresql-major-upgrade.yml) | Moves a cluster to a new major version |
| [`playbooks/postgresql-major-upgrade-cleanup.yml`](../../playbooks/postgresql-major-upgrade-cleanup.yml) | Removes the old cluster after that |

All three run against the `postgresql` group and take their variables from
inventory. [`inventories/lab-postgresql`](../../inventories/lab-postgresql) is a
worked example.

Requirements
------------

- A Rocky Linux 9/10 or Ubuntu 24.04/26.04 host, reachable over SSH by an account
  that can use `become`. The role escalates per task.
- The collections in [`requirements.yml`](requirements.yml) on the control node:
  `community.postgresql`, `community.general` (`ufw`, `seport`,
  `dnf_versionlock`, and `version_sort` for Molecule) and `ansible.posix`
  (`firewalld`).

Pinning and updating
--------------------

```yaml
postgresql_version: "17"
postgresql_package_version: "17.10"
```

The same value works on both OS families. The packages are installed at that
release and held: `dpkg` hold on Ubuntu, `versionlock` on Rocky Linux. A package
update on the host, including the [`patching`](../patching) role, leaves them
alone.

To update, raise `postgresql_package_version` and run
`playbooks/postgresql-install.yml`. It replaces the packages, restarts
PostgreSQL and checks that the running server reports the new release. Hosts are
updated one at a time, and a failed update leaves the hold in place.

The role refuses:

- **A downgrade.** A pin below the installed release stops the run.
- **A major version upgrade.** With `postgresql_version` changed on a host that
  runs another major, packages alone would give a second, empty cluster. That
  takes the separate path below.

**The apt repository only keeps the last few releases of a major**, so an old pin
stops being installable on a new Ubuntu host. A host that already has that
release keeps it. Keep the pin reasonably current, or point
`postgresql_apt_repository_url` at a mirror of your own.

Major version upgrades
----------------------

```shell
ansible-playbook -i inventories/lab-postgresql/hosts.yml \
  playbooks/postgresql-major-upgrade.yml \
  -e postgresql_version=18 -e postgresql_major_upgrade_enabled=true
```

Read the release notes of the version you are moving to first, and try it on a
copy. **The database is down from the moment the old server stops until the new
one is up.** In `copy` mode that scales with the size of the data.

The playbook checks that there is one lower major version installed and room for
a second copy of the data, then:

1. dumps the whole server with `pg_dumpall` into `postgresql_major_upgrade_backup_dir`;
2. installs the new packages next to the old ones, and holds them;
3. creates a new cluster with the old one's encoding, locale and data checksums;
4. stops the old server, and runs `pg_upgrade --check` and then `pg_upgrade`;
5. starts the new server, and stops the old one from starting at boot;
6. updates the extensions `pg_upgrade` lists, rebuilds the planner statistics, and
   points the scheduled backups at the new `pg_dump`.

Users, passwords, databases and extensions all come along.

- **Extensions with a package of their own** need the new major's package in
  `postgresql_extra_packages`, for example `postgresql-18-pgvector`. Without it,
  `pg_upgrade --check` stops the upgrade and the old server is started again.
- **Afterwards**, set the new `postgresql_version` in inventory and pin
  `postgresql_package_version` to a release of it. Running the normal playbooks
  with the old version is refused.
- **A rerun** of the playbook on an upgraded host does nothing.

### If it fails

Up to and including `pg_upgrade` in `copy` mode, a failure removes the new cluster
and its packages, and starts the old server again with its data untouched. Fix what it reports and
run the playbook again.

In `link` mode the old cluster is lost once `pg_upgrade` starts linking. A failure
after that point means restoring the dump, which is why `copy` is the default.

If the run itself is interrupted, the old server may be left stopped and a new
cluster may be left behind. Check that cluster, then rerun with
`-e postgresql_major_upgrade_discard_target=true`, which removes it. The role
never removes one it did not just create without that being set.

### Going back

Until the cleanup, the old cluster is intact in `copy` mode, **but it does not have
anything written since the upgrade**. To go back, stop the new server, start the old
one and set it to start at boot again:

```shell
# Ubuntu
sudo systemctl stop postgresql@18-main
echo auto | sudo tee /etc/postgresql/17/main/start.conf
sudo systemctl daemon-reload && sudo systemctl start postgresql@17-main

# Rocky Linux
sudo systemctl disable --now postgresql-18
sudo systemctl enable --now postgresql-17
```

### Cleaning up

```shell
ansible-playbook -i inventories/lab-postgresql/hosts.yml \
  playbooks/postgresql-major-upgrade-cleanup.yml \
  -e postgresql_version=18 -e postgresql_major_upgrade_cleanup_enabled=true
```

This removes the old cluster and its packages, after checking the upgrade finished
and the new server answers. **After this the only way back is the dump.** It stays
where it was written, so remove that yourself when you no longer need it.

Databases, users and extensions
-------------------------------

```yaml
postgresql_databases:
  - name: awx
    plugins:
      - pg_trgm
      - btree_gist
    users:
      - user: awx_test
        password: "{{ vault_awx_test_password }}"
      - user: management
        password: "{{ vault_management_password }}"
```

| Key | Required | Meaning |
| --- | --- | --- |
| `name` | yes | The database |
| `users` | yes, one or more | Each with `user` and `password` |
| `plugins` | no | PostgreSQL extensions to install in this database |
| `owner` | no | Owner of the database. Defaults to `postgres` |
| `encoding` | no | Overrides `postgresql_database_encoding` |
| `lc_collate`, `lc_ctype` | no | Locale of the database. An encoding other than UTF-8 usually needs them too, for example `C`, because PostgreSQL refuses an encoding that does not fit the locale |

- **Privileges.** Each user gets all privileges on its database and on the
  `public` schema. Tables belong to the user who created them, so a second user
  on the same database does not automatically get access to them.
- **Isolation.** Only a database's own users may connect to it
  (`postgresql_database_private`), where PostgreSQL's default lets every user
  connect to every database.
- **One password per user.** A user is one account on the whole server. One that
  appears under several databases needs the same password in each, and the role
  refuses it otherwise.
- **Extensions with a package of their own.** The contrib extensions (`pg_trgm`,
  `btree_gist`, `pg_stat_statements`, ...) are always there. Anything else also
  needs its package in `postgresql_extra_packages`, for example
  `postgresql-17-pgvector` on Ubuntu or `pgvector_17` on Rocky Linux, and that
  list belongs to the install playbook. An extension that has to be preloaded
  also needs `shared_preload_libraries` in `postgresql_settings`.
- **Nothing is removed.** A database or user taken out of the list stays on the
  server.

The role also refuses placeholder and empty passwords, for these and for
`postgresql_root_password`.

Role Variables
--------------

Every variable, with what it is for, is in [`defaults/main.yml`](defaults/main.yml).

| Variable | Default | Purpose |
| --- | --- | --- |
| `postgresql_version` | `"17"` | Major version |
| `postgresql_package_version` | `""` | Minor release to pin, e.g. `"17.10"`. Empty installs the newest and leaves an installed host alone |
| `postgresql_package_hold` | `true` | Hold the packages at their installed release |
| `postgresql_extra_packages` | `[]` | Packages extensions need |
| `postgresql_apt_repository_url`, `_key_url`, `_suite`, `_components` | PGDG | The apt repository, or a mirror |
| `postgresql_yum_repository_url`, `_key_url` | PGDG | The yum repository, or a mirror |
| `postgresql_conf_dir`, `postgresql_hba`, `postgresql_conf`, `postgresql_bin_dir` | per OS family | Paths |
| `postgresql_port` | `5432` | Port the server listens on |
| `postgresql_listen_addresses` | `"*"` | Addresses the server listens on |
| `postgresql_auth_method` | `scram-sha-256` | Method for password logins in `pg_hba.conf` |
| `postgresql_password_encryption` | `scram-sha-256` | How passwords are stored |
| `postgresql_settings` | `{}` | Anything else `postgresql.conf` accepts |
| `postgresql_allow_subnet` | `127.0.0.1/32` | Network allowed to log in, in `pg_hba.conf` and the firewall |
| `postgresql_hba_entries` | one rule for `postgresql_allow_subnet` | The network rules in `pg_hba.conf` |
| `postgresql_firewall_enabled` | `true` | Open the port in `ufw` or `firewalld` |
| `postgresql_root_password` | placeholder | Password of the `postgres` superuser. Has to be set |
| `postgresql_databases` | `[]` | See above |
| `postgresql_database_encoding` | `UTF-8` | Encoding of new databases |
| `postgresql_database_private` | `true` | Only a database's own users may connect to it |
| `postgresql_no_log` | `true` | Hide tasks that handle passwords. Switching it off prints them |
| `postgresql_backup_enabled` | `true` | Schedule a `pg_dump` per database |
| `postgresql_backup_dir` | `/backup` | Where the dumps go |
| `postgresql_backup_minute`, `_hour`, `_day`, `_month`, `_weekday` | Sunday 02:00 | Cron schedule |
| `postgresql_backup_keep_days` | `30` | Days a dump is kept. `0` keeps them all |
| `postgresql_major_upgrade_enabled`, `postgresql_major_upgrade_cleanup_enabled` | `false` | Guards on the major upgrade and on removing the old cluster |
| `postgresql_major_upgrade_mode` | `copy` | `copy` keeps the way back and needs room. `link` is fast and gives it up |
| `postgresql_major_upgrade_backup_before` | `true` | `pg_dumpall` before anything is touched |
| `postgresql_major_upgrade_backup_dir` | `postgresql_backup_dir` | Where that dump goes |
| `postgresql_major_upgrade_extra_arguments` | `[]` | Anything else `pg_upgrade` accepts, e.g. `["--jobs=4"]` |
| `postgresql_major_upgrade_discard_target` | `false` | Remove a new-version cluster an interrupted upgrade left behind |
| `postgresql_validate_retries`, `postgresql_validate_delay` | `12`, `5` | How long to wait for the server after a restart |

`postgresql.conf` is left as the package wrote it, apart from the line that loads
`conf.d`. The role's settings are in `conf.d/90-ansible.conf`, which is rendered
from these variables, as is the whole of `pg_hba.conf`: local edits to either
are overwritten.

Tags
----

| Tag | Runs |
| --- | --- |
| `preflight` | The checks that change nothing |
| `install` | Repository, packages, hold, service |
| `firewall` | The firewall rule |
| `hardening` | `postgresql.conf`, `pg_hba.conf` and the superuser password |
| `validate` | Checks that the running server is the release installed |
| `createusr` | Users and their privileges |
| `createdb` | Databases |
| `plugins` | Extensions |
| `backup` | The backup directory and the cron jobs |
| `report` | Prints what the run left behind |

Testing
-------

One Molecule scenario, `default`, in VirtualBox VMs via Vagrant, on Rocky Linux 9
and 10 and Ubuntu 24.04 and 26.04. It installs the release below the newest one
that the repository offers, proves a second run changes nothing, updates to the
newest, reboots and then checks:

- the running release, and the hold or lock at that release;
- the settings, port and `pg_hba.conf` rule from the variables;
- each user logging in over TCP and creating a table in its own database, while
  another database and a wrong password are refused;
- owner, encoding and extensions per database, including one with a package of
  its own;
- a backup job that writes a real dump when run the way cron runs it;
- refusal of a downgrade, a major upgrade, a placeholder password and a user with
  two passwords.

A second scenario, `major-upgrade`, on the same platforms, builds PostgreSQL 17 with
data in it, and then:

- makes an upgrade fail, and checks the old server is back with its data;
- upgrades to 18, and checks the data, logins, extensions, settings and hold,
  also after a reboot, with the old server stopped;
- proves the normal playbooks and a second upgrade run change nothing;
- refuses an upgrade without the guard, a move back down and a cleanup without
  the guard;
- removes the old cluster, and checks the new one is unaffected.

```shell
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml

# Works around molecule-plugins not registering its custom vagrant module
# with newer molecule releases.
export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

molecule test
molecule test -s major-upgrade
```

Requires Vagrant and VirtualBox installed locally.

License
-------
license (BSD, MIT)

Author Information
------------------

for questions, Contact guido-_@live.nl
