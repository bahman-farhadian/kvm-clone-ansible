# KVM VM Cloning with Ansible

Ansible playbook to clone Debian 12 VMs from a template on a KVM hypervisor.

## Features

- Clones VMs from a template using `virt-clone`
- Pre-boot customization with `virt-customize`:
  - Sets hostname
  - Configures static IP address
  - Regenerates `/etc/machine-id` (avoids duplicate machine IDs)
  - Regenerates SSH host keys (avoids SSH key conflicts)
- Configures vCPU and RAM per VM
- Waits for SSH availability before continuing
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
├── clone-vms.yml         # Main playbook
├── vars/
│   └── vms.yml           # VM definitions (edit this!)
└── tasks/
    └── clone-single-vm.yml
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

# Clone only specific VMs (by name)
ansible-playbook clone-vms.yml -e '{"vms": [{"name": "test-vm", "ip": "192.168.32.50", "vcpu": 1, "ram_mb": 1024}]}'
```

### 3. Connect to your VMs

```bash
ssh qwerty@192.168.32.10
```

## Configuration Options

In `vars/vms.yml`:

| Variable | Description | Default |
|----------|-------------|---------|
| `template_vm` | Name of the template VM | `debian-bookworm` |
| `disk_pool` | Path to store VM disks | `/btbssd/QEMUKVM/stgPool` |
| `guest_interface` | Network interface inside guest | `enp1s0` |
| `network_prefix` | Subnet prefix (CIDR) | `24` |
| `network_gateway` | Gateway IP | `192.168.32.1` |
| `dns_servers` | DNS servers | `8.8.8.8` |
| `restart_template_after` | Restart template after cloning | `false` |
| `wait_for_ssh` | Wait for SSH before next VM | `true` |

## Cleanup

To remove a cloned VM:

```bash
ssh Silenus "virsh destroy vm-name; virsh undefine vm-name --remove-all-storage"
```

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
