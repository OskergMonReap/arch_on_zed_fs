# Arch Linux install on ZFS
Living document outlining base installation for home/work PCs, encrypted with `/` on ZFS
- - -
**TO-DO**:
- *refactor into shell scripts for automation, likely one for zsh shell of live env and one for user customizations after reboot*
- *implement CI/CD to pre-build Arch ISO, serve from S3*
- - -
### PART 1
#### Bake our own ISO with ZFS in tow
From our existing Arch system, we'll start by pulling down the `archiso` package.
To avoid issues, all `archiso` steps are to be ran as root:
```
pacman -S archiso
```

We'll make use of the customizable profile included with the package. First, we'll create a build directory, then we'll copy the contents (building blocks) of the profile to the new directory:
```
mkdir ~/Build
cp -r /usr/share/archiso/configs/releng/* ~/Build
```

After `cd`'ing into our new build directory, we will see several new files/folders:

```
Build
|
├── airootfs # Make customizations here
│   ├── etc
│   │   ├── # configuration files...
│   │   └── # ...
│   └── root
│       ├── customize_airootfs.sh
│       └── install.txt
├── build.sh # Used to build ISO
├── packages.both
├── packages.i686
├── packages.x86_64
├── pacman.conf
├── mkinitcpio.conf
├── efiboot
│   └── # bootloader files...
├── isolinux
│   └── isolinux.cfg
└── syslinux
    └── # syslinux files...
```
#### Archiso File Structure Summary
- `airootfs`
    - translates to `/` on final image
    - all files/folders wanted should be copied here
- `packages.{both,i686,x86_64}`
    - list any packages you want installed here
- `pacman.conf`
    - the package manager configuration file
    - list additional repo's here

#### Customizations
Add ZFS repository to the `pacman.conf` file, above all other repo's add `[archzfs]`
Also, go ahead and uncomment `[multilib]`:
```
#
# /etc/pacman.conf
#
[archzfs]
SigLevel = Optional TrustAll
Server = http://archzfs.com/$repo/x86_64
```
Add ZFS package to bottom of our desired `packages` file, `packages.x86_64` in my case:
```
zfs-linux
```

#### Build the image
I tend to reuse these Build directories, so per some other great posts online I will also ship this to my `/tmp` directory to be finally built.. keeping the original clean/in-tact:
```
cp -r ~/Build/ /tmp
cd /tmp/Build
mkdir out
./build.sh -v
```
This will build the image and automatically place it in the `out/` directory we just created!

- - -
## PART 2
### The Installation
#### Partitioning, Encrypting
Assuming all went well in creating our custom ISO, boot into the image and get networking up (if no ethernet, `wifi-menu` it up), then lets partition that disk:
```
parted /dev/sda
(parted) mklabel gpt
(parted) mkpart ESP fat32 1MiB 513MiB
(parted) set 1 boot on
(parted) mkpart primary ext2 513MiB 99%
(parted) align-check optimal 1
(parted) align-check optimal 2
(parted) quit
```
Now, format EFI partition we just created:
```
mkfs.fat -F32 /dev/sda1
```
Now, lets partition for our ZFS pool:
```
parted /dev/sda2
(parted) mklabel gpt
(parted) mkpart ext2 0% 8GB
(parted) mkpart ext2 8GB 100%
(parted) quit
```
Finish setting up first partition as swap:
```
mkswap /dev/disk/by-id/<swap-id>
swapon /dev/disk/by-id/<swap-id>
```

#### Setting up ZFS
Create the `zpool.cache` file, this resides within the live env:
```
touch /etc/zfs/zpool.cache
```
Create the pool:
```
zpool create -o encryption=on -o keyformat=passphrase -o keylocation=prompt cachefile=/etc/zfs/zpool.cache -m none -R /mnt zroot /dev/disk/by-id/<id>
modprobe zfs
```
Now we can create the ZFS filesystems in the new pool:
```
SYS_ROOT=zroot
DATA_ROOT=zroot/data
zfs set compression=on zroot
zfs set relatime=on zroot
zfs create -o mountpoint=none -p ${SYS_ROOT}/ROOT
zfs create -o mountpoint=/ ${SYS_ROOT}/ROOT/default
zfs create -o mountpoint=legacy ${SYS_ROOT}/home
zfs create -o canmount=off -o mountpoint=/var -o xattr=sa ${SYS_ROOT}/var
zfs create -o canmount=off -o mountpoint=/var/lib ${SYS_ROOT}/var/lib
zfs create -o canmount=off -o mountponit=/var/lib/systemd ${SYS_ROOT}/var/lib/systemd
zfs create -o canmount=off -o mountpoint=/usr ${SYS_ROOT}/usr
SYS_DATASETS='var/lib/systemd/coredump var/log var/log/journal var/lib/docker var/lib/machines var/lib/libvirt var/cache usr/local'
for ds in ${SYS_DATASETS}; do zfs create -o mountpoint=legacy ${SYS_ROOT}/${ds}; done
zfs create -o mountpoint=legacy -o acltype=posixacl ${SYS_ROOT}/var/log/journal
USER_DATASETS='oskr_grme oskr_grme/local oskr_grme/config oskr_grme/cache'
for ds in ${USER_DATASETS}; do zfs create -o mountpoint=legacy ${SYS_ROOT}/home/${ds}; done
zfs create -o mountpoint=/home/oskr_grme/.local/share -o canmount=off ${SYS_ROOT}/home/oskr_grme/local/share
zfs create -o mountpoint=none ${DATA_ROOT}
DATA_DATASETS='Books Development Pictures Documents Work'
for ds in ${DATA_DATASETS}; do zfs create -o mountpoint=legacy ${DATA_ROOT}/${ds}; done
zfs allow oskr_grme create,mount,mountpoint,snapshot ${SYS_ROOT}/home/oskr_grme
zfs umount -a
```
Export/import dance:
```
zpool export zroot
zpool import -d /dev/disk/by-id -R /mnt -l zroot
cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache
```
If this cache does not exist, create one:
```
zpool set cachefile=/etc/zfs/zpool.cache zroot
```
Now, get '/dev/sda1` and grab the `UUID` for below mount command:
```
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
mkdir /mnt/home
mount -t zfs ${SYS_ROOT}/home /mnt/home
mkdir /mnt/usr/local
mount -t zfs ${SYS_ROOT}/usr/local /mnt/usr/local
...repeat for all datasets
```
Install base system and generate fstab entries:
```
genfstab -U -p /mnt >> /mnt/etc/fstab
pacstrap -i /mnt base base-devel vim
```

