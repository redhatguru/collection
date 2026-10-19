gitlab
======

Installs and configures a single-node [GitLab](https://about.gitlab.com)
(Omnibus) on Ubuntu, and then proves it works.

What it delivers, in one run, with no manual step afterwards:

- A **hardware and platform preflight** that refuses a host GitLab cannot run
  on — before a single package is installed
- The official Omnibus package from `packages.gitlab.com`, at a **pinned
  release**, installed without piping a downloaded script into a shell
- The whole of `/etc/gitlab/gitlab.rb` **rendered from role variables**, so the
  configuration of the instance is one reviewable file and a change to it is a
  diff
- **Both halves of a restorable backup** — the data archive and the
  configuration archive that holds `gitlab-secrets.json` — scheduled, pruned,
  and available on demand
- A **guarded upgrade path**: backup first, downgrades and unsupported version
  jumps refused, then reconfigure, restart and verify
- **A host firewall that stays enabled**, opening exactly the ports this
  instance serves — derived from the external URL and the enabled components
- **Validation after every run**: services up, the running release, the web
  interface answering, no pending migrations, credentials root-only, and the
  instance actually reachable from outside the host

One GitLab per host. A reference architecture that splits Rails, Gitaly,
PostgreSQL and Redis across machines is a different shape of deployment and
deliberately out of scope.

Contents
--------

1. [Requirements](#1-requirements)
2. [Inventory](#2-inventory)
3. [Installation](#3-installation)
4. [Variables](#4-variables)
5. [Configuration: how `gitlab.rb` is built](#5-configuration-how-gitlabrb-is-built)
6. [http or https](#6-http-or-https)
7. [Host firewall](#7-host-firewall)
8. [Backup](#8-backup)
9. [Restore](#9-restore)
10. [Upgrade](#10-upgrade)
11. [Validation](#11-validation)
12. [Tags](#12-tags)
13. [Testing](#13-testing)
14. [Troubleshooting](#14-troubleshooting)

1. Requirements
---------------

**The host.** These are GitLab's own published minimums for a single-node
Omnibus install serving up to around 500 users, and
[`tasks/preflight.yml`](tasks/preflight.yml) asserts every one of them:

| Resource | Minimum | Variable |
| --- | --- | --- |
| vCPU | 4 | `gitlab_preflight_min_cpu` |
| Memory | 3900 MB | `gitlab_preflight_min_memory_mb` |
| Swap | 2040 MB | `gitlab_preflight_min_swap_mb` |
| Free disk, fresh install | 20480 MB, on the filesystem holding each of `/opt`, `/var/opt/gitlab`, `/etc/gitlab` | `gitlab_preflight_min_disk_free_mb`, `gitlab_preflight_disk_paths` |
| Free disk, instance already installed | 5120 MB on the same filesystems | `gitlab_preflight_min_disk_free_installed_mb` |

The memory and swap figures are GitLab's round 4 GB and 2 GB minus what a host
never reports: the kernel and firmware reserve a few percent of RAM, and
`mkswap` reserves a page for its header, so a swap device of exactly 2 GiB
reports 2047 MB. Thresholds set at 4096 and 2048 would reject a host
provisioned exactly as GitLab documents, while these still reject the next
size down.

Below these, the first `gitlab-ctl reconfigure` either takes hours or has its
Puma workers killed by the OOM reaper part way through the database
migrations, which leaves an instance no later run repairs. Swap is not
optional for the same reason — it is what turns a memory spike into slowness
instead of a half-migrated database.

Disk is checked against two different figures because the question changes
once GitLab is there: a fresh host needs room for the package and the data,
while a running instance has already spent most of that and instead needs
enough left to take the next package upgrade (~3 GB unpacked) and a backup.
The role looks at whether the package is installed and applies the figure that
applies.

The preflight also refuses to continue when:

- the host is not Ubuntu, or not x86\_64/aarch64;
- the Ubuntu release has no Omnibus packages published for it — the role
  supports **22.04 (jammy), 24.04 (noble) and 26.04 (resolute)**
  (`gitlab_supported_ubuntu_releases`). 20.04 is deliberately not among them:
  packages.gitlab.com still serves it, but stops at 18.11.x, so an instance
  there cannot be upgraded further without moving the host first;
- `gitlab_external_url` is not an http/https URL with a real host name, or
  that name does not resolve on the host;
- `gitlab_version` is set but is not a package version like `19.5.0-ce.0`;
- GitLab would have no database, or two;
- the URL is https but there is neither a certificate nor Let's Encrypt;
- an enabled component has no address (registry, Pages, SMTP);
- `gitlab_backup_skip` names a component `gitlab-backup` does not know — it
  ignores an unknown name and backs the component up anyway, so a backup you
  believed was small quietly is not;
- `packages.gitlab.com` is unreachable.

**The controller.** ansible-core >= 2.15, plus the collections in
[`requirements.yml`](requirements.yml) — `community.general` and
`ansible.posix`, both only for the Molecule scenario.

**The network.** The host needs HTTPS to `packages.gitlab.com` for the
install. The role manages the host firewall itself — see
[section 7](#7-host-firewall).

2. Inventory
------------

One group, `gitlab_servers`, and one host in it per instance:

```yaml
# inventories/lab-gitlab/hosts.yml
all:
  children:
    gitlab_servers:
      hosts:
        gitlab-01:
          ansible_host: 10.10.60.10
```

Several hosts in the group means several independent instances, not a
cluster. A worked example with the variables that matter is in
[`inventories/lab-gitlab`](../../inventories/lab-gitlab).

3. Installation
---------------

```yaml
# inventories/lab-gitlab/group_vars/gitlab_servers.yml
gitlab_version: "19.5.0-ce.0"
gitlab_external_url: "https://gitlab.example.com"
gitlab_initial_root_password: "{{ vault_gitlab_root_password }}"
gitlab_nginx_ssl_certificate: "/etc/gitlab/ssl/gitlab.example.com.crt"
gitlab_nginx_ssl_certificate_key: "/etc/gitlab/ssl/gitlab.example.com.key"
```

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventories/lab-gitlab/hosts.yml playbooks/gitlab.yml
```

Two playbooks install GitLab, and they differ only in where the variables
live:

| | Variables from | Use it when |
| --- | --- | --- |
| [`playbooks/gitlab.yml`](../../playbooks/gitlab.yml) | inventory `group_vars`/`host_vars` | More than one instance, or anything you want to override per host |
| [`playbooks/gitlab-example.yml`](../../playbooks/gitlab-example.yml) | the play itself | One instance specified in one file — and as the checklist: every variable the role accepts is set or shown there |

Play `vars:` outrank inventory, so the example playbook's values cannot be
overridden per host from `group_vars`. That is the right trade for a single
worked instance and the wrong one for a fleet.

The first run takes 10–20 minutes, almost all of it in the one
`gitlab-ctl reconfigure` that initialises the database. Sign in at the
external URL as `root` with `gitlab_initial_root_password` — which is applied
on the **first** reconfigure only, so changing it afterwards does nothing;
change the password in GitLab itself.

Re-running the playbook is safe and changes nothing unless a variable
changed. It will **not** replace the GitLab package: if `gitlab_version` names
a release other than the installed one, the run says so and leaves the
instance alone, because replacing the package runs database migrations. That
is what [`playbooks/gitlab-upgrade.yml`](../../playbooks/gitlab-upgrade.yml)
is for.

4. Variables
------------

Full list with comments: [`defaults/main.yml`](defaults/main.yml). Values the
role derives itself (package names, the paths Omnibus owns, the parsed
external URL) are in [`vars/`](vars/) and are deliberately not overridable —
overriding the parsed URL separately from `gitlab_external_url` would let the
preflight checks pass while GitLab is configured from something else.

Every variable is prefixed `gitlab_`.

### Release and edition

| Variable | Default | Purpose |
| --- | --- | --- |
| `gitlab_edition` | `ce` | `ce` or `ee` |
| `gitlab_version` | `""` | The release to converge on, e.g. `19.5.0-ce.0`. Empty tracks whatever the repository calls newest; an upgrade refuses to run unpinned |
| `gitlab_package_hold` | `true` | Hold the package in dpkg, so `unattended-upgrades` cannot run a GitLab upgrade at 06:00 on its own |
| `gitlab_upgrade_enabled` | `false` | Second lock on the upgrade path |
| `gitlab_upgrade_backup_before` | `true` | Back up before replacing the package |
| `gitlab_upgrade_allow_path_skip` | `false` | Accept a version jump bigger than one minor release |

### Instance identity

| Variable | Default | Purpose |
| --- | --- | --- |
| `gitlab_external_url` | `http://<fqdn>` | The URL clients use. Every clone URL, email link and OAuth redirect is built from it |
| `gitlab_time_zone` | `UTC` | |
| `gitlab_initial_root_password` | `ChangeMe123!` | Applied on the first reconfigure only. Override from Vault |
| `gitlab_shell_ssh_port` | `22` | Port advertised in SSH clone URLs. Does not move `sshd` |
| `gitlab_trusted_proxies` | `[]` | Reverse proxies in front of GitLab, so it logs and rate-limits the real client address |

### nginx and TLS

| Variable | Default | Purpose |
| --- | --- | --- |
| `gitlab_nginx_enabled` | `true` | Set false when a load balancer terminates TLS and proxies to GitLab |
| `gitlab_nginx_listen_https` | `""` | Empty follows the scheme of the external URL |
| `gitlab_nginx_listen_port` | `""` | Empty follows the port of the external URL |
| `gitlab_nginx_redirect_http_to_https` | `true` | Only applies to an https instance |
| `gitlab_tls_certificate_source` | `file` | `file`, `self-signed` or `letsencrypt` — see section 6. Ignored over plain HTTP |
| `gitlab_nginx_ssl_certificate`, `..._key` | `""` | The `file` source: a certificate you manage. Not created or copied by this role |
| `gitlab_tls_self_signed_dir` | `/etc/gitlab/ssl` | Where the `self-signed` source writes the key and certificate |
| `gitlab_tls_self_signed_days` | `3650` | Validity of the generated certificate |
| `gitlab_tls_self_signed_key_size` | `4096` | |
| `gitlab_tls_self_signed_extra_sans` | `[]` | Extra `DNS:`/`IP:` names on the generated certificate |
| `gitlab_nginx_hsts_enabled`, `..._max_age` | `true`, `31536000` | Understand the max-age before raising it: a browser that has seen the header refuses plain HTTP for that long, with no way to revoke |
| `gitlab_nginx_client_max_body_size` | `250m` | The practical limit on an HTTP push or a large upload |
| `gitlab_letsencrypt_contact_emails` | `[]` | Where Let's Encrypt sends expiry warnings. Required by that source |

### Email

`gitlab_smtp_enabled` (default `false`) plus `gitlab_smtp_address`, `_port`,
`_user_name`, `_password`, `_domain`, `_authentication`,
`_enable_starttls_auto`, `_tls`, `_openssl_verify_mode`, and
`gitlab_email_from` / `_display_name` / `_reply_to`.

With SMTP off, GitLab tries to deliver through a local MTA that this role does
not install, so password resets and notifications go nowhere. Configure SMTP
or install an MTA yourself.

### Backup

| Variable | Default | Purpose |
| --- | --- | --- |
| `gitlab_backup_enabled` | `true` | Manage the destinations and the schedule |
| `gitlab_backup_path` | `/var/opt/gitlab/backups` | Data archives |
| `gitlab_backup_config_path` | `/etc/gitlab/config_backup` | Configuration and secrets archives |
| `gitlab_backup_keep_time` | `604800` (7 days) | GitLab prunes the data archives itself |
| `gitlab_backup_config_keep_days` | `7` | GitLab prunes nothing here, so the role installs a job that does |
| `gitlab_backup_archive_permissions` | `0o600` | Ruby literal, rendered verbatim |
| `gitlab_backup_skip` | `[]` | Components to leave out, for data that is backed up elsewhere |
| `gitlab_backup_schedule_enabled` | `true` | |
| `gitlab_backup_minute`/`_hour`/`_day`/`_month`/`_weekday` | `0 2 * * *` | Data backup schedule |
| `gitlab_backup_config_minute`/`_hour` | `0 1` | Configuration backup, an hour earlier so a half-finished one is never the newest |
| `gitlab_backup_upload_connection`, `_remote_directory` | `{}`, `""` | Fog settings for uploading archives to object storage from GitLab itself |

### Sizing and components

| Variable | Default | Purpose |
| --- | --- | --- |
| `gitlab_puma_worker_processes` | `2` | |
| `gitlab_puma_min_threads`, `_max_threads` | `4`, `4` | |
| `gitlab_puma_per_worker_max_memory_mb` | `1024` | |
| `gitlab_sidekiq_max_concurrency` | `10` | |
| `gitlab_postgresql_enabled` | `true` | Bundled PostgreSQL |
| `gitlab_postgresql_shared_buffers` | `512MB` | |
| `gitlab_postgresql_max_connections` | `200` | |
| `gitlab_database_external_enabled` | `false` | Use an external PostgreSQL — for example one built by this collection's [`postgresql`](../postgresql) role — with `gitlab_database_host`, `_port`, `_name`, `_user`, `_password`, `_sslmode` |
| `gitlab_redis_enabled`, `gitlab_redis_maxmemory` | `true`, `""` | |
| `gitlab_registry_enabled`, `gitlab_registry_external_url` | `false`, `""` | Container registry |
| `gitlab_pages_enabled`, `gitlab_pages_external_url` | `false`, `""` | GitLab Pages |
| `gitlab_kas_enabled` | `true` | Agent server for Kubernetes |
| `gitlab_mattermost_enabled` | `false` | |
| `gitlab_prometheus_enabled`, `gitlab_node_exporter_enabled` | `true`, `true` | What GitLab's own admin dashboards read from |
| `gitlab_alertmanager_enabled` | `false` | |
| `gitlab_monitoring_whitelist` | `127.0.0.0/8`, `::1/128` | Addresses allowed to read `/-/health`, `/-/readiness`, `/-/liveness`, `/-/metrics`. The role's validation reads them over the loopback interface, so keep `127.0.0.0/8` |

### Authentication

`gitlab_ldap_enabled` with `gitlab_ldap_servers` (a dict, rendered as the YAML
block GitLab documents — keep the bind password in Vault), and
`gitlab_omniauth_enabled` with `gitlab_omniauth_providers`,
`_allow_single_sign_on` and `_block_auto_created_users`.

### Validation

`gitlab_validate_enabled` (default `true`) and the individual checks:
`gitlab_validate_check_services`, `_check_version`, `_check_http`,
`_check_migrations`, plus `gitlab_validate_run_gitlab_check` (GitLab's own
full self-test — thorough, slow, opt-in),
`gitlab_validate_http_validate_certs`, `gitlab_validate_retries` and
`gitlab_validate_delay`.

5. Configuration: how `gitlab.rb` is built
------------------------------------------

`/etc/gitlab/gitlab.rb` is rendered from
[`templates/gitlab.rb.j2`](templates/gitlab.rb.j2) and from nothing else. The
file on the host is therefore disposable: edit a variable, re-run the role,
and the previous content is replaced. The template writes it `0600`, root-only
— it holds the initial root password and any SMTP, LDAP or database
credentials — and the task that writes it does not print its diff.

Values are converted to Ruby literals by one macro, so a plain YAML value
configures a Ruby file without the caller thinking about quoting:

| YAML | Rendered |
| --- | --- |
| `"250m"` | `"250m"` |
| `30` | `30` |
| `false` | `false` |
| `["a", "b"]` | `["a", "b"]` |
| `{x: 1}` | `{ "x" => 1 }` |

Before the file is applied it is checked with
`/opt/gitlab/embedded/bin/ruby -c`, so a typo in an override is caught there
rather than half way through a reconfigure with the previous configuration
already torn down.

For settings the named variables do not cover there are three escape hatches,
in increasing order of bluntness:

```yaml
# Any gitlab_rails['...'] setting.
gitlab_rails_extra_configuration:
  gitlab_default_theme: 2
  env:
    GITLAB_LOG_LEVEL: "warn"

# Any setting of one bundled component: *_extra_configuration exists for
# nginx, puma, sidekiq, postgresql, redis, registry, registry_nginx,
# gitlab_pages and prometheus.
gitlab_nginx_extra_configuration:
  ssl_protocols: "TLSv1.2 TLSv1.3"

# Raw Ruby, appended verbatim and never parsed by the role. For the rare
# setting that is not a simple assignment.
gitlab_extra_configuration: |
  gitlab_rails['object_store']['enabled'] = true
```

Applying a change is a handler, so a run that changes nothing leaves GitLab
alone, and several changes in one run cost one `gitlab-ctl reconfigure`.

6. http or https
----------------

The scheme of `gitlab_external_url` is the choice, and the only one — there is
no second "enable TLS" switch, because GitLab builds every clone URL, email
link and OAuth redirect from that one value and a separate flag could only
contradict it.

```yaml
# Plain HTTP. Nothing else to configure.
gitlab_external_url: "http://gitlab.example.com"

# TLS, terminated by GitLab's own nginx. Then pick where the certificate
# comes from:
gitlab_external_url: "https://gitlab.example.com"
gitlab_tls_certificate_source: "file"   # file | self-signed | letsencrypt
```

| Source | Where the certificate comes from | Use it when |
| --- | --- | --- |
| `file` (default) | `gitlab_nginx_ssl_certificate` and `..._key`, files you put on the host. The role points nginx at them and creates nothing | You have a certificate from your own CA or a public one |
| `self-signed` | Generated on the host by the role: 4096-bit key, valid for `gitlab_tls_self_signed_days` (10 years), covering the external URL's host as a DNS name or an IP address, whichever it is | No public CA can certify the name at all — an IP address, a `.local` name, an instance with no route from the internet |
| `letsencrypt` | Issued and renewed by GitLab itself | A publicly resolvable name, with port 80 reachable from the internet for the HTTP-01 challenge |

The source is ignored entirely over plain HTTP, and the preflight checks tell
you which choices you have when the one you picked cannot work — an https URL
with `file` and no paths set, or `letsencrypt` on an IP address, both stop the
run with the alternatives named.

### self-signed

```yaml
gitlab_external_url: "https://10.11.0.250"
gitlab_tls_certificate_source: "self-signed"

# Optional: more names and addresses clients reach it on.
gitlab_tls_self_signed_extra_sans:
  - "DNS:gitlab.internal.example"
  - "IP:10.11.0.251"
```

The key and certificate land in `/etc/gitlab/ssl/<host>.key` and `.crt`
(`gitlab_tls_self_signed_dir`), which is where GitLab itself looks, and the
key is `0600` root. Changing the external URL or the extra names regenerates
the certificate; nothing else does, so a normal run leaves it alone. A
certificate change restarts nginx rather than reconfiguring GitLab — seconds
instead of minutes.

A self-signed certificate is a deliberate choice, not a fallback. Clients warn
until it is trusted, and **git refuses to clone over HTTPS** until it is:

```bash
# On each client, once — or distribute the certificate through your own trust store.
sudo cp gitlab.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates
```

### Behind a load balancer

When something in front of this host terminates TLS, keep the external URL
`https` and tell nginx to serve plain HTTP:

```yaml
gitlab_external_url: "https://gitlab.example.com"
gitlab_nginx_listen_https: false
gitlab_nginx_listen_port: 80
gitlab_trusted_proxies: ["10.0.0.10"]
```

Both nginx settings, not just the first: `listen_https` decides whether nginx
speaks TLS, and the port still follows the external URL unless you say
otherwise — so without `listen_port` nginx would serve plain HTTP on 443.
That is GitLab's own reverse-proxy recipe, and the role follows it rather than
guessing a port for you.

The certificate source then does not apply — the balancer has the
certificate — and the role's validation notices, dialling the port nginx
actually listens on rather than the one in the URL.

7. Host firewall
----------------

The firewall stays on. `ufw` is installed, the ports this instance actually
serves are opened, unsolicited incoming traffic is denied by default, and the
firewall is enabled — the same model as this collection's
[`k3s`](../k3s) role, and for the same reason: a service is not finished
until the host it runs on is defensible.

The ports are **derived, not listed**, so they follow the configuration
instead of drifting from it:

| Opened | Where it comes from |
| --- | --- |
| The port nginx serves on | `gitlab_external_url` (or `gitlab_nginx_listen_port`) |
| `80` | only when an https instance redirects plain HTTP (`gitlab_nginx_redirect_http_to_https`) |
| The registry port | `gitlab_registry_external_url`, when the registry is enabled |
| The Pages port | `gitlab_pages_external_url`, when Pages is enabled |
| SSH | the port this play connects over, **and** `gitlab_shell_ssh_port` for git-over-SSH |
| Anything else | `gitlab_firewall_extra_tcp_ports`, `..._udp_ports` |

An http instance therefore gets `80` and not `443`, and an instance on
`https://host:8443` gets `8443` and `80` — not a guess at both.

| Variable | Default | Purpose |
| --- | --- | --- |
| `gitlab_firewall_enabled` | `true` | Manage the rules at all |
| `gitlab_firewall_manage_state` | `true` | Also enable ufw and set its default incoming policy to deny. Set false where something else owns the firewall's state and you only want GitLab's rules added |
| `gitlab_firewall_ssh_sources` | `[]` | Networks allowed to reach SSH. Empty means anywhere |
| `gitlab_firewall_http_sources` | `[]` | Networks allowed to reach the web interface and registry. Empty means anywhere |
| `gitlab_firewall_extra_tcp_ports`, `..._udp_ports` | `[]` | Anything else this host serves |

SSH is allowed **before** ufw is enabled, and the port the play is connected
over is always among the allowed ones, so turning the firewall on cannot drop
the session doing it. The rules are written after the GitLab package is
installed, so nothing is mid-download when the ruleset is rebuilt, and before
GitLab serves anything.

Two things worth knowing if you turn this off:

- A `ufw` that is already active keeps blocking GitLab, and the role will not
  tell you — the HTTP validation goes over the loopback interface, which no
  firewall touches.
- That is what `gitlab_validate_check_reachable` exists for: at the end of a
  run the controller dials GitLab's web port on
  `gitlab_validate_reachable_address` (by default the address Ansible connects
  over). It is the only check that leaves the host, and the only one that
  notices a firewall, a cloud security group or a routing mistake between
  clients and the instance. Switch it off where the controller reaches the
  host through a bastion that does not carry HTTP.

8. Backup
---------

A GitLab backup is two separate things, and a restore needs both:

| | Written by | To | Pruned by |
| --- | --- | --- | --- |
| Data — repositories, database, uploads, artifacts | `gitlab-backup create` | `gitlab_backup_path` | GitLab, on `gitlab_backup_keep_time` |
| Configuration — `/etc/gitlab`, including `gitlab-secrets.json` | `gitlab-ctl backup-etc` | `gitlab_backup_config_path` | a cron job this role installs |

Without the secrets file the database restores but everything encrypted with
it — CI/CD variables, access tokens, 2FA secrets, integration credentials —
stays unreadable. A data-only backup is not a backup of this instance.

The role schedules both (`/etc/cron.d/gitlab-backup` and
`/etc/cron.d/gitlab-backup-config`). To take one now:

```bash
ansible-playbook -i inventories/lab-gitlab/hosts.yml playbooks/gitlab-backup.yml
```

That runs [`tasks/backup-create.yml`](tasks/backup-create.yml), which lists the
archives before and after and **asserts that one of each actually appeared** —
both commands exit 0 on a backup that skipped what it could not read, so the
archives are what gets checked, not the exit codes. It prints what it wrote.

The archives are only proof against this host failing once they are copied off
it. `gitlab_backup_upload_connection` does that from GitLab itself; otherwise
it is a job for whatever backs up the rest of your estate.

9. Restore
----------

Restoring is a deliberate, destructive operation on a running instance, so it
is not automated here. The procedure, for an instance built by this role:

```bash
# 1. Same GitLab version as the backup was taken on. Check with:
#      sudo cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
#    and the version in the archive's file name.

# 2. Restore /etc/gitlab first — the secrets in it are what make the
#    database readable.
sudo tar -xf /etc/gitlab/config_backup/gitlab_config_<timestamp>.tar -C /

# 3. Stop the services that write to the database.
sudo gitlab-ctl reconfigure
sudo gitlab-ctl stop puma
sudo gitlab-ctl stop sidekiq

# 4. Restore the data. BACKUP= is the archive name without
#    _gitlab_backup.tar.
sudo gitlab-backup restore BACKUP=<timestamp>_<date>_<version>

# 5. Bring it back and check it.
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart
sudo gitlab-rake gitlab:check SANITIZE=true
```

Then run the role again (`playbooks/gitlab.yml`): it re-renders `gitlab.rb`
from your variables and runs the full validation.

10. Upgrade
-----------

```bash
ansible-playbook -i inventories/lab-gitlab/hosts.yml playbooks/gitlab-upgrade.yml \
  -e gitlab_version=19.5.0-ce.0 -e gitlab_upgrade_enabled=true
```

Set `gitlab_version` in your inventory instead of on the command line to keep
it as the pinned release for every later run.

What [`tasks/upgrade.yml`](tasks/upgrade.yml) does, in this order:

1. Refuses to run unless `gitlab_upgrade_enabled` is true and `gitlab_version`
   names a release.
2. Refuses a **downgrade** — the migrations the newer release ran have already
   changed the schema, and the older application cannot read it. Restore a
   backup instead.
3. Refuses a jump that skips a stop on GitLab's
   [upgrade path](https://docs.gitlab.com/update/upgrade_paths/): more than one
   minor release, or a major boundary. Check your jump against the
   [upgrade path tool](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/)
   and set `gitlab_upgrade_allow_path_skip: true` to override.
4. Takes a full backup (data **and** secrets) while the instance still works,
   and stops if it did not produce both archives.
5. Lifts the dpkg hold, installs the pinned package, puts the hold back, and
   asserts that the release installed is the one that was asked for.
6. `gitlab-ctl reconfigure`, which runs the database migrations, then
   `gitlab-ctl restart`.
7. Runs the full validation from section 11 — including that the version now
   *serving requests* is the new one, not just the version installed.

GitLab is unavailable from the moment the package is replaced until the
migrations have finished. `playbooks/gitlab-upgrade.yml` uses `serial: 1`, so
with more than one instance in the group a release that breaks on your data
stops the run at the first instance instead of all of them.

The upgrade deliberately leaves `gitlab.rb` alone — it changes the
application, not the configuration. Apply configuration changes with
`playbooks/gitlab.yml`, before or after.

After an upgrade GitLab finishes **background migrations** on its own
schedule. Let them drain (Admin → Monitoring → Background migrations) before
the next upgrade.

11. Validation
--------------

[`tasks/validate.yml`](tasks/validate.yml) runs at the end of a normal play
and again at the end of an upgrade. It is read-only, and it checks the
instance rather than the tasks that built it:

| Check | Catches |
| --- | --- |
| `gitlab-ctl status` has no `down:` service | Puma or Sidekiq killed by the OOM reaper, a service that will not start |
| The `VERSION` GitLab serves equals the installed package's | A package replaced while the reconfigure that applies it never finished |
| `/-/liveness` and `/-/readiness` answer 200 | GitLab up but unable to reach the database or Redis |
| `/users/sign_in` answers 200 and the page is GitLab's | nginx listening while Rails is broken, or something else on the port |
| `gitlab-rake db:abort_if_pending_migrations` | An interrupted reconfigure |
| `gitlab.rb` and `gitlab-secrets.json` are `0600` root | Credentials readable by anyone with a shell |
| The web port answers from the **controller** | A firewall rule, security group or route that leaves a healthy instance unreachable |

The HTTP checks dial the loopback interface with a `Host` header, so they do
not depend on DNS or on the certificate matching the address dialled — which
is why `gitlab_validate_http_validate_certs` defaults to `false` here, and
only here. They dial the scheme and port **nginx** listens on, not the ones in
the external URL, so they still work behind a load balancer that terminates
TLS; with `gitlab_nginx_enabled: false` there is nothing local to dial and they
are skipped.

12. Tags
--------

`preflight`, `os`, `repository`, `install`, `firewall`, `tls`, `config`,
`backup-schedule`, `validate`, `report`, plus two entry points that also carry
`never`, so they are unreachable unless asked for explicitly:

| Tag | Runs |
| --- | --- |
| `backup` | A backup of the running instance, and nothing else |
| `upgrade` | The upgrade path, and only when `gitlab_upgrade_enabled` is true |

```bash
# Re-render the configuration and apply it, nothing else.
ansible-playbook playbooks/gitlab.yml --tags config

# Check the instance without changing anything.
ansible-playbook playbooks/gitlab.yml --tags validate
```

13. Testing
-----------

One Molecule scenario, `default`: an Ubuntu 24.04 VirtualBox VM via Vagrant,
sized to GitLab's published minimum (4 vCPU, 8 GB, plus the swap file
`prepare.yml` creates — a scenario that lowered the role's requirements would
be testing a different role than the one anyone runs). 24.04 rather than
26.04, because the repository's noble suite carries GitLab's whole release
history, which is what the upgrade half of the test needs.

It covers install, backup and upgrade in one VM, against two **adjacent
releases resolved from the repository at run time** rather than two literal
versions that slowly rot:

| Step | What it does |
| --- | --- |
| `converge` | Installs the release below the newest one |
| `idempotence` | Proves a second configuration run does not reconfigure GitLab for nothing |
| `side_effect` | Takes an on-demand backup, then upgrades to the newest release |
| `verify` | Checks the result, and that two guards still refuse |

`verify.yml` reads everything back off the host: the installed package and the
version GitLab serves, every service running, the sign-in page and
`/-/readiness`, each configured setting present in `gitlab.rb` (a string, a
number, a boolean, a derived default and the raw-Ruby escape hatch), `0600`
credentials, both backups from both halves on disk, both cron files, the dpkg
hold — and then makes the role fail on purpose twice: the hardware preflight
asked for more memory than the VM has, and the upgrade path asked to move
GitLab backwards. A check that only ever passes proves nothing about whether
it would fail.

```bash
cd roles/gitlab
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml

# Works around molecule-plugins not registering its custom vagrant module
# with newer molecule releases.
export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

molecule test
```

Expect it to take a while: two GitLab reconfigures and a set of migrations on
one VM.

14. Troubleshooting
-------------------

```bash
# Why is a service down?
sudo gitlab-ctl status
sudo gitlab-ctl tail puma

# What does GitLab think of itself?
sudo gitlab-rake gitlab:check SANITIZE=true
sudo gitlab-rake gitlab:env:info

# What is actually configured? (This file is rendered by the role — change
# the variables, not the file.)
sudo cat /etc/gitlab/gitlab.rb

# Which release is installed, and which one is serving?
dpkg-query -W gitlab-ce
sudo cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

**The first run takes forever or dies.** Almost always memory. Check
`dmesg | grep -i oom`; the preflight minimums exist because of this.

**`502` from nginx after an upgrade.** Puma is not up. `gitlab-ctl tail puma`,
and check for pending migrations with
`sudo gitlab-rake db:abort_if_pending_migrations`.

**The role says an upgrade is pending and does nothing.** By design — see
section 9.

**`apt` refuses to upgrade `gitlab-ce`.** Also by design: the package is held
(`gitlab_package_hold`). The upgrade path lifts the hold for as long as it
needs it.
