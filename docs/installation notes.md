# About

These are my personal notes for installing Arch Linux with:

- an encrypted root partition;
- automatic unlocking with the help of a TPM chip;
- using the UEFI as the bootloader.

# Disk setup

## Partitioning

```sh
lsblk # to get the device
export disk=/dev/nvme2n1
cfdisk $disk
```

Partition types:
- EFI System
- Linux root (x86-64)

## Encryption

```sh
export bootpart="$disk"p1
export rootpart="$disk"p2
cryptsetup luksFormat --sector-size=4096 $rootpart # leave an empty passphrase as it will be wiped with --wipe-slot=empty
systemd-cryptenroll $rootpart --recovery-key
systemd-cryptenroll $rootpart --wipe-slot=empty --tpm2-device=auto
cryptsetup --perf-no_read_workqueue --perf-no_write_workqueue --persistent open $rootpart root
```

Initramfs hooks are written in the relevant [section](#initramfs).

## Format partitions

```sh
mkfs.ext4 -b 4096 /dev/mapper/root
mkfs.fat -F 32 $bootpart
```

## Mount the partitions

```sh
mount /dev/mapper/root /mnt
mount --mkdir $bootpart /mnt/boot
```

## Set up a swapfile

Create two files, one for actual swap, and the other for suspend to disk.
Set a lower priority on the latter one to reduce its usage.

```sh
dd if=/dev/zero of=/mnt/swapfile bs=1G count=16 status=progress
dd if=/dev/zero of=/mnt/swapfile.suspend bs=1G count=32 status=progress

chmod 0600 /mnt/swapfile /mnt/swapfile.suspend

mkswap -U clear /mnt/swapfile
mkswap -U clear /mnt/swapfile.suspend

swapon --priority 100 /mnt/swapfile
swapon --priority 0 /mnt/swapfile.suspend
```

TODO seems like optimizations can be applied to swapfiles for
hibernation [source](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#About_swap_partition/file_size).

# Installation

## Set mirrors using refrector

```sh
reflector \
--sort rate \
--country Sweden,Germany,Lithuania \
--fastest 5 \
--protocol https \
--save /etc/pacman.d/mirrorlist
```

## Install essential packages

```sh
pacstrap -K /mnt base base-devel linux linux-firmware vim intel-ucode efibootmgr sbctl sudo bash-completion openssh python networkmanager
```

These packages are needed to facilitate automation with Ansible:
- `openssh` for SSH server
- `python` as Ansible requires it to be installed on hosts
- `networkmanager` for DHCP, also Plasma uses it

## System config

```sh
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
echo LBook > /etc/hostname
```

## Timezone

Windows uses localtime by default. Paste this into an admin cmd to
set it to UTC.

```powershell
reg add "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_DWORD /f
```

```sh
ln -sf /usr/share/zoneinfo/Europe/Vilnius /etc/localtime
hwclock --systohc
```

## Localization

```sh
vim /etc/locale.gen
locale-gen
vim /etc/locale.conf
```

Tried these options when installing:

```
LANG=en_US.UTF-8

LC_COLLATE=C
LC_MONETARY=C
LC_MEASUREMENT=C
LC_PAPER=C
LC_TIME=C
```

Console fonts for the `sd-vconsole` hook. Paste this into `/etc/vconsole.conf`:

```conf
FONT=Lat2-Terminus16
```

## Initramfs

Set the encrypt hook ([link](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Examples)).

```
HOOKS=(systemd autodetect modconf kms block keyboard sd-vconsole sd-encrypt filesystems fsck)
```

List of other common hooks ([link](https://wiki.archlinux.org/title/Mkinitcpio#Common_hooks)).

### Boot times: further investigation needed

Need to investigate udev vs systemd based initramfs. Systemd seems to be faster.
About optimizing mkinitcpio: [link 1](http://blog.falconindy.com/articles/optmizing-bootup-with-mkinitcpio.html),
[link 2](https://wiki.archlinux.org/title/Mkinitcpio/Minimal_initramfs).

Command to analyze boot times:

```sh
systemd-analyze
```

### Kernel parameters

Paste this into `/etc/kernel/cmdline`:

```
rd.luks.name=a2170876-099f-4177-8508-59f8f2109c12=root
rd.luks.options=a2170876-099f-4177-8508-59f8f2109c12=tpm2-device=auto,discard
root=/dev/mapper/root rw
resume=/dev/mapper/root
resume_offset=436224
```

To get the `resume_offset`, use:

```sh
filefrag -v /swapfile.suspend | awk '$1=="0:" {print substr($4, 1, length($4)-2)}'
```

To get the UUIDs (not PARTUUID) in `rd.luks` values, use:

```sh
blkid $rootpart
```

### Unified kernel image

This is my `/etc/mkinitcpio.d/linux.preset` file:

```conf
# mkinitcpio preset file for the 'linux' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"
ALL_microcode=(/boot/*-ucode.img)

PRESETS=('default')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux.img"
default_uki="/boot/arch-linux.efi"
#default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-fallback.img"
#fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
#fallback_options="-S autodetect"
```

Only kept the default preset, as I'm using EFISTUB - I wouldn't be able to boot
into the fallback image without extra effort regardless. And I carry a USB stick
with a live environment anyway.

### Final steps

Remove the soon to be redundant previously generated `initramfs-*.img` images.

```sh
rm /boot/initramfs-linux*
```

Finally, to build all of the presets, use:

```sh
mkinitcpio -P
```

Create a boot entry for the newly created EFI file:

```sh
efibootmgr --create --disk $disk --loader arch-linux.efi --unicode
```

### Sign UKI for secure boot

Check that the device is in setup mode using this command:

```sh
sbctl status
```

Then do this:

```sh
sbctl create-keys
sbctl enroll-keys --microsoft
sbctl sign --save /boot/arch-linux.efi
```

The boot into UEFI settings and enable secure boot.

To automatically sign the UKI, create a mkinitcpio post
hook `/etc/initcpio/post/sign-uki`:

```sh
#!/usr/bin/env bash

uki="$3"
[[ -n "$uki" ]] || exit 0

sbctl sign --save $uki
```

Set this file as executable:

```sh
chmod +x /etc/initcpio/post/sign-uki
```

Disable sbctl's built-in pacman hook to avoid duplicate signing during `linux` updates.

```sh
mkdir -p /etc/pacman.d/hooks
ln -s /dev/null /etc/pacman.d/hooks/zz-sbctl.hook
```

## Finishing up

Set the root password with:

```sh
passwd
```

Exit the `chroot` environment and unmount partitions:

```sh
swapoff /mnt/swapfile /mnt/swapfile.suspend
umount -R /mnt
```

## Post-installation steps

After the installed OS has been booted up:

- Connect to the network
- Permit root login
- Start SSH server for Ansible to connect

```sh
systemctl enable NetworkManager.service
systemctl start NetworkManager.service
systemctl start sshd.service
```

# Maintenance

## Reenrolling the TPM key

```sh
systemd-cryptenroll $rootpart --wipe-slot=tpm2
systemd-cryptenroll $rootpart --tpm2-device=auto
```

If the `--tpm2-device=auto` command fails, either try again later, or try
installing `tpm2-tss`.

## Windows fucking up the boot partition

If the `Linux` boot entry inside the UEFI is gone, boot into a live USB and
re-add it using these commands. Use a tool like `lsblk` to find the
required partition.

```sh
export disk=/dev/nvme2n1
export bootpart="$disk"p1
fsck.fat -aw $bootpart
efibootmgr --create --disk $disk --loader arch-linux.efi --unicode
```

# For later investigation

## Plasma recommendations

[Here's](https://community.kde.org/Distributions/Packaging_Recommendations) a list of
packages and other settings recommended for Plasma. This also contains a section
about PulseAudio/PipeWire.

Wayland setup on Nvidia GPUs ([link](https://community.kde.org/Plasma/Wayland/Nvidia)).

## Pipewire vs PulseAudio

`plasma-pa` depends on `pulseaudio` package, but `pipewire-pulse` can cover for it.
Need to check that on a fresh installation explicitly defining the `pipewire-pulse`
package won't show any prompts or errors.

## Bluetooth pairing and dual booting

https://wiki.archlinux.org/title/Bluetooth#Dual_boot_pairing

## Firefox profiles

https://ffprofile.com for creating new profiles with selected defaults.
