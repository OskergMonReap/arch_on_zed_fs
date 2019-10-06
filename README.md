# Arch Linux install on ZFS
Living document outlining base installation for home/work PCs, encrypted with `/` on ZFS
- - -
**TO-DO**:
- *refactor into bash script for automation*
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
# other repositories...
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
Assuming all went well in creating our custom ISO, boot into the image and get networking up (if no ethernet, `wifi-menu` it up), then lets partition that disk:
```
parted /dev/sda
(parted) mklabel gpt
(parted) mkpart ESP fat32 1MiB 513MiB
(parted) set 1 boot on
(parted) mkpart primary ext2 513MiB 99%
(parted) align-check optimal 1
(parted) align-check optimal 2
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
With our luks encrypted partition open, lets repartition for our system on ZFS:
```
parted /dev/mapper/cryptroot
(parted) mklabel gpt
(parted) mkpart ext2 0% 512MiB
(parted) mkpart ext2 512MiB 100%
```
If we're working on a laptop, lets make swap (size of RAM at least) for hibernation purposes:
```
mkswap /dev/mapper/cryptroot1
swapon /dev/mapper/cryptroot1
```

#### Setting up ZFS
Create the `zpool.cache` file:
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
PACKAGE FREE FOR ALL!!!!! Meaning, just install w/e else you want here:
```
pacman -S rsync terminator iw dialog wpa_supplicant i3 volumeicon copyq py3status ....
```

#### Finishing Touches
Edit `/etc/pacman.conf` and place `[archzfs]` above other repos:
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
# /etc/initcpio/install/load_part:
#---------------------------------
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
# /etc/initcpio/hooks/load_part:
#------------------------------
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

Set `/etc/hostname` and `/etc/hosts`

Set root password:
```
passwd
```
Copy zpool, unmount and export!
```
cp /etc/zfs/zpool.cache /mnt/etc/zfs
umount /mnt/boot
zpool export zroot
```
