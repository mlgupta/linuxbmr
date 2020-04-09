# Linux Backup and Restore (BMR) procedure using fsarchiver

## Backup
1)	Mount remote volume on the machine where backup will be performed (Can be USB disk, or a NFS volume) on /mnt/backup
2)	Create a directory (if does not exists) for hostname
```
# mkdir /mnt/backup/<hostname>
```
3)	Backup disk partition information:
```
# sfdisk -d /dev/sd[a-z] > /mnt/backup/<hostname>sd[a-z].sfd
```
4)	Backup LVM information:
```
# vgcfgbackup -f /mnt/backup/<hostname>/%s.conf
```
5)	Backup LVM PV UUIDs (for each PV)
```
pvdisplay /dev/<path> > /mnt/backup/<hostname>/<pv>.conf
```
6)	Backup boot partition
```
# fsarchiver -A -o savefs /mnt/backup/<hostname>boot.fsa /dev/sda1
```
7)	Backup logical volumes using LVM snapshots (for each volume)
```
# lvcreate -s /dev/<vg name>/<lv name> -n snap_lv -L <size>
# fsarchiver -o savefs /mnt/backup/<lv name>.fsa /dev/<vg name>/snap_lv
# lvremove -f /dev/<vg name>snap_lv
```

## Restore:
1)	Boot the machine to be restored using SystemRescueCD (or RHEL)
2)	Mount remote volume, where backups are located, to /mnt/backup
3)	Restore disk partition table
```
# sfdisk /dev/sd[a-z] < /mnt/backup/<hostname>/sd[a-z].sfd
```
4)	Recreate physical volumens, using PV UUID (from step 5 of backup)
```
# pvcreate –uuid <UUID> --restorefile <vg conf from step 4> /dev/sda2
```
5)	Restore VG configuration
```
# vgcfgrestore -f <vg conf from step 4> <vg name>
```
6)	Re-scan LVM
```
# vgscan
# lvscan
```
7)	Restore boot partition
```
# fsaruchiver restfs /mnt/backup/<hostname>/boot.fsa id=0,dest=/dev/sda1
```
8)	Restore each logical volume
```
# fsarchiver restfs /mnt/backup/<hostname>/<filename>.fsa id=0,dest=/dev/<vg name>/<lv name>
```
9)	Configure System
    - Recreate swap file system (please refer to backup to find where swap is)
    ```
    # mkswap /dev/<vg name>/<lv name>
    ```
    - Mount all volumens to /mnt/target
    - Rebuild MBR
    ```
    # /sbin/grub-install --root-directory=/mnt/target '(hd0)'
    ```
    - Regenerate /boot/initrd file
    ```
    # chroot /mnt/target mkinitrd -f –fstab=/etc/fstab /boot/initrd-<version info> <kernel version>
    ```
