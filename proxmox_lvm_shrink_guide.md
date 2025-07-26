# How to Safely Shrink LVM Volumes for Windows VMs in Proxmox

This guide walks you through safely reducing the size of LVM-backed virtual disks attached to Windows 10/11 VMs in Proxmox. This process allows you to reclaim unused storage space that was over-provisioned during initial VM setup.

## Prerequisites

- **Proxmox VE** with basic administrative knowledge
- **LVM storage backend** (local-lvm or custom LVM setup)
- **Windows 10/11 VM** with NTFS filesystem
- **Administrative access** to both Proxmox host and Windows VM
- **Niubi Partition Editor Free Edition** (or similar partition management tool) - https://www.hdd-tool.com/partition-manager/partition-editor-free.html

## Critical Safety Warning

**ALWAYS create a full backup or snapshot before starting this process.** Disk operations carry inherent risk of data loss. This guide assumes you understand the risks and have proper backups in place.

#### Create a snapshot before starting
```bash
qm snapshot <vmid> pre-shrink-backup
```
#### Or create a full backup
```bash
vzdump <vmid>
```

## Example Scenario

This guide demonstrates reducing a 90GB LVM volume by 25GB (final size: 65GB) while maintaining a safety buffer throughout the process.

---

## Phase 1: Prepare Windows Partition

#### Step 1: Shrink the Windows C: Drive

1. **Boot your Windows VM** and log in as administrator
2. **Open Disk Management** (`diskmgmt.msc`)
3. **Right-click on C: drive** → Select "Shrink Volume"
4. **Calculate shrink amount:**
   - Target reduction: 25GB
   - Safety buffer: 3GB
   - **Shrink by: 28GB** (28,000 MB)
5. **Execute the shrink operation**
6. **Verify unallocated space** appears after C: drive
   ```
   [EFI Partition] [C: Drive] [Unallocated Space] [Recovery Partition]
   ```

#### Step 2: Move Recovery Partition

The unallocated space must be at the very end of the disk for LVM reduction to work properly.

1. **Download and install Niubi Partition Editor Free Edition** in the Windows VM
2. **Open Niubi Partition Editor**
3. **Locate/select the Windows Recovery partition** (usually small, ~500MB-1GB)
4. **Move the Recovery partition (drag the block to the far left)**
   - This places the recovery partition immediately after C: drive
5. **Apply the changes** and allow the operation to complete
6. **Verify disk layout in Windows Disk Management:**
   ```
   [EFI Partition] [C: Drive] [Recovery Partition] [Unallocated Space]
   ```

---

## Phase 2: Reduce LVM Volume

### Step 3: Shutdown VM and Identify Disk

#### Shutdown the VM
```bash
qm shutdown <vmid>
```
#### Check current disk configuration
```bash
qm config <vmid>> | grep lvm
```
#### List logical volumes to find the correct disk
```bash
lvs | grep vm-<vmid>
```
#### Example output:
```
  vm-101-disk-0   pve Vwi-aotz--   4.00m data        14.06                                  
  vm-101-disk-1   pve Vwi-aotz--  90.00g data        40.79                                  
  vm-101-disk-2   pve Vwi-aotz--   4.00m data        1.56
```

### Step 4: Reduce the LVM Volume

### Reduce the logical volume
#### Target: 65GB final size
```bash
lvreduce -L 65G /dev/pve/vm-<vmid>-disk-X
```
#### Confirm the reduction when prompted
**Expected warning:**
```
WARNING: Reducing active logical volume to 65.00 GiB.
THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce pve/vm-<vmid>-disk-X? [y/n]: y
```

### Step 5: Update Proxmox Configuration

#### Check current disk configuration
```bash
qm config <vmid>> | grep lvm
```
#### Update VM configuration (this ensures the webui matches our changes)
```bash
qm set <vmid> --scsi0 local-lvm:vm-<vmid>-disk-X,size=65G,iothread=1,discard=on
```
#### Verify the change
```bash
lvs | grep vm-<vmid>
```
#### Example output:
```
  vm-101-disk-0   pve Vwi-aotz--   4.00m data        14.06                                  
  vm-101-disk-1   pve Vwi-aotz--  65.00g data        40.79                                  
  vm-101-disk-2   pve Vwi-aotz--   4.00m data        1.56
```

---

## Phase 3: Finalize & Clean Up

### Step 6: Start VM and Verify Boot

#### Start the VM
```bash
qm start <vmid>
```
#### Monitor boot process for errors

### Step 7: Reclaim Safety Buffer

Once Windows boots successfully:

1. **Open Niubi Partition Editor** again
2. **Move the Recovery partition** back to the far right
3. **Extend the C: drive** to reclaim the safety buffer
4. **Apply changes** and wait for completion
5. **Verify disk layout in Windows Disk Management:**
   ```
   [EFI Partition] [C: Drive] [Recovery Partition]
   ```

### Step 8: Final Verification

**In Windows:**
- Open Disk Management and verify sizes
- Run `chkdsk C: /f` to check filesystem integrity
- Test application functionality

### Step 9: Clean Up

#### After thorough testing, remove the snapshot
```bash
qm delsnapshot <vmid> pre-shrink-backup
```
#### Verify space reclamation
```bash
df -h /dev/pve
```

---

## Quick Troubleshooting

### VM Won't Boot After LVM Reduction

#### Reset VM configuration
```bash
qm set <vmid> --scsi0 local-lvm:vm-<vmid>-disk-X,size=90G,iothread=1,discard=on
```
#### Restore from snapshot
```bash
qm rollback <vmid> pre-shrink-backup
```
```bash
qm start <vmid>
```

### Wrong Disk Identified
- Double-check `qm config <vmid> | grep lvm` output
- Verify disk numbers match between shell commands and web UI
- **Do not guess** - always confirm before running `lvreduce`

---

## Results

Upon successful completion:
- **LVM volume reduced** from 90GB to 65GB
- **25GB storage reclaimed** in Proxmox volume group
- **Windows VM functions normally** with optimized disk configuration

## Best Practices

- **Always test thoroughly** before removing snapshots
- **Document disk numbers** for multiple VMs to avoid confusion
- **Use safety buffer** throughout the process
- **Verify each step** before proceeding to the next
- **Keep backups** until certain the operation was successful

---

### This process can be repeated for multiple VMs. Each VM may have different disk numbers, so always verify the correct disk identifier before making changes.