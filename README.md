# KVM VM Cloning with Ansible

Ansible playbook to clone Debian 12 VMs from a template on a KVM hypervisor.

## Features

- **Parallel execution** - All operations run concurrently for maximum speed
- Clones VMs from a template using `virt-clone` (`clone_mode`: `auto`, `sequential`, `parallel`)
- Pre-boot customization with `virt-customize`
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

On the control node (any Linux host):
- Python 3.10+ (or compatible Python 3)
- SSH key access to hypervisor as root (or a user with required libvirt permissions)
- A local Python virtual environment for this project (recommended and documented below)

## Control Node Setup (Version-Pinned venv)

Use a project-local virtual environment so this repo does not depend on the system-wide Ansible version.

```bash
cd kvm-clone-ansible

# Create and activate a local virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Upgrade pip and install a pinned Ansible version
python -m pip install --upgrade pip
python -m pip install "ansible-core==2.16.14"

# Verify toolchain from the venv
ansible --version
ansible-playbook --version
```

Notes:
- Keep the venv activated while running playbooks from this repository.
- If you open a new shell, run `source .venv/bin/activate` again.
- To remove the environment later: `rm -rf .venv`

## Quick Start After Git Clone

```bash
git clone <your-repo-url>
cd kvm-clone-ansible
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install "ansible-core==2.16.14"
```

Edit only one file:
```bash
vim vars/vms.yml
```

Run:
```bash
ansible-playbook clone-vms.yml --check
ansible-playbook clone-vms.yml
```

## Directory Structure

```
kvm-clone-ansible/
ansible.cfg
inventory.ini
clone-vms.yml
cleanup-vms.yml
vars/vms.yml
```

## Usage

### 1. Edit one file only: `vars/vms.yml`

All user-editable settings live in `vars/vms.yml`:
- Hypervisor target (`kvm_host`)
- Template/storage/network settings
- Clone behavior flags (snapshot, SSH wait, template restart)
- Cleanup safety flag (`confirm_delete`)
- VM list (`vms`)

Example:

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

### 2. Run the playbook (from activated venv)

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

## Possible Commands

Run from the project root with the venv activated:

```bash
# Activate local environment
source .venv/bin/activate

# Validate playbook YAML/syntax
ansible-playbook clone-vms.yml --syntax-check
ansible-playbook cleanup-vms.yml --syntax-check

# Preview clone changes (safe dry run)
ansible-playbook clone-vms.yml --check

# Execute clone workflow
ansible-playbook clone-vms.yml

# Execute clone for verbose troubleshooting
ansible-playbook clone-vms.yml -vv

# Override one variable without editing file (example)
ansible-playbook clone-vms.yml -e wait_for_ssh=false

# Cleanup preview (default because confirm_delete=false in vars/vms.yml)
ansible-playbook cleanup-vms.yml

# Actual cleanup, option 1: temporary override
ansible-playbook cleanup-vms.yml -e confirm_delete=true

# Actual cleanup, option 2: set confirm_delete=true in vars/vms.yml, then run
ansible-playbook cleanup-vms.yml
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
| 1 | Clone VMs with `virt-clone` | Configurable (`clone_mode`) |
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
| `hypervisor_python_interpreter` | Python path on KVM host for Ansible modules | `/usr/bin/python3` |

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
| `confirm_delete` | Allow destructive cleanup in `cleanup-vms.yml` | `false` |
| `clone_mode` | Clone behavior: `auto`, `sequential`, `parallel` | `auto` |
| `clone_async_timeout_sec` | Timeout per `virt-clone` job (seconds) | `1800` |
| `clone_wait_retries` | Poll retry count while waiting for clone jobs | `180` |
| `clone_wait_delay_sec` | Poll delay between clone status checks (seconds) | `10` |

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
# Dry run (see what would be deleted; default with confirm_delete=false in vars/vms.yml)
ansible-playbook cleanup-vms.yml

# Actually delete (temporary override)
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

### `virt-clone` fails with `cannot open directory ... No such file or directory`
Your `disk_pool` in `vars/vms.yml` does not exist on the KVM host.
Find a valid storage path on the hypervisor and update `disk_pool`:
```bash
virsh pool-list --all
virsh pool-dumpxml <pool-name> | grep -oP '(?<=<path>).*?(?=</path>)'
```
Then run again:
```bash
ansible-playbook clone-vms.yml --check
ansible-playbook clone-vms.yml
```

### `virt-clone` fails with `pool '<name>' has asynchronous jobs running`
Your libvirt storage pool does not allow concurrent clone jobs.
Set this in `vars/vms.yml`:
```yaml
clone_mode: "auto"
```
`auto` first tries parallel clone jobs, then retries only pool-locked VMs sequentially.
Use `clone_mode: "parallel"` only if your pool supports concurrent jobs.
Use `clone_mode: "sequential"` for strict predictability.

### Snapshot revert doesn't restore IP
This is expected! Each clone's `snapshot-a` contains its own hostname/IP, not the template's.
Reverting restores the clone to its initial customized state.
