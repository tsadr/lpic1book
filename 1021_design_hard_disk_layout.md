# 102.1. Design hard disk layout 


*Weight: 2*

Description: Candidates should be able to design a disk partitioning scheme for a Linux system.

## Key Knowledge Areas

- Allocate filesystems and swap space to separate partitions or disks.
- Tailor the design to the intended use of the system.
- Ensure the /boot partition conforms to the hardware architecture requirements for booting.
- Knowledge of basic features of LVM


- / (root) filesystem
- /var filesystem
- /home filesystem
- swap space
- mount points
- partitions

## Basics
As any other OS, Linux uses *files* and *directories* to operate. But unlike *Windows*, it does not use A:, C:, D:, etc. In Linux everything is in **one big tree*, starting with / (called root). Any partion, disk, CD, USB, network drive, ... will be placed somewhere in this huge tree. 

> Note: Most of external devices (USB, CD, ..) are mounted at /media/ or /mnt/ .

## Unix directories
Directory	Description
bin	Essential command binaries
boot	Static files of the boot loader
dev	Device files
etc	Host-specific system configuration
lib	Essential shared libraries and kernel modules
media	Mount point for removable media
mnt	Mount point for mounting a filesystem temporarily
opt	Add-on application software packages
sbin	Essential system binaries
srv	Data for services provided by this system
tmp	Temporary files
usr	Secondary hierarchy
var	Variable data

## Partitions
In linux world, devices are defined at /dev/. First SCSI disk is /dev/sda, seccond SCSI disk is /dev/sdb, ... and first SATA disk (older systems) is /dev/hda.

You have to *PARTITION* the disks, that is creating smaller parts on a big disk and calling them /dev/sda1 (first partition on first SCSI disk) or /dev/hd**b**3 (3rd partition on seccond disk). 


````
# fdisk /dev/sda
Welcome to fdisk (util-linux 2.25.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p
Disk /dev/sda: 298.1 GiB, 320072933376 bytes, 625142448 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x000beca1

Device     Boot     Start       End   Sectors   Size Id Type
/dev/sda1  *         2048  43094015  43091968  20.6G 83 Linux
/dev/sda2        43094016  92078390  48984375  23.4G 83 Linux
/dev/sda3        92080126 625141759 533061634 254.2G  5 Extended
/dev/sda5        92080128 107702271  15622144   7.5G 82 Linux swap / Solaris
/dev/sda6       107704320 625141759 517437440 246.8G 83 Linux
````

### Primary, Extended & Logical Partitions
The partition table is located in the master boot record (**MBR**) of a disk. The MBR is the first sector on the disk, so the partition table is not a very large part of it. This limits the primary partitions to 4 and the max size of a disk to around 2TBs. If you need more partitions you have a define one extended and then create logicals *inside* them. 

Linux numbers the primary paritions 1, 2, 3 & 4. If you defin an extended partitions, logical partitions inside it will be called 5, 6, 7. 

>Note: an Extended partition is just an empty box for creating Logical partitions inside it.

So:
- /dev/sda3 is the 3rd primary partition on the first disk
- /dev/sdb5 is the first logical partition on the seccond disk
- /dev/sda7 is the 3rd logical partition of the first physical disk

The newer **GUID Partition Table (or GPT)** solves this problems. If you format your disk with GTP you can have 128 primary partitions (no need to extended and logical). 

## commands
### parted
````
jadi@funlife:~$ sudo parted /dev/sda p
Model: ATA ST320LT000-9VL14 (scsi)
Disk /dev/sda: 320GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type      File system     Flags
 1      1049kB  22.1GB  22.1GB  primary   ext4            boot
 2      22.1GB  47.1GB  25.1GB  primary   ext4
 3      47.1GB  320GB   273GB   extended
 5      47.1GB  55.1GB  7999MB  logical   linux-swap(v1)
 6      55.1GB  320GB   265GB   logical
````

#### fdisk
````
# sudo fdisk /dev/sda
[sudo] password for jadi: 

Welcome to fdisk (util-linux 2.25.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p
Disk /dev/sda: 298.1 GiB, 320072933376 bytes, 625142448 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x000beca1

Device     Boot     Start       End   Sectors   Size Id Type
/dev/sda1  *         2048  43094015  43091968  20.6G 83 Linux
/dev/sda2        43094016  92078390  48984375  23.4G 83 Linux
/dev/sda3        92080126 625141759 533061634 254.2G  5 Extended
/dev/sda5        92080128 107702271  15622144   7.5G 82 Linux swap / Solaris
/dev/sda6       107704320 625141759 517437440 246.8G 83 Linux
````
>Note: parted does not understands GPT


#### gparted
A gparhical tool. 

![http://www.ibm.com/developerworks/linux/library/l-lpic1-v3-102-1/gparted-1s.jpg]

 
### LVM
In many cases you need to resize your partitions or even add new disks and *add* them to your mount points. LVM is designed for this.

LVM helps you create one partition from different disks and add or remove space to them. The main concepts are:

- Physical Volume (pv): a whole drive or a partition. It is better to define paritions and **not use whole disks - unpartitioned**.
- Volume Garoups (vg): this is the collection of one or more **pv**s. OS will see the vg as one big disk. PVs in one vg, can have different sizes or even be on different physical disks.
- Logical Volumes (lv): OS will see lvs as paritions. You can format an lv wit your OS and use it. 

## Design Hard disk layout
Disk layout and allocation partitions to directories depends on you usage. First we will discuss *swap* and *boot* and then will see three different cases.

#### swap
swap in linux works like an extended memory. Kernel will *page* memory to this paritoin / file. It is enoungh to format one partion with **swap file system** and define it in /etc/fstab (you will see this later in 104 modules). 

> Note: swap size is 1 or 2 times the system memory but not more than 8GBs. So if you have 2GB of RAM, swap will be 4GB but if you have 6GB of RAM, it is recommended to have a 8GB swap partition.

Older linuxes were not able to handle HUGE disks (say Terrabytes) so there were /boot. 
Swap

on setup
/
/boot
swap



network workstation
/
/boot
/home (might be NFS, SMB, SSH)
swap


server
/
/home/
/var
/usr (sometimes read only from a network so all machine will be updated in one move)
