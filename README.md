
# Unattended automatic backup via Clonezilla bootable media

## Background

The purpose of this repo is to build a fully automated backup solution using [Clonezilla](https://clonezilla.org/) engine.

The solution provides the ability to create backups of one/multiple partitions or one/multiple drives of a computer system in a completely unattended mode.

The system boots, determines what should be backed, where the backup should stored, then performs the backup and powers the equipment down.

## How it works

The solution is based on the bootable clonezilla media.

The only user intervention is to boot the target computer into Clonezilla bootable media. The automatic process take over from there.

  

### Algorithm

- Boot into Clonezilla

- Automatically start Custom backup option

- Inspect local partitions looking for `backup.conf` configuration file in the root folder

- Compare hardware ID of the target hardware to the hardware ID in the `backup.conf` (correct backup configuration check)

- Determine backup target partition (backup folder is created in the root folder on that partition)

- Determine partitions or drives to be backed up

- Execute the backup

- Powerdown or reboot the target computer

### Outcome

The resulting backup is created in a folder (folder name is supplied in the `backup.conf` file) on the backup partition (partition is supplied in the `backup.conf` file). Backup parameters are also supplied in the `backup.conf` file. Those are passed to Clonezilla engine at runtime.


## Bill of Materials

In order to have a self-sufficient backup system, I opted to have a 2TB external USB HDD split into two partitions:

  

- Bootable Clonezilla partition with Clonezilla application and backup scripts

- Data partition - where the resulting backup is stored

  

### You will need:

- 2TB USB hard drive example is this: [Seagate Backup Plus Slim 2TB External Hard Drive Portable HDD](https://www.amazon.com/gp/product/B00FRHTTJE)

- USB/USBC cable

- A computer (this repo relies on you knowing Linux command line)

  

## Build process

  

### Partition hard drive

Partition hard drive into 2 separate partitions:
- FAT32 Clonezilla Partition - 512MB
- exFAT Data Partition - 1.8T

Relevant fdisk output for this disk is:

    fdisk -l /dev/sdc
    Disk /dev/sdc: 1.84 TiB, 2000398934016 bytes, 3907029168 sectors
    Disk model:
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 4096 bytes
    I/O size (minimum/optimal): 4096 bytes / 4096 bytes
    Disklabel type: dos
    Disk identifier: 0xcb37c45a
      
    Device Boot Start End Sectors Size Id Type
    /dev/sdc1 * 2048 1050623 1048576 512M b W95 FAT32
    /dev/sdc2 1050624 3907028991 3905978368 1.8T 7 HPFS/NTFS/exFAT

- Follow this setup guide to install Clonezilla on the first partition: [setup guide](https://clonezilla.org/liveusb.php)
- Copy `custom-ocs` file from the `/clonezilla/live/` folder to the `/live` folder on Clonezilla partition
- Navigate to the `/boot` folder on the Clonezilla partition
- Backup `grub.cfg` files (e.g. `grub.cfg.bak`) in case you make a mistake
- Edit the `grub.cfg` file making the following changes:

 1. Set timeout to 5 seconds (default is 30):

		set timeout = "5"

 2. Location this line (usually the first Clonezilla menu item):

		menuentry --hotkey=r "Clonezilla live (VGA 800x600 & To RAM)" --id live-toram {

 3. Copy the entire `menuentry` block ahead of the first menublock - the default menu item is 0 (first on the list)

		menuentry --hotkey=r "Clonezilla live (VGA 800x600 & To RAM)" --id live-toram {
		  search --set -f /live/vmlinuz
		  $linux_cmd /live/vmlinuz boot=live union=overlay username=user hostname=noble config quiet loglevel=0 noswap edd=on nomodeset enforcing=0 noeject locales= keyboard-layouts= ocs_live_run="ocs-live-general" ocs_live_extra_param="" ocs_live_batch="no" vga=788 toram=live,syslinux,EFI,boot,.disk,utils net.ifnames=0  splash i915.blacklist=yes radeonhd.blacklist=yes nouveau.blacklist=yes vmwgfx.enable_fbdev=1
		  $initrd_cmd /live/initrd.img
		}

 4. In the new block, modify the menuitem description and the boot command according to the following:

		menuentry --hotkey=r "Clonezilla Autobackup (VGA 800x600 & To RAM)" --id live-autobackup {
		  search --set -f /live/vmlinuz
		  $linux_cmd /live/vmlinuz boot=live union=overlay username=user hostname=noble config quiet loglevel=0 noswap edd=on nomodeset enforcing=0 noeject locales=locales=en_EN.UTF-8 keyboard-layouts=NONE ocs_live_run="/lib/live/mount/medium/live/custom-ocs" ocs_live_extra_param="" ocs_live_batch="no" vga=788 toram=live,syslinux,EFI,boot,.disk,utils net.ifnames=0  splash i915.blacklist=yes radeonhd.blacklist=yes nouveau.blacklist=yes vmwgfx.enable_fbdev=1
		  $initrd_cmd /live/initrd.img
		}

 5. Save `grub.cfg` file

The autobackup setup of Clonezilla is now complete.

### Prepare target computer

#### Backup configuration file
Backup configuration file tells automatic Clonezilla what kind of backup needs to be performed
A sample backup file is below:

		4c4c4544-0054-3610-804a-c7c04f444b33
		7B0DFCE74B228860
		-q2 -j2 -nogui -z1p -i 2000000 -p poweroff savedisk xps15dev_system
		D2DE19FADE19D815
 
Let's review it line by line

##### Line 1 - hardware ID of the target machine:

		00020003-0004-0005-0006-000700080009

This ID could be looked up with this command: `sudo lshw -quiet -class system | grep configuration | grep uuid`

		$ sudo lshw -quiet -class system | grep configuration | grep uuid
		configuration: boot=normal chassis=desktop family=Default string sku=Default string uuid=00020003-0004-0005-0006-000700080009


The easiest way to build `backup.conf` file is to boot into interactive Clonezilla, drop into shell instead of starting the backup and look it up there. 
Clonezilla checks if correct configuration file has been found and ignores the ones with unmatched hardware IDs


##### Line 2 - UUID of the partition to store back up file on

		7B0DFCE74B228860

If the target computer has mulitple disk drives, you can specify on of the partitions on the drives that are not part of the backup.
Typically this is the partition UUID of the DATA partition of the 2TB backup drive we are building.
Note that `7B0DFCE74B228860` is UUID of an exFAT partition (windows). Linux ext4 partition UUID will look like `38d9e8cd-deda-4735-ad97-a795665e77fe`
You can look up partition UUIDs using `sudo blkid` command.


##### Line 3 - Clonezilla command line arguments

		-q2 -j2 -nogui -z1p -i 2000000 -p poweroff savedisk xps15dev_system

One by one:

- `-q2` is
- `-j2` is
- `-nogui` is to disable fancy graphical progress reporting and stick to simple text only output
- `-z1p` is compression method
- `-i 2000000` is to set the size in MB to split the partition image file into multiple volumes files.
- `-p poweroff` is to shutdown the target computer after backup
- `savedisk` is command to backup the entire drive (`saveparts` saves individual partitions)
- `xps15dev_system` is the name of the target folder for the backup

The esiest way to build this line is to run an interactive Clonezilla session, select all desired parameters and save the resulting command line.
There is also a command line reference here:  [Man page of ocs-sr](https://clonezilla.org/fine-print-live-doc.php?path=./clonezilla-live/doc/98_ocs_related_command_manpages/01-ocs-sr.doc#google_vignette)

##### Line 4, 5, 6 and so on

		D2DE19FADE19D815

Lines 4 onward list UUID of partitions being backed up. 
If you are using `savedisk` command - any partion on a disk will make Clonezilla backup the entire disk.
If you are using `saveparts` - you need to specify each partition separately
