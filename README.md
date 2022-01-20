# Manjaro btrfs Encryption without /boot encryption
This guide shows how to install any Manjaro with disk encryption without encrypting the boot partition for faster boot time.
It is based on two great guides ([Here](https://forum.manjaro.org/t/root-tip-do-a-manual-manjaro-installation/12507), [Here](https://archived.forum.manjaro.org/t/howto-install-manjaro-using-uefi-systemd-boot-luks-and-btrfs/116534)) for manually installing manjaro and combines both of them. For further information please look at them.

## 1. Getting an Image
Download an image of Majarom from their official [website](https://manjaro.org/).

## 2. Booting into Live-Image
Create a bootable device and boot into the live image. At startup select language, timezone and keyboard.

## 3. Partitioning disk
Make sure you are targeting the right disk!
### 3.1 Clear drive (optional)
You can overwrite the old data on your disk with random data
```
dd if=/dev/urandom of=/dev/nvme1n1 status=progress bs=10M
```
When your previous partition was encrypted you can skip this step.
### 3.2 Create Partition
1. boot
* At least 512M
* Mark as `EFI boot`
2. root
* Remaining space
* Mark as `Linux File System`

```
cfdisk /dev/nvme1n1
```
### 3.3 Setting up the encryption container
```
cryptsetup luksFormat /dev/nvme1n1
```
Open up the container
```
cryptsetup open /dev/nvme1n1p2 luks
```
### 3.4 Format
```
mkfs.vfat -F32 /dev/nvme1n1p1
mkfs.btrfs /dev/mapper/luks
```
### 3.5 Creating Subvolumes
```
mount /dev/mapper/luks /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
umount /mnt
mount -o subvol=@,ssd,compress=zstd,noatime,nodiratime /dev/mapper/luks /mnt
mkdir /mnt/{boot,home,var}
mount -o subvol=@home,ssd,compress=zstd,noatime,nodiratime /dev/mapper/luks /mnt/home
mount -o subvol=@var,ssd,compress=zstd,noatime,nodiratime /dev/mapper/luks /mnt/var
mount /dev/nvme1n1p1 /mnt/boot
```

## 4. Installation
### 4.1 Base Install of Manjaro
```
basestrap /mnt base btrfs-progs sudo manjaro-zsh-config intel-ucode networkmanager linux54 nano vim systemd-boot-manager mkinitcpio
```
### 4.2 Generate fstab
```
fstabgen -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

## 5. Configure system
### Chroot
```
manjaro-chroot /mnt /bin/zsh
```
### Hostname
```
echo YOU-HOST-NAME > /etc/hostname
```
### Edit /etc/hosts
```
nano /etc/hosts
```
Change to:
```
127.0.0.1   localhost
::1     localhost
127.0.1.1   YOU-HOST-NAME.localdomain   YOU-HOST-NAME
```
### Shell
```
chsh -s /bin/zsh
```
### Timezone
```
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
```
### NetworkManager
```
systemctl enable NetworkManager
```
### Enable ntp client
```
systemctl enable systemd-timesyncd
```
### Console keyboard
```
nano /etc/vconsole.conf
```
to
```
KEYMAP=de-latin1
FONT=
FONT_MAP=
```
### Locale
```
nano /etc/locale.gen
```
uncomment `de_DE.UTF-8`
Generate locale
```
locale-gen
echo LANG=de_DE.UTF-8 > /etc/locale.conf
```
### Set root password
```
passwd
```
### Configure mkinitcpio
```
nano /etc/mkinitcpio.conf
```
to
```
HOOKS=(base udev btrfs keyboard keymap encrypt autodetect modconf block filesystems fsck)
mkinitcpio -p linux54
```
### systemd-boot
```
bootctl --path=/boot install
```
### Create Manjaro loaders
```
sdboot-manage gen
```
### Base config done
```
exit
```
### Install Live Pakages (optional)
#### Read all pakages and save them into a file
```
cat /rootfs-pkgs.txt | awk '{print $1;}' > ~/iso-pkglist.txt
cat /desktopfs-pkgs.txt | awk '{print $1;}' >> ~/iso-pkglist.txt
```
#### Install them
```
pacman -Syy --noconfirm --needed --root /mnt - < ~/iso-pkglist.txt
```
### Reboot
```
umount -R /mnt
reboot
```
## 6. Config the installed System
### 6.1 Login
Login as `root` with the set password.
### 6.2 Add user
```
useradd -mUG lp,network,power,sys,wheel -s /bin/zsh USERNAME
passwd USERNAME
```
### 6.3 Add user to sudo
```
visudo
```
uncomment
```
%wheel ALL=(ALL) ALL
```
### 6.4 Enable Disply Manager
Depending of your selected Desktop Enviroment enable the display manager.
* KDE `systemctl enable sddm`
* Xfce `systemctl enable lightdm`
* Gnome `systemctl enable gdm`





