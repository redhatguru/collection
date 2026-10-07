k3s_app
=======

Deploys applications onto an existing Kubernetes cluster from a declarative
inventory definition.

This role talks to the Kubernetes API and nothing else — it never configures a
node, installs a package on the cluster's behalf or touches K3s. That
separation is the point: applications can be added, changed and removed without
going near the [`k3s`](../k3s) infrastructure role, and this role
works just as well against a cluster it did not build.

Three ways to deploy, side by side:

| | For | Variable |
| --- | --- | --- |
| **Application model** | Ordinary workloads — a container, a Service, an Ingress, some configuration | `k3s_app_applications` |
| **Helm releases** | Anything that already ships a chart | `k3s_app_helm_releases` |
| **Raw manifests** | The cases neither covers — CRDs, RBAC, a Job | `k3s_app_manifests` |

Secure by default
-----------------

Every workload this role generates gets, unless the definition overrides it:

```yaml
runAsNonRoot: true                    # restricted Pod Security Standard
runAsUser: 1000
seccompProfile: {type: RuntimeDefault}
allowPrivilegeEscalation: false
readOnlyRootFilesystem: true
privileged: false
capabilities: {drop: [ALL]}
automountServiceAccountToken: false   # most applications never call the API
resources: {requests: …, limits: …}   # always set
```

Overrides are **merged**, not substituted, so changing one field leaves the
rest in place. Every namespace this role creates is labelled
`pod-security.kubernetes.io/enforce: restricted`, and the generated workloads
are written to satisfy that without needing an exception.

Also on by default:

- A **default-deny NetworkPolicy** per namespace, together with the allow-rules
  that keep it working — DNS egress and ingress from the ingress controller.
  Never the deny without the rest: that is an outage, not hardening.
  Application-to-application traffic stays denied until a policy allows it.
- **No RBAC at all.** Each application gets a ServiceAccount with no
  RoleBinding. No `cluster-admin`, no wildcards. Permissions are granted
  deliberately, through `k3s_app_manifests`.
- **ClusterIP Services.** With Traefik routing on hostname, a NodePort or a
  per-application LoadBalancer buys nothing and widens the exposed surface.
- **Pinned images required.** A missing tag, or `latest`, is rejected in
  preflight: without a pin the same playbook deploys different software
  tomorrow.

Requirements
------------

- `ansible-core` >= 2.15, collection `kubernetes.core`
- A reachable cluster and a kubeconfig for it. The default is
  `/etc/rancher/k3s/k3s.yaml`, which is why the playbook normally targets a
  control-plane node; point `k3s_app_kubeconfig` elsewhere to drive the
  cluster from a management host
- The Python Kubernetes client, which the role installs itself: the
  distribution package first, then pip if the distribution has none (RHEL and
  Rocky Linux without EPEL). Set
  `k3s_app_manage_python_dependencies: false` to manage it yourself
- Helm, only if `k3s_app_helm_releases` is used. The `k3s` role
  installs it on the control-plane nodes

Variables
---------

Full list with comments: [`defaults/main.yml`](defaults/main.yml).

| Variable | Default | Purpose |
| --- | --- | --- |
| `k3s_app_kubeconfig` | `/etc/rancher/k3s/k3s.yaml` | Credentials for the cluster |
| `k3s_app_state` | `present` | `absent` removes what is defined |
| `k3s_app_applications` | `[]` | The application model (below) |
| `k3s_app_helm_releases` | `[]` | Helm releases |
| `k3s_app_manifests` | `[]` | Literal Kubernetes objects |
| `k3s_app_ingress_class` | `traefik` | IngressClass for generated Ingresses |
| `k3s_app_ingress_namespace` | `traefik` | Namespace the NetworkPolicies let in |
| `k3s_app_pod_security_level` | `restricted` | Enforced on every namespace this role creates |
| `k3s_app_network_policy_enabled` | `true` | Default-deny plus its allow-rules |
| `k3s_app_api_server_cidr` | `10.43.0.0/16` | Service CIDR, for the opt-in API egress policy |
| `k3s_app_wait` | `true` | Wait for each workload to become available |
| `k3s_app_wait_timeout` | `600` | Seconds to wait |
| `k3s_app_debug` | `false` | Print replica status per application |

Tags: `dependencies`, `preflight`, `applications`, `helm`, `manifests`,
`report`.

Application schema
------------------

