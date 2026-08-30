pxeboot
=======

Sets up a PXE network boot server: proxy-DHCP + TFTP via `dnsmasq`, with a
PXELINUX boot menu for BIOS clients and direct UEFI boot support, both
serving [Memtest86+](https://www.memtest.org/) as the first (and so far
only) boot option.

Proxy-DHCP mode means this role never hands out IP leases itself — it only
answers PXE-specific DHCP requests — so it's safe to run alongside an
existing DHCP server on the same network. There is no full standalone-DHCP
mode (yet); proxy mode is the deliberate, safe default for a first PXE
rollout.

How clients boot
-----------------

- **BIOS clients** are handed `pxelinux.0`, which loads its own menu from
  `pxelinux.cfg/default` on the TFTP server. That menu currently has one
  entry: Memtest86+.
- **UEFI x86_64 clients** (detected via DHCP option 93, client-arch 7) boot
  straight into Memtest86+'s `.efi` build — no menu yet, since it's the
  only thing to boot. A UEFI GRUB netboot menu can be added once there's
  more than one entry to choose between.

Requirements
------------

- A network interface on the subnet PXE clients boot from
  (`pxeboot_interface`), and the DHCP-proxy range for that subnet
  (`pxeboot_dhcp_range_start`/`pxeboot_dhcp_range_end`).
- Outbound internet access to `www.memtest.org` to fetch the Memtest86+
  release archive (its SHA256 checksum is verified against the value in
  `pxeboot_memtest_checksum` — see defaults/main.yml).
- Client hardware/firmware that actually attempts a PXE boot (enabled in
  BIOS/UEFI setup, and typically requires the client to be on the same L2
  segment as this server, or a DHCP/PXE relay in between).

### Collections

This role needs `ansible.posix` and `community.general` (already declared
in the collection's own `requirements.yml`/`galaxy.yml`, and in this
role's own `requirements.yml`):

```shell
ansible-galaxy collection install -r requirements.yml
```

Role Variables
--------------

- `pxeboot_interface` (default: the host's default-route interface) —
  network interface dnsmasq listens for DHCP-proxy and TFTP requests on.
- `pxeboot_dhcp_range_start` / `pxeboot_dhcp_range_end` (no default, must
  both be set) — the subnet range PXE clients boot from. Proxy-DHCP mode
  still needs to know which subnet to answer PXE requests on, even though
  it never hands out leases itself.
- `pxeboot_tftp_root` (default `/var/lib/tftpboot`) — where boot files
  (bootloader, menu config, boot images) are served from via TFTP.
- `pxeboot_memtest_version` (default `"8.10"`) — Memtest86+ release fetched
  from memtest.org.
- `pxeboot_memtest_checksum` (default matches `pxeboot_memtest_version`'s
  own published SHA256) — **must be updated together with
  `pxeboot_memtest_version`**, from
  `https://www.memtest.org/download/v<version>/sha256sum.txt` (the
  `mt86plus_<version>.binaries.zip` entry). The download fails closed if
  they don't match.
- `pxeboot_firewall_enabled` (default `true`) — open the host firewall
  (`ufw`/`firewalld`) for DHCP-proxy (67, 4011) and TFTP (69) traffic.

Example Playbook
----------------

```yaml
---
- name: Deploy a PXE boot server
  hosts: pxe_server
  become: true
  vars:
    pxeboot_dhcp_range_start: "10.0.0.50"
    pxeboot_dhcp_range_end: "10.0.0.100"
  tasks:
    - name: PXE boot server
      ansible.builtin.include_role:
        name: pxeboot
```

Testing
-------

This role has a Molecule scenario that brings up two VirtualBox VMs (via
Vagrant) — Ubuntu 24.04 and Rocky Linux 9 — each configured independently
as its own PXE boot server, so both of this role's `os_family` branches
are actually exercised:

```shell
pip install molecule "molecule-plugins[vagrant]" python-vagrant
ansible-galaxy collection install -r requirements.yml

# Works around molecule-plugins not registering its custom vagrant module
# with newer molecule releases.
export ANSIBLE_LIBRARY="$(python3 -c 'import molecule_plugins, os; print(os.path.join(os.path.dirname(molecule_plugins.__file__), "vagrant", "modules"))')"

molecule test
```

Verification downloads `pxelinux.0` back over real TFTP from each VM (not
just checking the file exists on disk) and diffs it against the source
file, to confirm the server actually serves what it claims to.

Requires Vagrant and VirtualBox installed locally.

License
-------

license (BSD, MIT)

Author Information
------------------

for questions, Contact guido-_@live.nl
