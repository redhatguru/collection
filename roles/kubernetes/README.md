kubernetes
=========

Installs Kubernetes via `kubeadm` (containerd runtime, Calico CNI) on Rocky
Linux 9/10 or Ubuntu 24.04/26.04, as either a single all-in-one node or a
multi-node cluster, and installs the Kubernetes Dashboard, the
ingress-nginx Ingress controller, and some cluster hardening on top. At the
end of the run it prints a report of every node's status plus the restart
count of each control-plane component (`kube-system`, `tier=control-plane`)
— a quick way to spot etcd/apiserver instability, which on
underpowered/shared storage is usually a disk-latency problem (see
`kubernetes_etcd_heartbeat_interval` below) rather than anything this role
misconfigured. That tuning is reconciled into the running etcd/kube
-apiserver static pod manifests on every run (not just applied once at
`kubeadm init`), so an existing cluster picks up a changed value too.

Which mode a host gets is driven entirely by the inventory groups it's a
member of:
- `single` — the host gets its own independent, all-in-one cluster
  (control-plane taint removed so it can run workloads too).
- `masters` — the first host in this group runs `kubeadm init` and becomes
  the control-plane endpoint (its own address, unless
  `kubernetes_control_plane_endpoint` is overridden); any further hosts in
  the group join it as additional control-plane nodes.
- `workers` — hosts in this group join the cluster formed by the `masters`
  group as workers.

A host that isn't in any of these three groups is left untouched by this
role. `single` is independent of `masters`/`workers` — don't mix a host
into both.

Requirements
------------

- A Rocky Linux 9/10 or Ubuntu 24.04/26.04 host, reachable with
  `become: true` and outbound internet access (container images, the
  `pkgs.k8s.io` and `download.docker.com` package repos, the Calico and
  Dashboard manifests are all fetched from the internet).