```yaml
k3s_app_applications:
  - name: nginx                     # required — object names and labels
    namespace: nginx                # required
    enabled: true                   # false skips it entirely (not a delete)
    kind: Deployment                # or StatefulSet
    replicas: 2

    image:                          # required
      repository: nginxinc/nginx-unprivileged
      tag: "1.27-alpine"            # or digest: sha256:…  — one is required
      pull_policy: IfNotPresent

    command: []                     # container entrypoint override
    args: []
    env:                            # literal Kubernetes env entries
      - name: API_TOKEN
        valueFrom:
          secretKeyRef: {name: app-credentials, key: api_token}
    env_from: []

    ports:
      - {name: http, container_port: 8080, protocol: TCP}

    service:
      enabled: true
      type: ClusterIP
      port: 80
      target_port: http
      annotations: {}
      extra_ports: []

    ingress:
      enabled: true
      host: nginx.example.local     # required when enabled
      path: /
      tls: true
      tls_secret: nginx-tls         # defaults to <name>-tls
      cluster_issuer: internal-ca   # cert-manager issues into tls_secret
      class_name: traefik
      annotations: {}
      paths: []                     # several paths, each with its own backend

    config_maps:
      - name: nginx-config
        data:
          nginx.conf: |
            …

    secrets:                        # values from Ansible Vault
      - name: nginx-credentials
        type: Opaque
        string_data:
          api_token: "{{ vault_nginx_api_token }}"

    persistent_volume_claims:       # standalone PVCs (Deployment)
      - name: data
        size: 10Gi
        access_modes: [ReadWriteOnce]
        storage_class: local-path

    volume_claim_templates:         # per-replica volumes (StatefulSet)
      - name: data
        size: 10Gi
        access_modes: [ReadWriteOnce]

    volumes: []                     # literal Kubernetes volumes
    volume_mounts: []

    resources:                      # merged over the secure defaults
      requests: {cpu: 100m, memory: 128Mi}
      limits: {cpu: 1000m, memory: 512Mi}

    pod_security_context: {}        # merged over the secure defaults
    security_context: {}            # merged over the secure defaults

    service_account:
      name: nginx
      automount: false

    node_selector: {}
    tolerations: []
    affinity: {}
    topology_spread_constraints: []

    readiness_probe: {}             # literal Kubernetes probes
    liveness_probe: {}
    startup_probe: {}

    labels: {}                      # on the workload
    pod_labels: {}                  # on the pods
    pod_annotations: {}
    namespace_labels: {}

    pod_security_level: restricted  # lower it per application, deliberately

    network_policy:
      enabled: true
      allow_dns: true
      allow_from_ingress: true
      allow_api_server: false       # only for applications that call the API
      extra:                        # explicit allow-rules
        - name: allow-to-database
          spec:
            podSelector: {}
            policyTypes: [Egress]
            egress:
              - to:
                  - namespaceSelector:
                      matchLabels:
                        kubernetes.io/metadata.name: postgres
```

Objects created, in order: Namespace, ServiceAccount, ConfigMaps, Secrets,
PVCs, the workload, Service, Ingress, NetworkPolicies.

Example
-------

A worked example, deployable as-is, is in
[`inventories/lab-single-node/group_vars/all.yml`](../../inventories/lab-single-node/group_vars/all.yml):
unprivileged nginx on its own hostname, with TLS from the internal CA and a
default-deny NetworkPolicy.

```bash
ansible-playbook -i inventories/lab-single-node/hosts.yml \
  playbooks/applications.yml
```

The image is `nginxinc/nginx-unprivileged` rather than `nginx` for a concrete
reason: the stock image runs as root and writes to `/var/cache`, so it cannot
satisfy the `restricted` standard this role enforces. An application that needs
a writable path gets an `emptyDir`, not a writable root filesystem.

Helm releases
-------------

```yaml
k3s_app_helm_releases:
  - name: grafana
    namespace: monitoring
    chart_ref: grafana/grafana
    chart_repo_url: https://grafana.github.io/helm-charts
    chart_version: "8.5.0"          # pin it
    pod_security_level: baseline    # charts often cannot meet restricted
    values:
      adminPassword: "{{ vault_grafana_admin_password }}"
      ingress:
        enabled: true
        ingressClassName: traefik
        hosts: [grafana.example.local]
```

The namespace is created with Pod Security labels first, so a chart's workloads
are held to a standard even though the chart knows nothing about it. Many
charts cannot meet `restricted` — set `pod_security_level: baseline` for those
deliberately, per release, rather than lowering it everywhere.

Removing an application
-----------------------

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/applications.yml \
  -e k3s_app_state=absent
```

Every object is removed except the namespace, which is created but never
deleted: deleting a namespace takes everything else in it along, including
objects this role never created.

To stop deploying an application without deleting anything, set
`enabled: false` on it. That is a skip, not a delete.

Testing
-------

The Molecule scenario builds a minimal single-node cluster with the
infrastructure role's shared converge playbook, then exercises this role
against it: a Deployment with a ConfigMap, a Secret, environment from that
Secret, volumes, probes, an Ingress and a custom NetworkPolicy; a StatefulSet
with a volume claim template; a disabled application that must not be created;
and finally `state: absent` on one of them.

It then verifies that every object exists, that the namespace really enforces
`restricted`, that the ingress serves the ConfigMap's content on the
application's hostname, and that `absent` removed the workload.

```bash
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml

export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

cd roles/k3s_app
molecule test
```

License
-------

GPL-2.0-or-later
