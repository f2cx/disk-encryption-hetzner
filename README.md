## disk-encryption-hetzner

### Hardware Setup
- 2x HDD HWR
- RAM ECC

### 1) Boot into the Rescue System

- Boot to the rescue system via hetzners server management page using SSH Key
- install a minimal Ubuntu 18.04 LTS with hetzners "installimage"

```
#SET HOSNTAME: XXX

PART /boot ext3 1G
PART lvm vg0 all

LV vg0 root / ext4 1250G
LV vg0 storage /storage ext4 1250G
LV vg0 swap swap swap 20G
```
- after you adjusted all parameters in the install config file, press F10 to install the ubuntu minimal system
- reboot and ssh into your fresh installed ubuntu

### 2) Boot into the Ubuntu System

```
apt update && apt-get dist-upgrade && apt install busybox dropbear
sed -i 's/BUSYBOX=auto/BUSYBOX=y/g' /etc/initramfs-tools/initramfs.conf
mkdir -p /etc/initramfs-tools/root/.ssh/
cp /root/.ssh/authorized_keys /etc/initramfs-tools/root/.ssh/
```

### 3) Boot again into the Rescue System

```
screen
mkdir /oldroot/ && mount /dev/mapper/vg0-root /mnt/ && rsync -avz /mnt/ /oldroot/
umount /mnt/
```

```
vgcfgbackup vg0 -f vg0.freespace
vgremove vg0
CRYPTPASSWORD=`pwgen 64 1`
echo "Please save the encryption key: $CRYPTPASSWORD"
cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha256 --iter-time 6000 luksFormat /dev/sda2
cryptsetup luksOpen /dev/sda2 cryptroot
pvcreate /dev/mapper/cryptroot
```

```
UUID=`blkid /dev/mapper/cryptroot | awk -F'"' '{print $2}'`
cp vg0.freespace /etc/lvm/backup/vg0
sed -i "25 c\                        id = \"$UUID\"" /etc/lvm/backup/vg0
sed -i "26 c\                        device = \"/dev/mapper/cryptroot\"        # Hint only" /etc/lvm/backup/vg0
vgcfgrestore vg0
vgchange -a y vg0
mkfs.ext4 /dev/vg0/root
mkswap /dev/vg0/swap
mount /dev/vg0/root /mnt/
rsync -avz /oldroot/ /mnt/
mount /dev/sda1 /mnt/boot
mount --bind /dev /mnt/dev
mount --bind /sys /mnt/sys
mount --bind /proc /mnt/proc
chroot /mnt
```

```
echo "cryptroot /dev/sda2 none luks" > /etc/crypttab
update-initramfs -u
update-grub
grub-install /dev/sda

echo '#!/bin/sh -e' >> /etc/rc.local
echo "/sbin/ifdown --force eth0" >> /etc/rc.local
echo "/sbin/ifup --force eth0" >> /etc/rc.local
echo "exit 0" >> /etc/rc.local

sed -i 's/#Port 22/Port 2201/g' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
exit
```

```
umount /mnt/boot /mnt/proc /mnt/sys /mnt/dev
umount /mnt
sync
reboot
```

```
ssh -P22 root@<yourserverip>
/lib/cryptsetup/askpass "passphrase: " > /lib/cryptsetup/passfifo
exit
```