- At least 2 CPUs and 2GB RAM per node (kubeadm's own minimum).
- For a `masters`/`workers` cluster, all nodes need to be able to reach
  each other, and the workers/additional masters need to be able to reach
  the first master on port 6443.

### Collections

This role needs the following collections on the control node (already
declared in the collection's own `requirements.yml`/`galaxy.yml`, and in
this role's own `requirements.yml`):
- `ansible.posix` (`sysctl`)
- `community.general` (`modprobe`)

Install them with:
```shell
ansible-galaxy collection install -r requirements.yml
```

Role Variables
--------------

- `kubernetes_version` (default `"1.30"`) — Kubernetes minor version; selects
  the `pkgs.k8s.io` package repository kubelet/kubeadm/kubectl are installed
  from.
- `kubernetes_node_ip` (default: the address of the node's default route) —
  address this node advertises itself on, and the one other nodes join it
  through. The default is correct for a normal single-NIC server; override
  it per-host on multi-NIC hosts where the default route isn't the
  interface other cluster nodes should reach this one on.
- `kubernetes_pod_network_cidr` (default `"192.168.0.0/16"`) — pod network
  CIDR passed to kubeadm and to the Calico manifest.
- `kubernetes_calico_version` (default `"3.28.0"`) — Calico release to
  install as the CNI.
- `kubernetes_control_plane_endpoint` — host/IP the control plane is
  reachable on. Defaults to the first `masters` host's address, so a
  cluster works without a separate load balancer in front of it; only
  relevant to the `masters`/`workers` mode.
- `kubernetes_hold_packages` (default `true`) — pin kubelet/kubeadm/kubectl
  (`dnf versionlock` / `apt-mark hold`) so a generic OS package update
  can't bump them out from under the cluster.
- `kubernetes_etcd_heartbeat_interval` / `kubernetes_etcd_election_timeout`
  (default `"250"` / `"2500"`, ms) — etcd's tolerance for slow disk fsync
  latency, applied via kubeadm's `ClusterConfiguration` at init time (and
  inherited by joining control-plane nodes). etcd's own defaults (100/1000)
  are tuned for fast local disks; on underpowered/shared storage, requests
  routinely exceed them, causing spurious leader elections and
  kube-apiserver/etcd instability. Only lower these if the underlying disk
  is genuinely fast.
- `kubernetes_apiserver_etcd_timeout` (default `"5s"`) — kube-apiserver's
  `--etcd-healthcheck-timeout`/`--etcd-readycheck-timeout` (both default
  `2s` upstream); how long it waits on an etcd health/readiness check
  before giving up.
- `kubernetes_wait_timeout` (default `600`, seconds) — how long to wait for
  control-plane/Dashboard/ingress-nginx readiness and `kubeadm init`/`join`/
  `kubectl drain` to complete. Generous by default for the same slow-disk
  reason as the etcd settings above.
- `kubernetes_worker_join_command` / `kubernetes_control_plane_join_command`
  — computed by the role on the first master/single node; do not set these
  yourself.
- `kubernetes_dashboard_enabled` (default `true`) — install the Kubernetes
  Dashboard.
- `kubernetes_dashboard_version` (default `"2.7.0"`) — Dashboard release to install.
- `kubernetes_dashboard_admin_user` (default `true`) — the dashboard's
  service account is bound to `cluster-admin`. Set to `false` to bind it to
  the built-in read-only `view` ClusterRole instead.
- `kubernetes_ingress_enabled` (default `true`) — install the ingress-nginx
  Ingress controller, so other roles (e.g. `awx`) can expose services on a
  real hostname instead of a raw NodePort.
- `kubernetes_ingress_version` (default `"1.11.3"`) — ingress-nginx release
  to install.
- `kubernetes_ingress_http_node_port` / `kubernetes_ingress_https_node_port`
  (default `30880` / `30943`) — fixed NodePorts the ingress-nginx
  controller's Service is patched to use, so they're predictable enough to
  open on the firewall and to point DNS records at.
- `kubernetes_hardening_enabled` (default `true`) — master switch for all
  hardening below.
- `kubernetes_harden_kubelet` (default `true`) — disables kubelet anonymous
  authentication and the kubelet read-only port. Baked into the cluster at
  `kubeadm init` time (via a `KubeletConfiguration`), so it applies to
  every node that joins, not just the one that ran `init`. Deliberately
  does *not* set `protectKernelDefaults: true` — it makes kubelet refuse to
  start unless the host's kernel sysctls exactly match Kubernetes'
  hardcoded expectations, which varies enough across base images to be
  unreliable as a default.
- `kubernetes_harden_network_policy` (default `true`) — applies a
  default-deny-all `NetworkPolicy` in the `default` namespace.
- `kubernetes_upgrade_version` (default: `kubernetes_version`) — target
  minor version for a rolling upgrade. Only used by the `upgrade` tag; see
  "Rolling Upgrade" below.

Not covered by `kubernetes_harden_*` yet: API server flags (audit logging,
anonymous auth) aren't hardened, since that means hand-editing kubeadm's
static pod manifest, which is too fragile to do reliably across kubeadm
versions. Something to revisit if this role grows a dedicated variant per
supported kubeadm version.

Example Playbook
----------------

Single all-in-one node:
```yaml
---
- name: Install a single-node Kubernetes cluster
  hosts: single
  become: true

  tasks:
    - name: Kubernetes
      ansible.builtin.include_role:
        name: kubernetes
```

Multi-node cluster, using inventory groups `masters` and `workers`:
```yaml
---
- name: Install a Kubernetes cluster
  hosts: masters:workers
  become: true
  vars:
    kubernetes_dashboard_admin_user: false
    kubernetes_harden_network_policy: true

  tasks:
    - name: Kubernetes
      ansible.builtin.include_role:
        name: kubernetes
```

with an inventory like:
```ini
[masters]
k8s-master-1
k8s-master-2

[workers]
k8s-worker-1
k8s-worker-2
k8s-worker-3
```

Rolling Upgrade
----------------

Upgrading kubelet/kubeadm/kubectl to a new minor version is opt-in only —
the upgrade tasks are tagged `[upgrade, never]`, so they never run as part
of a normal play, only when explicitly requested with `--tags upgrade`.
For a single host this cordons and drains the node, upgrades the
`pkgs.k8s.io` repo and packages to `kubernetes_upgrade_version`, runs
`kubeadm upgrade apply` (on the primary control-plane node/`single`) or
`kubeadm upgrade node` (everywhere else), restarts kubelet, then
uncordons the node.

To upgrade a whole cluster as a genuine *rolling* upgrade — one node at a
time, keeping the cluster available throughout — add `serial: 1` to the
play, and make sure the primary master (`groups['masters'][0]`) is
upgraded first, since `kubeadm upgrade apply` has to run there before any
other node can `kubeadm upgrade node`:

```yaml
---
- name: Rolling-upgrade the Kubernetes cluster
  hosts: masters:workers
  become: true
  serial: 1
  vars:
    kubernetes_upgrade_version: "1.31"

  tasks:
    - name: Kubernetes
      ansible.builtin.include_role:
        name: kubernetes
      tags: upgrade
```

`ansible-playbook` preserves inventory order within `hosts: masters:workers`,
so as long as the primary master is listed first under `[masters]` in your
inventory, `serial: 1` upgrades it first, then the rest of `masters`, then
`workers`, one at a time. After a successful upgrade, update your own
`kubernetes_version` to match `kubernetes_upgrade_version` so future runs
(without `--tags upgrade`) don't try to reconcile packages back to the old
version.

Testing
-------

This role has three Molecule scenarios:
- `default` — installs a `single` all-in-one cluster on Rocky Linux 9,
  Rocky Linux 10, Ubuntu 24.04 and Ubuntu 26.04 (one independent cluster
  per VM).
- `cluster-ubuntu` — a `masters`/`workers` cluster (2 masters, 3 workers)
  on Ubuntu 24.04.
- `cluster-rockylinux` — the same 2 masters/3 workers topology, on Rocky
  Linux 10.

All of them run in VirtualBox VMs via Vagrant, closely mirroring real
servers:

```shell
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml

# Works around molecule-plugins not registering its custom vagrant module
# with newer molecule releases.
export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

molecule test                          # default scenario
molecule test -s cluster-ubuntu
molecule test -s cluster-rockylinux
```

Requires Vagrant and VirtualBox installed locally. The cluster scenarios
bring up 5 VMs each, so expect them to take considerably longer than the
`default` scenario.

License
-------

license (BSD, MIT)

Author Information
------------------

for questions, Contact guido-_@live.nl
