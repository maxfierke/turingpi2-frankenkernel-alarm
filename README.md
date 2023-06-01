# UNDERCONSTRUCTION, PROBABLY NOT WORKING YET

# Arch Linux ARM for Turing Pi 2 SoCs

Maintainer: Max Fierke

README based on from: Cole Smith & Yatao Li's ["Arch Linux ARM for ClockworkPi A06"](https://github.com/yatli/arch-linux-arm-clockworkpi-a06/tree/main)

License: LGPL-2.1

# Introduction

This document will walk you through installing [Arch Linux ARM](https://archlinuxarm.org/) on a few of the supported SoCs for the Turing Pi 2 cluster board.

We will create a root file system for supporting ~~three~~ two different SoCs:
  * ~~**soquartz** (rk3566)~~
    * Just use Manjaro, it's easier: https://github.com/manjaro-arm/soquartz-cm4-images/releases
  * **NVIDIA Jetson TX2 NX** (tegra186)
  * **NVIDIA Jetson Nano** (tegra210)

I may expand the supported SoCs if/when I acquire them. (e.g. I suspect I'll get a Turing RK1 at some point)

## Caution
If you are running advanced filesystems on your host (for example `zfs`), don’t try to prepare the image on that host. 
`mkinitcpio` in chroot doesn’t like that, and may cause damage to your host filesystem. Please use a virtual machine, or use a prebuilt image instead.

## Quickstart

Pre-built root filesystem(s) are provided in the **Releases** tab. Skip to the
**Prepare to Flash** if using a pre-built image for eMMC on **NVIDIA Jetson**.

**NOTE:** Please note the following defaults for the release filesystem:

* Root password: `root`
* Timezone: `US/Central`
* Locale: `en_US UTF8`
* Hostname: `turingpi2-node0x`

# Setup

This guide **assumes you are already using Arch Linux**. Some package names or procedures may differ depending on your
distribution.

In order to build the root filesystem, we will set up the following:

1. `aarch64` chroot environment + necessary configuration
2. `arm-none-eabi-gcc` build tools from ARM
3. Linux kernel
4. U-Boot bootloader
5. Additional packages for running on Turing Pi 2

## Setting up Chroot Environment

We will start by creating an ARM chroot environment to build our root filesystem using
[this guide](https://nerdstuff.org/posts/2020/2020-003_simplest_way_to_create_an_arm_chroot/).

1. Install the required packages

```
$ yay -S base-devel qemu-user-static qemu-user-binfmt arch-install-scripts
# systemctl restart systemd-binfmt.service
```

2. Verify that an `aarch64` executable exists in

```
$ ls /proc/sys/fs/binfmt_misc
```

3. Download the base root FS to use. We will use the `aarch64` tarball from Arch Linux ARM

```
$ wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
```

4. Create the mount point and extract the files as root (not via sudo)

```
$ sudo su
# mkdir root 

# bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C root
# mount --bind root root
```

**NOTE**: It's very important that this `root` folder is owned by **root**. Otherwise, you will
get `unsafe path transition`
errors in the final build.

5. Chroot into the newly created environment

```
# arch-chroot root
```

6. Inside the chroot, populate and init the pacman keyring

```
# pacman-key --init
# pacman-key --populate archlinuxarm
```

7. Finally, update the packages

```
# pacman -Syu
```

## Configuring The Root Filesystem

We will start with lightly configuring our system before compiling the packages.

For this section, **all commands will be run inside the chroot**.

1. Install some useful tools

```
# pacman -S base-devel git vim wget ranger sudo man networkmanager
```

2. Enable `networkmanager` and `dhcpcd` for networking on first boot

```
# systemctl enable NetworkManager dhcpcd
```

3. Set the Locale by editing `/etc/locale.gen` and uncommenting your required locales.
4. Run

```
# locale-gen
```

5. Reference partitions in `/etc/fstab`

```
# echo 'LABEL=ROOT_ARCH    /    f2fs    defaults,noatime    0    0' >> /etc/fstab
# echo 'LABEL=BOOT_ARCH    /boot    ext4    defaults    0    0' >> /etc/fstab
```

6. Set the time using `timedatectl`. To list supported timezones: `timedatectl list-timezones`

```
# timedatectl set-timezone "US/Central"
# timedatectl set-ntp true
```

**NOTE**: This may affect the timezone of your system outside the chroot, if so, you may reset
your host system timezone after finishing the root tarball. 


7. Set the system clock

```
# hwclock --systohc
```

8. Set the hostname to whatever you like

```
# echo 'turingpi2-node0x' > /etc/hostname
```

9. Assign the root password

```
# passwd
```

### Switch to the `alarm` user

1. We can avoid working with the root account by granting `alarm`, the default Arch Linux ARM user, `sudo` privileges.

```
# EDITOR=/usr/bin/vim visudo
```

2. And add the corresponding line for `alarm` after the one for `root`

```
alarm ALL=(ALL) ALL
```

3. Switch to the `alarm` user

```
# su alarm

$ cd
```

**NOTE**: The default password for the **alarm** user is **alarm**

## Acquiring GCC Build Tools

U-Boot depends on the `arm-none-eabi-gcc` executable to be built, and since this program is not available in the Arch
Linux ARM repositories, we will download it directly from
[ARM's website](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads)
.

1. Download the binaries

```
$ wget https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-aarch64-arm-none-eabi.tar.xz
```

2. Extract the binaries to another directory

```
$ mkdir gcc 
$ tar -xJf gcc-arm-10.3-2021.07-aarch64-arm-none-eabi.tar.xz -C gcc
```

3. Add the toolchain to your `PATH`

```
$ cd gcc/gcc-arm-10.3-2021.07-aarch64-arm-none-eabi/bin
$ export PATH=$PATH:$(pwd)
$ cd
```

## Compiling The Packages

This repository contains pre-configured and patched Arch Linux packages for
the relevant SoCs. The Linux kernel and U-Boots are based off the multi-platform
AArch64 kernel from Manjaro ARM, which is based on the kernel already available
in Arch Linux ARM, with patches provided from various sources for relevant SoCs
and additional configuration for Tegra devices.

### Download This Repository

1. Inside the `alarm` home folder of your `aarch64` chroot environment, clone this repository

```
$ git clone https://github.com/maxfierke/turingpi2-frankenkernel-alarm.git
$ cd turingpi2-frankenkernel-alarm
```

### Compiling The Linux Kernel

1. Build the package. **This can take a long time!!** Especially since we are emulating an `aarch64`
   architecture. The package build tool `makepkg`, supports a flag called `MAKEFLAGS`. Below, we will append
   `MAKEFLAGS="-j$(nproc)"` to the `makepkg` command to instruct the compiler to use one worker for each core.

```
$ cd linux-turingpi2
$ MAKEFLAGS="-j$(nproc)" makepkg -si 
$ cd ..
```

### Compiling U-Boot

1. Build each u-boot package, similar to above.

```
$ cd uboot-jetson-nano # or uboot-jetson-tx2-nx
$ MAKEFLAGS="-j$(nproc)" makepkg -si 
$ cd ..
```

### Compiling Additional Packages

For each additional package directory in this repository

```
$ cd <package directory> 
$ MAKEFLAGS="-j$(nproc)" makepkg -si 
$ cd ..
```

### Clean up the build dependencies
```
# pacman -Rs base-devel git vim wget ranger xmlto docbook-xsl inetutils bc dtc
# rm /var/cache/pacman/pkg/*.pkg.tar.xz*
# rm -rf /home/alarm/gcc* /home/alarm/.cache/ /home/alarm/.bash_history
```

### Exit the chroot

1. Exit `alarm`

```
$ exit
```

2. Exit `root`

```
# exit
```

### Unmount The Root Filesystem

```
 # umount root
```

# Tar The Root Filesystem

We are now ready to package up the root filesystem into a compressed tarball.
Optionally, we can save the built packages.

```
 # cd root
 # mv home/alarm/turingpi2-frankenkernel-alarm ../
 # tar cpJf ../turingpi2-frankenkernel-alarm-rootfs.tar.xz .
 # cd ..
```

Change ownership of the tarball and exit the `root` account

```
 # chown <user>:<user> turingpi2-frankenkernel-alarm-rootfs.tar.xz
 # exit
```

**You now have a root filesystem tarball to bootstrap the SD card!**

## Prepare to Flash

TBD

## Next Steps

Check out the [post-install suggestions](https://wiki.archlinux.org/title/General_recommendations) from Arch Linux for
further configuration.

## Troubleshooting

If you run into issues where you cannot SSH or it does not appear to be booting, please check the debugging output
via UART:

1. Connect a USB-to-serial cable on the UART header for the specific node on Turing Pi 2 board
  - Node 1 requires UART through the GPIO pins, see the Turing Pi 2 docs for more details
3. Connect the other end to your Linux system, you should now see a new device: `/dev/ttyUSB0`
4. Monitor the connection with `sudo stty -F /dev/ttyUSB0 115200 && sudo cat /dev/ttyUSB0`
  * Or use `picocom`, `sudo picocom -b 115200 /dev/ttyUSB0`
6. Power on your Turing Pi 2 and/or the specific node and monitor for errors

# Acknowledgements

I stole this README format from Cole Smith and Yatao Li's guide on running
Arch Linux ARM on the ClockworkPi A06.

The Linux and PKGBUILD is based on the Manjaro ARM kernel PKGBUILD from the
Manjaro ARM team (Dan Johansen, Furkan Salman Kardame, and others)
