# Arch linux + Framework 13 AMD 7040 + Full-Disk Encryption + TMP2.0 + Xfce4 

## Hardware
* Framework 13 
* AMD Ryzen 7 AMD Ryzen 7 7840
* 16GB RAM
* 1Tb SSD
* Dongle usb-c with ETH

## Preparation

Boot up [Arch Linux ISO](https://archlinux.org/download/) and do the following:
* Device connected by ethernet interface

## Configuration file
All configuration file modified are in src folder

## Boot by usb stick

### Load keyboard map
```sh
loadkeys it
```

### Check connectivity
```sh
ip addr
```
the command should return
```
2: enp195s0f3u1u4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether XX:XX:XX:XX:XX:XX brd ff:ff:ff:ff:ff:ff
    inet 192.168.XXX.XXX/24 brd 192.168.12.255 scope global dynamic noprefixroute enp195s0f3u1u4
       valid_lft 86395sec preferred_lft 86395sec
	   ...
```
```sh
ping archlinux.org
```

### Disk partitioning
I keep /home dir in separate partition and i would like set the disk like this:  
1 EFI   512Mb  
2 /	100Gb  
3 /home 850Gb  

```sh
fdisk /dev/nvme0n1
```
With the following sequence of characters we will obtain the desired partitioning (I assume the disk has 512 byte sectors): 
 - Command: g
 - Command: n
 - Partition number: <enter>
 - First sector: <enter>
 - Last sector ...: 1046529
 - Command: t
 - Partition type or alias: 1 _(set EFI type it's very important)_
 - Command: n
 - Partition number: <enter>
 - First sector: <enter>
 - Last sector ...: 208664577
 - Command: n
 - Partition number: <enter>
 - First sector: <enter>
 - Last sector ...: <enter>
 - Command: p (check if all partition have a right dimensioning)
 - Command: w

to set the first EFI partition when fdisk is still open:   
 - t
 - 1
 - 1
 - w

> [!WARNING]  
> The first partition must be EFI type

### Format EFI partiton
```sh
mkfs.fat -F32 -n EFI /dev/nvme0n1
```
### Encrypt and format root partiton
```sh
cryptsetup luksFormat -h sha256 /dev/nvme0n1p2
or
cryptsetup luksFormat --type luks1 --use-random -S 1 -s 512 -h sha512 -i 5000 /dev/nvme0n1p2
cryptsetup luksHeaderBackup /dev/nvme0n1p2 --header-backup-file /root/system-header-backup.img
cryptsetup open /dev/nvme0n1p2 system
mkfs.ext4 -L system /dev/mapper/system
```

### Encrypt and format home partiton
```sh
cryptsetup luksFormat -h sha256 /dev/nvme0n1p3
or
cryptsetup luksFormat --type luks1 --use-random -S 1 -s 512 -h sha512 -i 5000 /dev/nvme0n1p3
cryptsetup luksHeaderBackup /dev/nvme0n1p3 --header-backup-file /root/home-header-backup.img
cryptsetup open /dev/nvme0n1p3 home
mkfs.ext4 -L home /dev/mapper/home
```

### Mount partitions
```sh
mount LABEL=system /mnt
mkdir /mnt/boot
mkdir /mnt/home
mount LABEL=EFI /mnt/boot
mount LABEL=home /mnt/home
```

### Install system base
```sh
pacstrap /mnt base linux linux-firmware vim
```

### Update mirror list
```sh
reflector -c it > /mnt/etc/pacman.d/mirrorlist
```

### Populate fstab 
```sh
genfstab -L /mnt >> /mnt/etc/fstab
```

### Copy header of encrypted partitions
```sh
cp /root/system-header-backup.img /mnt/root
cp /root/home-header-backup.img /mnt/root
```
### Chroot to new system
```sh
arch-chroot /mnt
```

#### Set localtime
```sh
ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime
```

#### Uncomment desired locales
```sh
vim /etc/locale.gen
```
exaple uncomment:  
	en_GB.UTF-8 UTF-8

#### Generate the locales
```sh
locale-gen
```
#### Set locale envirorment
```
vim /etc/locale.conf
```
exaple:  
	LANG="en_GB.UTF-8"  
	LANGUAGE="en_GB.UTF-8"  
	LC_ALL="en_GB.UTF-8"  
	LC_COLLATE="C.UTF-8"  
	LC_CTYPE="C.UTF-8"  

#### Set the RTC from the system time
```sh
hwclock --systohc
```

#### Set terminal keyboard map 
```sh
vim /etc/vconsole.conf
```
exaple set: KEYMAP=it

#### Set hostname
```sh
vim /etc/hostname
```
exaple set: 
	XXXX-linux

#### Set hosts
```sh
vim /etc/hosts
```
example set:  
	127.0.0.1	localhost  
	127.0.1.1	XXXX-linux.local	XXXX-linux  
 
#### Install and set networking
```sh
pacman -S wpa_supplicant networkmanager
systemctl enable NetworkManager
```
#### Install and set Midnight Commander
(Optional)
```sh
pacman -S mc
vim /etc/profile.d/editor.sh
```
set:  
	EDITOR=/usr/bin/mcedit  

#### Configure HOOKS for initramfs
```sh
vim /etc/mkinitcpio.conf 
```
insert the follow config:  
HOOKS=(systemd autodetect modconf kms keyboard sd-vconsole block sd-encrypt filesystems resume fsck)  
> [!WARNING]  
> Maintain the right module sequengce

#### Install AMD microcode 
```sh
pacman -S amd-ucode
```

#### Create initramfs
```sh
mkinitcpio -p linux
```

#### Install systemd-boot
```sh
bootctl install
```

#### Config systemd-boot loader
```sh
vim /boot/loader/loader.conf
```
insert the follow config:  
	default arch*.conf  
	timeout 5  
	editor yes  
	console-mode auto  

#### Config systemd-boot deafult entry
```sh
vim /boot/loader/entries/arch.conf
```
insert the follow config:  
	title Arch Linux  
	linux /vmlinuz-linux  
	initrd /amd-ucode.img  
	initrd /initramfs-linux.img  
	options rd.luks.name=</dev/disk/by-uuid>=system rd.luks.name=</dev/disk/by-uuid>=home root=/dev/mapper/system acpi_osi="!Windows 2000" amdgpu.sg_display=0 nowatchdog rw splash

> [!WARNING]  
> Substitute this </dev/disk/by-uuid> with right uuid partition identifier

#### Config systemd-boot fallback entry
```sh
vim /boot/loader/entries/arch-fallback.conf
```
insert the follow config:  
	title Arch Linux  
	linux /vmlinuz-linux  
	initrd /amd-ucode.img  
	initrd /initramfs-linux-fallback.img
	options rd.luks.name=</dev/disk/by-uuid>=system rd.luks.name=</dev/disk/by-uuid>=home root=/dev/mapper/system acpi_osi="!Windows 2000" amdgpu.sg_display=0 nowatchdog rw splash   
> [!WARNING]  
> Substitute this </dev/disk/by-uuid> with right uuid partition identifier

#### Set root passwd
```sh
passwd
```

#### Reboot system
```sh
exit
reboot
```

## Boot from internal NVMe
Login with root user  

#### Force keyboard mapp
If necessary set keyboard map
```sh
localectl set-keymap it
```

#### Update the date by ntp protocol at system boot 
```sh
timedatectl set-ntp 1
```

#### Add user
```sh
useradd -m wheel,storage -G johndoe
passwd johndoe
```

#### Install base services
```sh
pacman -S cronie apparmor avahi nss-mdns reflector sudo ntp logrotate

systemctl enable cronie  apparmor avahi-daemon reflector ntpd
systemctl start cronie  apparmor avahi-daemon reflector ntpd
```

#### Configure base services

Set DNS Multicast in Name Service Switch congihuration file  
```sh
vim /etc/nsswitch.conf
```
Edit hosts key like this:  
	hosts: mymachines mdns_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] files myhostname dnsloadk  

Add user to sudoers
```sh
vim /etc/sudoers.d/johndoe
```
Insert the follow row:  
	johndoe	ALL=(ALL:ALL) ALL  

```sh
vim /boot/loader/entries/arch.conf
```
Update the follow config:  
	options rd.luks.name=</dev/disk/by-uuid>=system ... __lsm=landlock,lockdown,yama,intergrity,apparmor,bpf__ rw splash  
> [!WARNING]  
> Substitute this </dev/disk/by-uuid> with right uuid partition identifier

```sh
vim /boot/loader/entries/arch-fallback.conf
```
Update the follow config:  
	options rd.luks.name=</dev/disk/by-uuid>=system ...  __lsm=landlock,lockdown,yama,intergrity,apparmor,bpf__ rw splash  
> [!WARNING]  
> Substitute this </dev/disk/by-uuid> with right uuid partition identifier


#### Enable swap 
~~touch /var/swap.img~~  
~~chmod 600 /var/swap.img~~  
~~swapoff /dev/mapper/server--vg-swap_1~~  
~~mkswap /var/swap.img~~  
~~swapon /var/swap.img~~  
```sh
fallocate -l 16G /var/swap.img
```
Update fstab  
```sh
vim /etc/fstab
```
Append this:  
	/swap.img	none swap defaults 0 0 
```sh
systemctl daemon-reload
```

Optimize full ram utilization
```sh
vim  /etc/sysctl.d/swap.conf 
```
Add:
	vm.swappiness=20
 	vm.page-cluster=0

#### Enable zram
(optional)
```sh
vim /etc/modules-load.d/zram.conf
```
Add:
	zram  

```sh
 vim /etc/fstab
```
Add:
	...
	dev/zram0 none swap defaults,pri=100 0 0
 
```sh
vim  /etc/sysctl.d/99-zram.rules
```
Add:  
	ACTION=="add", KERNEL=="zram0", ATTR{comp_algorithm}="lz4", ATTR{disksize}="4G", RUN="/usr/bin/mkswap -U clear /dev/%k", TAG+="systemd"  
 
#### Enable PNIN (old if name like ETH0)
(optional)
```sh
vim /boot/loader/entries/arch.conf
```
Update the follow config:  
	options rd.luks.name= ... __net.ifnames=0__  

```sh
vim /boot/loader/entries/arch-fallback.conf
```
Update the follow config:  
	options rd.luks.name= ... __net.ifnames=0__  

#### Enable mounf filesystem in /media/VolumeName
```sh
vim  /etc/sysctl.d/99-udisk2.rules
```
Add:  
	# UDISKS_FILESYSTEM_SHARED  
	# ==1: mount filesystem to a shared directory (/media/VolumeName)  
	# ==0: mount filesystem to a private directory (/run/media/$USER/VolumeName)  
	# See udisks(8)  
	ENV{ID_FS_USAGE}=="filesystem|other|crypto", ENV{UDISKS_FILESYSTEM_SHARED}="1"  

#### Set tpm2
Check if tpm2 has been detected
```sh
systemd-cryptenroll --tpm2-device=list
```
then
```sh
systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "1+7" /dev/nvme0n1p2
systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "1+7" /dev/nvme0n1p3
```
#### Enable FSTrim
(optional) for SSD Optimization
```sh
systemctl enable --now fstrim.timer
```
then edit
```sh
vim /boot/loader/entries/arch.conf
```
Add:  
	options rd.luks.name=</dev/disk/by-uuid>=system ... __rd.luks.options=discard__  

#### For saving some SSD cycles
```sh
pacman -S profile-sync-daemon
systemctl --user enable psd --now
```

#### Enable Timeshift
(optional)
```sh
pacman -S timeshift
```
Attach external drive and create snapshot 
```sh
timeshift --create --snapshot 'clean-distr' --snapshot-device /dev/sda1
```

#### ~~Set suspension on low battery~~
~~vim /etc/udev/rules.d/99-lowbat.rules~~
~~Add this:~~  
	~~#Suspend the system when battery level drop to 5% or lower~~  
	~~SUBSTYSTEM=="power_supply", ATTR{status}="Discharging", ATTR{capacity}="[0-5]", RUN="/run/bin/systemctl hibernate"~~  

#### Install XFCE4
```sh
pacman --needed -S xorg-server xorg-xinit xterm xf86-video-amdgpu xfce4 xfce4-goodies xarchiver network-manager-applet lightdm lightdm-gtk-greeter alsa-utils pulseaudio pavucontrol dbus xdg-dbus-proxy xdg-desktop-portal xdg-desktop-portal-gtk xdg-user-dirs xdg-utils ls man-db man-pages catfish gvfs 
pacman -Rncsu xfburn xfce4-notes-plugin parole xfce4-dict
```

#### Configure LightDM
```sh
/etc/lightdm/lightdm.conf
```
Add under [Seat:*]  
	greeter-session=lightdm-gtk-greeter  

#### Install Bluez
```sh
pacman -S bluez bluez-utils bluemanmanager  pulseaudio-bluetooth
systemctl start bluetooh.service
systemctl enable bluetooh.service
```

#### Install Cups
```sh
pacman -S cups cups-pdf
systemctl enable cups
systemctl start cups
```

#### Update finger reader device
You have to do this only one time if installed on your Framework 13 AMD a firmware older than 01000330
```sh
wget (https://archive.archlinux.org/packages/f/fwupd/fwupd-1.9.5-2-x86_64.pkg.tar.zst)
wget (https://github.com/FrameworkComputer/linux-docs/raw/main/goodix-moc-609c-v01000330.cab)
pacman -U fwupd-1.9.5-2-x86_64.pkg.tar.zst
fwupdtool install --allow-reinstall --allow-older goodix-moc-609c-v01000330.cab
fwupdtool get-history
```
This operation will return an error as reported in the link but eventually the firmware should be updated:  
(https://knowledgebase.frame.work/en_us/updating-fingerprint-reader-firmware-on-linux-for-13th-gen-and-amd-ryzen-7040-series-laptops-HJrvxv_za)  
```sh
reboot
```

#### Update login with fingerprint
```sh
vim /etc/pam.d/system-login
vim /etc/pam.d/xfce4-screensaver
vim /etc/pam.d/system-auth
```
Add this in the first position may must be placed after #%PAM-1.0:  
```sh
auth      sufficient pam_fprintd.so  
```

#### Add developer base tools
```sh
pacman -S base-devel cmake git gdb
```

##### Secure boot
TODO

#### Configuration PPD
```sh
pacman -S power-profiles-daemon
systemctl start power-profiles-daemon.service
systemctl enable power-profiles-daemon.service
```

## Install laptop feaures references
* (https://wiki.archlinux.org/title/laptop)
* (https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate)

## Framerowk configuration references
* (https://community.frame.work/t/arch-linux-on-the-framework-laptop/3843)
* (https://wiki.archlinux.org/title/Framework_Laptop_13#Graphics)

## Arch Linux references
* (https://wiki.archlinux.org/title/installation_guide)
* (https://gist.github.com/orhun/02102b3af3acfdaf9a5a2164bea7c3d6)
* (https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system#Simple_encrypted_root_with_TPM2_and_Secure_Boot)
* (https://wiki.archlinux.org/title/dm-crypt/Device_encryption)

## A big thank to Orhun
A big thank you to [orhun](https://gist.github.com/orhun) who thanks to his [guide](https://gist.github.com/orhun/02102b3af3acfdaf9a5a2164bea7c3d6) gave me inspiration for this

## Thanks to fix 
* [trail744](https://community.frame.work/t/guide-my-procedure-for-installing-arch-linux-xfce-on-amd-framework-13/41451/7)
