
# Disk Partitioning and LVM Management Guide

## Author
This guide is authored by [GitHub.com/sariamubeen](https://github.com/sariamubeen).

## 1. **Overview**
This guide covers the entire process of adding a new disk, creating partitions, managing logical volumes using LVM (Logical Volume Management), and resizing filesystems in a Linux environment. The document includes the steps to extend storage by adding a new disk, partitioning it, and using it within LVM.

---

## 2. **Disk Setup Process**

### **2.1. Verify Existing Disk Configuration**
Before adding a new disk, verify the current disk and volume group configuration using the `lsblk` command.

#### **Command:**
```bash
lsblk
```

#### **Expected Output:**
This command will display all connected disks and their partitions. Here's a typical output structure:

```bash
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
fd0                       2:0    1    4K  0 disk
sda                       8:0    0  25G  0 disk
├─sda1                    8:1    0  500M 0 part /boot
└─sda2                    8:2    0 24.5G 0 part
  ├─root_lv              253:0  0  22G  0 lvm  /
  └─swap_lv              253:1  0   2G  0 lvm  [SWAP]
sdb                       8:16   0  100G 0 disk
└─sdb1                    8:17   0  100G 0 part
sr0                       11:0   1  1024M 0 rom
```

- **sda**: Existing disk.
- **sda1**: Partition for `/boot`.
- **sda2**: Partition used for LVM.
- **root_lv**: Logical volume for root (`/`).
- **swap_lv**: Logical volume for swap.
- **sdb**: Newly added disk.
- **sdb1**: Partition on the new disk for LVM.

---

## 3. **Adding a New Disk**

### **3.1. Add New Disk**
For virtualized environments (e.g., VMware, KVM, VirtualBox), you can add a new virtual disk via the management interface. If using a physical server, ensure the new disk is connected to the system.

### **3.2. Verify New Disk**
Once the disk is added, verify it using the `lsblk` or `fdisk` commands.

#### **Command:**
```bash
lsblk
```

- The new disk should appear as `/dev/sdb` (or `/dev/sdc`, `/dev/sdd`, etc., depending on the number of disks in the system).
- It may show as unallocated space without partitions.

---

## 4. **Partitioning the New Disk**

### **4.1. Create Partition on New Disk Using `fdisk`**
To create a partition on the new disk, use `fdisk` or `cfdisk`. 

#### **Using `fdisk`:**
1. **Open `fdisk` to modify the new disk:**
   ```bash
   sudo fdisk /dev/sdb
   ```

2. **Create a new partition:**
   - Type `n` to create a new partition.
   - Choose `p` for a primary partition.
   - Select the partition number (usually `1`).
   - Accept the default starting and ending sectors, or specify the desired size.
   - Type `w` to write the changes and exit `fdisk`.

#### **Using `cfdisk`:**
1. **Open `cfdisk` to modify the new disk:**
   ```bash
   sudo cfdisk /dev/sdb
   ```

2. **Create the new partition:**
   - Navigate using the arrow keys to create a new partition.
   - After partitioning, write the changes.

#### **Expected Output:**
After creating the partition, run `lsblk` to verify the new partition (`/dev/sdb1`):

```bash
lsblk
```

You should now see `/dev/sdb1` listed under `/dev/sdb`.

---

## 5. **Create Physical Volume (PV)**

### **5.1. Create Physical Volume on Partition**
After partitioning, initialize the partition as a physical volume (PV) for LVM.

#### **Command:**
```bash
sudo pvcreate /dev/sdb1
```

#### **Expected Output:**
```bash
  Physical volume "/dev/sdb1" successfully created.
```

---

## 6. **Extend Volume Group (VG)**

### **6.1. Verify Existing Volume Groups**
Before adding the new physical volume to the volume group, check the available volume groups using:

#### **Command:**
```bash
sudo vgdisplay
```

#### **Expected Output:**
This will list all the volume groups. Note the name of the volume group (VG), which is used for adding physical volumes.

Example output:
```bash
  --- Volume group ---
  VG Name               vg_name
  System ID             
  Format                lvm2
  Metadata Areas        2
  ...
```

### **6.2. Add Physical Volume to Volume Group**
Once you know the volume group name, extend it by adding the new physical volume.

#### **Command:**
```bash
sudo vgextend vg_name /dev/sdb1
```

#### **Expected Output:**
```bash
  Volume group "vg_name" successfully extended.
```

This command adds the new disk partition to the existing volume group.

---

## 7. **Extend Logical Volume (LV)**

### **7.1. Verify Existing Logical Volumes**
Use the following command to list the existing logical volumes (LVs):

#### **Command:**
```bash
sudo lvdisplay
```

#### **Expected Output:**
This command will display the logical volumes and their sizes. Note the LV you want to extend (e.g., `lv_name`).

### **7.2. Extend Logical Volume**
Extend the logical volume to use all the free space available in the volume group.

#### **Command:**
```bash
sudo lvextend -l +100%FREE /dev/vg_name/lv_name
```

- **`-l +100%FREE`**: This extends the LV to use 100% of the free space in the volume group.

#### **Expected Output:**
```bash
  Extending logical volume lv_name to size <new size>.
```

---

## 8. **Resize the Filesystem**

After extending the logical volume, resize the filesystem to make use of the new space.

#### **8.1. Resize Filesystem for XFS**
If your system uses `XFS`, run the following command:

#### **Command:**
```bash
sudo xfs_growfs /dev/vg_name/lv_name
```

#### **Expected Output:**
The filesystem should be resized and the additional space available.

#### **8.2. Resize Filesystem for EXT4**
If the filesystem is `ext4`, use:

#### **Command:**
```bash
sudo resize2fs /dev/vg_name/lv_name
```

#### **Expected Output:**
```bash
resize2fs: new size is <new size>
```

---

## 9. **Verify the Changes**

Finally, verify that the filesystem has been resized and the new space is available.

#### **9.1. Check Disk Usage**
Use the `df` command to verify the available disk space:

#### **Command:**
```bash
df -h /
```

#### **Expected Output:**
```bash
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/vg_name-lv_name   200G  50G  150G  25% /
```

This confirms that the logical volume and filesystem have been successfully extended.

---

## 10. **Troubleshooting**

- **If `lvextend` or `vgextend` fails**, ensure the correct volume group and logical volume names are used.
- **Ensure there is no data loss**: Always back up important data before modifying partitions, logical volumes, or filesystems.
- **Use `partprobe` if partitions aren't detected**: If a newly created partition doesn't show up, you can re-read the partition table using `partprobe`:

  #### **Command:**
  ```bash
  sudo partprobe /dev/sdb
  ```

  This ensures the partition table is updated, and the system can recognize new partitions without rebooting.

- If partitions still aren't showing up, reboot the system.

---

## 11. **Summary**

- **Create a partition** using `fdisk` or `cfdisk`.
- **Create a physical volume** (`pvcreate`) on the new partition.
- **Extend the volume group** (`vgextend`) to include the new physical volume.
- **Extend the logical volume** (`lvextend`) to use the additional space.
- **Resize the filesystem** (`xfs_growfs` or `resize2fs`) to take advantage of the newly available space.
- **Verify** the changes using `df -h`.

---

## **Appendix**

- **LVM commands:**
  - `pvcreate`: Initialize a physical volume.
  - `vgextend`: Add a physical volume to a volume group.
  - `lvextend`: Extend a logical volume.
  - `xfs_growfs`: Resize an XFS filesystem.
  - `resize2fs`: Resize an EXT4 filesystem.
  - `partprobe`: Re-read the partition table without rebooting.

This documentation can serve as a standardized process for disk management and LVM usage in your environment.
