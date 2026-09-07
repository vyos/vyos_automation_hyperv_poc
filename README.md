# vyos-hyperv-automation (POC)

Proof-of-concept pipeline for automating the lifecycle of VyOS network
appliances running as Hyper-V VMs: provisioning the VM, and configuring it
according to its business function (edge router, firewall, etc.), with
Netbox as the single source of truth for device data.

## Status

This is a proof of concept. Several things are deliberately simplified for
now and called out explicitly below — see **Known limitations**.

## Architecture

The pipeline is split into two independent stages, each its own playbook,
each independently triggerable:

1. **Provision / deprovision the VM** (`site/hyperv_vm.yml`)
   Creates the Hyper-V VM from a golden VyOS image for every device Netbox
   marks `staged`, attaches a cloud-init seed ISO for Day-0 bootstrap,
   starts it, waits for SSH to come up, and marks the device `active` in
   Netbox. Devices marked `decommissioning` are reported only, unless the
   run is explicitly invoked with `confirm_destroy=true` (see **Rundeck
   integration** below), in which case that specific device is torn down.

2. **Configure the appliance** (`site/vyos_configure.yml`)
   Applies Day-1 configuration over SSH (`network_cli`) using the
   `vyos.vyos` resource modules, driven by the device's role and Config
   Context data in Netbox. Runs independently of stage 1 — reconfiguring an
   existing appliance never touches the VM lifecycle.

Netbox is the source of truth throughout: VM sizing (memory/vCPU),
interfaces and IP addressing, business-function role, lifecycle status,
and role-specific configuration data (via Config Context / Local Context
Data) all come from Netbox, not hardcoded playbook values or manually
maintained inventory files.

### Day-0 / Day-1 split

- **Day-0** (cloud-init, baked into the seed ISO at provision time):
  hostname, minimal management interface, initial credentials. This is
  what gets the VM to a state where Ansible can reach it at all.
- **Day-1** (`vyos_configure.yml`, over SSH): everything else — full
  interface set, firewall, routing, NAT, VPN, HA, depending on role.

## Repo structure

```
.
├── ansible.cfg
├── requirements.yaml       # Ansible collections
├── requirements.txt        # Python packages the collections' connection
│                           # plugins need at runtime (not auto-installed
│                           # by ansible-galaxy — see note below)
├── inventory/
│   └── netbox.yml           # dynamic inventory via netbox.netbox.nb_inventory
├── group_vars/
│   ├── device_roles_hypervisor.yml   # WinRM/PSRP connection vars
│   └── platforms_vyos.yml            # network_cli connection vars
├── roles/
│   ├── hyperv_vm/           # provision/deprovision the VM (WinRM → Hyper-V)
│   ├── vyos_common/         # hostname, interfaces, addressing — every appliance
│   ├── vyos_edge_router/    # WAN, default route, NAT masquerade
│   ├── vyos_dual_wan/       # (scaffolded, not yet implemented)
│   ├── vyos_firewall/       # (scaffolded, not yet implemented)
│   ├── vyos_ha/             # (scaffolded, not yet implemented)
│   └── vyos_ipsec_vpn/      # (scaffolded, not yet implemented)
└── site/
    ├── hyperv_vm.yml         # stage 1 playbook
    └── vyos_configure.yml    # stage 2 playbook
```

## Prerequisites

- Python 3.x with `pip`
- A Hyper-V host reachable over WinRM (HTTPS, port 5986), with a golden
  VyOS image already present
- A Netbox instance, set up per **Netbox setup** below
- SSH reachability from the runner to provisioned VyOS appliances

## Setup

```bash
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yaml
```

`requirements.txt` deliberately lists Python packages that specific
collections' connection plugins need at runtime (e.g. `pywinrm`/`pypsrp`
for `microsoft.hyperv`'s WinRM/PSRP plugins, `pytz` for
`netbox.netbox`'s inventory plugin). `ansible-galaxy` only installs
collection content — it does not read or install a collection's Python
dependencies, even when a collection ships its own `requirements.txt`
internally. Those have to be discovered and listed here by hand; if a run
fails with `No module named '<x>'`, add `<x>` here rather than assuming
it's a one-off environment problem.

## Required environment variables

