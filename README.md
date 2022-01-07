# Arch Linux

## Basic command line install
### Download the image and liveboot
The images can be found at https://www.archlinux.org/download/

The image can be flashed on a USB with any software (balena etcher works just fine)

Boot the device from USB (UEFI if your computer supports it)

### Check the bootmode
Check whether you booted in UEFI mode or not with 

```# ls /sys/firmware/efi/efivars```

If the directory exists you're in UEFI mode, otherwise you're in BIOS

### Network(1)
If you're connecting to the network via ethernet, check whether internet is working or not:

```# ping archlinux.org```

Instead if you're trying to connect to a wifi, do so: 

```# wifi-menu```

# Sync the system clock

```# timedatectl set-ntp true```

### Disk partitioning
Check current disks: 

```# fdisk -l```

For the partitioning scheme you can choose if you want to go with MBR or GPT as partition table.

The partition scheme:

BIOS:

  MBR:

  _With MBR as partition table in BIOS mode you need to let some space unallocated before the first partition. The space left by fdisk by default should be enough_

    /dev/sdX1 mounted to /mnt with partition type "Linux" (root partition, you choose the size)

    /dev/sdX2 mounted as swap with partition type "Linux swap" (swap partition, more than 512MiB)

  GPT:

  _With GPT as partition table in BIOS mode instead you need an extra boot partition_

    /dev/sdX1 mounted to /mnt with partition type "Linux" (root partition, you choose the size)

    /dev/sdX2 mounted as swap with partition type "Linux swap" (swap partition, more than 512MiB)

    /dev/sdX3 with partition type "BIOS boot" (size of +1M, partition type GUID: 21686148-6449-6E6F-744E-656564454649)

  UEFI:

  GPT:

  _With GPT as partition table in UEFI mode, you need an EFI partition with type "EFI system"_

    /dev/sdX1 mounted to /mnt with partition type "Linux" (root partition, you choose the size)

    /dev/sdX2 mounted as swap with partition type "Linux swap" (swap partition, more than 512MiB)

    /dev/sdX3 mounted to /mnt/efi with partition type "EFI system partition" (UEFI partition, 260-512MiB, GUID: C12A7328-F81F-11D2-BA4B-00A0C93EC93B)
 
  MBR:

  _With MBR as partition table in UEFI mode, you need an EFI partition with type "EFI (FAT-12/16/32)"_

    /dev/sdX1 mounted to /mnt with partition type "Linux" (root partition, you choose the size)

    /dev/sdX2 mounted as swap with partition type "Linux swap" (swap partition, more than 512MiB)

    /dev/sdX3 mounted to /mnt/efi with partition type "EFI (FAT-12/16/32)" (UEFI partition, 260-512MiB, partition type ID: EF)


To start partitioning the disk you want you can use ```fdisk```:

```# fdisk /dev/sdX```

Use the help menu that you can get with ```m``` to create a new partition table, create partitions like said above and write changes to the disk.

### Format the partitions
Format the root partition as ext4: 

```# mkfs.ext4 /dev/sdX1```
Format the swap partition as swap:
```
# mkswap /dev/sdX2
# swapon /dev/sdX2
```
If you are in **UEFI** mode also format the UEFI partition as fat32:

```# mkfs.fat -F32 /dev/sdX3```

### Install the base
Mount the root partition to /mnt:

```# mount /dev/sdX1 /mnt```

Install essential packages:

```# pacstrap /mnt base linux linux-firmware```

These are just the most basic packages needed. We will install others later

### Generate the fstab
Generate the fstab simply with just one command:

```# genfstab -U /mnt >> /mnt/etc/fstab```

Take a look at it to see whether everything is fine, if it isn't just edit the file manually

### Chroot
It's time to switch the root folder of the bootable enviroment to the installation root:

```# arch-chroot /mnt```

From now on you can install packages simply with ```pacman -S <pkg>```

### Install secondary packages
Install some other basic packages:
```# pacman -S sudo nano networkmanager```

### Time zone and localization
To set the time zone and generate ```/etc/adjtime```:
```
# ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
# hwclock --systohc
```

Uncomment wanted locales in ```/etc/locale.gen```

The en_US one is ```en_US.UTF-8 UTF-8```

So regenerate the locales and create locale.conf (the LANG variable needs to be set accordingly):
```
# locale-gen
# echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### Network (2)
Create the hostname:

```# echo myhostname > /etc/hostname```

Add the needed entries to hosts:

*/etc/hosts*:
```
127.0.0.1	localhost
::1		localhost
127.0.1.1	myhostname.localdomain	myhostname
```
Change "myhostname" with the desired hostname in the two previous commands

### Intramfs
*This step is usually not requires since it was already done during the installation of the kernel*

To regenerate the intramfs:

```# mkinitcpio -P```

### Set the root password
To set the run password:

```# passwd```

### Creating a new user
This step is crucial since you wouldn't want to keep using the root user for everything

To create a new user:

```# useradd -m username```

Now you need to set a password to protect the new user:

```# passwd username```

To get sudo permissions on the recently created user:
```# usermod -aG sudo username```

Now you should be able to get sudo permissions on your new user.

## Bootloader installation
To boot into your system after the installation you need a bootloader. The most used and suggested one is ```grub```

### Grub installation
**Legacy BIOS installation**
_Remember that if you're installing on a disk with GPT and in BIOS mode, you need to create a specific partition for it. Check the "Disk partitioning" section_

Firstly install the ```grub``` package with pacman as said previously.

Now start the installation:

```# grub-install --target=i386-pc /dev/sdX```

Change _/dev/sdX_ with the disk where you want to install grub (not the partition)

**UEFI installation**
Firstly install ```grub``` and ```efibootmgr``` packages with pacman.

Mount the EFI partition to /efi:
```
# mkdir /efi
# mount /dev/sdX3 /efi
```
Now start the installation:

```# grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB```

This command assumes that the system is running as 64bit, if not just replace "x86_64-efi" with "i386-efi".

### Grub configuration
It's time to generate all configuration files needed for grub.

You can do so with ```grub-mkconfig```:

```# grub-mkconfig -o /boot/grub/grub.cfg```

**Multiboot installations**
For multiboot installation we need to install the package ```os-prober```.

After doing so, we need to mount the partitions containing other systems and re-run ```grub-mkconfig```.

Windows partitions will be discovered automatically by ```os-prober``` but the default Linux driver for NTFS may not be enough. If that's the case, install ```ntfs-3g``` and remount.

#### Install sysvcompat
The installation of this package might be needed to boot successfully in some devices and prevent ```ERROR: root device mounted successfully, but /sbin/init does not exist```

The package name is `systemd-sysvcompat`

## Boot into the system

Everything should be set-up perfectly by now. 

So, exit the chroot enviroment doing either ```Ctrl + D``` or typing ```exit```

Eventually unmount the root partition:

```umount -R /mnt```

Ultimately type ```reboot``` and plug out the USB drive.

Hopefully the system will boot correctly at first trial.

**Congratulations, you installed Arch!**

## First boot
### Login
If everything went according to plans, you've just booted into your new system.
The first thing you need to do is login into your user with the username and password that you created earlier in the installation.

### Network (3)
Now that you booted into your newly installed system, the first thing to get the connection working.

Firstly start NetworkManager:

```sudo NetworkManager```

Now connect to a network with the command line tool ```nmcli```

Check if the connection is working now:

```ping archlinux.org```

If everything works fine, enable NetworkManager automatically at boot:

```sudo systemctl enable NetworkManager.service```
