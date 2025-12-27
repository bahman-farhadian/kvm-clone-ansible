# KVM VM Cloning with Ansible

Ansible playbook to clone Debian 12 VMs from a template on a KVM hypervisor.

## Features

- **Parallel execution** - All operations run concurrently for maximum speed
- Clones VMs from a template using `virt-clone`
- Pre-boot customization with `virt-customize`:
  - Sets hostname
  - Configures static IP address
  - Regenerates `/etc/machine-id` (avoids duplicate machine IDs)
  - Regenerates SSH host keys (avoids SSH key conflicts)
- Configures vCPU and RAM per VM
- **Creates snapshot** (`snapshot-a`) on each clone for easy rollback
- Waits for SSH availability
- Idempotent: skips VMs that already exist

## Prerequisites

On the hypervisor (Silenus):
- `libvirt` and `qemu-kvm` installed
- `libguestfs-tools` installed (provides `virt-customize`)
- Template VM created and working

On the control node (Selene):
- Ansible installed
- SSH key access to hypervisor as root

## Directory Structure

```
kvm-clone-ansible/
├── ansible.cfg           # Ansible configuration
├── inventory.ini         # Hypervisor connection details
├── clone-vms.yml         # Main playbook (parallel)
├── cleanup-vms.yml       # Remove cloned VMs
└── vars/
    └── vms.yml           # ← Edit this to define your VMs
```

## Usage

### 1. Edit VM definitions

Edit `vars/vms.yml` to define your VMs:

```yaml
vms:
  - name: "web-server-01"
    ip: "192.168.32.10"
    vcpu: 2
    ram_mb: 2048

  - name: "db-server-01"
    ip: "192.168.32.11"
    vcpu: 4
    ram_mb: 4096
```

### 2. Run the playbook

```bash
# Dry run (check mode)
ansible-playbook clone-vms.yml --check

# Full run
ansible-playbook clone-vms.yml

# Clone specific VMs only (override vars)
ansible-playbook clone-vms.yml -e '{"vms": [{"name": "test-vm", "ip": "192.168.32.50", "vcpu": 1, "ram_mb": 1024}]}'

# Skip snapshot creation
ansible-playbook clone-vms.yml -e create_snapshot=false
```

### 3. Connect to your VMs

```bash
ssh qwerty@192.168.32.10
```

## Execution Phases

The playbook runs in 7 phases (most run in parallel):

| Phase | Operation | Parallel |
|-------|-----------|----------|
| 0 | Pre-flight checks, build VM list | - |
| 1 | Clone VMs with `virt-clone` | ❌ (storage pool locks) |
| 2 | Customize disks (`virt-customize`) | ✅ |
| 3 | Configure resources (vCPU/RAM) | ✅ |
| 4 | Start VMs | ✅ |
| 5 | Wait for SSH | ✅ |
| 6 | Create snapshots | ✅ |
| 7 | Cleanup and summary | - |

## Configuration Options

In `vars/vms.yml`:

### Hypervisor Connection

| Variable | Description | Example |
|----------|-------------|---------|
| `kvm_host` | SSH hostname or user@ip of KVM hypervisor | `Silenus`, `root@192.168.24.12` |

### Template and Storage

| Variable | Description | Default |
|----------|-------------|---------|
| `template_vm` | Name of the template VM | `debian-bookworm` |
| `disk_pool` | Path to store VM disks | `/btbssd/QEMUKVM/stgPool` |
| `guest_interface` | Network interface inside guest | `enp1s0` |
| `network_prefix` | Subnet prefix (CIDR) | `24` |
| `network_gateway` | Gateway IP | `192.168.32.1` |
| `dns_servers` | DNS servers | `8.8.8.8` |
| `create_snapshot` | Create snapshot after cloning | `true` |
| `snapshot_name` | Name of the snapshot | `snapshot-a` |
| `snapshot_description` | Snapshot description | `initial configuration` |
| `start_after_snapshot` | Start VMs after snapshot | `true` |
| `restart_template_after` | Restart template after cloning | `false` |
| `wait_for_ssh` | Wait for SSH before continuing | `true` |

## Useful Aliases

Add these to your shell profile on the hypervisor (e.g., `~/.bashrc` or `~/.zshrc`).
They exclude the `debian-bookworm` template VM:

```bash
# Start all VMs except template
alias start-vms='for vm in $(virsh list --name --inactive | grep -v "^debian-bookworm$"); do virsh start "$vm"; done'

# Graceful shutdown all VMs except template
alias stop-vms='for vm in $(virsh list --name --state-running | grep -v "^debian-bookworm$"); do virsh shutdown "$vm"; done'

# Revert to current snapshot and restart (excludes template)
alias restore-restart-vms='for vm in $(virsh list --name --state-running | grep -v "^debian-bookworm$"); do virsh snapshot-revert "$vm" --current && virsh start "$vm"; done'

# Revert to current snapshot without starting (excludes template)
alias restore-stop-vms='for vm in $(virsh list --name --state-running | grep -v "^debian-bookworm$"); do virsh snapshot-revert "$vm" --current; done'
```

## Cleanup

To remove cloned VMs:

```bash
# Dry run (see what would be deleted)
ansible-playbook cleanup-vms.yml

# Actually delete
ansible-playbook cleanup-vms.yml -e confirm_delete=true
```

Or manually:
```bash
ssh Silenus "virsh destroy vm-name; virsh undefine vm-name --remove-all-storage --snapshots-metadata"
```

## Troubleshooting

### VM stuck at old IP
The template must be **shut down** before cloning. The playbook handles this automatically.

### SSH host key warning
Expected on first connection to a new VM. The playbook regenerates SSH host keys during customization.

### virt-customize fails
Ensure `libguestfs-tools` is installed on the hypervisor:
```bash
apt install libguestfs-tools
```

### Snapshot revert doesn't restore IP
This is expected! Each clone's `snapshot-a` contains its own hostname/IP, not the template's.
Reverting restores the clone to its initial customized state.
