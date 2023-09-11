# A06 Linux patches for the uConsole

This repo allows you to get your uConsole up and running with Arch Linux ARM.
It mainly provides a patched Linux kernel with support for the screen of the
uConsole. The screen is different from that of the DevTerm, so the DevTerm
panel patches can't be used with the uConsole. They are still included here and
can be chosen with some adjustments of the PKGBUILD file of the Linux package.

If you experience problems with the build process, let me know. Unfortunately
I'm not able to fully validate the instructions at the moment. Errors should be
something of the type like mismatching paths or checksums, so if these occur,
you might be able to fix these easily. Just wanted to let you know.

Based on Yatli's https://github.com/yatli/arch-linux-arm-clockworkpi-a06.git
which in turn is based on Armbian's https://github.com/armbian/build.git

Using patches from ClockworkPi's https://github.com/clockworkpi/uConsole/tree/master/Code/patch/a06/20230630

## Compatibility

### What works

- screen
	- brightness as well
- HDMI out
- audio
	- speakers
	- headphone
	- it tries to work even more by supplying another audio device that
	  doesn't do anything though
- WiFi and Bluetooth
	- the supplied antenna limits its performance
	- needs extra firmware
- battery stats
	- charge level
	- charge/discharge estimation
	- state (charging, discharging)
	- voltage/current information
- battery life is pretty much the same if not better
- suspend
- shutdown
- power button handling

### What's untested

- mobile module

### What doesn't work right

- screen may need correction for rotation.
- audio has some slight clicks as if you would play a vinyl. Not sure whether
  this is on the ClockworkPi supplied images as well. It also sends loud pops
  via the Headphone output when powering up or down, be especially prepared if
  you're going to plug it into a PA.
- the screen panel code throws some errors in the `dmesg` but doesn't affect the
  observed operation of the panel.

## Compilation

### Prerequisites

Setup an (aarch64) Arch Linux ARM chroot environment on a preferably beefy
(x86_64) Arch Linux system. You may also set up a cloud server with native
ARM hardware and build the packages there.

The packages needed on the host system are:

```
pacman -S arch-install-scripts base-devel qemu-user-static qemu-user-static-binfmt
```

`arch-install-scripts` is needed to `arch-chroot` in to the Arch Linux ARM
chroot environment.

`base-devel` obviously provides some more packages regarding development.

The rest of the packages allow to run binaries of different architectures to
run on the host system.

After these are installed you'll need to make some adjustments to the `binfmt`
configuration.

Edit `/usr/lib/binfmt.d/qemu-aarch64-static.conf` and replace the uppercase
letters at the end from i.e. `:FP` to `:FPOC`. Don't ask me how that magic
works, refer to https://en.wikipedia.org/wiki/Binfmt_misc for more information
regarding the binfmt configuration. It is needed to get rid of `sudo` errors
complaining about the `suid` flag when it's issued inside the chroot session.
`sudo` has a `suid` flag that checks whether its file permissions are only
modifiable by root and if that isn't the case, `sudo` errors out. It apparently
also errors out if we don't have the binfmt flags `FPOC` set on the host.

Though that might be an Artix Linux issue on my system, so if you find that you
don't need these adjustments, just continue living as if nothing happened.

#### Caution for zfs users

Citing Yatli:

