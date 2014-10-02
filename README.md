# Getting started

This copy&paste manual allow you to set up fully encrypted Ubuntu 12.04 64-bit base system with Software RAID 1, with possibility to unlock encrypted root using SSH.
Commands below could be easily adjusted, to get the disk/partitions layout you want.
It might also work to create a Debian installation.
I'm using `debootstrap` utility here, so actual OS list depends on what your version of `debootstrap` support.
The partitions layout you get "by default":

| Block device | Filesystem | Mountpoint | Size | Comment
| ------------ | ---------- | ---------- | ---- | -------
| /dev/md0     | ext2       | /boot      | 200M | plain partition
| /dev/mapper/enc_sys | ext4| /          | 4G   | encrypted root partition built on top of /dev/md1
| /dev/mapper/enc_home| ext4| /home      | -    | will take rest of free space, encrypted /home partition built on top ot /dev/md2

Below I created a degraded RAID1 array, since I'm too lazy to add a 2nd HDD to my virtual machine.
This is easily adjustable though.

Boot a machine from some kind of LiveCD (I was using [SystemRescueCD 3.7.0](http://www.sysresccd.org/Download)), and go ahead :)

# Rescue environment

After booting a rescue environment we're going to take some preparation steps.

## Partitioning

First, we're going to create partition layout on disk(s).
We will be using *GNU parted* for this:

```
# parted
(parted) open /dev/sda
(parted) mklabel gpt
(parted) mkpart primary ext2 0 200MB
(parted) mkpart primary ext4 200MB 4296MB
(parted) mkpart primary ext4 4296MB 100%
(parted) set 1 boot on
```

Repeat the same steps for the other disk as well (if you have one).
Alternatively, you can just use `sgdisk`:

```
# sgdisk -R /dev/sdb /dev/sda
# sgdisk -G /dev/sdb
```

2nd command generates a new set of UUIDs for partitions, so don't confuse kernel.

## RAID creation

Now, we'll create Software RAID1.
Here you'll see I create a degraded RAID, since I only have 1 HDD.
2nd HDD can be added later using somethig like `mdadm /dev/md0 --add /dev/sdb1`.
Also note I'm using `--metadata=0.90` for the /boot partition - that's needed for GRUB to recognize our /boot partition.

```
# mdadm --create /dev/md0 --metadata=0.90 -l raid1 -f -n 1 /dev/sda1
# mdadm --create /dev/md1 -l raid1 -f -n 1 /dev/sda2
# mdadm --create /dev/md2 -l raid1 -f -n 1 /dev/sda3
```

## Filesystems and encryption

Time to create some filesystems.
We will go with *ext2* for /boot, and *ext4* for the rest.

- /boot partition:
```
# mkfs -t ext2 /dev/md0
```

- / and /home paritions - since we want these encrypted, we first create an encryption layer, on top of which we will build the actual *ext4* filesystem.
*NOTE:* `cryptsetup luksFormat` command generates encryption keys, and is using pseudo-random number generator for this - it can take some time for this command to complete.

```
# cryptsetup luksFormat /dev/md1
# cryptsetup luksFormat /dev/md2
# cryptsetup open /dev/md1 enc_sys
# cryptsetup open /dev/md2 enc_home
# mkfs -t ext4 /dev/mapper/enc_sys
# mkfs -t ext4 /dev/mapper/enc_home
```

## Installing Ubuntu/Debian into chroot

First, we'll create chroot directory, mount our encrypted root partition we prepared earlier and create some needed directories:

```
# mkdir /mnt/ubuntu_chroot
# mount /dev/mapper/enc_sys /mnt/ubuntu_chroot/
# mkdir -p /mnt/ubuntu_chroot/{boot,home,proc}
```

And now we're going to install base system with `debootstrap`:
*NOTE:* Choose mirror closest to you for faster downloads [here](https://launchpad.net/ubuntu/+archivemirrors)

```
# debootstrap --arch=amd64 precise /mnt/ubuntu_chroot http://us.archive.ubuntu.com/
```

After `deboostrap` finishes we need to create some needed files.

- `/etc/fstab`:

```
# cat > /mnt/ubuntu_chroot/etc/fstab<<EOFSTAB
proc                    /proc   proc    defaults    0 0
/dev/md0                /boot   ext2    defaults    0 0
/dev/mapper/enc_sys     /       ext4    defaults    0 1
/dev/mapper/enc_home    /home   ext4    defaults    0 2
EOFSTAB
```

- `/etc/apt/sources.list`:
*NOTE:* you can generate entries for sources.list [here](http://repogen.simplylinux.ch/generate.php)
Example below contains main, restricted, universe and multiverse repositories - which might not be what you want.

```
# cat > /mnt/ubuntu_chroot/etc/apt/sources.list <<EOSOURCES
# Ubuntu Main Repos
deb http://cz.archive.ubuntu.com/ubuntu/ precise main restricted universe multiverse 

# Ubuntu Update Repos
deb http://cz.archive.ubuntu.com/ubuntu/ precise-security main restricted universe multiverse 
deb http://cz.archive.ubuntu.com/ubuntu/ precise-updates main restricted universe multiverse 
deb http://cz.archive.ubuntu.com/ubuntu/ precise-proposed main restricted universe multiverse 
deb http://cz.archive.ubuntu.com/ubuntu/ precise-backports main restricted universe multiverse 
EOSOURCES
```

# Step 2. Time to go chroot

This is it, time to continue our OS setup - for this we will go to the chroot environment.
All further commands are executed in the context of our newly installed OS - keep that in mind:

```
# env LANG=C env HOME=/root chroot /mnt/ubuntu_chroot /bin/bash -i
```

## Initial configuration and software install

This is mandatory:

```
# mount -t proc /proc /proc
# mount -a
```

Get repository metadata, and configure locales:

```
# apt-get update
# apt-get -y install locales
# dpkg-reconfigure locales
```

Setup hostname and basic networking (adjust `HOST=` to your flavor):

```
# HOST='ubuntu-system'
# echo "$HOST" > /etc/hostname
# hostname $HOST
# echo -e "\n127.0.0.1 localhost $HOST" >> /etc/hosts
# echo "nameserver 8.8.8.8" >> /etc/resolv.conf
```

Now let's get the kernel, bootloader, and couple of tools:

```
# apt-get -y install linux-base linux-image-server \
           linux-headers-server grub mdadm cryptsetup \
           hashalot initramfs-tools \
           openssh-server dropbear
```

Install bootloader and update its config.
*NOTE:* We're installing GRUB on the actual HDD, not the Software RAID - that's because at the time BIOS boots a machine - it does not know anything about our Software RAID, it only knows that there're actual HDD(s).

```
# grub-install /dev/sda
# update-grub
```

## Setting up init filesystem

The fun part starts here.
For our init to be able to find our RAID partitions, and crypted parittions, we'd need some additional kernel modules.
Let's add them to the `initramfs-tools` configuration:

```
# cat >> /etc/initramfs-tools/modules<<EOMOD
dm_mod
dm_crypt
dm_raid
raid1
sha256
aes_i586
EOMOD
```

We will be authenticating to the temporary SSH server (to unlock encrypted volumes) using [SSH public key](https://help.github.com/articles/generating-ssh-keys).
Let's put these in place (don't forget to actually put your public key there :) ).
We will also copy Dropbear generated keys to initramfs, to avoid "Unknown host key" warnings every time we connect to the machine to unlock our volumes, and also put our public key for root account on the actual system to be able to log in to it as well:

```
# mkdir -p /root/.ssh
# mkdir -p /etc/initramfs-tools/root/.ssh/
# mkdir -p /etc/initramfs-tools/etc/dropbear/
# cp /etc/dropbear/dropbear_* /etc/initramfs-tools/etc/dropbear/
# cat >> /etc/initramfs-tools/root/.ssh/authorized_keys<<EOAK
[ PUT YOUR SSH PUBLIC KEY HERE ]
EOAK
# cat >> /root/.ssh/autorized_keys <<EOR
[ PUT YOUR SSH PUBLIC KEY HERE ]
EOR
```

Now, we will set up some hook scripts for `initramfs-tools`.
This one goes to `/etc/initramfs-tools/hooks/fix-login.sh`:

```bash
#!/bin/sh
 
PREREQ=""
 
prereqs()
{
    echo "$PREREQ"
}
 
case $1 in
# get pre-requisites
prereqs)
    prereqs
    exit 0
    ;;
esac
 
cp $(dpkg -L libc6 | grep libnss_ | tr '\n' ' ') "${DESTDIR}/lib/"
```

And this one - to `/etc/initramfs-tools/hooks/crypt_unlock.sh`:

```bash
#!/bin/sh
 
PREREQ="dropbear"
 
prereqs() {
    echo "$PREREQ"
}
 
case "$1" in
    prereqs)
        prereqs
        exit 0
    ;;
esac
 
. "${CONFDIR}/initramfs.conf"
. /usr/share/initramfs-tools/hook-functions
 
if [ "${DROPBEAR}" != "n" ] && [ -r "/etc/crypttab" ] ; then
    cat > "${DESTDIR}/bin/unlock" << EOF
#!/bin/sh
if PATH=/lib/unlock:/bin:/sbin /scripts/local-top/cryptroot; then
    kill \`ps | grep cryptroot | grep -v "grep" | awk '{print \$1}'\`
    exit 0
fi
exit 1
EOF
 
    chmod 755 "${DESTDIR}/bin/unlock"
 
    mkdir -p "${DESTDIR}/lib/unlock"
cat > "${DESTDIR}/lib/unlock/plymouth" << EOF
#!/bin/sh
[ "\$1" == "--ping" ] && exit 1
/bin/plymouth "\$@"
EOF
 
    chmod 755 "${DESTDIR}/lib/unlock/plymouth"
 
    echo >> ${DESTDIR}/etc/motd
    echo >> ${DESTDIR}/etc/motd
    echo To unlock root-partition run "unlock" >> ${DESTDIR}/etc/motd
 
fi
```

Give them *execute* permissions, otherwise `initramfs-tools` won't be able to pick these up:

```
# chmod +x /etc/initramfs-tools/hooks/crypt_unlock.sh
# chmod +x /etc/initramfs-tools/hooks/fix-login.sh
```

Now we will set up `crypttab(5)`, to let OS know which encrypted volumes we want to have:

```
# cat > /etc/crypttab<<EOCT
# name   device   options type
enc_sys  /dev/md1 none luks
enc_home /dev/md2 none luks
EOCT
```

Finally, we will update bootloader configuration, initramfs, and reboot :)

```
# update-grub
# update-initramfs -u
# #reboot
```

When booting up, system will spawn Dropbear to allow you to SSH login to it, and unlock your drives.

# References

Most of the information is taken from:

- [hacksr blog](http://hacksr.blogspot.cz/2012/05/ssh-unlock-with-fully-encrypted-ubuntu.html)
- [Hanrahabr](http://habrahabr.ru/post/147522/)
- [ArchLinux Wiki](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_a_non-root_file_system)

