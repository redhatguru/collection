# Security

What this role hardens, what it deliberately does not, and how to check that
any of it is actually in force. The working assumption throughout is "it works
and it is secure until proven otherwise", not "it works, therefore it is fine".

## Threat model

| Assumed | Not assumed |
| --- | --- |
| Nodes sit on a network with other hosts on it | The network is trusted |
| Anyone with the cluster token can join a node | The token is hard to guess |
| A compromised pod will try to reach the node, the API server and other pods | Pods are benign |
| An operator reads playbook output, logs and git history | Those places are private |
| A stolen etcd snapshot or disk discloses every Secret | Disks are safe |

The controls below follow from that list, in roughly that order.

## Host

| Control | How | Variable |
| --- | --- | --- |
| Firewall stays enabled | `firewalld` / `ufw`, rules per node role | `k3s_firewall_enabled` |
| Swap off | `swapoff` plus the `/etc/fstab` entries | `k3s_disable_swap` |
| CNI-breaking services masked | `nm-cloud-setup` (RHEL) | `k3s_disable_unused_services` |
| No ICMP redirects, no source routing, SYN cookies, martian logging | `/etc/sysctl.d/90-k3s-hardening.conf` | `k3s_sysctl_hardening_enabled` |
| Kernel parameters the kubelet depends on are pinned | `/etc/sysctl.d/90-k3s-kubelet.conf` | `k3s_protect_kernel_defaults` |
| SELinux left enforcing | `k3s-selinux` policy installed, never relaxed | — |
| Credentials root-only | `0600` files, `0700` directories | `k3s_file_permissions_hardening` |

### The firewall is the control, not an obstacle

The one rule this role will not break: no `systemctl disable firewalld`, no
`ufw disable`, no "allow everything from the pod network to the whole host".
Pod traffic is permitted by trusting the pod and service CIDRs — which is what
upstream K3s documents — and node-to-node control traffic gets explicit
per-port rules restricted to the cluster node addresses.

The consequence is that trusting a peer node for etcd does not trust it for
every other port, and the exposed surface of a worker is genuinely smaller than
that of a control-plane node: no 6443, no 2379, no 2380.

SSH is always permitted *before* the firewall is enabled, so turning the
firewall on cannot drop the session the play is running over.

## Cluster

| Control | How | Variable |
| --- | --- | --- |
| Secrets encrypted at rest | K3s `secrets-encryption` | `k3s_secrets_encryption_enabled` |
| API audit log | Policy file + `kube-apiserver-arg` | `k3s_api_audit_enabled` |
| Pod Security Admission | Cluster-wide `AdmissionConfiguration` | `k3s_pod_security_enabled` |
| TLS everywhere, correct SANs | `tls-san` from the inventory | `k3s_api_extra_tls_san` |
| kubeconfig not world-readable | `write-kubeconfig-mode: "0600"` | `k3s_write_kubeconfig_mode` |
| Token never in a config file | A root-only file, referenced as `token-file` | `k3s_token` |
| NetworkPolicies enforced | K3s' policy controller left enabled | `k3s_network_policy_controller_enabled` |
| No NodePorts | Off by default | `k3s_firewall_nodeport_enabled` |
| Ingress TLS 1.2 minimum, HTTP redirected | Traefik chart values | `k3s_traefik_tls_min_version` |

### The audit policy never logs a Secret's body

Secrets, ConfigMaps and token reviews are logged at `Metadata` only. Logging
them at `Request` or `RequestResponse` would write the secret itself into the
audit log — a file that is then backed up, shipped to a log collector and read
by more people than the Secret ever was.

What *is* logged at `RequestResponse`: RBAC changes, and `pods/exec`,
`pods/attach` and `pods/portforward`. Those are the operations worth being able
to reconstruct afterwards. High-volume read traffic (`get`/`list`/`watch`,
`system:nodes`, `kube-proxy`, events) is dropped entirely, so the log stays
small enough to actually be read.

The log lives in a `0700` directory under the K3s server directory.

### Pod Security Standards

Cluster-wide: `enforce: baseline`, `audit: restricted`, `warn: restricted`.
Per application namespace: `enforce: restricted`, applied by
[`k3s_app`](../../k3s_app).

`baseline` cluster-wide already blocks privileged containers, `hostPID`,
`hostIPC`, host ports, host paths beyond a small allow-list, and dangerous
capabilities. `restricted` additionally requires non-root, a seccomp profile,
dropping `ALL` capabilities and no privilege escalation.

**Why not `restricted` cluster-wide?** It stops MetalLB's speaker from
starting. The speaker needs `hostNetwork` and `NET_RAW` to answer ARP for the
service addresses; `baseline` already forbids `hostNetwork`, so its namespace
needs an exemption either way. Setting the cluster default to a level that
nothing can satisfy, and then exempting half the cluster, is worse than setting
a level the cluster components meet and enforcing the strict level where it can
actually hold.

The `audit` and `warn` levels stay at `restricted`, so a workload that would
fail the strict standard is still recorded even in a namespace that only
enforces `baseline`.

### Exemptions, in full

| Namespace | Why |
| --- | --- |
| `kube-system` | CoreDNS, the CNI and K3s' own controllers |
| `kube-public`, `kube-node-lease` | Cluster bookkeeping |
| `metallb-system` | The speaker needs `hostNetwork` and `NET_RAW` (added only when MetalLB is enabled) |

There are no others, and `k3s_pod_security_exempt_namespaces` is the
only way to add one — a single list, visible in the inventory, rather than a
label scattered on a namespace.

