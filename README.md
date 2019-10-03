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


