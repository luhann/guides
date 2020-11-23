# Installation guide for Void Linux with LUKS-encrypted btrfs root

## Introduction
In this guide you will find:
- btrfs with Zstandard compression
- LUKS-encrypted root and swapfile
- GRUB with UEFI

You will **not** find:
- Instructions for file systems other than btrfs
- Full disk encryption (there's an official guide [here](https://docs.voidlinux.org/installation/guides/fde.html))
- Explanation for all choices I've made (sometimes I don't know the true reason behind my choices)

## Index
1. [Setting up the live ISO](#setting-up-the-live-iso)
    1. [Logging in](#logging-in)
    2. [Configuring the keyboard layout](#configuring-the-keyboard-layout)
    3. [Connecting to the internet](#connecting-to-the-internet)
2. [Formatting disks](#formatting-disks)
    1. [Partitioning](#partitioning)
    2. [Creating the file systems](#creating-the-file-systems)
        1. [EFI partition](#efi-partition)
        2. [Boot partition](#boot-partition)
        3. [LUKS-encrypted root partition](#luks-encrypted-root-partition)
    3. [Mounting partitions](#mounting-partitions)
        1. [Root partition](#root-partition)
        2. [EFI and boot partitions](#efi-and-boot-partitions)
3. [Installing the system](#installing-the-system)
    1. [Base installation](#base-installation)
    2. [Running `chroot`](#running-chroot)
    3. [Basic configuration](#basic-configuration)
        1. [Hostname](#hostname)
        2. [System configuration information](#system-configuration-information)
        3. [Configuring locales](#configuring-locales)
        4. [Root password](#root-password)
        5. [Configuring `fstab`](#configuring-fstab)
        6. [Setting up Dracut](#setting-up-dracut)
    4. [Finishing installation](#finishing-installation)
        1. [Intel microcode](#intel-microcode)
        2. [GRUB](#grub)
        3. [Swapfile](#swapfile)
        4. [Regenerating configurations](#regenerating-configurations)
4. [Post-installation](#post-installation)
    1. [Creating the main user](#creating-the-main-user)
    2. [Session management](#session-management)

## Setting up the live ISO
### Logging in
There are two available users, `root` (superuser) and `anon`. The password of both is `voidlinux`. I like to log in using the superuser so I don't have to type `sudo` at all. I highly suggest you run `exec bash` so you don't have to deal with `dash`'s limitations.

### Configuring the keyboard layout
If you need a different layout other than `en-US`, you can do the following:
```console
# loadkeys $(ls /usr/share/kbd/keymaps/i386/**/*.map.gz | grep <your-layout>)
```

### Connecting to the internet
```console
# cp /etc/wpa_supplicant/wpa_supplicant.conf /etc/wpa_supplicant/wpa_supplicant-<wlan-interface>.conf
# wpa_passphrase <ssid> <passphrase> >> /etc/wpa_supplicant/wpa_supplicant-<wlan-interface>.conf
# sv restart dhcpcd
# ip link set up <interface>
```

## Formatting disks
### Partitioning
The minimum number of partitions is three:
- The EFI partition (`/efi`)
- The boot partition, where kernels are stored (`/boot`)
- The LUKS-encrypted btrfs root partition

So, first we need to generate the partition tables. Check which device is the one you want to install Void into. For this guide I'll simply use `/dev/sda`, but it can change depending on your setup, so watch out! Back to the partition tables:
```console
# fdisk /dev/sda
```

After running `fdisk`, it will prompt you with a menu, so follow these steps:
1. Select `g` to generate a GTP table
2. Select `n` to create the EFI partition with size of +200M
3. After creating the partition, change its type by selecting `t` and then selecting the option that represents EFI Partition (generally 1)
4. Select `n` to create the boot partition with size of +500M (more space means more kernels, I like using +800M)
5. Select `n` to create the btrfs partition with the remaining size

### Creating the file systems
#### EFI partition
```console
# mkfs.vfat -nBOOT -F32 /dev/sda1
```

#### Boot partition
```console
# mkfs.ext2 -L grub /dev/sda2
```

#### LUKS-encrypted root partition
```console
# cryptsetup luksFormat --type=luks -s=512 /dev/sda3
# cryptsetup open /dev/sda3 cryptroot
# mkfs.btrfs -L void /dev/mapper/cryptroot
```

### Mounting partitions
#### Root partition
First, let's mount the main btrfs partition:
```console
# BTRFS_OPTS="rw,noatime,ssd,compress=zstd,space_cache,commit=120"
# mount -o $BTRFS_OPTS /dev/mapper/cryptroot /mnt
# btrfs subvolume create /mnt/@
# btrfs subvolume create /mnt/@home
# btrfs subvolume create /mnt/@snapshots
# umount /mnt
```

Then, let's mount the top-level partitions:
##### `/`
```console
# mount -o $BTRFS_OPTS,subvol=@ /dev/mapper/cryptroot /mnt
```

##### `/home`
```console
# mkdir -p /mnt/home
# mount -o $BTRFS_OPTS,subvol=@home /dev/mapper/cryptroot /mnt/home
```

##### `/.snapshots`
```console
# mkdir -p /mnt/.snapshots
# mount -o $BTRFS_OPTS,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
```

> **NOTE:** Configure mount options according to your needs.

After that, let's mount some nested partitions, which won't have a snapshot taken, since snapshots don't work resursively:
```console
# mkdir -p /mnt/var/cache
# btrfs subvolume create /mnt/var/cache/xbps
# btrfs subvolume create /mnt/var/tmp
# btrfs subvolume create /mnt/srv
```
You also need to create a nested subvolume for the swapfile:
```console
# btrfs subvolume create /mnt/var/swap
```

#### EFI and boot partitions
Once the root partition is mounted, it is time to mount the remaining ones:
##### `/efi`
```console
# mkdir /mnt/efi
# mount -o rw,noatime /dev/sda1 /mnt/efi
```

##### `/boot`
```console
# mkdir /mnt/boot
# mount -o rw,noatime /dev/sda2 /mnt/boot
```

## Installing the system
### Base installation
Set the appropriate variables (this may vary depending on your needs):
```console
# REPO=https://alpha.us.repo.voidlinux.org/current
# ARCH=x86_64
```

If using `musl`, the values might be something like:
```console
# REPO=https://alpha.us.repo.voidlinux.org/current/musl
# ARCH=x86_64-musl
```

> **NOTE:** [Here is a handful of mirrors](https://docs.voidlinux.org/xbps/repositories/mirrors/index.html).

Then run:
```console
XBPS_ARCH=$ARCH xbps-install -S -R "$REPO" -r /mnt base-system btrfs-progs cryptsetup
```

The command above installs the base system along with btrfs utilites, GRUB and dm-crypt utility, which are core parts of this setup.

### Running `chroot`
Mount the pseudo file systems needed for a `chroot`:
```console
# for dir in dev proc sys run; do mount --rbind /$dir /mnt/$dir; mount --make-rslave /mnt/$dir; done
```

Copy the DNS configuration into the new root so that XBPS can still download new packages inside the `chroot`:
```console
# cp /etc/resolv.conf /mnt/etc/
```

Then `chroot` into the new installation:
```console
# BTRFS_OPTS=$BTRFS_OPTS PS1='(chroot) # ' chroot /mnt/ /bin/bash
```

### Basic configuration
#### Hostname
Write the desired hostname to `/etc/hostname`.

#### System configuration information
Refer to [this documentation](https://docs.voidlinux.org/config/rc-files.html#rcconf) in order to configure your `rc.conf` file.

#### Configuring locales
For glibc installations, edit `/etc/default/libc-locales`, then run:
```console
(chroot) # xbps-reconfigure -f glibc-locales
```

#### Root password
```console
(chroot) # passwd
```

#### Configuring `fstab`
```console
(chroot) # UEFI_UUID=$(blkid -s UUID -o value /dev/sda1)
(chroot) # GRUB_UUID=$(blkid -s UUID -o value /dev/sda2)
(chroot) # ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/cryptoroot)
(chroot) #  cat <<EOF > /etc/fstab
UUID=$ROOT_UUID / btrfs $BTRFS_OPTS,subvol=@ 0 1
UUID=$UEFI_UUID /efi vfat defaults,noatime 0 2
UUID=$GRUB_UUID /boot ext2 defaults,noatime 0 2
UUID=$ROOT_UUID /home btrfs $BTRFS_OPTS,subvol=@home 0 2
UUID=$ROOT_UUID /.snapshots btrfs $BTRFS_OPTS,subvol=@snapshots 0 2
tmpfs /tmp tmpfs defaults,nosuid,nodev 0 0
EOF
```

#### Setting up Dracut
I advise doing a "hostonly" install, that is, Dracut will generate a lean initramfs with everything you might need, including `i915` drivers if you have an Intel CPU with integrated graphics:
```console
(chroot) # echo hostonly=yes >> /etc/dracut.conf
```

### Finishing installation
#### Intel microcode
```console
(chroot) # xbps-install -Su void-repo-nonfree intel-ucode
```

#### GRUB
```console
(chroot) # xbps-install grub-x86_64-efi
(chroot) # grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id="Void Linux"
```

#### Swapfile
In order to have an encrypted swap, let's use a more modern approach by using a swapfile as our swap partition. For this example, I'll create a swapfile of 16 GiB, but you can choose the best size for your installation:
```console
(chroot) # btrfs subvolume create /var/swap
(chroot) # truncate -s 0 /var/swap/swapfile
(chroot) # chattr +C /var/swap/swapfile
(chroot) # btrfs property set /var/swap/swapfile compression none
(chroot) # chmod 600 /var/swap/swapfile
(chroot) # dd if=/dev/zero of=/var/swap/swapfile bs=1G count=16 status=progress
(chroot) # mkswap /var/swap/swapfile
(chroot) # swapon /var/swap/swapfile
```

After that, follow [this Arch's guide](https://wiki.archlinux.org/index.php/Power_management/Suspend_and_hibernate#Hibernation_into_swap_file_on_Btrfs) on calculating the `resume_offset` kernel parameter for btrfs.

> **HINT:** You can use XBPS to compile the `btrfs_map_physical` for you by using [my own template](https://github.com/gbrlsnchs/void-packages/tree/btrfs_map_physical). Just clone the branch and use `xbps-src` as usual to `pkg` the `btrfs_map_physical` package.

After calculating it, append the following line to GRUB's config:
```console
(chroot) # RESUME_OFFSET=<calculated-offset-from-tutorial-above>
(chroot) # cat <<EOF >> /etc/default/grub
GRUB_CMDLINE_LINUX="resume=UUID=$ROOT_UUID resume_offset=$RESUME_OFFSET"
EOF
```

> **NOTE:** You need Linux 5.0+ in order to use a swapfile with btrfs.

#### Regenerating configurations
```console
(chroot) # xbps-reconfigure -fa
(chroot) # exit
# shutdown -r now
```

### Post-installation
#### Creating the main user
Log in as root and then run:
```console
# xbps-install -S zsh
# useradd -m -G wheel,input,video -s /bin/zsh <username>
# passwd <username>
# visudo
```

After running `visudo`, uncomment the line that contains `%wheel`. Log out and then log in with the newly created user.

> **NOTE:** If you want to lock down the root account, you can run `sudo passwd -dl root`. Be careful though, since you won't be able to log in using the root account anymore.

#### Session management
Please refer to [this official guide](https://docs.voidlinux.org/config/session-management.html) from the handbook.
