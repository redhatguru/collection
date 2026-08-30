pxeboot
=======

Sets up a PXE network boot server: TFTP via `dnsmasq`, with an optional
proxy-DHCP service, a PXELINUX boot menu for BIOS clients, and direct UEFI
boot support, both serving [Memtest86+](https://www.memtest.org/) as the
first (and so far only) boot option.

Proxy-DHCP (`pxeboot_dhcp_proxy_enabled`, default `true`) never hands out
IP leases itself — it only answers PXE-specific DHCP requests — so it's
safe to run alongside an existing DHCP server on the same network. If you
already run your own DHCP server and would rather configure PXE booting
there directly, set `pxeboot_dhcp_proxy_enabled: false`: this host then
only serves TFTP, and the role's final report prints the exact
next-server/filename settings to add to your DHCP server instead. There is
no full standalone-DHCP mode (this role never hands out leases itself
either way).

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
  (`pxeboot_interface`). If using the built-in proxy-DHCP (the default),
  also an address within that subnet (`pxeboot_dhcp_proxy_address`).
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
  network interface dnsmasq listens for TFTP (and DHCP-proxy, if enabled)
  requests on.
- `pxeboot_dhcp_proxy_enabled` (default `true`) — run dnsmasq's
  proxy-DHCP service. Set to `false` if you already run your own DHCP
  server and want to configure PXE booting there instead — see "Using
  your own DHCP server" below.
- `pxeboot_dhcp_proxy_address` (no default, must be set when
  `pxeboot_dhcp_proxy_enabled` is true) — an address within the subnet
  PXE clients boot from. Proxy-DHCP mode still needs to know which
  (directly-connected) subnet to answer PXE requests on, even though it
  never hands out leases itself — dnsmasq derives the netmask from the
  interface automatically.
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
  (`ufw`/`firewalld`) for TFTP (69), and for DHCP-proxy (67, 4011) too
  when `pxeboot_dhcp_proxy_enabled` is true.

Using your own DHCP server
---------------------------

With `pxeboot_dhcp_proxy_enabled: false`, this host only serves TFTP —
add these to your own DHCP server's configuration instead (the role's
final report prints this too, with the actual IP filled in):

- **next-server** / TFTP server address: this host's IP.
- **filename** (BIOS / default): `pxelinux.0`.
- **filename for UEFI x86_64 clients** (DHCP option 93 = 7):
  `images/memtest/memtest.efi`. How to make this conditional on client
  architecture depends on your DHCP server software — e.g. a client class
  matching option 93 in ISC dhcpd/Kea, or a vendor-class policy on
  Windows Server DHCP.

Example Playbook
----------------

```yaml
---
- name: Deploy a PXE boot server
  hosts: pxe_server
  become: true
  vars:
    pxeboot_dhcp_proxy_address: "10.0.0.50"
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
