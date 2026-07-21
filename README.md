# Foreman Automated Installation and Configuration

This toolset installs and configures Foreman on a Rocky 9 host using
`ansible-foreman-create` (Python CLI) backed by Ansible playbooks.

## Local setup (control node)

You need: `ansible-foreman-create`, the `ansible/` directory, and a Python venv.
The `venv/` directory is not portable — recreate it on the control node:

```bash
python3 -m venv venv
source venv/bin/activate
pip install ansible-core PyYAML
```

### Local pip packages

| Package | Why |
|---------|-----|
| `ansible-core` | Ansible engine and all `ansible.builtin` modules |
| `PyYAML` | Required by `ansible-core` internally; also used by local Jinja2 `\| from_yaml` filters |

Note: `ansible-core` declares `PyYAML` as a dependency and will pull it in
automatically, but listing it explicitly prevents surprises if the version
constraint ever changes.

### Remote packages (installed by Ansible on the Foreman host)

The `foreman_configure` role installs these on the Foreman host during the
playbook run. They are only needed while the playbook is running — Foreman
itself (a Ruby on Rails application) does not use them at runtime.

| Package | Installed via | Why |
|---------|---------------|-----|
| `apypie` | `pip` (PyPI) | HTTP client used by all `theforeman.foreman` collection modules; no RPM available in Foreman 3.x repos for EL9 |
| `python3-pyyaml` | `dnf` (standard Rocky 9 repos) | Required by `theforeman.foreman.partition_table` and other collection modules |

### Ansible collections

Collections are stored in `~/.ansible/collections/`, not inside the venv.
Install with:

```bash
venv/bin/ansible-galaxy collection install -r ansible/requirements.yml
```

`ansible-foreman-create` also installs a `hammer` wrapper into `venv/bin/` after a
successful run. The wrapper SSHes to the Foreman host and runs the real `hammer`
CLI there. If you recreate the venv you can reinstall it manually:

```bash
cat > venv/bin/hammer << 'EOF'
#!/usr/bin/env bash
FOREMAN_HOST=$(python3 << 'PYEOF'
import re, os
f = os.path.expanduser('~/.hammer/cli.modules.d/foreman.yml')
c = open(f).read()
m = re.search(r":host:\s+'https?://([^/']+)/?'", c)
print(m.group(1) if m else '')
PYEOF
)
if [[ -z "$FOREMAN_HOST" ]]; then
    echo "hammer: cannot determine Foreman host from ~/.hammer/cli.modules.d/foreman.yml" >&2
    exit 1
fi
CMD="hammer"
for arg in "$@"; do
    CMD="$CMD $(printf '%q' "$arg")"
done
exec ssh -q -o BatchMode=yes root@"$FOREMAN_HOST" "$CMD"
EOF
chmod +x venv/bin/hammer
```

Passwordless SSH to root on the target host must be in place before running.

## Installation

### Variables

Set these shell variables to match your environment before running any commands
in this section. All subsequent commands reference them.

```bash
FOREMAN_HOST=foreman.example.com       # FQDN of the target Foreman host
DNS_DOMAIN=subdomain.example.com       # DNS domain for managed hosts
SUBNET_CIDR=192.168.200.0/24           # Managed subnet (CIDR notation)
DHCP_IFACE=eth1                        # Network interface on Foreman host for DHCP
UPSTREAM_DNS=8.8.8.8,8.8.4.4           # Comma-separated upstream DNS forwarder IPs
SSL_CERT=/path/to/wildcard.pem         # SSL certificate file (PEM bundle: server cert + CA chain)
SSL_KEY=/path/to/wildcard.key          # SSL private key file
```

Optional — override subnet defaults:

```bash
SUBNET_GATEWAY=192.168.200.254         # Gateway IP (default: last usable host IP in subnet)
SUBNET_PRIMARY_DNS=192.168.200.1       # Primary DNS IP (default: Foreman host IP on subnet)
SUBNET_SECONDARY_DNS=                  # Secondary DNS IP (default: none)
```

