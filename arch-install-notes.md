Here are some notes about installing ArchLinux on a Asus Zenbook UX410UA with UEFI/GPT that had Windows 10 installed (no dual boot).

This is not a tutorial. [ArchLinux installation guide](https://wiki.archlinux.org/index.php/installation_guide) is the reference.
These notes help me detail some steps that need jumping between pages on the wiki.

## Preparation steps
- Backup the Windows installation if/as needed
- Download and [burn the ISO in DD mode](https://wiki.archlinux.org/index.php/USB_flash_installation_media#BIOS_and_UEFI_bootable_USB) to a USB key so that it can be booted in UEFI mode (see Rufus on Windows)
- Decide what [partitions](https://wiki.archlinux.org/index.php/Partitioning) are needed, what type, size etc.
- Verify hardware, will there be any firmware issue (ex: wifi, video). If for instance there might be a wifi firmware issue, plan for a wired installation (ex: ethernet to USB adapter)
- Decide if [encryption](https://wiki.archlinux.org/index.php/Disk_encryption) is needed since there are some prerequisites like wiping the disk and partitioning
- Decide if/what a [bootloader](https://wiki.archlinux.org/index.php/Category:Boot_loaders) is needed (here: efibootmgr, no specific bootloader like grub, ec)
- For Intel processors, [Microcode](https://wiki.archlinux.org/index.php/Microcode) updates will be needed (here : yes)
- Read the [wiki](https://wiki.archlinux.org/) and the [forums](https://bbs.archlinux.org/), know where to find information
- Look up the internet for installation of Linux on similar hardware

## Installation steps
### Boot 
- Boot on the live USB
- Verify `/sys/firmware/efi/efivars` has a list of variables as well as `efivar -l` to confirm the boot is UEFI boot
- Configure keyboard mapping with `loadkeys`

**References**: [UEFI](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface),
[UEFI Bootable USB](https://wiki.archlinux.org/index.php/USB_flash_installation_media#BIOS_and_UEFI_bootable_USB)

### Network
Set up wifi if needed :
- `ip link` => identify the name of the interface (ex: wlan0, wlp2s0...), it should start by w
- `ip link set <interface> up`
- `wpa_passphrase SSID passphrase > /etc/wpa_supplicant/wpa_supplicant.conf` (+ remove the line that has the WPA key in clear)
- `wpa_supplicant -B -i <interface> -c /etc/wpa_supplicant/wpa_supplicant.conf`
- `dhcpd <interface>`

NB : `netctl` and `wifi-menu` can be used once archlinux is installed for automatic configuration

**References**: [Wireless Configuration](https://wiki.archlinux.org/index.php/Wireless_network_configuration),
[wpa_supplicant](https://wiki.archlinux.org/index.php/WPA_supplicant)

### Create the partitions
To create the partitions use `gdisk` (there are [others](https://wiki.archlinux.org/index.php/Fdisk) that work as well)

`o` -> overwrite everything (this assumes the existing system is wiped out and archlinux is installed in its place)

`n` -> new partition
- accept default numbering (unless needed otherwise)
- first sector : enter
- last sector : enter +size (ex: +20G)
- type : **8300** for a data partition, **EF00** for the esp, **8200** for swap

`p` -> to see what you've done so far

`w` -> to write the final result

#### Notes
- About partition alignment: no need to take care of that, gdisk handles it.
- Boot partition shall be the first partition on disk in some cases.
- About the size of the boot partition (esp): seems it needs to be at least **512M.**
- About the size of swap : ideally equals the amount of RAM, ex **8G**.

### Create the filesystems
- `mkfs.ext4 /dev/sdXY` for /, /var, /home
- `mkswap /dev/sdXY` for swap then `swapon /dev/sdXY`
- `mkfs.vfat -F32 /dev/sdXY` for /boot

### Mount the filesystems
It's preferrable to have the esp mounted to `/boot` but some people prefer to have it on `/boot/efi`. 
In that case some extra steps are needed when configuring the boot loader/manager.
Ex:
```
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot
```

**References:** [Partitioning](https://wiki.archlinux.org/index.php/Partitioning), 
[GPT](https://wiki.archlinux.org/index.php/Partitioning#GUID_Partition_Table),
[EFI System Partition ESP](https://wiki.archlinux.org/index.php/EFI_System_Partition)

### Follow the wiki instructions
`timedatectl`, `pacstrap`, `genfstab`, `arch-chroot`, `hostname`, various `locale` stuff, and compile the kernel.

Set up the root password.

### Configure the boot manager (efibootmgr)

[Microcode](https://wiki.archlinux.org/index.php/Microcode) updates need to be configured in the case of an Intel processor:
`pacman -S intel-ucode`

#### Configure efibootmgr
- Install efibootmgr : `pacman -S efibootmgr`
- Then configure it the following way

`efibootmgr -d /dev/sdX -p Y -c -L "Arch Linux" -l /vmlinuz-linux -u "root=/dev/sdBZ rw initrd=/initramfs-linux.img"`

For instance if the boot (esp) is `/dev/sda1` and root is `/dev/sda2` replace X by a, Y by 1 and `/dev/sdBZ` by `/dev/sda2`

For microcode the following command has to be used instead, that adds another initrd as kernel parameter BEFORE the linux initramfs option:

`efibootmgr -d /dev/sdX -p Y -c -L "Arch Linux" -l /vmlinuz-linux -u "root=/dev/sdBZ rw initrd=/intel-ucode.img initrd=/initramfs-linux.img"`

If the esp is not `/boot` but `/boot/efi` for instance, the boot files have to be copied from `/boot` to `/boot/efi` manually and a hook is needed to copy them each time the kernel is recompiled.

Verify `efibootmgr -v` makes sense.

**References:** [Boot Loaders](https://wiki.archlinux.org/index.php/Category:Boot_loaders), 
[EFI Stub](https://wiki.archlinux.org/index.php/EFISTUB)

### Done
Reboot, remove the install media, and archlinux will start. Yay!
