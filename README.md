# Guide for install Arch Linux KDE 

## 1. Download the ISO file form archlinux.org
[Download ISO file](https://archlinux.org/releng/releases/)

## 2. Enter to BIOS and set USB bootlive into highest priority
This is really depend on type of laptop to enter the BIOS, I don't know but Lenovo just repeat F2 when you power on the laptop 

## 3. Check the partition
```bash
lsblk

# My partition is 

NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 931.5G  0 disk
└─sda1        8:1    0 931.5G  0 part /run/media
nvme0n1     259:0    0 238.5G  0 disk
├─nvme0n1p1 259:1    0   529M  0 part
├─nvme0n1p2 259:2    0  65.5M  0 part
├─nvme0n1p3 259:3    0    16M  0 part
├─nvme0n1p4 259:4    0 102.8G  0 part
├─nvme0n1p5 259:5    0   5.3G  0 part
├─nvme0n1p6 259:6    0   512M  0 part /boot
└─nvme0n1p7 259:7    0 129.3G  0 part /home
                                      /var
                                      /cache
                                      /log
                                      /


```
I'm use p6 as EFI system and p7 as Linux FileSystem, you maybe different, just set the EFI and Linux FileSystem in what partition you want 


## 4. Create and format the partition 
```bash 
cfdisk /dev/nvme0n1 
```
Create that EFI system min 512MB and Linux Filesystem min 30GB, set the type, write and quit 

```bash
/dev/nvme0n1p6 512MB EFI system (partition1)
/dev/nvme0n1p7 30GB Linux FileSystem (partition2)

#if you use swap
/dev/nvme0n1p5 4GB Swap (partition3)
```

### 4.1 Swap (Not Recommendation)
```bash 
mkfs.ext4 /dev/partition3 
mkswap /dev/partition3 
swapon /dev/partition3 
```

## 5. Create the partition with btrfs
```bash 
mkfs.fat -F32 /dev/nvme0n1p6
mkfs.btrfs -f /dev/nvme0n1p7 
mount /dev/nvme0n1p7 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
umount /mnt

# If you create swap 
mkswap /dev/nvme0n1p5
swapon /dev/nvme0n1p5

```

## 6. Mount the partition 
```bash
mount -o noatime,compress=zstd,subvol=@ /dev/nvme0n1p7 /mnt
mkdir -p /mnt/{home,var,log,cache,boot}
mount -o noatime,compress=zstd,subvol=@home /dev/nvme0n1p7 /mnt/home
mount -o noatime,compress=zstd,subvol=@var /dev/nvme0n1p7 /mnt/var
mount -o noatime,compress=zstd,subvol=@log /dev/nvme0n1p7 /mnt/log
mount -o noatime,compress=zstd,subvol=@cache /dev/nvme0n1p7 /mnt/cache
mount /dev/nvme0np6 /mnt/boot 
```

### 6.1 Connect the network with iwd
```bash 
iwctl station wlan0 connect "Wifi_name"

# If this doesn't work enter iwctl and input command `device list` to know if tour adapter on or off 

# If off set this command `device name set-property Powered on` and continue with 'adapter adapter_name set-property Powered on'

# Scan the station name with 'station name scan' and get the wifi list with 'station station_name get-networks' in my case station_name is wlan0 

# And then 'station station_name connect "Wifi_name'

```
For more detail check this [wiki](https://wiki.archlinux.org/title/Iwd)


## 7. Install the core with pacstrap
```bash 
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs
```

## 8. Generate fstab
```bash 
genfstab -U /mnt >> /mnt/etc/fstab 
```

## 9. Enter into root
```bash 
arch-chroot /mnt 
```

# 10. Set the base configuration 
```bash 
# Set the country you live 
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc

# Set the language
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Set the hostname(Don't change, this is not not user name)
echo "archlinux" > /etc/hostname

# Set the locale.gen 
# With nvim 
nvim /etc/locale.gen 

# delete # from 
#en_US.UTF-8 UTF-8 -> en_US.UTF-8 UTF-8
#id_ID.UTF-8 UTF-8 -> id_ID.UTF-8 UTF-8

:wq

# With nano 
nano /etc/locale.gen 

# delete # from 
#en_US.UTF-8 UTF-8 -> en_US.UTF-8 UTF-8
#id_ID.UTF-8 UTF-8 -> id_ID.UTF-8 UTF-8

CTRL+o (save) CTRL+x (exit) 

# And then 
locale-gen 

# Make sure nvim or nano installed with pacman -S nvim nano 
```

## 11. Set the root user password
```bash 
passwd 
```

## 12. Install Bootloader(GRUB)
GRUB is the simplest bootloader, you can use rEFInd modern bootloader, but that's kinda pain to set 
```bash
pacman -S grub efibootmgr NetworkManager wpa_supplicant openssh

# Set the GRUB into EFI Arch 
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# Set btrfs option in GRUB 
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="[^"]*/& rootflags=subvol=@/' /etc/default/grub

# Generate GRUB config 
grub-mkconfig -o /boot/grub/grub.cfg 

```

## 13. Enable sysmted services
```bash 
systemctl enable systemd-networkd
systemctl enable systemd-resolved
systemctl enable sshd

```

## 14. Exur and Reboot
```bash
exit
umount -R /mnt
reboot

```

## 15. Enter into root account
After reboot use user 'root' and enter your passwd you set before and create new user 
```bash
Username: root 
Password: 'your password'

# Create new user 
useradd -m -G wheel -s /bin/bash name_user
passwd name_user 

# Give sudo privileges 
EDITOR=nvim visudo

# And uncomment this line 
#%wheel ALL=(ALL:ALL) ALL -> %wheel ALL=(ALL:ALL) ALL

:wq 
```

### 15.1 If you can't connect into internet check this 
```bash
sudo systemctl enable NetworkManager
sudo systemctl enable wpa_supplicant 
sudo systemctl start NetworkManager 
sudo systemctl start wpa_supplicant

# Connect the network with nmcli
nmcli device status # For make sure everything is ON 

# For connect into network 
nmcli device wifi connect "Wifi_name" --ask 

# Note: Don't install iwd, iwd caused conflict for NetworkManager 

```
## 16. Install KDE plasma 
```bash 
pacman -S xorg plasma-meta kde-applications sddm && systemctl enable sddm 

# If you confuse what the repo you use when install KDE, just enter use default 
```
Don't ask for GPU, I can't test it because i don't have any amd or nvidia gpu 

For audio I use PipeWire, that's just work for me but if you use for audio software apps pulseaudio maybe better, I don't know that will work but PulseAudio is more modern than PipeWire 
```bash
pacman -S pipewire pipewire-pulse wireplumber pipewire-alsa pipewire-jack
systemctl enable --now pipewire wireplumber

```
Set ~/.xinitrc file to automate kde plasma
```bash 
nvim ~/.xinitrc
exec startplasma-x11

:wq
```

and REBOOT 

## 17. Set snapshots
```bash 
sudo pacman -S snapper grub-btrfs

# Set the snapper
sudo snapper -c root create-config / 

# Check the result 
sudo snapper list-configs

# If you want create manual snapshots 
sudo snapper -c root create --description "Stable(Maybe)"

# For see what snapshot you have 
sudo snapper -c root list 

# The output
 # | Type   | Pre # | Date                      | User | Description
---+--------+-------+---------------------------+------+-----------------
 1 | single |       | 2025-03-30 10:00:00       | root | Stable(Maybe)

# How to rollback (if you break your system after update)
sudo napper -c root rollback ID_snapshot (you can see id in that COLUMN #)

# If you have too much snapshot, you can delete with
sudo snapper -c root delete ID_snapshot 

# Or if you want delete all 
sudo snapper -c root delete --sync number < $(snapper -c root list | awk '$4 < "'$(date -d "-10 days" +%F%T)'" {print $1}')

# If you tired create manual snapshot just automate 
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer 

# Don't forget to install this package 
sudo pacman -S snap-pac # for automate 

# Integrity with GRUB, you can select the rollback version you want very clearly
sudo systemctl enable --now grub-btrfsd.service 
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