### Run

```bash
source venv/bin/activate
./ansible-foreman-create \
  --host "$FOREMAN_HOST" \
  --no-new-name \
  --dns "$DNS_DOMAIN" \
  --subnet "$SUBNET_CIDR" \
  --dhcp-interface "$DHCP_IFACE" \
  --upstream-dns "$UPSTREAM_DNS" \
  --ssl-cert "$SSL_CERT" \
  --ssl-key "$SSL_KEY"
```

Pass the optional flags to override subnet defaults:

```bash
./ansible-foreman-create \
  --host "$FOREMAN_HOST" \
  --no-new-name \
  --dns "$DNS_DOMAIN" \
  --subnet "$SUBNET_CIDR" \
  --subnet-gateway "$SUBNET_GATEWAY" \
  --subnet-primary-dns "$SUBNET_PRIMARY_DNS" \
  --subnet-secondary-dns "$SUBNET_SECONDARY_DNS" \
  --dhcp-interface "$DHCP_IFACE" \
  --upstream-dns "$UPSTREAM_DNS" \
  --ssl-cert "$SSL_CERT" \
  --ssl-key "$SSL_KEY"
```

`ansible-foreman-create` derives all subnet values (network address, mask, reverse
DNS zone, first-host IP, gateway, DNS servers, DHCP range) from `--subnet`
automatically. Override individual values with the optional flags above.

### Post-run

- Hammer CLI config is written to `~/.hammer/cli.modules.d/foreman.yml`
- `$DHCP_IFACE` is configured with a static IP (first host address in `$SUBNET_CIDR`)
- Configure the switch port for the correct VLAN before provisioning managed hosts

---

## Running individual roles via Ansible

The playbook roles are tagged so they can be run independently with `--tags`.
This is useful for re-applying a specific piece of configuration without
running the full install (which re-runs foreman-installer and takes 5–10 min).

### Variables

Define these once in your shell session. All role commands below reference them.
Derived values (network address, mask, prefix, reverse zone) must be filled in
to match the chosen subnet — `ansible-foreman-create` computes these automatically
during a full install, but manual role runs require them spelled out.

```bash
# Adjust all values to match your environment
FOREMAN_HOST=foreman.example.com
DNS_DOMAIN=subdomain.example.com
SUBNET_CIDR=192.168.200.0/24
SUBNET_NET=192.168.200.0
SUBNET_MASK=255.255.255.0
SUBNET_PREFIX=24
SUBNET_GATEWAY=192.168.200.254    # last usable host IP; override if needed
SUBNET_PRIMARY_DNS=192.168.200.1  # Foreman host IP; override if needed
SUBNET_SECONDARY_DNS=             # leave blank for no secondary DNS
DHCP_IP=192.168.200.1
DHCP_RANGE_START=192.168.200.2    # first IP after Foreman host
DHCP_RANGE_END=192.168.200.253    # last IP before gateway
DHCP_DNS_SERVERS="192.168.200.1"  # primary only; append ", <ip>" for secondary
DHCP_IFACE=eth1
UPSTREAM_DNS_LIST='["8.8.8.8","8.8.4.4"]'  # JSON list of upstream DNS forwarder IPs
REVERSE_ZONE=200.168.192.in-addr.arpa
SSL_CERT=/path/to/wildcard.pem
SSL_KEY=/path/to/wildcard.key
```

### Inventory and extra-vars

Create the inventory file:

```bash
cat > /tmp/foreman_inv.ini << EOF
[foreman]
${FOREMAN_HOST} ansible_user=root
EOF
```

Build the extra-vars JSON (shell variables are expanded into the heredoc):

