# Arch Linux install on ZFS
Living document outlining base installation for home/work PCs, encrypted with `/` on ZFS

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
| # Important files and folders:
├── airootfs # Make customizations here
│   ├── etc
│   │   ├── # configuration files...
│   │   └── # ...
│   └── root
│       ├── customize_airootfs.sh
│       └── install.txt
├── build.sh # Used to build ISO
| # Customizable files:
├── packages.both
├── packages.i686
├── packages.x86_64
├── pacman.conf
| # Other files and directories
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
Add ZFS repository to the `pacman.conf` file, above all other repo's add `[archzfs]`:
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
#### Partitioning, Encrypting and Partitioning (again)..
Assuming all went well in creating our custom ISO, boot into the image and get networking up (if no ethernet, `wifi-menu` it up), set system time, then lets partition that disk:
```
timedatectl set-ntp true
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
LUK'in those crypts:
```
cryptsetup luksFormat /dev/sda2
cryptsetup luksOpen /dev/sda2 cryptroot
```
With our luks encrypted partition open, lets repartition for our system on ZFS,
If on laptop and hibernation is wanted (in this case, yes); then we'll make two
partitions... first, is 8GB to match my laptops RAM, then rest of disk for our system:
```
parted /dev/mapper/cryptroot
(parted) mklabel gpt
(parted) mkpart ext2 0% 8GB
(parted) mkpart ext2 8GB 100%
(parted) quit
```
Finish setting up first partition as swap:
```
mkswap /dev/mapper/cryptroot1
swapon /dev/mapper/cryptroot1
```

#### Setting up ZFS
Create the `zpool.cache` file, this resides within the live env:
```
touch /etc/zfs/zpool.cache
```
Create the pool:
```
zpool create -o cachefile=/etc/zfs/zpool.cache -m none -R /mnt zroot /dev/mapper/cryptroot2
```
Now we can create the ZFS filesystems in the new pool:
```
zfs create -o mountpoint=none -o compression=lz4 zroot/ROOT
zfs create -o mountpoint=/ zroot/ROOT/default
zfs create -o mountpoint=/opt zroot/opt
zfs create -o mountpoint=/var zroot/var
zfs create -o mountpoint=/home zroot/home
zfs create -o mountpoint=/root zroot/home/root
zpool set bootfs=zroot zroot
```
Export/import dance:
```
zpool export zroot
zpool import -R /mnt zroot
```

Now, run `blkid /dev/sda1` and grab the `UUID` for below mount command:
```
mkdir /mnt/boot
mount /dev/disk/by-uuid/${UUID} /mnt/boot
```
Install base system and generate fstab entries:
```
pacstrap -i /mnt base base-devel vim
genfstab -U -p /mnt | grep boot >> /mnt/etc/fstab
genfstab -U -p /mnt | grep swap >> /mnt/etc/fstab
```

#### Chroot in, wrap up install
First, chroot on in so we can interact with our new system:
```
arch-chroot /mnt /bin/bash
```
First, setup locale by uncommenting UTF-8 US option in `/etc/locale.gen` then run
```
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
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
Install some packages to assist in finishing installation later:
```
pacman -S rsync iw dialog wpa_supplicant dhcp bash-completion reflector
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
#### Bootup hook for ZFS:
Create `/etc/initcpio/install/load_part`:
```
#!/bin/bash

build() {
        add_binary 'partprobe'

        add_runscript
}

help() {
        cat <<HELPEOF
Probes mapped LUKS container for partitions.
HELPEOF
}
```
Create `/etc/initcpio/hooks/load_part`:
```
run_hook() {
        partprobe /dev/mapper/cryptroot
}
```
Edit `/etc/mkinitcpio.conf`, edit line with `HOOKS=` to match:
```
HOOKS="base udev autodetect modconf block keyboard encrypt load_part resume zfs filesystems"
```
Create boot image:
```
mkinitcpio -p linux
```

#### Home Stretch
For bootloader, rEFInd will be used:
```
pacman -S refind-efi
refind-install
```
Now, for this next part.. we'll need to gather some info..
1. `blkid /dev/sda2` grab `UUID` for cryptdevice
2. `blkid /dev/mapper/cryptroot1` grab `UUID` for resume

With that info we can now edit `/boot/refind_linux.conf`:
```
# /boot/refind_linux.conf:
#-------------------------
"Boot with defaults" "cryptdevice=/dev/disk/by-uuid/<uuid>:cryptroot zfs=zroot/ROOT/default rw resume=UUID=<swap UUID>"
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
cp /etc/zfs/zpool.cache /mnt/etc/zfs
umount /mnt/boot
zpool export zroot
```

#### Reboot! One small order of business then customize to taste (including display server/wm etc)
We need to generate the hostid and set the cachefile for the zpool:
```
zgenhostid $(hostid)
zpool set cachefile=/etc/zfs/zpool.cache zroot
```

Lets create out personal account:
```
useradd -m -g users -G audio,video,network,wheel,storage,rfkill -s /bin/bash my_username
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
sudo pacman -S i3 lxappearance nitrogen py3status terminology rsync copyq volumeicon \
               git python-pip noto-fonts ttf-font-awesome newsboat

# Install aur helper
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

---
- - -
- - -
# TROUBLESHOOTING
If for whatever reason you find yourself not able to boot... follow these steps to get back into your system from liveusb
```
cryptsetup luksOpen /dev/sda2 cryptroot
partprobe /dev/mapper/cryptroot
swapon /dev/mapper/cryptroot1
zpool import -R /mnt zroot
mount /dev/sda1 /mnt/boot
arch-chroot /mnt /bin/bash
```
Typically, cachefile is missing/corrupt.. after above steps to chroot, run following:
```
zpool set cachefile=/etc/zfs/zpool.cache zroot
```
When done poking around, exit like so:
```
exit
umount /mnt/boot
zpool export zroot
```
Reboot, removing the liveusb/media in the process.. enjoy
