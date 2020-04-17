# Arch Linux with LUKS and btrfs on a hetzner server (DRAFT)
This guide should show you how to set up an System with the following specifications on an hetzner server:
* Arch Linux
* btrfs
* LUKS

## Guide
### 1. Configure and Install Image
#### 1.1
Login to Hetzner Rescue System
```bash
ssh root@your_server_ip
```
#### 1.2
Create the autosetup by executing

```bash
nano /autosetup
```

and saving the following content into this file:

```bash
##  Hetzner Online GmbH - installimage - config

DRIVE1 /dev/sda
DRIVE2 /dev/sdb

##  SOFTWARE RAID:
## activate software RAID?  < 0 | 1 >
SWRAID 1

## Choose the level for the software RAID < 0 | 1 | 10 >
SWRAIDLEVEL 1

##  BOOTLOADER:
BOOTLOADER grub

##  HOSTNAME:
HOSTNAME hetzner-arch-luks
#Adapt the hostname to your needs

##  PARTITIONS / FILESYSTEMS:
PART /boot  btrfs     512M
PART lvm    vg0       all
LV vg0   swap   swap     swap         8G
LV vg0   root   /        btrfs        10G

##  OPERATING SYSTEM IMAGE:
IMAGE /root/.oldroot/nfs/install/../images/archlinux-latest-64-minimal.tar.gz
```
#### 1.3
Afterwards install the image by executing the following command:

```bash
installimage
```
#### 1.4
When the setup finished restart the server via
```bash
reboot
```

### 2. Setup System
#### 2.1
Login to your server:

```bash
ssh-keygen -f "$HOME/.ssh/known_hosts" -R your_server_ip #revokes old ssh_host
ssh root@your_server_ip
```
#### 2.2
Update the system:
```bash
pacman -Syyu
```
#### 2.3
Install basic administration software:
```bash
pacman -Syyu nano
```

#### 3. Prepare System for Unlocking via SSH
#### 3.1 Execute the following script
```bash
# Install software
pacman -Syyu busybox mkinitcpio-dropbear mkinitcpio-utils mkinitcpio-netconf
#Copy ssh-key
cp -v ~/.ssh/authorized_keys /etc/dropbear/root_key
```
#### 3.2
Replace the following line in **/etc/mkinitcpio.conf**
```
HOOKS=(base udev autodetect modconf block mdadm_udev lvm2 filesystems keyboard fsck)
```
with
```
HOOKS=(netconf dropbear encryptssh base udev autodetect modconf block mdadm_udev lvm2 filesystems keyboard fsck)
```

### 4. Activate Encryption
#### 4.1
Activate the rescue system https://robot.your-server.de/server
#### 4.2
Afterwards reboot the system by entering:

```bash
reboot
```
#### 4.3
Login to the rescue system:
```bash
ssh-keygen -f "$HOME/.ssh/known_hosts" -R your_server_ip #revokes old ssh_host
ssh root@your_server_ip
```

#### 4.4
Mount the "system" by:
```bash
vgscan -v
vgchange -a y
mount /dev/mapper/vg0-root /mnt
```
#### 4.5
Copy "system":
```bash
# Resync unterbrechen
echo 0 >/proc/sys/dev/raid/speed_limit_max
mkdir /oldroot
cp -va /mnt/. /oldroot/.
# Resync fortsetzen
echo 200000 >/proc/sys/dev/raid/speed_limit_max
```
#### 4.6
Unmount the "system" by:
```bash
umount /mnt
```

#### 4.7
Delete unencrypted LVM-Volume-Group by executing:
```bash
vgremove vg0
```

#### 4.8
Check drive state:
```bash
cat /proc/mdstat
```
#### 4.9
Encrypt MD1 by executing:
```bash
cryptsetup --cipher aes-xts-plain64 --key-size 256 --hash sha256 --iter-time=10000 luksFormat /dev/md1
cryptsetup luksOpen /dev/md1 cryptroot
pvcreate /dev/mapper/cryptroot
vgcreate vg0 /dev/mapper/cryptroot
lvcreate -n swap -L8G vg0
lvcreate -n root -L10G vg0
mkfs.btrfs /dev/vg0/root
mkswap /dev/vg0/swap
```

#### 4.10
Mount encrypted :
```bash
mount /dev/vg0/root /mnt
```

#### 4.12
Copy "system":
```bash
# Resync unterbrechen
echo 0 >/proc/sys/dev/raid/speed_limit_max
cp -av /oldroot/. /mnt/.
# Resync fortsetzen
echo 200000 >/proc/sys/dev/raid/speed_limit_max
```

#### 4.13
Integrate finale installation:
```bash
mount /dev/md0 /mnt/boot
mount --bind /dev /mnt/dev
mount --bind /sys /mnt/sys
mount --bind /proc /mnt/proc
chroot /mnt
```

#### 4.14
```bash
echo "cryptroot /dev/md1 none luks" >> /etc/crypttab
```

# I think here the mess starts....
#### 4.15
rewrite initramfs ***?assume this should be right???***
```bash
mkinitcpio
mkinitcpio -p linux
```
Missing **initramfs neu schreiben** **GRUB neu schreiben**

ssh-keygen -b 4096 -t rsa -m PEM -f /etc/ssh/ssh_host_rsa_key
dropbearconvert openssh dropbear /etc/ssh/ssh_host_rsa_key /etc/dropbear/dropbear_rsa_host_key
* https://github.com/random-archer/mkinitcpio-systemd-tool/issues/21
* https://github.com/random-archer/mkinitcpio-systemd-tool/issues/17
* https://bbs.archlinux.org/viewtopic.php?id=250512

from point 4 on I have questions:
https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#chkboot

check this one also out:
* https://blog.simonszu.de/set-up-luks-on-a-physical-hetzner-server-with-debian/ -> specially the part about dropbear configuration and ssh keys

https://gist.github.com/HardenedArray/31915e3d73a4ae45adc0efa9ba458b07

https://code.trafficking.agency/arch-linux-remote-unlock-root-volume-with-mdraid-and-dmcrypt.html

#### 4.15
```bash
exit
umount /mnt/boot /mnt/proc /mnt/sys /mnt/dev
umount /mnt
sync

#Neustart
reboot


```

## Sources
The code is adapted from the following guides:

* http://daemons-point.com/blog/2019/10/20/hetzner-verschluesselt/
* https://www.howtoforge.com/using-the-btrfs-filesystem-with-raid1-with-ubuntu-12.10-on-a-hetzner-server
* https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#Remote_unlocking_(hooks:_netconf,_dropbear,_tinyssh,_ppp)
