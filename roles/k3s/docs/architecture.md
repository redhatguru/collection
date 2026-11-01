# Architecture

Why this role is built the way it is, and what was traded away to get there.
The [README](../README.md) covers how to use it.

## Why K3s rather than kubeadm

Three reasons decided it, in order of weight.

**The firewall can stay on.** K3s documents a working configuration with the
host firewall enabled: trust the pod and service CIDRs, open the control ports
between nodes. With `kubeadm` plus Calico the equivalent runs into
[projectcalico/calico#4382](https://github.com/projectcalico/calico/issues/4382),
and the common "fix" is to disable `firewalld` — which is exactly the
workaround this role is not allowed to use.

**One code path from one node to an HA cluster.** K3s needs `cluster-init: true`
on the first server, `server: https://…` on the rest, and the same token
everywhere. `kubeadm` needs `kubeadm init`, `kubeadm join --control-plane` and
`kubeadm join`, with a different token flow for each — three code paths, three
sets of failure modes, and a role that quietly becomes three roles.

**The hardening hooks are in the distribution.** `secrets-encryption`,
`protect-kernel-defaults`, `etcd-snapshot-schedule-cron` and arbitrary
`kube-apiserver-arg` are configuration keys, not patches. And because K3s runs
the API server as a process rather than a static pod, the audit policy and the
admission configuration are ordinary files on disk — no `hostPath` mount
choreography.

The honest limitation: K3s' embedded etcd suits 3 to 5 control-plane nodes.
Beyond that, a dedicated etcd cluster is the right answer, and this role is not
it.

## Topology from the inventory

Group membership is the only input. [`vars/main.yml`](../vars/main.yml) derives
the rest, and it lives in `vars/` rather than `defaults/` on purpose: if the
derived topology could be overridden from inventory, the preflight assertions
could pass while the cluster was built from different facts.

```text
groups[k8s_control_plane]  ──►  k3s_control_plane_hosts
groups[k8s_workers]        ──►  k3s_worker_hosts   (minus any overlap)
                                k3s_topology
                                k3s_node_role      (per host)
                                k3s_init_host
                                k3s_schedulable_node_count
                                k3s_traefik_replica_count
                                k3s_firewall_cluster_tcp_ports
                                k3s_firewall_cluster_udp_ports
                                k3s_tls_sans
```

`k3s_node_role` is what makes one role cover three topologies. A
control-plane node gets a server configuration, 6443 and the etcd ports; a
worker gets an agent configuration and neither. Nothing in the role branches on
`k3s_topology` except the things that genuinely differ at the cluster
level — Traefik's replica count and whether a drain is worth doing.

### Why two control-plane nodes is rejected

An etcd quorum is `floor(n/2) + 1`. With three members the quorum is two, so one
node can fail. With **two** members the quorum is also two, so *either* failure
stops the cluster — strictly worse than running one node. The role refuses the
configuration instead of building something that looks redundant and is not.

## Bootstrap sequence

The order matters more than any individual step, which is why it lives in the
role rather than in whatever order Ansible happens to reach the hosts.

```text
  all nodes   preflight ─► OS prep ─► host hardening ─► firewall
                  │
                  ▼
  all nodes   config.yaml, audit policy, admission config, token file
                  │              (written before K3s ever starts: the API
                  │               server refuses to come up if the policy
                  ▼               files it is told to read are missing)
  all nodes   verified binary + systemd unit   (INSTALL_K3S_SKIP_START=true)
                  │
                  ▼
  init host   start K3s ─► wait for /readyz ─► wait for Ready
                  │
                  ▼
  other CPs   start K3s, throttle: 1 ─► wait for Ready        (one at a time)
                  │
                  ▼
  workers     start K3s ─► wait for Ready                     (in parallel)
                  │
                  ▼
  init host   Helm ─► storage ─► MetalLB ─► Traefik ─► cert-manager
              ─► NetworkPolicies ─► validate ─► report
```

The upstream install script is run with `INSTALL_K3S_SKIP_START=true`
specifically so the role, and not the script, decides when each node starts.
Joining several etcd members at once to a cluster that has only just been
initialised races on the member list. The K3s unit is `Type=notify`, so
`systemctl start` does not return until that node's API server is serving —
which is what makes `throttle: 1` a real gate rather than a pause.

## Installing K3s without piping anything into a shell

```text
GitHub release ──► sha256sum-<arch>.txt  ──► checksum extracted and asserted
                                                     │
                   k3s binary ──────────────────────► get_url verifies it
                                                     │
                   get.k3s.io install script ────────► run from disk with
                                                        INSTALL_K3S_SKIP_DOWNLOAD
```

The script still runs, because it is the supported way to get a correct systemd
unit and the `k3s-selinux` policy on RHEL — but with `SKIP_DOWNLOAD` it neither
downloads nor executes anything that was not already verified. That is also why
SELinux is left enforcing: the policy is installed rather than worked around.

When `k3s_version` is empty, the release is resolved from
`update.k3s.io` once, on the bootstrap node, and the result is propagated to
every host — so a multi-node cluster cannot end up with two nodes on different
releases because the channel moved mid-run.

## How cluster objects are applied

Two mechanisms, each where it fits:

| What | How | Why |
| --- | --- | --- |
| MetalLB, Traefik, cert-manager | `kubernetes.core.helm` | Version pinning, `--wait`, and upstream charts that are far better maintained than a hand-written manifest |
| MetalLB pools, StorageClasses, ClusterIssuers, cluster NetworkPolicies | K3s' auto-deploying manifests directory | Declarative and reconciled on every K3s start, survives a reboot, removing the file removes the objects — and it needs no Python Kubernetes client on the node |
| Applications | `kubernetes.core.k8s` (in `k3s_app`) | Proper diffing, check mode and `wait_condition`, which an application deployment actually benefits from |

The split keeps the infrastructure role's dependency list to "Helm", while the
application role — which is the one that genuinely needs diffing — carries the
Python client requirement and installs it itself.

## Firewall model

Pod traffic and node traffic are different problems and get different answers.

**Pod and service traffic** is allowed by trusting the CIDRs it originates
from. A pod reaching CoreDNS or the API server arrives from `10.42.0.0/16` or
`10.43.0.0/16`; trusting those is what lets the overlay work with the firewall
still enabled. On `firewalld` that is the `trusted` zone; on `ufw` it is
`allow from` *plus* `route allow`, because `ufw`'s default forward policy is
`DROP` and without the routed rules the pod network silently breaks. The routed
rules are scoped to those two CIDRs rather than set as a blanket
`default allow routed`.

**Node-to-node control traffic** gets explicit per-port rules, with the cluster
node addresses as the source — derived from the inventory, so adding a node
updates every other node's rules. Trusting a peer node for the etcd port does
not mean trusting it for everything else.

The CNI interfaces (`cni0`, `flannel.1`, or `flannel-wg`) are added to the
trusted zone permanently rather than immediately, because they do not exist
until K3s has started once; `firewalld` binds them when they appear.

The full port table, and what each port is for, is in the
[README](../README.md#15-firewall) and in the comments at the top of the
firewall section of [`vars/main.yml`](../vars/main.yml).

## MetalLB layer 2: highly available, not load balanced

In layer-2 mode one node answers ARP for a given service address at a time.
If that node fails, another takes over within a few seconds. That is genuine
high availability — and it is *not* load balancing: all traffic for that
address enters through one node, whose bandwidth becomes the ceiling.

This is why every node has 80 and 443 open: which node announces the address
is MetalLB's decision and it changes on failure.

For real distribution, BGP spreads the address across nodes via ECMP, at the
cost of needing a router that will peer. Layer 2 is the default because it
needs nothing from the network.

### `externalTrafficPolicy`: the trade-off

`Cluster` (the default here) sends traffic to any Traefik pod, hiding the
client address behind SNAT. `Local` preserves the client address but only
delivers to a pod on the node that received the packet — and with layer-2
MetalLB, that is whichever node currently announces the address. If no Traefik
pod runs there, the traffic is dropped.

`Local` is therefore safe only when Traefik runs on every node that can
announce, which in practice means a DaemonSet or a replica count equal to the
node count. The role defaults to the option that cannot blackhole traffic and
documents the other rather than silently picking the clever one.

## Traefik sizing

```text
replicas = min(schedulable nodes, k3s_traefik_max_replicas)
```

where "schedulable" excludes the control plane when it is tainted. One node
gives one replica; three gives three. The PodDisruptionBudget is only created
above one replica, because a 1-of-1 budget blocks every node drain — including
the ones this role's own upgrade path performs.

Anti-affinity is `preferred`, not `required`: `required` with more replicas than
nodes leaves the surplus permanently `Pending`. Topology spread uses
`ScheduleAnyway` for the same reason. Both prefer spreading and neither can
deadlock the scheduler.

One consequence worth stating: tainting the control plane
(`k3s_taint_control_plane`) restricts Traefik to the workers. With a
single worker that makes the ingress a single point of failure again. Dedicated
workers and HA ingress together need at least two or three workers.

## Pod Security Standards: why `baseline` and not `restricted`

Enforcing `restricted` cluster-wide stops MetalLB's speaker from starting: it
needs `hostNetwork` and `NET_RAW` to answer ARP, and `baseline` already forbids
`hostNetwork`. So the speaker's namespace is exempted in the admission
configuration — one explicit, auditable place — rather than labelled
`privileged`, where nobody would look for it.

The cluster-wide `enforce` is `baseline`, which still blocks privileged
containers, `hostPID`, `hostIPC`, host ports and dangerous capabilities.
`audit` and `warn` are `restricted`, so a violation anywhere is recorded.
Application namespaces are labelled `restricted` individually by
`k3s_app`, and the workloads it generates are written to satisfy that
without an exception.

## Storage

The local-path provisioner is the default class because it is what a single
node needs and it works with no configuration. It is node-local: a volume is a
directory on one node's disk, and a pod rescheduled elsewhere does not find its
data.

The role will not pretend otherwise, and it will not install a distributed
storage system on your behalf either. `k3s_storage_extra_classes`
declares a real CSI driver's class, and preflight refuses to leave two default
classes in place, because which one an unqualified PVC gets would then depend
on alphabetical luck.

## Upgrades

One node at a time, control plane first, with a health gate between nodes.
`serial: 1` is a play-level concern, so it lives in
[`playbooks/upgrade.yml`](../../../playbooks/upgrade.yml) rather than in the
role — and the role asserts that `ansible_play_batch` really is one host, so
running it without `serial: 1` fails loudly instead of taking down the quorum.

```text
per node:  etcd snapshot ─► drain ─► verified binary ─► restart ─► Ready
                                                           │
                                                           ▼
                                                 uncordon ─► wait for the
                                                             WHOLE cluster
                                                             to be Ready
                                                                  │
                                                           next node ◄┘
```

A node already on the target release is skipped, so a re-run after a failure
resumes rather than starting over. The drain is skipped on a single-node
cluster, where it only means a longer outage than the restart itself.

## Idempotency

A second run changes nothing, and a few specific decisions are what make that
true rather than nearly true:

- **The token** is resolved in a fixed order: an explicitly supplied value, then
  the file this role wrote, and only then the one K3s wrote. K3s stores the
  token in its long `K10<ca-hash>::server:<secret>` form, so reading *that* back
  on every run would rewrite the token file — and restart the cluster — each
  time.
- **The K3s binary** is only downloaded when the installed version differs, so
  the normal run makes no network request for it at all.
- **The install script** is guarded by the systemd unit it creates.
- **`swapoff`** is guarded by `ansible_facts['swaptotal_mb']`, which is already
  zero on the second run.
- **Secrets** generated by `k3s_app` use `data` with base64 rather than
  `stringData`. `stringData` is write-only — the API server stores it under
  `data` and never returns it — so a comparison is impossible and the object
  would report as changed on every single run.

## Known limitations

- `k3s_api_registration_address` defaults to the bootstrap node. Running
  nodes are unaffected if it is lost, but *joining* needs it. A VIP or external
  load balancer in front of the control plane removes this; the role adds
  whatever address you set to the API server certificate.
- MetalLB layer 2 does not distribute load (above).
- Local-path storage is not HA storage (above).
- etcd snapshots are written locally. Copying them off the cluster is left to
  you — a snapshot that only exists on the node you lost is not a backup.
- The embedded etcd is appropriate to roughly five control-plane nodes.
