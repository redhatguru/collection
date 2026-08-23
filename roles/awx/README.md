awx
=========

Deploys [AWX](https://github.com/ansible/awx) on an existing Kubernetes
cluster (see the `kubernetes` role) via the
[AWX Operator](https://github.com/ansible/awx-operator), using an external
PostgreSQL database (see the `postgresql` role) rather than the operator's
bundled one.

Installs the AWX Operator into `awx_namespace` via its published kustomize
manifests, creates the admin-password and postgres-configuration secrets
the operator expects, then applies an `AWX` custom resource and waits for
it to become available.

Requirements
------------

- A working Kubernetes cluster with `kubectl` and `/etc/kubernetes/admin.conf`
  already present on the host this role runs on — i.e. run this role on a
  host that already has the `kubernetes` role applied (`single` or a
  `masters` node).
- An external, already-reachable PostgreSQL server with a database and user
  for AWX — i.e. the `postgresql` role, applied to a different host.
- Outbound internet access from the Kubernetes host (the AWX Operator's
  manifests are fetched from GitHub, and AWX/operator container images are
  pulled from Quay/Docker Hub).

### Collections

This role only uses modules built into `ansible-core` (`ansible.builtin.*`),
so no extra collections need to be installed.

Role Variables
--------------

- `awx_namespace` (default `"awx"`) — Kubernetes namespace the AWX Operator
  and the AWX instance are both deployed into.
- `awx_operator_version` (default `"2.19.1"`) — AWX Operator release to
  install, applied via its `github.com/ansible/awx-operator/config/default`
  kustomize manifests for that git ref.
- `awx_operator_kube_rbac_proxy_image` (default
  `"quay.io/brancz/kube-rbac-proxy:v0.15.0"`) — the 2.19.1 operator
  manifests pin `gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0`, which no
  longer exists now that Google has deprecated old GCR image paths; this
  redirects it to the same image on its original upstream registry. Set to
  `""` to disable the override once that's no longer needed.
- `awx_name` (default `"awx"`) — name of the AWX custom resource, and the
  prefix for the resources the operator creates for it.
- `awx_service_type` (default `"NodePort"`) — how the AWX web service is
  exposed: `ClusterIP`, `NodePort` or `LoadBalancer`.
- `awx_nodeport_port` (default `30080`) — only used when `awx_service_type`
  is `NodePort`.
- `awx_admin_user` (default `"admin"`) — AWX superuser account created on
  first boot.
- `awx_admin_password` (default `"ChangeMe123!"`) — password for
  `awx_admin_user`. Change this.
- `awx_replicas` (default `1`) — number of AWX replicas.
- `awx_postgres_host` (no default, must be set) — address of the external
  PostgreSQL server (e.g. the host running the `postgresql` role).
- `awx_postgres_port` (default `5432`)
- `awx_postgres_database` (default `"awx"`) — matches `postgresql_database_name`'s
  own default, so the two roles line up out of the box.
- `awx_postgres_user` (default `"dbuser"`) — matches `postgresql_user`'s own default.
- `awx_postgres_password` (default `"dbuser"`) — matches `postgresql_user_password`'s
  own default.
- `awx_postgres_sslmode` (default `"prefer"`)
- `awx_wait_timeout` (default `900`) — seconds to wait for the operator and
  AWX itself to become ready. AWX's images are large and the operator does
  several reconcile passes, so this is generous by default.

Example Playbook
----------------

```yaml
---
- name: Deploy PostgreSQL for AWX
  hosts: postgres_server
  become: true
  vars:
    postgresql_database_name: awx
    postgresql_user: awx
    postgresql_user_password: "S3cretPassword"
    postgresql_allow_subnet: "10.0.0.0/24"
  tasks:
    - name: PostgreSQL
      ansible.builtin.include_role:
        name: postgresql

- name: Deploy a single-node Kubernetes cluster
  hosts: k8s_server
  become: true
  tasks:
    - name: Kubernetes
      ansible.builtin.include_role:
        name: kubernetes

- name: Deploy AWX
  hosts: k8s_server
  become: true
  vars:
    awx_postgres_host: "{{ hostvars['postgres_server']['ansible_default_ipv4']['address'] }}"
    awx_postgres_user: awx
    awx_postgres_password: "S3cretPassword"
    awx_admin_password: "AnotherS3cretPassword"
  tasks:
    - name: AWX
      ansible.builtin.include_role:
        name: awx
```

Testing
-------

This role has a Molecule scenario that brings up two VirtualBox VMs (via
Vagrant), on Ubuntu 26.04, closely mirroring a real deployment:
1. A single-node Kubernetes cluster (`kubernetes` role, `single` group).
2. A PostgreSQL server for AWX (`postgresql` role).
3. AWX itself, deployed from the Kubernetes node against that PostgreSQL
   server.

```shell
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml

# Works around molecule-plugins not registering its custom vagrant module
# with newer molecule releases.
export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

molecule test
```

Requires Vagrant and VirtualBox installed locally. AWX's images are large
and the AWX Operator does several reconcile passes, so expect this to take
considerably longer than the other roles' scenarios.

License
-------

license (BSD, MIT)

Author Information
------------------

for questions, Contact guido-_@live.nl