#### Chroot in, wrap up install
First, chroot on in so we can interact with our new system:
```
arch-chroot /mnt /bin/bash
```
First, setup locale by uncommenting UTF-8 US option in `/etc/locale.gen` then run
```
echo en_US.UTF-8 >> /etc/locale.gen
locale-gen
```
Now lets take care of several other initial setup tasks:
```
echo LANG=en_US.UTF-8 > /etc/locale.conf
ln -s /usr/share/zoneinfo/America/New_York /etc/localtime
hwclock --systohc --utc
pacman -S ntp
ntpd -q
hwclock -w
```
Package collector...
```
pacman -S rsync iw dialog wpa_supplicant dhcp bash-completion wifi-menu reflector
```

#### Finishing Touches
Edit `/etc/pacman.conf` and place `[archzfs]` above other repos and uncomment multilib as well:
```
[archzfs]
Server = http://archzfs.com/$repo/x86_64
```
Both the db and packages are signed, so add key to pacman's trusted key list:
```
pacman-key -r F75D9D76
pacman-key --lsign-key F75D9D76
pacman -Syyu
```
Install and enable ZFS:
```
pacman -S zfs-linux parted
systemctl enable zfs.target
systemctl enable zfs-import-cache
systemctl enable zfs-mount
systemctl enable zfs-import.target
```
Edit `/etc/mkinitcpio.conf`, edit line with `HOOKS=` to match:
```
HOOKS="base udev autodetect modconf block keyboard zfs usr filesystems shutdown"
```
Create boot image:
```
mkinitcpio -p linux
```

#### Home Stretch
For bootloader, systemd-boot will be used:
```
bootctl --path=/boot install
```
Now, the bootloader entry, `vim /boot/loader/entries/arch.conf` with the following:
```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options zfs=zroot/ROOT/default rw
```

Set `/etc/hostname` and `/etc/hosts`:
```
echo archez > /etc/hostname
```
and then enter the following into `/etc/hosts`:
```
127.0.0.1   localhost
::1         localhost
127.0.1.1   archez.reapnet      archez
```

Set root password, make networking simple upon next boot:
```
passwd
pacman -S networkmanager broadcom-wl
systemctl enable NetworkManager
```
Exit chroot, copy zpool, unmount and export!
```
exit
umount /mnt/boot
zfs umount -a
zpool export zroot
```
#### Reboot, some minor fixes to ensure future boots go ok:
```
zpool set cachefile=/etc/zfs/zpool.cache zroot
systemctl enable zfs.target
systemctl enable zfs-import-cache
systemctl enable zfs-mount
systemctl enable zfs-import.target
```
Generate hostid:
```
zgenhostid $(hostid)
```
Regenerate initramfs now that system will properly remember its host ID:
```
mkinitcpio -p linux
```

#### Now its customize to taste (including display server/wm etc)
Lets create out personal account:
```
useradd -m -g users -G audio,video,network,wheel,storage,rfkill,docker -s /bin/bash my_username
passwd my_username
```
Since we want our user to have admin privileges via sudo, we need to run the below command:
```
EDITOR=vim visudo
```
and uncomment the line with:
```
%wheel ALL=(ALL) ALL
```
Logout and log backin to test our new account, run `exit` and log backin with user creds
Test privileges, run `sudo pacman -Syu`, if it worked after entering password we're good here!
- - -
#### Gettin Graphical
Lets install our display server:
```
sudo pacman -S xorg-server xorg-apps xorg-xinit xterm x86-video-intel
```
I'll be using lightdm as my login manager:
```
sudo pacman -S lightdm
sudo pacman -S lightdm-gtk-greeter lightdm-gtk-greeter-settings
sudo systemctl enable lightdm.service
```

Finally, installing my window manager of choice as my desktop environment.. i3.
Also tacking on several other apps of choice:
```
sudo pacman -S i3 lxappearance nitrogen py3status terminator rsync copyq volumeicon git

# Install aur helper
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```