```bash
EVARS=$(python3 << PYEOF
import json
d = {
    "foreman_fqdn":       "$FOREMAN_HOST",
    "new_fqdn":           "",
    "dns_domain":         "$DNS_DOMAIN",
    "subnet_cidr":        "$SUBNET_CIDR",
    "subnet_network":     "$SUBNET_NET",
    "subnet_mask":        "$SUBNET_MASK",
    "subnet_gateway":     "$SUBNET_GATEWAY",
    "subnet_primary_dns": "$SUBNET_PRIMARY_DNS",
    "dhcp_interface":     "$DHCP_IFACE",
    "dhcp_interface_ip":  "$DHCP_IP",
    "dhcp_prefix":        "$SUBNET_PREFIX",
    "dhcp_range_start":   "$DHCP_RANGE_START",
    "dhcp_range_end":     "$DHCP_RANGE_END",
    "dhcp_dns_servers":   "$DHCP_DNS_SERVERS",
    "upstream_dns_servers": $UPSTREAM_DNS_LIST,
    "dns_reverse_zone":   "$REVERSE_ZONE",
    "ssl_cert_src":       "$SSL_CERT",
    "ssl_key_src":        "$SSL_KEY",
}
if "$SUBNET_SECONDARY_DNS":
    d["subnet_secondary_dns"] = "$SUBNET_SECONDARY_DNS"
print(json.dumps(d))
PYEOF
)
```

### Quick cheat sheet

All commands below assume `/tmp/foreman_inv.ini` and `$EVARS` are set as described above.

```bash
venv/bin/ansible-playbook -i /tmp/foreman_inv.ini ansible/site.yml [--tags <tag>] --extra-vars "$EVARS"
```

| Tag | Scope |
|-----|-------|
| _(none)_ | Full install: runs all four roles in order (firewall, install, distro, configure) |
| `foreman_firewall` | Open required ports in iptables; reorder rules if needed |
| `foreman_install` | Full Foreman installation: hostname, SSL, sysctl, foreman-installer, named, DHCP, hammer config |
| `foreman_distro` | Scan `/mnt/iso`, mount ISOs under `/srv/distro`, write Apache distro config |
| `foreman_configure` | Domain, subnet, OS entries, templates, partition tables, hostgroups, global settings |
| `foreman_sysctl` | Kernel sysctl parameters only (subset of `foreman_install`) |
| `foreman_named` | named/DNS options only (subset of `foreman_install`) |
| `foreman_dhcp` | DHCP subnet options only (subset of `foreman_install`) |

---

### Check and correct firewall rules

Detects whether the required ports are open and positioned before the default
REJECT rule. Removes and re-inserts all rules if any are missing or out of
order. Idempotent — prints "unchanged" and makes no changes if already correct.

```bash
venv/bin/ansible-playbook -i /tmp/foreman_inv.ini ansible/site.yml \
  --tags foreman_firewall --extra-vars "$EVARS"
```

To inspect the current iptables INPUT chain on the host directly:

```bash
ssh root@"$FOREMAN_HOST" 'iptables -L INPUT --line-numbers -n'
```

### Mount ISOs and configure the distro HTTP server

Scans `/mnt/iso` for ISO files, mounts any found under `/srv/distro/<Distro>/<version>/<arch>`,
adds `/etc/fstab` entries, and writes `/etc/httpd/conf.d/distro.conf`.
Idempotent — already-mounted ISOs and existing fstab entries are left unchanged.
Run this after dropping new ISOs into `/mnt/iso` to pick them up without a full
install.

```bash
venv/bin/ansible-playbook -i /tmp/foreman_inv.ini ansible/site.yml \
  --tags foreman_distro --extra-vars "$EVARS"
```

To inspect what is currently mounted under `/srv/distro`:

```bash
ssh root@"$FOREMAN_HOST" 'findmnt -t iso9660 -o SOURCE,TARGET'
```

### Re-apply domain, subnet, and OS configuration

