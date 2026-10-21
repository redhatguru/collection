gitlab_runner
=============

Installs [GitLab Runner](https://docs.gitlab.com/runner/) and the Docker
engine on one Ubuntu host, configures the docker executor, and registers the
runner to a GitLab instance, group or project.

What it delivers, in one run, with no manual step afterwards:

- A **preflight** that refuses a host that cannot carry a runner, and a GitLab
  URL it cannot reach
- **Docker CE** from Docker's own repository, with log rotation configured —
  the engine's default is none, and a long job's output otherwise fills the
  disk
- GitLab Runner at a **pinned release**, from the official repository, without
  piping a downloaded script into a shell
- `/etc/gitlab-runner/config.toml` **rendered from role variables**, so the
  runner's whole configuration is one reviewable file
- The **runner created through GitLab's API**, which is where its
  authentication token comes from — see [section 4](#4-the-token)
- The **certificate of a GitLab no public CA signed for** installed where the
  runner looks for it — supplied, or fetched with `openssl` when you ask for
  that — so both the runner and the git inside its jobs trust it
- **Verification**: GitLab itself confirms the runner is registered, a
  throwaway container proves a job could actually start, and both systemd
  units are checked to be enabled so the host comes back as a working runner
  after a reboot

One runner per host. A host with several `[[runners]]` sections is a different
shape of deployment and deliberately out of scope — run the role against more
hosts, or raise `gitlab_runner_concurrent` to run more jobs on one.

Contents
--------

1. [Requirements](#1-requirements)
2. [Inventory](#2-inventory)
3. [Installation](#3-installation)
4. [The token](#4-the-token)
5. [Variables](#5-variables)
6. [Docker](#6-docker)
7. [A GitLab with a self-signed certificate](#7-a-gitlab-with-a-self-signed-certificate)
8. [Tags](#8-tags)
9. [Testing](#9-testing)
10. [Troubleshooting](#10-troubleshooting)

1. Requirements
---------------

**The host.** Ubuntu 22.04, 24.04 or 26.04, x86\_64 or ARM64, and:

| Resource | Minimum | Variable |
| --- | --- | --- |
| vCPU | 2 | `gitlab_runner_preflight_min_cpu` |
| Memory | 1900 MB | `gitlab_runner_preflight_min_memory_mb` |
| Free disk | 20480 MB on the filesystems holding `/var/lib/docker` and `/home/gitlab-runner` | `gitlab_runner_preflight_min_disk_free_mb` |

Those are a floor, not a recommendation: a runner's appetite is its jobs', and
`gitlab_runner_concurrent` decides how many run at once. Disk is the one that
bites — images, layers and build caches accumulate under `/var/lib/docker`
quietly until a job fails with *no space left on device*.

The preflight also refuses to continue when the executor is `docker` and
there is no engine (neither installed by the role nor already present), when
the scope is `group` or `project` with nothing naming it, when a supplied
token is not a `glrt-` token, and when GitLab's web port is unreachable from
this host — a runner that cannot reach GitLab registers nothing and takes no
jobs.

**The controller.** ansible-core >= 2.15 and the collections in
[`requirements.yml`](requirements.yml) (only the Molecule scenario needs them).

2. Inventory
------------

```yaml
all:
  children:
    gitlab_runners:
      hosts:
        runner-01:
          ansible_host: 10.11.0.60
```

Several hosts means several runners, each created separately in GitLab. That
is usually what you want: capacity comes from more hosts.

3. Installation
---------------

```yaml
# group_vars/gitlab_runners.yml
gitlab_runner_url: "https://gitlab.example.com"
gitlab_runner_version: "19.5.0-1"
gitlab_runner_scope: "group"
gitlab_runner_group: "platform/infra"
gitlab_runner_tags: ["docker"]
gitlab_runner_api_token: "{{ vault_gitlab_create_runner_token }}"
```

```bash
ansible-playbook -i inventories/lab-gitlab/hosts.yml playbooks/gitlab-runner.yml
```

Keep the runner on the same major.minor release as the GitLab it talks to —
that is the combination GitLab supports.

4. The token
------------

**Short answer: yes, the role fetches it from the API — but not the token you
may be thinking of.**

Registration tokens are gone. Until GitLab 15.x you read a *registration
token* off a group or project (`runners_token` in the API) and handed it to
`gitlab-runner register`. Those were deprecated in 15.6 and no longer work on
a current GitLab, so there is nothing to look up on a group or project any
more.

The current flow is the other way round: the runner is **created first**, and
creating it returns an **authentication token** (`glrt-…`) which is the only
credential the runner ever needs. This role does that creation for you:

```
POST /api/v4/user/runners
  runner_type=group_type  group_id=<resolved from the path>
  description=…  tag_list=…  run_untagged=…  access_level=…
→ { "id": 9171, "token": "glrt-…" }
```

What you supply is `gitlab_runner_api_token`: a **personal or group access
token** (`glpat-…`), held by someone with permission on the target. Keep it in
Ansible Vault. It is used once, for that one call, and is never written to the
host.

Its scopes depend on how you name the target:

| You set | Scopes the token needs | Why |
| --- | --- | --- |
| `gitlab_runner_group: 77` (numeric ID) | `create_runner` | The create call takes the ID as it is |
| `gitlab_runner_group: "platform/infra"` (path) | `create_runner` **and** `read_api` | Resolving a path is an API read, which `create_runner` does not grant |

Do not confuse it with the `glrt-…` token GitLab shows you when you create a
runner in its UI. That is a runner authentication token: it cannot call the
API at all, and it belongs in `gitlab_runner_authentication_token`, which
skips the API entirely. Putting it in `gitlab_runner_api_token` produces a
401, and the preflight checks refuse it with that explanation.

The role takes the token from the first of these it finds:

| | Source | When |
| --- | --- | --- |
| 1 | The token already in `config.toml` on the host | Always, on a host this role (or `gitlab-runner register`) has already set up — which is what makes a second run change nothing and never create a duplicate runner in GitLab |
| 2 | `gitlab_runner_authentication_token` | A runner you created in the UI yourself |
| 3 | `gitlab_runner_api_token` | The role creates the runner and reads the token out of the answer |

`gitlab_runner_group` and `gitlab_runner_project` take either a numeric ID or
a full path (`platform/infra`); a path is resolved to its ID first, because
the create call takes an integer.

Tags, `run_untagged`, `locked`, `access_level`, `paused` and
`maximum_timeout` are properties **GitLab** holds, so the role sets them when
it creates the runner. Changing them afterwards is a job for GitLab's UI or
API — the role will not delete and recreate a runner to move a tag.

5. Variables
------------

Full list with comments: [`defaults/main.yml`](defaults/main.yml). Derived
values (package names, paths, the API call's shape) are in [`vars/`](vars/)
and are deliberately not overridable.

### The instance and the runner's identity

| Variable | Default | Purpose |
| --- | --- | --- |
| `gitlab_runner_url` | `""` | Base URL of the GitLab instance. Required |
| `gitlab_runner_ca_certificate_fetch` | `false` | Fetch GitLab's certificate with openssl when the host's trust store cannot verify it — see [section 7](#7-a-gitlab-with-a-self-signed-certificate) |
| `gitlab_runner_ca_certificate_fingerprint` | `""` | SHA-256 the fetched certificate must match |
| `gitlab_runner_ca_certificate_src` / `_content` | `""` | A certificate you supply instead of fetching |
| `gitlab_runner_version` | `""` | Package release, e.g. `19.5.0-1`. Empty installs the newest |
| `gitlab_runner_package_hold` | `true` | Hold the package in dpkg |
| `gitlab_runner_scope` | `instance` | `instance`, `group` or `project` |
| `gitlab_runner_group` / `_project` | `""` | Numeric ID or full path |
| `gitlab_runner_description` | the hostname | Name shown in GitLab |
| `gitlab_runner_tags` | `[]` | Tags a job must ask for; with `run_untagged` empty takes anything |
| `gitlab_runner_run_untagged` | `true` | |
| `gitlab_runner_locked` | `false` | Lock to its project |
| `gitlab_runner_access_level` | `not_protected` | `ref_protected` keeps the runner off unprotected branches |
| `gitlab_runner_maximum_timeout` | `0` | Seconds; 0 leaves GitLab's own limit |

### The runner's own configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `gitlab_runner_concurrent` | `4` | Jobs this host runs at once |
| `gitlab_runner_check_interval` | `3` | Seconds between asking for work |
| `gitlab_runner_executor` | `docker` | `docker` or `shell` |
| `gitlab_runner_limit` | `0` | Jobs this one runner takes; 0 is unlimited |
| `gitlab_runner_output_limit` | `4096` | Largest job log, in kilobytes |
| `gitlab_runner_environment` | `[]` | `"KEY=value"` for every job |
| `gitlab_runner_docker_image` | `alpine:3.21` | Image for jobs that name none |
| `gitlab_runner_docker_privileged` | `false` | **Read [section 6](#6-docker) before turning this on** |
| `gitlab_runner_docker_volumes` | `["/cache"]` | |
| `gitlab_runner_docker_pull_policy` | `["if-not-present"]` | |
| `gitlab_runner_docker_allowed_images` | `[]` | Restrict which images jobs may run |
| `gitlab_runner_cache_type` | `""` | `s3` with `gitlab_runner_cache_s3` for a shared cache |
| `gitlab_runner_*_extra_configuration` | `{}` | Escape hatches for `[[runners]]`, `[runners.docker]`, and the top of the file |
| `gitlab_runner_raw_configuration` | `""` | Raw TOML appended verbatim |

`config.toml` is rendered from these and nothing else — the file on the host
is disposable, and the task that writes it does not print its diff because it
holds the token. `gitlab-runner register` is deliberately not used: the token
is all a runner needs, and `gitlab-runner verify` afterwards is what proves
GitLab agrees.

### Docker engine

| Variable | Default | Purpose |
| --- | --- | --- |
| `gitlab_runner_docker_engine_install` | `true` | Install Docker CE from Docker's repository |
| `gitlab_runner_docker_engine_packages` | `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin` | |
| `gitlab_runner_docker_engine_users` | `[]` | Accounts added to the `docker` group |
| `gitlab_runner_docker_engine_daemon_configuration` | log rotation, `live-restore` | Rendered to `/etc/docker/daemon.json` |

6. Docker
---------

The engine comes from Docker's own repository rather than Ubuntu's
`docker.io`, because the docker executor follows the engine's releases closely
and that is the version GitLab tests against. `/etc/docker/daemon.json` is
rendered from `gitlab_runner_docker_engine_daemon_configuration`, which by
default sets log rotation — Docker's own default is none, and one chatty job
can fill the disk.

The runner service runs as `root` (that is how the package installs it), so it
reaches `/var/run/docker.sock` without being in the `docker` group. That is
why `gitlab_runner_docker_engine_users` is empty by default: membership of
that group is equivalent to root on the host, so add a human to it
deliberately or not at all.

**Privileged jobs.** `gitlab_runner_docker_privileged: true` is what
Docker-in-Docker needs, and it means any job on this runner can do anything to
the host — including reading every other job's secrets. If you need it, give
it a runner host of its own that you can rebuild, and keep that runner off
protected branches (`gitlab_runner_access_level: ref_protected` does the
opposite: it reserves the runner *for* protected refs, so use a separate
runner for each purpose rather than one for both).

7. A GitLab with a self-signed certificate
------------------------------------------

A runner against a GitLab whose certificate no public CA signed fails in the
least obvious way: every call is refused, and the runner looks installed but
registers nothing. The role handles it without a manual step:

```yaml
gitlab_runner_url: "https://gitlab.devlab.local"
gitlab_runner_ca_certificate_fetch: true
```

The certificate is fetched with `openssl s_client -showcerts`, so an internal
CA's intermediates come along rather than only the leaf, and installed at
`/etc/gitlab-runner/certs/<the URL's host>.crt` — the path the runner reads
**on every call**, so a renewed certificate needs no re-registration. The
runner also hands it to each job through `CI_SERVER_TLS_CA_FILE`, which is
what makes `git clone` inside a job work.

It only fetches when it has to. First it asks whether this host's own trust
store already verifies that GitLab; when it does, nothing is installed and
the message says so — an instance with a publicly signed certificate needs no
pinned file, and pinning one would break the day its chain changes.

### Why fetching is off by default

Asking a server for the certificate you will then trust it by is
**trust-on-first-use**. Whoever answers that first connection becomes
permanently trusted, and the access token that creates the runner travels
over it. That is a reasonable trade on a network you control and a bad one to
inherit without noticing, so the role asks for it rather than assuming it —
hence `gitlab_runner_ca_certificate_fetch: true` above.

With nothing pinned, the run prints the fingerprint it accepted, so there is
one place to notice what was trusted. Close the gap in one of two ways.

Pin the fingerprint, which turns the fetch into a verification:

```bash
# On the GitLab host, once:
sudo openssl x509 -in /etc/gitlab/ssl/gitlab.devlab.local.crt -noout -fingerprint -sha256
```

```yaml
gitlab_runner_ca_certificate_fingerprint: "AA:BB:CC:…"   # case and colons ignored
```

The fingerprint is read from what came over the wire **before anything is
written**, so a certificate that does not match leaves nothing behind on the
host — and nothing is registered either.

A pin also narrows *what* is installed: only the certificate you named
becomes a trust anchor. That is the difference between a pin that means
something and one that does not. A leaf certificate is public — it is sent in
every handshake — so anything in the middle could present GitLab's real leaf
next to a certificate of its own, pass a leaf-only pin, and have its own
certificate installed as an anchor it can sign anything with. Nothing
unpinned is installed, so that does not work.

Which certificate to pin follows from how GitLab's was issued:

| GitLab's certificate | Pin | Why |
| --- | --- | --- |
| Self-signed (what the [`gitlab`](../gitlab) role generates) | that certificate | It is its own anchor |
| Issued by your own CA | **the CA**, not the leaf | The server sends its intermediates in every handshake, so the CA alone verifies the leaf — and a leaf installed on its own is not an anchor OpenSSL-based clients will build a chain from |

The role does not leave you to work that out from a TLS error: right after
installing, it verifies GitLab's endpoint against exactly what it installed,
and says to pin the CA instead if that is the problem.

Without a pin the whole presented chain is installed, which is the strongest
argument for setting one.

Or copy the certificate out of band and skip the fetch entirely:

```yaml
gitlab_runner_ca_certificate_src: "files/gitlab.devlab.local.crt"
# or
# gitlab_runner_ca_certificate_content: "{{ lookup('file', 'gitlab.crt') }}"
```

Either of those takes precedence over fetching, and
`gitlab_runner_ca_certificate_fetch: false` refuses it outright. A CA file you
manage yourself elsewhere on the host goes in `gitlab_runner_tls_ca_file`.

Whichever route, `gitlab_runner_validate_check_certificate` (default true)
afterwards verifies GitLab's endpoint against exactly the file the runner
will use — which is what catches a fetch that picked up a captive portal's
certificate, or an out-of-band copy of the wrong host's.

The one API call the role makes is verified against that same certificate, so
there is no need to turn certificate verification off anywhere. If you do turn
it off (`gitlab_runner_api_validate_certs: false`), you are trusting whatever
answers that address for the one call that hands over an access token — pin
the fingerprint instead.

With a GitLab built by this collection's [`gitlab`](../gitlab) role and
`gitlab_tls_certificate_source: self-signed`, the certificate is at
`/etc/gitlab/ssl/<host>.crt` on the GitLab host, which is where the
fingerprint above comes from.

8. Tags
-------

`preflight`, `os`, `repository`, `docker`, `install`, `certificate`, `token`,
`config`, `service`, `validate`, `report`.

`gitlab_runner_service_enabled` (default `true`) keeps the runner's and
Docker's systemd units enabled rather than merely running, and
`gitlab_runner_validate_check_boot` checks that rather than assuming it — a
unit that is running but not enabled looks identical until the host
restarts.

```bash
# Re-render the configuration and restart the runner, nothing else.
ansible-playbook playbooks/gitlab-runner.yml --tags config

# Check an existing runner without changing anything.
ansible-playbook playbooks/gitlab-runner.yml --tags validate
```

9. Testing
----------

One Molecule scenario, `default`, with **two** VMs: a GitLab built by the
[`gitlab`](../gitlab) role and a runner registered against it. Registration
is the whole point of this role and cannot be tested without a real GitLab —
a mocked API would prove only that the role can talk to a mock.

The GitLab is deliberately built with https and a self-signed certificate,
because that is where a runner fails in the least obvious way. The runner
**fetches that certificate itself**, and the scenario pins the fingerprint it
reads off the GitLab host — so the test proves the runner picked up exactly
the right certificate rather than whatever answered. The access token is
minted on the GitLab host too, so nothing is pre-seeded:

| Step | What it does |
| --- | --- |
| `converge` | Builds GitLab, mints a `create_runner` token on it, then installs and registers the runner |
| `idempotence` | Proves a second run changes nothing — and, with the check below, that it creates no second runner |
| `verify` | Checks both hosts |

`verify.yml` reads everything back: the service, `gitlab-runner verify`
(GitLab confirming the token), the rendered `config.toml` including
`tls-ca-file`, `0600` credentials, the certificate reaching GitLab's API, a
container actually running — and then asks **GitLab** what it knows: exactly
one runner with this description (two would mean the role created a new runner
on its second run), its type, that it reports itself online, and that the tags
the role asked for are the tags GitLab holds.

```bash
cd roles/gitlab_runner
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml
export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

molecule test
```

It costs what a GitLab install costs: reckon on an hour, and 10 GB of RAM for
the two VMs.

10. Troubleshooting
-------------------

```bash
# Is it registered, and does GitLab still accept the token?
sudo gitlab-runner verify

# What is it actually configured with? (Rendered by the role — change the
# variables, not the file.)
sudo cat /etc/gitlab-runner/config.toml

# Why is it not taking jobs?
sudo journalctl -u gitlab-runner -n 100 --no-pager

# Can the host run a job container at all?
sudo docker run --rm alpine:3.21 /bin/true
```

**`x509: certificate signed by unknown authority`.** The runner does not trust
GitLab's certificate — see [section 7](#7-a-gitlab-with-a-self-signed-certificate).
The same error inside a job means the job's git does not trust it either,
which the same certificate fixes.

**`ERROR: Verifying runner... is removed`.** The runner was deleted in GitLab,
so the token in `config.toml` belongs to nothing. Remove that file and re-run
with `gitlab_runner_api_token` set; the role will create a fresh runner.

**Jobs stay queued.** The runner is online but no job matches it: check the
tags GitLab holds against the tags the job asks for, and whether
`run_untagged` is true.

**`no space left on device`.** `/var/lib/docker`. `docker system prune -af`
clears it for now; the preflight's disk minimum exists because of this.
