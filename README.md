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
   Creates (or removes) the Hyper-V VM from a golden VyOS image, attaches a
   cloud-init seed ISO for Day-0 bootstrap, starts it, waits for SSH to come
   up, and marks the device `active` in Netbox.

2. **Configure the appliance** (`site/vyos_configure.yml`)
   Applies Day-1 configuration over SSH (`network_cli`) using the
   `vyos.vyos` resource modules, driven by the device's role and Config
   Context data in Netbox. Runs independently of stage 1 — reconfiguring an
   existing appliance never touches the VM lifecycle.

Netbox is the source of truth throughout: VM sizing (memory/vCPU),
interfaces and IP addressing, business-function role, and role-specific
configuration data (via Config Context / Local Context Data) all come from
Netbox, not hardcoded playbook values or manually maintained inventory
files.

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
- A Netbox instance with devices/VMs modeled, and an API token
- SSH reachability from the runner to provisioned VyOS appliances

## Setup

```bash
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yaml
```

`requirements.txt` deliberately lists Python packages that specific
collections' connection plugins need at runtime (e.g. `pywinrm`/`pypsrp`
for `microsoft.hyperv`'s WinRM/PSRP plugins). `ansible-galaxy` only
installs collection content — it does not read or install a collection's
Python dependencies, even when a collection ships its own
`requirements.txt` internally. Those have to be discovered and listed here
by hand.

## Required environment variables

| Variable | Used by |
|---|---|
| `NETBOX_API` | inventory plugin, `hyperv_vm` role's Netbox status update |
| `NETBOX_TOKEN` | same |
| `HYPERV_ADMIN_PASSWORD` | `group_vars/device_roles_hypervisor.yml` |

None of these are stored in the repo. See **Known limitations** re:
credential handling.

## Running locally

```bash
# Provision (or deprovision) a VM
ansible-playbook -i inventory/netbox.yml site/hyperv_vm.yml \
  --forks 1 \
  -e "vm_name=<name>" -e "appliance_state=present"

# Configure an appliance
ansible-playbook -i inventory/netbox.yml site/vyos_configure.yml \
  -e "target_host=<name>"
```

`--forks 1` on the Hyper-V play is not optional — see the WinRM note
below.

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
- **Single Hyper-V host.** `hyperv_target` defaults to one host; no
  load-spreading logic across multiple hypervisors yet.
- **No formal change management.** Requests are lodged by directly running
  the relevant job with the appropriate parameters. There is currently no
  separate request/approval step upstream of execution.
- **Terraform is not used.** An earlier iteration of this pipeline used
  Terraform (`taliesins/hyperv` provider) for VM provisioning. This was
  descoped in favor of Ansible end-to-end (`microsoft.hyperv`) for a
  single, consistent tool and better idempotency around partial failures.
- **Roles beyond `vyos_edge_router` are scaffolded but not implemented**
  (`vyos_firewall`, `vyos_dual_wan`, `vyos_ha`, `vyos_ipsec_vpn`).

## Rundeck integration

_To be documented._