If you are running advanced filesystems on your host (for example `zfs`), don’t
try to prepare the image on that host. `mkinitcpio` in chroot doesn’t like that,
and may cause damage to your host filesystem. Please use a virtual machine, or
use a prebuilt image instead \[I can't provide one yet\].

### Arch Linux ARM chroot environment setup

To compile the packages using `makepkg` an aarch64 environment is needed. The
chroot environment just used to build the packages. The real end image will be
created later.

#### On the host

Download the Arch Linux ARM rootfs from http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz

Unpack it as root:

```
mkdir root
bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C ./root
```

Do some `mount` black magic:

```
mount --bind root root
```

Chroot into it:

```
arch-chroot root
```

#### In the chroot session as root

Initialise and update it:

```
pacman-key --init && pacman-key --populate && pacman -Syu
```

`gpg-agent` may complain that the version it is currently running may be older
than what got updated. Use

```
gpgconf --kill all
```

to let it restart with its updated version.

Install a text editor of your choice for editing configuration files if not
already present:

```
pacman -S vim
```

Make appropiate changes to `/etc/locale.gen` and `/etc/locale.conf` and
generate locales:

```
locale-gen
```

Give the chroot environment a hostname (you may replace the name `aarch64-build`
with whatever you like, doesn't really matter):

```
printf '%s\n' aarch64-build > /etc/hostname
```

Install the packages needed for the package compilation:

```
pacman -S base-devel git vim wget curl sudo
```

Configure `sudo` to allow the default `alarm` user to run sudo.

Edit `/etc/sudoers` via (replace the texteditor if you like):

```
EDITOR=vim visudo
```

Uncomment the line `%wheel ALL=(ALL:ALL) ALL` to allow `wheel` users to use
`sudo` and save the file.

Add the user `alarm` to the `wheel` group:

```
usermod -a -G wheel alarm
```

Change to the user `alarm`:

```
su alarm
```

#### In the chroot session as the `alarm` user

Test whether `sudo` is working:

```
sudo printf 'a\n'
```

You should be asked for the password of the `alarm` user which is `alarm` by
default.

There then should be `a` printed on the screen.

Switch to the home directory of the `alarm` user, without any arguments:

```
cd
```

Clone this repo and `cd` into it:

```
git clone https://github.com/autianic/clockworkpi-linux.git
cd clockworkpi-linux
```

`cd` into each of the directories and build them using:

```
MAKEFLAGS="-j$(nproc)" makepkg -si
```

The options `--noextract --noprepare --skipchecksums` may additionally be useful
for troubleshooting or to continue an aborted buld session. These instruct
`makepkg` to not recreate the extracted, patched and compiled kernel source when
rebuilding the package.

If you want to compile the U-Boot images from source:

```
rkbin-aarch64-hack
uboot-clockworkpi-a06
```

Omit the `-i` `makepkg` option from the `uboot-clockworkpi-a06` package and
disallow the build process from installing the U-Boot images on your system (if
it asks you).

The `uboot-clockworkpi-a06` package is a bit of a moving target because one of
its sources are pulled from their master branch. The U-Boot images will be later
used to write them into an area of the first 32K sectors. You may get these from
the releases section instead:

- `idbloader.img`
- `uboot.img`
- `trust.img`

Place them next to the directory of the chroot. If you built the U-Boot package,
copy them from there:

```
# executing from the host:
cp ./root/home/alarm/clockworkpi-linux/uboot-clockworkpi-a06/pkg/uboot-clockworkpi-a06/boot/idbloader.img ./
cp ./root/home/alarm/clockworkpi-linux/uboot-clockworkpi-a06/pkg/uboot-clockworkpi-a06/boot/uboot.img ./
cp ./root/home/alarm/clockworkpi-linux/uboot-clockworkpi-a06/pkg/uboot-clockworkpi-a06/boot/trust.img ./
```

The most interesting packages are:

```
audio-clockworkpi-a06
networking-clockworkpi-a06
linux-clockworkpi-a06
```

I didn't compile the rest of the packages so it's up to you if you want the
other packages.

Prefer to compile the Linux package last. If you started building the Linux
kernel, continue with the following steps in the mean time.

It is probably worth to temporarily set a power management option to prevent the
system from sleeping. The building process takes a long time considering that we
have an chroot session of a different architecture running. On my laptop with an
Intel i7 2630QM (Sandy Bridge (2nd Gen), 4 Cores, 8 Threads) it took a day to
complete.

Periodically check the progress, there may be a prompt to ask what to do with
new compiling options that aren't handled in our `config` file. You can accept
the default answer by just hitting Enter, though you might like to have a closer
look and compile some drivers as kernel modules (answer `m`). Eventually the
terminal cursor will stop at the start of a new line and it will then start to
build the kernel.

## On the host: preparing the Arch Linux ARM image

For this we are going to create an image that we then flash onto the SD card if
everything goes well.

Open a new terminal session on your host, change to the directory where you
downloaded the U-Boot images and create an empty image file using
`dd`:

```
dd if=/dev/zero of=./mmc.img bs=1M count=16K status=progress
```

Although the `count` is `16K` it multiplies with the `bs` (block size) and
results in 16GiB.

You may use a different size based on what the system will take up after you did
configure the system. Just account for the time it takes to transfer the image
on to the target medium.

Create a GPT partition table with MBR compatibility using gdisk:

```
gdisk ./mmc.img
```

In the prompt, use `o` to initialise a partition table. Use `n` to create a new
partition. This is going to hold the U-Boot configuration along with the Linux
kernel and its initramfs. Set its starting offset to sector `32768`. The U-Boot
binaries will later be written into the preceding sectors. Size the partiton
`+1GiB` (include the `+` sign. Otherwise it is seen as an absolute end offset).
When asked for a partition type, set it to Linux filesystem which would be
gdisk's `8300` or GUID `0FC63DAF-8483-4772-8E79-3D69D8477DE4`.

Create another partition that is going to hold the actual OS. From that point
you can pretty much add any other partition you want.

Make sure the first partition is going to be the U-Boot partition that is going
to hold the Linux kernel. Don't create a BIOS partition as if you were going to
prepare a BIOS OS install with UEFI upgradeability potential (just catching
habits here). It would cause U-Boot to run into the BIOS partition otherwise.

Write the changes with `w` and gdisk will drop you to the shell afterwards if
all went well.

As a root user, bind the `./mmc.img` file as a loop device so that we have
better access to the partitions inside that image:

```
udisksctl loop-setup -f ./mmc.img
```

You should see a new `/dev/loopn` entry where `n` is a number. There also should
be further devices resembling individual paritions, i.e. `/dev/loop0p1`. If you
want to remove the loop, issue:

```
udisksctl loop-delete -b /dev/loopn
```

Format the first partition as ext4 and the second one to whatever you like as
long as the Linux kernel can read from there, i.e. `f2fs`, `ext4` or `btrfs`.

When formatting `ext4`, omit the Journaling to reduce overall writes. The
`mkfs.ext4` option for that would be `-O ^has_journal`.

Example:

```
mkfs.ext4 -L boot -O ^has_journal /dev/loop0p1
mkfs.btrfs -m dup -d single -L OS /dev/loop0p2
```

Mount both partitions. You may use mounting options i.e. to enable filesystem
based compression. For `btrfs`, `compress-force=zstd:1` may be of interest.

```
mkdir boot
mkdir OS
mount /dev/loop0p1 ./boot
mount /dev/loop0p2 ./OS
```

Extract the Arch Linux ARM rootfs into the second mounted partition using the
same tar archive that was used for the build chroot environment.

```
bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C ./OS
```

Move `/boot/*` of the second partition into the first partition.

```
mv ./OS/boot/* ./boot/
```

Unmount the first partiton and mount it into the `/boot` directory of the
second partition, i.e.

```
umount ./boot
rmdir ./boot
mount /dev/loop0p1 ./OS/boot
```

Create a subdirectory called `extlinux` in the mounted boot partition and create
a file named `extlinux.conf` with the following content:

```
LABEL Arch ARM
KERNEL /Image
FDT /dtbs/rockchip/rk3399-clockworkpi-a06.dtb
APPEND initrd=/initramfs-linux.img root=UUID=aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa rw rootwait audit=0
```

Replace the UUID with the one matching the OS partition. The partitions may be
listed with `lsblk`. You may need to run it as root to get full information:

```
lsblk -o NAME,UUID,LABEL,MOUNTPOINTS
```

Copy the `/boot/Image` and `/boot/initramfs-linux.img` to the same directory as
the `./mmc.img` on the host so that these can be later used for a QEMU VM, i.e.:

```
cp ./OS/boot/Image ./
cp ./OS/boot/initramfs-linux.img ./
```

Change the permissions on the copied files so that you can access these with
your user. Replace `user` and `group` with your regular user name and group.

```
chown user:group ./Image ./initramfs-linux.img
```

Append `fstab` entries as root so that UUIDs can be resolved like in the `lsblk`
call:

```
genfstab -t UUID ./OS >> ./OS/etc/fstab
```

You may now like to chroot into `./OS` and proceed with the initialisation of
the Arch installation as you like. If you are not planning to use QEMU to do a
graphical installation or expect to be able to finish the installation on the
uConsole itself, copy the built packages into the OS partition and install these
as shown further down in the next sections. When you're done, continue here.

Unmount the first and second partition and remove the loop device:

```
umount ./OS/boot
umount ./OS
udisksctl loop-delete -b /dev/loopn
```

`dd` the U-Boot binaries into the `./mmc.img`:

```
dd if=./idbloader.img of=./mmc.img bs=512 seek=64 conv=notrunc,fsync
dd if=./uboot.img of=./mmc.img bs=512 seek=16384 conv=notrunc,fsync
dd if=./trust.img of=./mmc.img bs=512 seek=24576 conv=notrunc,fsync
```

Exit root if you haven't already and proceed with the next steps. Keep Linux
compiling in the background.

### Run Arch Linux ARM in a VM

The VM can be used to configure the system with graphical output. This also
ensures that problems unrelated to the uConsole are detected earlier on before
the image is flashed to the SD card. If you already made a base installation
and also happen to have installed the built packages including the custom built
Linux kernel, you may skip this section.

Create a subdirectory called `shared`, which may be used to conveniently
transfer files in and out of the VM.

Create a `start.sh` shell script next for the VM:

```
#/bin/sh

# can't get QEMU to work in such a way that it would boot directly off the
# drive containing U-Boot, so I've extracted the Linux kernel image and the
# initramfs from the /boot partition and appended the kernel options from the
# /boot/extlinux/extlinux.conf file and use these direcly in the QEMU
# parameters.

qemu-system-aarch64 \
	-display sdl,gl=on \
	-machine virt \
	-cpu cortex-a72 \
	-smp $(nproc) \
	-m size=8G,maxmem=8G \
	-serial stdio \
	-k us \
	-usb \
	-net nic \
	-net user \
	-device qemu-xhci,id=xhci \
	-device virtio-gpu-gl-pci \
	-device usb-tablet \
	-device usb-kbd \
	-device usb-audio \
	-kernel ./Image \
	-initrd ./initramfs-linux.img \
	-append 'root=UUID=aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa rw rootwait audit=0' \
	-drive file=./mmc.img,format=raw,media=disk,index=0,readonly=off \
	-virtfs local,path=./shared,mount_tag=vfs0,security_model=mapped-xattr,id=vfs0 \

```

Replace the UUID of the OS partition in the value of the  `-append` argument.
The `-append` argument serves as a kernel parameter set.

The QEMU parameters `-kernel` and `-initrd` also refer to the Linux-Image and
initramfs that were extracted earlier, so change its paths if it differs.

Set execution permissions for the script and start the VM and do the usual Arch
Linux installing process. Refer to the Arch Linux ARM and Arch Linux wiki pages.

The startup will take a while, give it a bit of time and it will start logging
into the terminal. Later also the graphical output will be initialised. When the
startup process has settled, you can start working on the OS.

If you want to automatically mount the share directory across reboots, use the
`fstab` entry:

```
vfs0	/mnt/share	9p	defaults,noauto,user,trans=virtio,version=9p2000.L,msize=8388608,nofail 0 2
```

### Package installation

Copy over the previously generated packages and replace the Linux kernel on the
target system via (adjust the paths as needed):

```
pacman -U linux-clockworkpi-a06-x.x.x-1-aarch64.pkg.tar.xz
pacman -U linux-clockworkpi-a06-headers-x.x.x-1-aarch64.pkg.tar.xz
```

If there are conflicting files, those should be the dts files that reside in the
`/boot/dtbs` directory. Delete the conflicting files and retry the installation.

Watch the pacman install logs and adjust the mkinitcpio configuration to load
the modules needed for the display panel and more.

Also install the other packages as needed to get WiFi, bluetooth and audio
working.

Once everything is done, shut the VM down and make sure outstanding writes to
the `mmc.img` are done and nothing is writing to it.

### Flashing the prepared image onto the SD card

You may now `dd` the image to the SD card. Due to buffering the first bunch of
Gigabytes will go through very fast and `dd` will seemingly hang at the end of
the transfer. If your card reader has an activity light, you should see it
indicating accesses. Wait for its completion and the card should then idle and
the command shell should drop you back to the prompt.

After that, you probably want to expand the OS filesystem. Use gdisk or some
other tool to update the partition table to the new size, otherwise partitoning
tools may still think that the SD card is of a small size.

Using `gdisk`, perform `r` to get into recovery and transformation options, `d`
to use the main GPT header to create the backup GPT header at the end of of the
space and write the changes using `w`, accept with `y` and continue. The
partitions are not changed.

Using `gdisk`, use `w` to let gdisk detect the new medium size and place the
second GPT header to a new location, accept the correction with `y` and accept
with `y` again to write the changes to the SD card. The partitions are not
yet changed.

Afterwards the OS partition can be resized using gparted or partitionmanager.

When done, eject the SD card using `eject /dev/disk/by-id/...` (you may also use
the classic `/dev/sdx` path pattern instead of `/dev/disk/...`), insert it into
the uConsole and power it up.

It should take around 15 seconds until it displays something. You may also plug
an USB device that lights up when there is power to check if the USB ports are
initialised (some devices only light up if they are detected by the OS, use a
device which also lights up when it just receives power). If there is absolutely
nothing except for the glowing power button, then something went wrong up until
the load of the kernel.

If you are using a btrfs filesystem and the free space indicator is still the
same after the resize, executing `btrfs filesystem resize max /` from the booted
target system should fix that.

Exit all sessions created so far.

That's it.

## See also

If you want to further tweak your system, you may like to have a look at the
following pages:

- KDE Plasma 5
	- https://wiki.archlinux.org/title/SDDM#KDE_/_KWin
		- SDDM itself runs in X11 by default even if you are using a
		  Wayland session. You can configure SDDM so that it runs in
		  Wayland too using kwin. Should shave off a bit of memory
		  usage and get rid of dangling X11 processes.
- PipeWire
	- https://gitlab.freedesktop.org/pipewire/pipewire/-/wikis/Config-PipeWire#setting-buffer-size-quantum
		- If you get some stuttering due to buffer underruns, you may
		  like to increase the buffer sizes. Using `default.clock.min-quantum = 1024`
		  to set a minimum,
		  `default.clock.quantum = 4096` to set generic defaults and
		  `default.clock.max-quantum = 8192` to set a maximum pretty
		  much got rid of that problem, feel free to experiment.
- VLC
	- The default number of threads for the `dav1d` decoder might be too
	  low, resulting in as low as two threads. Raise it manually to 6.
	- Use the OpenGL ES output and the EGL backend for better performance.

# Credits

Yatli and all the other people that this work is based on. I wouldn't have known
where to pull all this stuff to get Arch Linux working. I just rearranged things
to get the most recent kernel version.

## Yatli's acks

Very special thanks to **Max Fierke (@maxfierke)** from the CPI Discord, and the
Manjaro team for their help in debugging and kernel patching. The Linux kernel
and u-boot ports in this repository uses their carefully designed patches, and
modified PKGBUILDs. This Arch Linux port would not be possible without their
hard work, and I make no claims or credit to it.

[Manjaro DevTerm A06 Linux Kernel](https://gitlab.manjaro.org/manjaro-arm/packages/core/linux-clockworkpi-a06)

[Manjaro DevTerm A06 U-Boot](https://gitlab.manjaro.org/manjaro-arm/packages/core/uboot-clockworkpi-a06)

