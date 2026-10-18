k3s
===

Installs, configures and hardens a [K3s](https://k3s.io) Kubernetes cluster.
One role covers every topology — single node, a single control plane with
workers, or a highly available control plane — because the topology is derived
from the Ansible inventory rather than selected by a variable.

What it delivers, in one run, with no manual step afterwards:

- OS preparation (packages, kernel modules, sysctl, time sync, `/etc/hosts`)
- K3s, from a checksum-verified binary, configured through
  `/etc/rancher/k3s/config.yaml`
- A host firewall that **stays enabled**, with rules that depend on the node's
  role in the cluster
- CIS-style cluster hardening: Secret encryption at rest, API audit logging,
  Pod Security Admission, root-only credentials, `protect-kernel-defaults`
- [MetalLB](https://metallb.universe.tf) for LoadBalancer addresses
- [Traefik](https://traefik.io) as the ingress controller, highly available on
  a multi-node cluster and a single replica on one node
- Optional [cert-manager](https://cert-manager.io) with self-signed, internal
  CA and ACME issuers
- Scheduled etcd snapshots
- Post-install validation, and an opt-in rolling upgrade

Applications are **not** this role's job — that is
[`k3s_app`](../k3s_app), so applications can be added without
touching the cluster.

Contents
--------

1. [Architecture](#1-architecture)
2. [Requirements](#2-requirements)
3. [Supported operating systems](#3-supported-operating-systems)
4. [Inventory](#4-inventory)
5. [Variables](#5-variables)
6. [Single-node installation](#6-single-node-installation)
7. [HA installation](#7-ha-installation)
8. [Worker nodes](#8-worker-nodes)
9. [MetalLB](#9-metallb)
10. [Traefik](#10-traefik)
11. [DNS](#11-dns)
12. [TLS](#12-tls)
13. [Applications](#13-applications)
14. [Helm](#14-helm)
15. [Firewall](#15-firewall)
16. [Hardening](#16-hardening)
17. [Storage](#17-storage)
18. [Backup](#18-backup)
19. [Upgrade](#19-upgrade)
20. [Troubleshooting](#20-troubleshooting)
21. [Disaster recovery](#21-disaster-recovery)

Deeper background lives in [docs/architecture.md](docs/architecture.md) (why
K3s, the full port matrix, the trade-offs taken) and
[docs/security.md](docs/security.md) (what is hardened, what is deliberately
not, and every exemption).

1. Architecture
---------------

```text
                            DNS
       app1.example.com ──┐
       app2.example.com ──┼──► 10.10.50.100   (one shared address)
     grafana.example.com ─┘          │
                                     ▼
                       MetalLB (layer 2, announced by one node,
                                failed over to another if it dies)
                                     │
                                     ▼
                            Traefik  (3 replicas, one per node,
                                      PodDisruptionBudget, probes)
                                     │
                   ┌─────────────────┼─────────────────┐
                   ▼                 ▼                 ▼
                k8s-01            k8s-02            k8s-03     control plane
             server + etcd     server + etcd     server + etcd  (quorum of 3)
                   └─────────────────┼─────────────────┘
                                     ▼
                              k8s-04 … k8s-NN                   workers
```

On a single node the same picture collapses without any configuration change:
MetalLB announces the address from the one node it has, Traefik runs one
replica without a disruption budget, and etcd runs as a single member.

2. Requirements
---------------

- `ansible-core` >= 2.15 on the controller
- Collections: `ansible.posix`, `community.general`, `kubernetes.core`
  (`ansible-galaxy collection install -r requirements.yml`)
- Each node reachable with `become: true`
- Outbound HTTPS from each node to `github.com`, `update.k3s.io`,
  `get.k3s.io`, `get.helm.sh`, `rpm.rancher.io` (SELinux policy on RHEL),
  the chart repositories, and the container registries
- Minimum per node (adjustable, see `k3s_preflight_*`): 2 vCPU,
  1536 MB RAM, 10 GB free under `/var`
- Free addresses on the nodes' own layer-2 segment for MetalLB, outside the
  DHCP range

The play must target **all** cluster nodes at once. The firewall rules and the
API registration address are derived from the other nodes' facts, so a run with
`--limit` is rejected by the preflight checks rather than silently producing a
half-configured firewall.

3. Supported operating systems
------------------------------

| Family | Releases | Firewall | Package manager |
| --- | --- | --- | --- |
| Debian | Ubuntu 22.04, 24.04 and newer; Debian 12+ | `ufw` | `apt` |
| RedHat | RHEL 9+, Rocky Linux 9+, AlmaLinux 9+ | `firewalld` | `dnf` |

Architectures: `x86_64` and `aarch64` (ARM64). The architecture only changes
which release asset is downloaded; everything else is identical.

The OS family is detected from `ansible_facts['os_family']`, and
`vars/Debian.yml` / `vars/RedHat.yml` supply the distribution-specific package
names, firewall backend and service names. Anything unsupported is rejected in
preflight with the reason, rather than failing halfway through an install.

4. Inventory
------------

Two groups decide everything. Their names are configurable
(`k3s_control_plane_group`, `k3s_worker_group`) so an existing
inventory only has to map onto them.

```yaml
all:
  children:
    k8s_control_plane:
      hosts:
        k8s-01: {ansible_host: 10.10.50.11}
        k8s-02: {ansible_host: 10.10.50.12}
        k8s-03: {ansible_host: 10.10.50.13}
    k8s_workers:
      hosts:
        k8s-04: {ansible_host: 10.10.50.14}
```

The derived topology:

| Control plane | Workers | `k3s_topology` | Result |
| --- | --- | --- | --- |
| 1 | 0 | `single-node` | Everything on one host, Traefik ×1 |
| 1 | 1+ | `single-server` | One API server, workloads on the workers |
| 3, 5, 7… | 0+ | `ha` | Embedded etcd quorum, Traefik spread over nodes |

Rejected by `assert`, with the reason:

- **2 control-plane nodes** — the etcd quorum becomes 2, so losing either node
  stops the cluster. Strictly worse than one node. Use 1 or 3.
- **An even number of control-plane nodes** — same problem.
- **A host in both groups** — puts unprivileged workloads next to etcd. Allow
  it deliberately with `k3s_allow_control_plane_as_worker: true`.
- **Duplicate node names or addresses** — two nodes would fight over one Node
  object.
- **An HA topology with `k3s_cluster_init: false`** — the sqlite
  backend cannot be shared.
- **A cluster node missing from the play** — see [Requirements](#2-requirements).
- **MetalLB enabled with no address pool** — there is no safe default for
  addresses on someone else's network, so the role will not guess.
- **MetalLB and K3s' ServiceLB both enabled** — they would fight over every
  LoadBalancer service.
- **Two default StorageClasses** — which class a PVC without an explicit
  `storageClassName` gets would become unpredictable.

Ready-made examples: [`inventories/lab-single-node`](../../inventories/lab-single-node)
and [`inventories/lab-ha`](../../inventories/lab-ha).

5. Variables
------------

Full list with comments: [`defaults/main.yml`](defaults/main.yml). Values the
role derives itself (topology, port matrix, package names) are in
[`vars/`](vars/) and are deliberately not overridable — overriding a derived
topology would let the preflight checks pass while the cluster is built from
different facts.

Every variable is prefixed `k3s_`. That is not cosmetic: `ansible-lint`'s
production profile fails a role that defines variables without its own name as
the prefix.

### Topology and identity

| Variable | Default | Purpose |
| --- | --- | --- |
| `k3s_control_plane_group` | `k8s_control_plane` | Inventory group holding the control-plane nodes |
| `k3s_worker_group` | `k8s_workers` | Inventory group holding the workers |
| `k3s_init_host` | first control-plane host | Node that bootstraps the cluster and runs all cluster-wide API work |
| `k3s_allow_control_plane_as_worker` | `false` | Accept a host in both groups |
| `k3s_taint_control_plane` | `false` | `CriticalAddonsOnly=true:NoExecute`, keeping workloads off the control plane |
| `k3s_node_name` | the host's hostname | Name this node registers under |
| `k3s_node_ip` | default-route address | Address the cluster reaches this node on |
| `k3s_node_external_ip` | `""` | Public address when it differs (NAT) |
| `k3s_api_registration_address` | bootstrap node's address | Endpoint nodes join through; point at a VIP for a fully HA endpoint |
| `k3s_api_extra_tls_san` | `[]` | Extra names on the API server certificate |

### K3s

| Variable | Default | Purpose |
| --- | --- | --- |
| `k3s_version` | `""` | Exact release, e.g. `v1.31.5+k3s1`. Empty resolves `k3s_channel` at run time |
| `k3s_channel` | `stable` | Release channel used when no version is pinned |
| `k3s_token` | `""` | Shared cluster secret — supply from Ansible Vault |
| `k3s_cluster_init` | `true` | Embedded etcd rather than sqlite |
| `k3s_cluster_cidr` | `10.42.0.0/16` | Pod network |
| `k3s_service_cidr` | `10.43.0.0/16` | Service network |
| `k3s_flannel_backend` | `vxlan` | `vxlan`, `host-gw`, `wireguard-native` or `none` |
| `k3s_write_kubeconfig_mode` | `"0600"` | Root-only kubeconfig on the node |
| `k3s_extra_config` | `{}` | Merged verbatim into `config.yaml` for options this role does not model |

### Components

| Variable | Default |
| --- | --- |
| `k3s_metallb_enabled` | `true` |
| `k3s_traefik_enabled` | `true` |
| `k3s_cert_manager_enabled` | `false` |
| `k3s_metrics_server_enabled` | `true` |
| `k3s_storage_local_path_enabled` | `true` |
| `k3s_servicelb_enabled` | `false` |
| `k3s_etcd_backup_enabled` | `true` |
| `k3s_network_policy_enabled` | `true` |
| `k3s_helm_enabled` | `true` |

K3s' bundled Traefik is **always** disabled, regardless of
`k3s_traefik_enabled`: K3s deploys it through its own HelmChart addon,
which cannot be version-pinned or spread across nodes the way the upstream
chart can. This role installs Traefik with Helm instead.

### Tags

`preflight`, `os`, `hardening`, `firewall`, `k3s`, `cluster`, `kubeconfig`,
`helm`, `storage`, `metallb`, `traefik`, `cert-manager`, `network-policies`,
`etcd-backup`, `validate`, `report`, and `upgrade` (which also carries `never`,
so it is unreachable unless asked for explicitly).

6. Single-node installation
---------------------------

```yaml
# inventories/lab/hosts.yml
all:
  children:
    k8s_control_plane:
      hosts:
        k8s-01: {ansible_host: 10.10.50.10}
    k8s_workers:
      hosts: {}
```

```yaml
# inventories/lab/group_vars/all.yml
k3s_metallb_address_pools:
  - name: default
    addresses: ["10.10.50.100-10.10.50.110"]
k3s_traefik_loadbalancer_ip: "10.10.50.100"
k3s_firewall_api_sources: ["10.10.0.0/16"]
```

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventories/lab/hosts.yml playbooks/k3s.yml
```

Afterwards, on the node:

```bash
kubectl get nodes          # ~/.kube/config is installed for the connecting user
sudo k3s kubectl get pods -A
```

7. HA installation
------------------

Add two more control-plane hosts and nothing else:

```yaml
k8s_control_plane:
  hosts:
    k8s-01: {ansible_host: 10.10.50.11}
    k8s-02: {ansible_host: 10.10.50.12}
    k8s-03: {ansible_host: 10.10.50.13}
```

The role then:

- bootstraps `k8s-01` with `cluster-init: true`
- joins `k8s-02` and `k8s-03` **one at a time** — adding several etcd members
  at once to a freshly initialised cluster races on the member list. The K3s
  unit is `Type=notify`, so systemd does not report a start complete until that
  node's API server is serving, which makes the next node's turn safe
- puts every control-plane name and address on the API server certificate
- runs Traefik with three replicas, pod anti-affinity, topology spread
  constraints and a PodDisruptionBudget

**One honest limitation.** `k3s_api_registration_address` defaults to
the bootstrap node, which is a single point of failure *for joining new nodes*
— running nodes keep working without it. For an endpoint that survives losing
that node, put a VIP or an external load balancer in front of all three and set:

```yaml
k3s_api_registration_address: "10.10.50.10"   # the VIP
```

The role adds it to the certificate automatically.

8. Worker nodes
---------------

Workers are agents: no API server, no etcd, and a firewall that reflects that
(no 6443, no 2379/2380). Add them to `k8s_workers` and re-run the playbook;
existing nodes are untouched.

To keep application workloads off the control plane:

```yaml
k3s_taint_control_plane: true
```

The role adds the matching tolerations to MetalLB, Traefik and cert-manager, so
turning this on cannot strand the add-ons. Do bear in mind what it means for
the ingress: with the control plane tainted, Traefik can only run on the
workers, so **HA ingress needs at least two or three workers**. With a single
worker, leave the taint off.

9. MetalLB
----------

```yaml
k3s_metallb_enabled: true
k3s_metallb_mode: layer2
k3s_metallb_address_pools:
  - name: default
    addresses:
      - 10.10.50.100-10.10.50.110
  - name: dedicated
    addresses:
      - 10.10.50.120/29
    auto_assign: false      # only handed out when asked for by name
```

The addresses must be free on the nodes' own layer-2 segment and outside the
DHCP range. Nothing has to be configured on the network.

Layer 2 is **highly available but not load balanced**: one node answers ARP for
a given address at a time, and another takes over if it fails. All ingress
traffic therefore enters through a single node. For real load distribution, use
BGP:

```yaml
k3s_metallb_mode: bgp
k3s_metallb_bgp_peers:
  - name: core
    my_asn: 64513
    peer_asn: 64512
    peer_address: 10.10.0.1
```

Pools and advertisements are written into K3s' auto-deploying manifests
directory, so they are reconciled on every K3s start and removing them from the
inventory removes them from the cluster.

`k3s_metallb_frr_enabled` is `false`, and stated explicitly in the chart values
rather than left to the chart. The chart defaults it to **true**, which deploys
an extra `frr-k8s` DaemonSet and status cleaner and pulls two more container
images — for BGP machinery that layer-2 mode never touches and that MetalLB's
own native BGP speaker does not need either. Turn it on only for FRR-specific
features such as BFD or multi-protocol BGP.

10. Traefik
-----------

Installed with Helm from the upstream chart. Highly available by default on a
multi-node cluster and a single replica on one node, without configuration:

```yaml
k3s_traefik_replicas: ""        # one per schedulable node, capped at 3
k3s_traefik_max_replicas: 3
k3s_traefik_pod_anti_affinity: true
k3s_traefik_topology_spread: true
k3s_traefik_pdb_min_available: 1
```

The disruption budget is only created when there is more than one replica — a
1-of-1 budget would block every node drain, upgrades included. Anti-affinity is
*preferred* rather than required, so more replicas than nodes does not leave
pods permanently `Pending`.

Other defaults worth knowing:

- HTTP redirects to HTTPS (`k3s_traefik_http_redirect_enabled`)
- TLS 1.2 minimum (`k3s_traefik_tls_min_version`)
- The dashboard and the insecure API are off
- `allowCrossNamespace: false` — an IngressRoute cannot point at a Service in
  another namespace
- `globalArguments: []` — the chart's defaults phone home for version checks
  and usage statistics
- Non-root, read-only root filesystem, all capabilities dropped
- `externalTrafficPolicy: Cluster`. `Local` preserves the client address but
  only works while a Traefik pod runs on the node MetalLB currently announces
  from, so it can blackhole traffic — see
  [docs/architecture.md](docs/architecture.md) before changing it

Anything the variables do not cover goes in
`k3s_traefik_extra_values`, merged into the chart values.

11. DNS
-------

No DNS provider is configured or assumed. Two models work:

**Model B — one shared address (recommended).** Every hostname resolves to the
same ingress address and Traefik routes on the hostname:

```text
app1.example.com    ─┐
app2.example.com    ─┼──►  10.10.50.100
grafana.example.com ─┘
```

Adding the fiftieth application costs one more DNS record and no extra address.
Set `k3s_traefik_loadbalancer_ip` explicitly so the address does not move
when the pool is re-ordered.

**Model A — an address per application.** Give an application its own
LoadBalancer Service from a non-auto-assigned pool. Supported, but it spends an
address per application for routing Traefik already does for free.

`kubectl get ingress -A` shows the address each hostname needs, because the role
enables Traefik's `publishedService`.

12. TLS
-------

Without cert-manager, Traefik serves its own self-signed certificate — fine for
a lab, and the validation step deliberately does not disable certificate
verification to paper over it.

With cert-manager:

```yaml
k3s_cert_manager_enabled: true
k3s_cert_manager_issuers:
  # An internal CA: a self-signed root created once, then real certificates
  # issued from it. The right choice for an internal zone.
  - name: internal-ca
    type: ca
    ca_common_name: "Lab Root CA"
    ca_duration: "87600h"

  # Let's Encrypt. Only works for names that resolve on the public internet.
  - name: letsencrypt
    type: acme
    acme_server: staging        # start here; production rate-limits failures
    acme_email: hostmaster@example.com
    solver: http01

  # Self-signed, for testing cert-manager itself.
  - name: selfsigned
    type: selfsigned
```

DNS-01 needs a provider-specific solver, which is passed through verbatim
rather than modelled:

```yaml
k3s_cert_manager_issuers:
  - name: letsencrypt-dns
    type: acme
    acme_server: production
    acme_email: hostmaster@example.com
    solver: dns01
    dns01:
      cloudflare:
        apiTokenSecretRef:
          name: cloudflare-api-token
          key: api-token
```

Applications request a certificate by naming the issuer on their Ingress —
see [`k3s_app`](../k3s_app).

**Let's Encrypt cannot issue for `.local`, `.lan` or any split-horizon
name.** Use the `ca` type there and distribute the root certificate to the clients —
it is in the `<issuer name>-ca` Secret in the cert-manager namespace, so
`internal-ca-ca` for the issuer above.

13. Applications
----------------

Deliberately out of scope. [`roles/k3s_app`](../k3s_app) deploys
applications — namespace, workload, Service, Ingress, ConfigMaps, Secrets, PVCs
and NetworkPolicies — from a declarative definition, plus raw manifests and
Helm releases. It knows nothing about how the cluster was built, which is what
lets applications be added, changed and removed without ever touching this
role.

14. Helm
--------

Helm is installed on the control-plane nodes from the official release server
and verified against the checksum published next to the tarball. The
`get-helm-3` convenience script is not used — it would mean piping a downloaded
script into a shell.

```yaml
k3s_helm_enabled: true
k3s_helm_version: ""     # empty resolves the latest release; pin it
```

This role uses Helm for MetalLB, Traefik and cert-manager only. Application
charts belong in `k3s_app_helm_releases`.

15. Firewall
------------

The firewall stays on. `firewalld` is never disabled and no rule is relaxed to
make Kubernetes work.

Two mechanisms, both needed:

1. **Pod and service traffic** is permitted by trusting the CIDRs it comes from
   (`firewalld`: the `trusted` zone; `ufw`: `allow from` plus `route allow`).
   This is the approach upstream K3s documents. On Debian it is not optional:
   `ufw`'s default forward policy is `DROP`, which silently breaks the pod
   network — and the rules written here are scoped to the pod and service CIDRs
   rather than a blanket `default allow routed`.
2. **Node-to-node control traffic** gets explicit per-port rules, restricted to
   the cluster node addresses, which the role derives from the inventory.
   Trusting a peer node does not mean trusting every port on it.

| Port | Protocol | Why | Where |
| --- | --- | --- | --- |
| 22 | TCP | SSH | `k3s_firewall_ssh_sources` |
| 6443 | TCP | Kubernetes API / K3s supervisor | control plane; cluster nodes + `k3s_firewall_api_sources` |
| 2379 | TCP | etcd client | control plane, cluster nodes only |
| 2380 | TCP | etcd peer | control plane, cluster nodes only |
| 2381 | TCP | etcd metrics (health checks) | control plane, cluster nodes only |
| 10250 | TCP | kubelet API — metrics, `logs`, `exec` | every node, cluster nodes only |
| 10256 | TCP | kube-proxy health endpoint | every node, cluster nodes only |
| 7946 | TCP+UDP | MetalLB speaker memberlist | every node, cluster nodes only; MetalLB only |
| 8472 | UDP | Flannel VXLAN overlay | every node, cluster nodes only; `flannel-backend: vxlan` |
| 51820/51821 | UDP | Flannel WireGuard overlay | every node, cluster nodes only; `flannel-backend: wireguard-native` |
| 80, 443 | TCP | Traefik ingress | every node (MetalLB announces from any of them) |
| 30000-32767 | TCP | NodePorts | off by default — with MetalLB and Traefik there is no reason to expose them |

The UDP overlay ports follow `k3s_flannel_backend` automatically;
switching to WireGuard changes the rules with it.

```yaml
k3s_firewall_enabled: true
k3s_firewall_backend: ""              # "" picks the native backend
k3s_firewall_ssh_sources: [10.10.0.0/16]
k3s_firewall_api_sources: [10.10.0.0/16]
k3s_firewall_ingress_sources: []      # [] means anywhere
k3s_firewall_extra_tcp_ports: [9100]  # e.g. a node exporter
k3s_firewall_nodeport_enabled: false
```

Empty source lists mean "any source". The SSH rule is always written *before*
the firewall is enabled, so enabling it cannot drop the session the play is
running over.

On RHEL, `nm-cloud-setup` is masked and NetworkManager is told to leave `cni0`
and `flannel.1` alone — both documented K3s requirements, and both things that
tear down the routes the CNI installed if left as they are.

16. Hardening
-------------

On by default and individually switchable. Full reasoning, including what is
deliberately *not* done: [docs/security.md](docs/security.md).

| Variable | Default | What it does |
| --- | --- | --- |
| `k3s_hardening_enabled` | `true` | Master switch |
| `k3s_disable_swap` | `true` | `swapoff` plus the `/etc/fstab` entries |
| `k3s_disable_unused_services` | `true` | Masks services that break the CNI (`nm-cloud-setup`) |
| `k3s_sysctl_hardening_enabled` | `true` | No redirects, no source routing, SYN cookies, martian logging |
| `k3s_file_permissions_hardening` | `true` | Root-only credentials, keys and datastore |
| `k3s_secrets_encryption_enabled` | `true` | Secret encryption at rest in etcd |
| `k3s_protect_kernel_defaults` | `true` | The kubelet refuses to start rather than rewrite kernel parameters |
| `k3s_api_audit_enabled` | `true` | API server audit log with a policy that never logs Secret bodies |
| `k3s_pod_security_enabled` | `true` | Pod Security Admission, cluster-wide defaults |

Two deliberate choices worth stating up front:

- **The cluster-wide Pod Security default enforces `baseline`, not
  `restricted`.** `restricted` cluster-wide stops MetalLB's speaker from
  starting — it genuinely needs `hostNetwork` and `NET_RAW` to answer ARP.
  `restricted` is enforced *per namespace* instead, and `k3s_app` labels
  every application namespace for it. Cluster-wide `audit` and `warn` are still
  `restricted`, so a violation anywhere is visible.
- **`rp_filter` is left alone.** Strict reverse-path filtering interacts badly
  with overlay networking, and the gain over a firewall that is already on is
  small. This is the "do not break the thing you are securing" line.

`protect-kernel-defaults` deserves a note: it makes the kubelet *refuse to
start* unless `vm.panic_on_oom`, `vm.overcommit_memory`, `kernel.panic` and
`kernel.panic_on_oops` already have the expected values. The role sets them in
the same step that enables the flag — never enable one without the other.

17. Storage
-----------

K3s' local-path provisioner is the default StorageClass and is all a single
node needs:

```yaml
k3s_storage_enabled: true
k3s_storage_local_path_enabled: true
k3s_storage_local_path: /var/lib/rancher/k3s/storage
```

**It is node-local storage, and it is not HA storage** no matter how many nodes
the cluster has: a volume lives on one node's disk, and a pod rescheduled
elsewhere does not find its data. For anything that has to survive
rescheduling, add a real CSI driver and declare its class:

```yaml
k3s_storage_local_path_enabled: false   # only one default class allowed
k3s_storage_extra_classes:
  - name: nfs
    provisioner: nfs.csi.k8s.io
    is_default: true
    reclaim_policy: Delete
    volume_binding_mode: Immediate
    allow_volume_expansion: true
    parameters:
      server: 10.10.0.5
      share: /export/k8s
```

Preflight refuses two default classes, because which one an unqualified PVC
gets would then be unpredictable.

18. Backup
----------

K3s takes etcd snapshots itself; the role configures them and owns the
directory:

```yaml
k3s_etcd_backup_enabled: true
k3s_etcd_backup_schedule: "0 */12 * * *"
k3s_etcd_backup_retention: 10
k3s_etcd_backup_dir: /var/lib/rancher/k3s/server/db/snapshots
```

The directory is root-only: **an etcd snapshot contains every Secret in the
cluster.** Snapshots are local to the node that took them — copy them off the
cluster, or configure K3s' S3 upload through
`k3s_extra_config`. Restore procedure:
[§21](#21-disaster-recovery).

Snapshots require embedded etcd. With `k3s_cluster_init: false` the
cluster uses sqlite, which has none, and preflight says so rather than letting
you believe you have backups.

19. Upgrade
-----------

Upgrades never happen as a side effect. They need the release, the flag, and
the dedicated playbook:

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/upgrade.yml \
  -e k3s_version=v1.32.1+k3s1 \
  -e k3s_upgrade_enabled=true
```

What that does, per node and one node at a time:

1. Takes an etcd snapshot (control-plane nodes)
2. Drains the node — skipped on a single-node cluster, where draining only
   means a longer outage than the restart
3. Installs the verified binary for the target release
4. Restarts K3s and waits for the node to be `Ready`
5. Uncordons it
6. **Waits for the whole cluster to be `Ready` before the next node is
   touched** — this health gate is what makes it a rolling upgrade

`playbooks/upgrade.yml` runs the control plane and the workers as two plays with
`serial: 1`, so the control plane is always finished first. The role asserts
that the batch really is one host: upgrading several control-plane nodes at once
loses the etcd quorum, and that is not something to leave to whether someone
remembered the flag.

A node already on the target release is skipped, so a re-run after a failure
resumes rather than repeating.

Chart versions upgrade independently, by pinning them and re-running the normal
playbook: `k3s_traefik_chart_version`,
`k3s_metallb_chart_version`, `k3s_cert_manager_chart_version`.

20. Troubleshooting
-------------------

```bash
# Everything, in one place
sudo k3s kubectl get nodes,pods -A -o wide
sudo journalctl -u k3s -f              # k3s-agent on a worker
sudo cat /etc/rancher/k3s/config.yaml  # the running configuration

# The role's own verbose report
ansible-playbook ... -e k3s_debug=true --tags report
```

| Symptom | Likely cause |
| --- | --- |
| The Traefik Service has no external address | The MetalLB pool is exhausted, or not on the nodes' own layer-2 segment. Check `kubectl -n metallb-system logs deploy/metallb-controller` |
| Pods cannot resolve anything | A default-deny NetworkPolicy without the allow-DNS rule. The templates here always write both |
| The kubelet will not start, complaining about kernel parameters | `protect-kernel-defaults` without the four sysctls. Re-run with `--tags os` |
| Nodes `NotReady`, pods stuck `ContainerCreating` | Overlay traffic blocked. Check 8472/UDP between nodes and that `cni0`/`flannel.1` are in the trusted zone |
| Routes disappear a minute after boot (RHEL) | `nm-cloud-setup`. Re-run with `--tags hardening` |
| A node joins and then goes `NotReady` | Clock skew, or 6443 unreachable from it |
| `x509: certificate is valid for …` | The address you are connecting on is not in `tls-san`. Add it to `k3s_api_extra_tls_san` |
| A pod is rejected with `violates PodSecurity` | Working as intended. Either fix the workload or lower that namespace's level deliberately |

21. Disaster recovery
---------------------

**One control-plane node lost, quorum intact (2 of 3 alive).** Remove it from
the cluster, rebuild the host, re-run the playbook. With a vaulted
`k3s_token` the rebuilt node rejoins on its own:

```bash
sudo k3s kubectl delete node k8s-02
# rebuild the host, then:
ansible-playbook -i inventories/prod/hosts.yml playbooks/k3s.yml
```

**Quorum lost (2 of 3 control-plane nodes gone).** etcd cannot elect a leader
and the API server stops serving writes. Restore from a snapshot on the
surviving node:

```bash
sudo systemctl stop k3s
sudo k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/<snapshot>
# wait for "Managed etcd cluster membership has been reset", then Ctrl-C
sudo systemctl start k3s
```

That leaves a single-member cluster. Delete the stale Node objects, rebuild the
other two hosts and re-run the playbook to grow back to three.

**Whole cluster lost.** Rebuild the hosts, restore the snapshot on the
bootstrap node as above, then re-run the playbook for the rest. This is the
reason the snapshots have to be copied off the cluster: a snapshot that only
exists on the node you lost is not a backup.

**What is not in an etcd snapshot:** PersistentVolume *data*. The snapshot has
the PVC and PV objects, not the bytes on disk. With the local-path provisioner
that data lives in `/var/lib/rancher/k3s/storage` on one node and needs backing
up separately.

Testing
-------

Three Molecule scenarios, all driving the same role and the same converge
playbook — only the inventory differs, which is the claim being tested:

| Scenario | Topology | OS | Firewall |
| --- | --- | --- | --- |
| `default` | single node | Ubuntu 24.04 | `ufw` |
| `rockylinux` | single node | Rocky Linux 9 | `firewalld`, SELinux enforcing |
| `cluster-ha` | 3 control plane + 1 worker | Rocky Linux 9 | `firewalld` |

The two single-node scenarios are separate rather than one scenario with two
platforms: both hosts would land in `k8s_control_plane` and be read as a single
cluster with two control-plane nodes, which the role rejects outright — an etcd
quorum of two means either failure stops the cluster.

Each one installs the cluster, runs again to prove idempotency, deploys an
application through `k3s_app`, and then verifies: every node `Ready`,
CoreDNS / metrics-server / Traefik / MetalLB available, the MetalLB speaker on
every node, the ingress answering on the expected address, the application
served by hostname, an unknown hostname getting a 404, the NetworkPolicies in
place, Secret encryption active, an audit log being written, swap off, the
firewall still running, credentials at `0600`, and the required kernel modules
and sysctls in effect.

It also checks that the hardening *refuses* something, rather than only that it
is configured: creating a privileged pod in an application namespace has to be
rejected by Pod Security Admission.

```bash
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml

export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

cd roles/k3s
molecule test                    # the default scenario: Ubuntu, single node
molecule test -s rockylinux
molecule test -s cluster-ha      # needs ~11 GB free memory on the host
```

License
-------

GPL-2.0-or-later