| Variable | Used by |
|---|---|
| `NETBOX_API` | inventory plugin, `hyperv_vm` role's Netbox status update |
| `NETBOX_TOKEN` | same |
| `HYPERV_ADMIN_PASSWORD` | `group_vars/device_roles_hypervisor.yml` |

None of these are stored in the repo. See **Known limitations** re:
credential handling.

## Netbox setup

Netbox holds every piece of data the pipeline needs — device/VM
inventory, connection targeting, sizing, and role-specific configuration.
Nothing about a specific appliance is meant to live in this repo.

**1. Device Role — `hypervisor`**
Devices → Device Roles → Add. Slug must be exactly `hypervisor` — this is
what `group_by: device_roles` in `inventory/netbox.yml` turns into the
`device_roles_hypervisor` inventory group, which `group_vars/device_roles_hypervisor.yml`
matches by name.

**2. Custom Field — `golden_vhdx_path`**
Customization → Custom Fields → Add. Type: Text. Object type: `DCIM >
Device`. This is where the Windows path to the golden VyOS image lives —
set it on the Hyper-V device itself (Devices → your device → Edit).
Surfaces in inventory under `custom_fields.golden_vhdx_path`.

**3. The Hyper-V host device**
Devices → Add Device, Role `hypervisor`. Add an interface, assign it an IP
address, then set that IP as the device's **Primary IPv4** (Edit → Primary
IPv4 address) — the inventory's `compose: ansible_host: primary_ip4.ip`
only reads from this field specifically, not any IP merely attached to an
interface.

**4. Platform — `vyos`**
Devices → Platforms → Add, slug `vyos`. Produces the `platforms_vyos`
group.

**5. Role — `edge_router`** (and similarly for other business functions as
those roles get implemented)
Same Role model, used for VMs. Slug `edge_router`.

**6. Config Context — role-level baseline**
Provisioning → Config Contexts → Add, assigned to Role `edge_router`:
```json
{
  "vyos": {
    "role": "edge_router",
    "wan_dhcp": true,
    "nat_enabled": true
  }
}
```
Every VM with this Role inherits this data automatically — surfaces
flattened at the top level of that host's inventory vars (e.g. `vyos.role`),
not nested under a `config_context` key, since `flatten_config_context: True`
is set in `inventory/netbox.yml`.

**7. Local Context Data — per-device override**
On an individual VM's own edit page, to override the shared baseline for
just that instance:
```json
{
  "vyos": {
    "wan_dhcp": false,
    "wan_gateway": "203.0.113.1"
  }
}
```
Netbox merges this on top of the Role-level Config Context, highest
precedence per key — every other `edge_router` VM without this override
keeps the shared default.

**8. The test/appliance VM itself**
Virtualization → Virtual Machines → Add: Role `edge_router`, Platform
`vyos`, Status `staged`. Add interfaces (e.g. `eth0` management with an
assigned IP, `eth1` WAN with none for DHCP), and memory/vCPU — or leave
those blank to exercise the role defaults' fallback (see `roles/hyperv_vm/defaults/main.yml`).

**Verify before running anything:**
```bash
ansible-inventory -i inventory/netbox.yml --host <name>
```
Confirm `netbox_id`, `netbox_name`, `interfaces`, `custom_fields`, `vyos.*`,
and `status.value` all render as expected for both the hypervisor and the
appliance VM.

## Running locally

```bash
# Reconcile: provisions every VM Netbox marks 'staged'.
# Devices marked 'decommissioning' are reported only, never destroyed here.
ansible-playbook -i inventory/netbox.yml site/hyperv_vm.yml \
  --forks 1 \
  -l "<optional -l pattern, omit to reconcile everything staged>"

# Destroy: only for a device already marked 'decommissioning' in Netbox.
# -l is required — this never runs unscoped.
ansible-playbook -i inventory/netbox.yml site/hyperv_vm.yml \
  --forks 1 \
  -l "<vm-name>" \
  -e "confirm_destroy=true"

# Configure: applies Day-1 config to already-provisioned appliance(s).
ansible-playbook -i inventory/netbox.yml site/vyos_configure.yml \
  -l "<optional -l pattern, omit to configure every active appliance>"
```

`--forks 1` on the Hyper-V play is not optional — see the WinRM note
below. Neither playbook takes a `target_host`/`appliance_state` variable;
targeting is entirely via `-l` and Netbox's own `status` field.

## Rundeck integration

