
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

### Target computer preparation

