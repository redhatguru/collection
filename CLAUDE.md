# Ansible style guide

Conventions for this collection (`redhatguru.general`). Follow these for every role/playbook
change — this file is loaded automatically, so treat it as standing instructions, not a
one-off checklist.

`ansible-lint -c ansible-lint.cfg` (production profile) is the automated gate — run it after
every change and keep it at 0 failures/0 warnings. Several rules below (FQCN, `truthy`
booleans, jinja spacing) are already enforced by that gate; they're restated here so the
reasoning is clear, not because lint might let them slip.

## General

- Keep it simple — complexity that isn't earning its keep hurts productivity, not helps it.
- Optimize for readability: easy to read means easy to understand and to troubleshoot.
- Playbooks are declarative desired state, not imperative code. Write task names and
  structure that describe the end state you're synchronizing the inventory to, not a list of
  steps.
- Reuse before writing new logic: this collection's own roles first, then Galaxy/RHEL system
  roles, before inventing something from scratch.
- Give every play, block, task, and handler a `name`.
- Prefer `import`/`include` to split large task files by concern (this repo's per-file
  task layout — `dashboard.yml`, `ingress.yml`, `vip.yml`, etc. — is the existing example).
- Use privilege escalation (`become`) only on the tasks that actually need it, not blanket at
  the play level unless every task genuinely requires it.

## Roles

- Keep each role focused on one purpose (`kubernetes`, `awx`, `postgresql`, ...) — resist
  scope creep into a role that does too much.
- Keep `README.md` and `meta/main.yml` current; they're the role's contract with callers.
- Declare inter-role dependencies in `meta/main.yml`, not just implicitly via playbook
  ordering.
- Never commit real secrets (passwords, keys, tokens) into a role. Defaults may only contain
  obviously-fake placeholders the caller is expected to override (e.g. `"ChangeMe"`,
  `"ChangeMe123!"`) — matches this repo's existing pattern.

## Variables

- Name a variable so its purpose is obvious without reading its default or the task that
  consumes it:
  ```yaml
  # Good
  kubernetes_dashboard_insecure_http_port: 8080

  # Bad
  http_port: 8080
  ```
- Every role variable is prefixed with the role name — the existing convention here
  (`kubernetes_*`, `awx_*`, `postgresql_*`) — so names never collide across roles and it's
  always obvious which role a variable belongs to.
- Avoid names that collide with module option names (e.g. don't name a variable `name` or
  `state`).
- Be deliberate about precedence when choosing where to define a variable. From lowest to
  highest priority: role `defaults/` → inventory/group_vars/host_vars → play `vars` →
  `vars_files` → role `vars/` → block/task `vars` → `set_fact`/registered vars → role params
  → `extra-vars` (`-e`, always wins). When in doubt, `defaults/main.yml` is the right home for
  anything a caller might reasonably want to override — full reference:
  <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable>.

## Booleans

Always `true`/`false`. Never `yes`/`no`, `Y`/`N`, `1`/`0`, `on`/`off`. (ansible-lint's
`truthy` rule enforces this already, so this should never actually show up as a failure —
just write it right the first time.)

## Comments

- Start with `#`, one space, then a capital letter.
- Per this project's own standing guidance: don't add a comment unless the *why* is
  non-obvious — a hidden constraint, a workaround for a specific upstream bug or issue
  (this repo already does this well, e.g. the firewalld/Calico comment in
  `roles/kubernetes/tasks/install-RedHat.yml` referencing `projectcalico/calico#4382`), or
  behavior that would genuinely surprise a future reader. Don't restate what the task name or
  module already say.

## Modules

### FQCN

Always the fully-qualified collection name: `ansible.builtin.file`, never bare `file`. This
makes the source collection unambiguous. Enforced by ansible-lint (`fqcn[action-core]`) — a
bare module name fails the production-profile gate here, not just a style nit.

### Task description

The task `name` is a full, declarative sentence describing the resulting state — not a terse
imperative label:

```yaml
# Bad
- name: Create user
  ansible.builtin.user:
    name: james
    shell: /bin/bash
    groups: admins,developers
    append: true

# Good
- name: Ensure the user james exists, uses bash as the default shell, and is a member of the admins and developers groups
  ansible.builtin.user:
    name: james
    shell: /bin/bash
    groups: admins,developers
    append: true
```

### `command`/`shell`

Only use `ansible.builtin.command` or `ansible.builtin.shell` when no dedicated module covers
the job — the recurring legitimate case in this repo is `kubectl`/`kubeadm` invocations, since
`kubernetes.core` has no equivalent for things like `kubectl apply -k` (kustomize). When you
do reach for `command`/`shell`, don't rely on the exit code alone — pair it with
`changed_when` (and `failed_when` when the exit code doesn't tell the real story):

```yaml
# Bad — always reports "changed", regardless of whether anything actually changed
- name: Print "changed" to the screen
  ansible.builtin.shell: |
    echo "changed"

# Good
- name: Print "changed" to the screen
  ansible.builtin.shell: |
    echo "changed"
  register: output
  changed_when: "'changed' in output.stdout"
```

### Task key order

This repo's own existing convention (hundreds of tasks already follow it, so this is what
keeps new/edited tasks consistent with everything around them): the module comes right after
`name`; modifiers (`when`, `loop`, `register`, `become`, `environment`, `changed_when`,
`failed_when`, ...) follow below it — **not** the "module last" ordering some style guides
recommend. Match this ordering for every task you touch.

## Testing

- Validate with `ansible-lint -c ansible-lint.cfg` (production profile) before considering
  any role/playbook change done.
- Roles with a `molecule/` directory have real VirtualBox/Vagrant test scenarios — run the
  relevant scenario(s) (`molecule test` / `molecule test -s <scenario>`) for anything you
  change there, on every OS family the role supports, not just one.