Re-runs the `theforeman.foreman` API calls that create the DNS domain, subnet,
and operating system entries in Foreman. Safe to run repeatedly — all modules
are idempotent. Useful if any of these were changed or deleted in the UI, or
after mounting new ISOs (OS entries are derived from currently mounted ISOs).

```bash
venv/bin/ansible-playbook -i /tmp/foreman_inv.ini ansible/site.yml \
  --tags foreman_configure --extra-vars "$EVARS"
```

To see what operating systems Foreman currently has defined:

```bash
ssh root@"$FOREMAN_HOST" 'hammer os list'
```

### Push custom templates and partition tables

Reads any content files in the drop-in directories and pushes them to Foreman.
Idempotent — existing items with unchanged content are not re-uploaded.
All of the following are picked up in a single `foreman_configure` run:

```bash
venv/bin/ansible-playbook -i /tmp/foreman_inv.ini ansible/site.yml \
  --tags foreman_configure --extra-vars "$EVARS"
```

| Directory | Foreman type |
|-----------|-------------|
| `ansible/partition_tables/` | Partition table |
| `ansible/provisioning_templates/` | Provisioning template (kind: provision) |
| `ansible/pxelinux_templates/` | Provisioning template (kind: PXELinux) |
| `ansible/pxegrub2_templates/` | Provisioning template (kind: PXEGrub2) |
| `ansible/snippets/` | Provisioning template (kind: snippet) |

To inspect what is currently in Foreman:

```bash
ssh root@"$FOREMAN_HOST" 'hammer partition-table list'
ssh root@"$FOREMAN_HOST" 'hammer template list'
```

### Re-run the full install

Runs all roles in order: firewall → install → configure.
`foreman-installer` is idempotent but takes 5–10 minutes even when nothing
changes. Prefer individual role tags for targeted fixes.

```bash
venv/bin/ansible-playbook -i /tmp/foreman_inv.ini ansible/site.yml \
  --extra-vars "$EVARS"
```

---

## Custom templates (drop-in mechanism)

All drop-in directories follow the same conventions:

- **Content files**: no file extension; the filename becomes the name in Foreman.
- **OS associations**: an optional `os_associations.yml` in the same directory.
- **Git**: content files and `*.yml` sidecar files are gitignored — never committed.
  `*.example` files are committed and serve as templates to copy from.

Each directory ships with an `os_associations.yml.example` showing the expected
format. Copy it to `os_associations.yml` and edit as needed. Similarly,
`pxelinux_templates/` and `pxegrub2_templates/` include a `local-boot.yml.example`.

Run `--tags foreman_configure` to push any newly added files to Foreman.

### Directory → Foreman type mapping

| Directory | Foreman type | Module |
|-----------|-------------|--------|
| `ansible/partition_tables/` | Partition table | `theforeman.foreman.partition_table` |
| `ansible/provisioning_templates/` | Provision template | `theforeman.foreman.provisioning_template` (kind: provision) |
| `ansible/pxelinux_templates/` | PXELinux template | `theforeman.foreman.provisioning_template` (kind: PXELinux) |
| `ansible/pxegrub2_templates/` | PXEGrub2 template | `theforeman.foreman.provisioning_template` (kind: PXEGrub2) |
| `ansible/snippets/` | Snippet | `theforeman.foreman.provisioning_template` (kind: snippet) |

### Adding a partition table

Create a file in `ansible/partition_tables/` with no extension (filename = Foreman name).
Content must be a valid Foreman partition table (Ruby ERB):

```
<%#
kind: ptable
name: RHEL9_RHEL10
%>
zerombr
clearpart --all --initlabel
autopart --type lvm
```

### Adding a provisioning template

Create a file in the appropriate directory. The content must be a valid Foreman
ERB template. Foreman reads the `kind:` metadata from the ERB header comment,
but the module's `kind` parameter (set by the directory) takes precedence:

```
<%#
kind: provision
name: My Kickstart
%>
... kickstart content ...
```

