# Arch Linux install on ZFS
Living document outlining base installation for home/work PCs, encrypted with `/` on ZFS
- - -
**TO-DO**:
-*refactor into bash script for automation*
-*implement CI/CD to pre-build Arch ISO, serve from S3*
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
    - list additional repo's here`

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
#### Partitiong, Encrypting and Partitioning (again).. Oh My!
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
```
arch-chroot /mnt /bin/bash
```