## Applications

Defaults applied by `k3s_app` to every workload it generates:

```yaml
runAsNonRoot: true
runAsUser: 1000
seccompProfile: {type: RuntimeDefault}
allowPrivilegeEscalation: false
readOnlyRootFilesystem: true
privileged: false
capabilities: {drop: [ALL]}
automountServiceAccountToken: false
resources: {requests: …, limits: …}   # always set
```

They are *merged* with whatever the application definition supplies, so a
partial override changes one field rather than discarding the rest.

Three further defaults worth naming:

- **No RBAC is granted.** Each application gets a ServiceAccount with no
  RoleBinding. No `cluster-admin`, no wildcard rules, nothing. Permissions have
  to be granted deliberately through `k3s_app_manifests`.
- **The service-account token is not mounted.** Most applications never call
  the API; mounting a credential they do not use is free risk.
- **Images must be pinned.** A missing tag, or `latest`, is rejected in
  preflight. Without a pin, the same playbook deploys different software
  tomorrow.

### NetworkPolicies

Every application namespace gets a default-deny policy **together with** the
allow-rules that keep it working:

| Policy | Effect |
| --- | --- |
| `default-deny` | Denies all ingress and egress in the namespace |
| `allow-dns` | Egress to CoreDNS on 53/udp and 53/tcp |
| `allow-ingress` | Ingress from the ingress controller's namespace only |
| `allow-apiserver` | Egress to the API server — opt-in, off by default |

A default-deny policy applied without the allow-rules is an outage, not a
hardening measure: pods lose DNS first and then everything that depends on
resolving a name. The role applies them as one set and never one without the
other, and `kube-system` is never touched — a default-deny there takes the
cluster down.

Application-to-application traffic stays denied until a policy allows it, which
is what `network_policy.extra` is for.

## Secrets

- `k3s_token` belongs in Ansible Vault. It is written to a
  root-only file and referenced as `token-file`, so it never appears in
  `config.yaml` — a file that gets diffed, copied into tickets and pasted into
  chat.
- Tasks that handle it are `no_log`, including the token slurp, the token file
  write and the Secret applies.
- When no token is supplied, one is generated with
  `community.general.random_string` and reused on later runs. That works, but a
  vaulted token is what makes a rebuilt node able to rejoin without reading the
  token off a surviving node.
- Secret values in an application definition should come from Vault. They are
  base64-encoded into `data` — not for secrecy, base64 is not encryption, but
  because `stringData` is write-only and would make the object report as
  changed on every run.
- The role never writes a kubeconfig to the controller unless
  `k3s_kubeconfig_fetch` is set explicitly. That credential is
  `cluster-admin`.
- `k3s_debug` output contains no token and no kubeconfig.

**An etcd snapshot contains every Secret in the cluster.** The snapshot
directory is `0700`, and copying snapshots somewhere else is a decision with
the same weight as copying the Secrets themselves.

## What is deliberately not done

Security settings that break the thing they protect are not security settings.

- **`rp_filter` is left alone.** Strict reverse-path filtering interacts badly
  with overlay networking, and the gain over a firewall that is already on is
  small.
- **No cluster-wide `restricted` enforcement.** See above.
- **No default-deny in `kube-system`.** It takes the cluster down.
- **No `--insecure`, no `validate_certs: false`, no disabled TLS verification
  anywhere.** The post-install validation checks hostname routing over HTTP
  with an explicit `Host` header, precisely so that a self-signed default
  certificate does not become a reason to turn verification off.
- **The firewall is never disabled**, including on RHEL, where that is the
  usual workaround.
- **Nothing is piped into a shell.** The K3s binary and the Helm archive are
  verified against published SHA-256 checksums; the upstream install script
  runs from disk with `INSTALL_K3S_SKIP_DOWNLOAD=true`.
- **SELinux stays enforcing.** The `k3s-selinux` policy is installed instead.
- **No NodePorts and no per-application LoadBalancer by default.** With Traefik
  routing on hostname, neither is needed, and both widen the exposed surface.

## Verifying it

The hardening is tested by making it refuse something, not by reading its
configuration back. The Molecule scenarios check, among other things, that
creating a privileged pod in an application namespace is **rejected** by Pod
Security Admission.

By hand:

```bash
# Secrets encrypted at rest
sudo k3s secrets-encrypt status

# Pod Security Admission actually enforcing
sudo k3s kubectl run probe -n <app-namespace> --image=busybox:1.36 \
  --restart=Never --privileged --command -- /bin/true
# expected: Error … violates PodSecurity "restricted:latest"

# The audit log is being written
sudo tail -1 /var/lib/rancher/k3s/server/logs/audit.log

# The firewall is still on, and what it permits
sudo firewall-cmd --list-all --zone=trusted   # RHEL
sudo ufw status verbose                       # Debian

# Credentials are root-only
sudo ls -l /etc/rancher/k3s/

# NetworkPolicies in force for an application
sudo k3s kubectl get networkpolicy -n <app-namespace>
```

## Reviewing a change to this role

Worth asking, in order:

1. Does it disable, relax or work around a control? If so, what replaces it?
2. Does it widen what is reachable — a port, a source range, a NodePort, a
   LoadBalancer, an RBAC rule?
3. Does a secret end up in `stdout`, a log, a generated file or git?
4. Does it add an exemption? Is it in the one list where exemptions live?
5. Does it introduce an unpinned image, chart or binary?
6. Is a download verified?
7. Does it still work with the firewall on, SELinux enforcing, and a
   default-deny NetworkPolicy in the namespace?