For PXELinux templates (`ansible/pxelinux_templates/`):

```
<%#
kind: PXELinux
name: My PXELinux
%>
DEFAULT linux
...
```

For snippets (`ansible/snippets/`):

```
<%#
kind: snippet
name: My Snippet
%>
... reusable snippet content ...
```

### OS associations

Each drop-in directory supports an optional `os_associations.yml` that maps
template/table names to the operating system titles as they appear in Foreman.
The file is also gitignored and need not exist until OS associations are required.

```yaml
# ansible/provisioning_templates/os_associations.yml
My Kickstart:
  - "Red Hat Enterprise Linux 9.4"
  - "Red Hat Enterprise Linux 10.0"
  - "Rocky Linux 9.1"
```

The same format applies to `partition_tables/`, `provisioning_templates/`, `pxelinux_templates/`, and `pxegrub2_templates/`. Snippets are included by other templates and do not have OS associations — `os_associations.yml` is not supported in `snippets/`.

OS titles must match exactly — check with:

```bash
ssh root@"$FOREMAN_HOST" 'hammer os list'
```

The OS association task queries each item's current OS list and **adds** the
specified OSes without removing existing ones. Idempotent. If a template name
or OS title does not exist in Foreman, the task prints a warning and skips that
entry without failing the playbook run.

### OS default templates

`provisioning_templates/`, `pxelinux_templates/`, and `pxegrub2_templates/` each
support an optional `OS_Defaults:` section in `os_associations.yml`. This sets
the per-OS default template of that kind — the template Foreman selects
automatically when building or PXE-booting a host with that OS and no per-host
override is configured.

```yaml
# ansible/pxelinux_templates/os_associations.yml
My PXELinux Template:
  - "RHEL 9.4"
  - "Rocky 9.1"

OS_Defaults:
  "RHEL 9.4":
    template: "My PXELinux Template"
  "Rocky 9.1":
    template: "My PXELinux Template"
```

Rules:
- `OS_Defaults` is optional. Omitting it leaves existing Foreman defaults unchanged.
- The `template:` value must be a template that is already associated with that OS
  (either listed in the same file or previously associated by another means). If it
  is not associated, a warning is printed and the default is not set.
- If the OS title or template name is not found in Foreman, a warning is printed
  and that entry is skipped. The playbook does not fail.
- Idempotent — if the default is already set to the named template, no change is made.

To inspect current OS default templates:

```bash
ssh root@"$FOREMAN_HOST" 'hammer os list'
ssh root@"$FOREMAN_HOST" 'hammer os info --title "RHEL 9.4"'
```

### Hostgroups

Hostgroups are defined in `ansible/hostgroups.yml` (gitignored — may contain
root passwords). Copy the example and edit:

```bash
cp ansible/hostgroups.yml.example ansible/hostgroups.yml
```

The file uses a `foreman_hostgroups:` top-level key containing a list of
hostgroup definitions. It is loaded with Ansible `include_vars`, so
Ansible Vault-encrypted strings work for sensitive values like `root_pass`.

**Field reference:**

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Hostgroup leaf name as it will appear in Foreman |
| `parent` | no | Full name of an existing parent hostgroup |
| `description` | no | Free-text description |
| `domain` | no | DNS domain name (must exist in Foreman) |
| `subnet` | no | IPv4 subnet name (must exist in Foreman) |
| `architecture` | no | CPU architecture, e.g. `x86_64` |
| `operatingsystem` | no | OS title — run `hammer os list` for exact values |
| `medium` | no | Installation media name (must exist in Foreman) |
| `ptable` | no | Partition table name (must exist in Foreman) |
| `pxe_loader` | no | `PXELinux BIOS`, `Grub2 UEFI`, `Grub2 UEFI SecureBoot`, etc. |
| `root_pass` | no | Root password for provisioned hosts |