**Job Source Control (SCM):** Rundeck's own job definitions are versioned
in a separate git repository from this one (job definitions vs. automation
content are reviewed independently). Configured under Project Settings →
Source Control Management, Git plugin, both Import and Export enabled.

**`_setup_environment`** — an internal job (not meant to be run directly),
referenced via job-ref from every payload job below. Clones this repo and
installs dependencies fresh on every execution, since the runner is
ephemeral:
```bash
rm -rf workspace
git clone --branch main --depth 1 <this-repo-url> workspace
cd workspace
pip install -r requirements.txt --break-system-packages
ansible-galaxy collection install -r requirements.yaml
```

**Three payload jobs**, each cloning fresh via `_setup_environment`, each
ending with an explicit `rm -rf /tmp/@job.execid@` cleanup step:

| Job | Playbook | Options |
|---|---|---|
| `Reconcile Appliance VMs` | `site/hyperv_vm.yml` | `netbox_api`, optional `limit` (plain text, maps to `-l`) |
| `Destroy Appliance VM` | `site/hyperv_vm.yml` `-e confirm_destroy=true` | `netbox_api`, **required** `limit` — never runs unscoped |
| `Configure Appliance` | `site/vyos_configure.yml` | `netbox_api`, optional `limit` |

`Destroy Appliance VM` should have a narrower ACL than the other two — it's
the one job in this set that permanently deletes a VM and its disk.
Reconcile only ever provisions or reports; it never deletes anything on
its own.

**Secrets:** `netbox_token` and `hyperv_admin_password` are Key
Storage-backed options (Input Type **Plain Text with Password Input**, not
**Secure Remote Authentication** — the latter withholds the value from
script steps entirely, which breaks the `export NETBOX_TOKEN=...` pattern
these jobs rely on).

**Script-step variable syntax:** inside inline script step bodies, use
Rundeck's own `@option.foo@` / `@job.execid@` token syntax, not shell-style
`${...}` — the latter is only substituted on native plugin step fields
(e.g. the Git Clone step's Base Directory), not on script content, and
will fail with `bad substitution` if used inside a script body.

**Example step content** (Reconcile job):
```bash
cd /tmp/@job.execid@/vyos-hyperv-automation

export NETBOX_API="@option.netbox_api@"
export NETBOX_TOKEN="@option.netbox_token@"
export HYPERV_ADMIN_PASSWORD="@option.hyperv_admin_password@"

LIMIT_ARG=""
if [ -n "@option.limit@" ]; then
  LIMIT_ARG="-l @option.limit@"
fi

ansible-playbook -i inventory/netbox.yml site/hyperv_vm.yml --forks 1 $LIMIT_ARG
```

## Known limitations (demo scope)

- **No dynamic credential issuance.** SSH access to VyOS appliances and
  WinRM access to the Hyper-V host both use static, pre-shared credentials
  for this POC. A production version should issue short-lived
  credentials per instance (e.g. via HashiCorp Vault) rather than reusing
  one static credential across the fleet.
- **WinRM reliability.** The Hyper-V provisioning leg communicates over
  WinRM, which has shown intermittent connection resets against the
  current lab host under concurrent load. `--forks 1` and idempotent,
  re-runnable tasks are the current mitigation. This is a network/host
  issue, not a bug in this pipeline's tasks — confirmed by reproducing
  identical failures across multiple independent client libraries
  (Terraform's provider, raw `pywinrm`, `pypsrp`, and Ansible's own WinRM
  connection plugin).
- **Single Hyper-V host.** `hyperv_target` defaults to one host (matching
  its Netbox device name); no load-spreading logic across multiple
  hypervisors yet.
- **No formal change management.** Requests are lodged by directly running
  the relevant job with the appropriate parameters. `Destroy Appliance VM`
  requires an explicit target and an explicit confirmation flag as its
  only gate; there is no separate upstream approval workflow yet.
- **Terraform is not used.** An earlier iteration of this pipeline used
  Terraform (`taliesins/hyperv` provider) for VM provisioning. This was
  descoped in favor of Ansible end-to-end (`microsoft.hyperv`) for a
  single, consistent tool and better idempotency around partial failures.
- **`vyos_edge_router` is the only implemented business-function role.**
  `vyos_firewall`, `vyos_dual_wan`, `vyos_ha`, `vyos_ipsec_vpn` are
  scaffolded but not yet implemented.