**Inheritance:** A child hostgroup lists only the fields it needs to override.
Any field omitted in the child is inherited from the parent. For example, to
have one hostgroup for BIOS boot and another for UEFI with everything else the
same:

```yaml
foreman_hostgroups:
  - name: "RHEL BIOS"
    domain: "subdomain.example.com"
    subnet: "192.168.200.0/24"
    architecture: "x86_64"
    operatingsystem: "RHEL 9.4"
    medium: "RHEL Local"
    ptable: "My Partition Table"
    pxe_loader: "PXELinux BIOS"
    root_pass: "changeme"

  - name: "RHEL UEFI"
    parent: "RHEL BIOS"
    pxe_loader: "Grub2 UEFI"
```

**Important:** list parent hostgroups before any children that reference them —
entries are processed in order, and a child whose parent does not yet exist in
Foreman will fail.

All hostgroups are associated with the default organization and location,
controlled by the `foreman_default_organization` and `foreman_default_location`
role variables (both default to `"Default Organization"` / `"Default Location"`).
Override via `--extra-vars` if your Foreman uses different names.

To run hostgroup configuration independently:

```bash
venv/bin/ansible-playbook -i /tmp/foreman_inv.ini ansible/site.yml \
  --tags foreman_configure --extra-vars "$EVARS"
```

To inspect current hostgroups:

```bash
ssh root@"$FOREMAN_HOST" 'hammer hostgroup list'
ssh root@"$FOREMAN_HOST" 'hammer hostgroup info --name "RHEL BIOS"'
```

### Local boot global settings (PXELinux and PXEGrub2 only)

Foreman has two global provisioning settings that control which template is used
when a managed host should boot locally (i.e. from its own disk rather than PXE):

| Foreman setting | API name |
|-----------------|----------|
| Local boot PXELinux template | `local_boot_PXELinux` |
| Local boot PXEGrub2 template | `local_boot_PXEGrub2` |

To configure these, create `local-boot.yml` in the relevant template directory:

```yaml
# ansible/pxelinux_templates/local-boot.yml
template: "PXELinux default local boot"
```

```yaml
# ansible/pxegrub2_templates/local-boot.yml
template: "PXEGrub2 default local boot"
```

The `template:` value must be the exact name of a template that already exists
in Foreman — either one of Foreman's built-in templates or a custom template
that has been uploaded from the same drop-in directory. The task verifies the
template exists and has the correct kind before applying the setting, and prints
a warning (without failing) if the named template is not found.

To check current values:

```bash
ssh root@"$FOREMAN_HOST" 'hammer settings list --search "local_boot"'
```

Both `local-boot.yml` files are gitignored (covered by the `*` exclusion in
`.gitignore`) and are optional — omitting the file leaves the Foreman setting
unchanged.

### File layout reference

```
ansible/
  partition_tables/              ← content files gitignored; *.example committed
    .gitkeep
    os_associations.yml.example  ← copy to os_associations.yml and edit
    RHEL9_RHEL10                 ← content file (no extension) — gitignored
    os_associations.yml          ← optional; maps names → OS list — gitignored
  provisioning_templates/        ← kind: provision
    .gitkeep
    os_associations.yml.example
    My_Kickstart                 ← content file — gitignored
    os_associations.yml          ← optional — gitignored
  pxelinux_templates/            ← kind: PXELinux
    .gitkeep
    local-boot.yml.example       ← copy to local-boot.yml and edit
    os_associations.yml.example
    local-boot.yml               ← optional; sets "Local boot PXELinux template" — gitignored
    os_associations.yml          ← optional — gitignored
  pxegrub2_templates/            ← kind: PXEGrub2
    .gitkeep
    local-boot.yml.example
    os_associations.yml.example
    local-boot.yml               ← optional; sets "Local boot PXEGrub2 template" — gitignored
    os_associations.yml          ← optional — gitignored
  snippets/                      ← kind: snippet; no OS associations
    .gitkeep
```
