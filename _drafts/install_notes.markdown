# Summary
## Needed features
* [X] Desktop environment, preferably GNOME+XMonad
* [X] Wifi, preferably using NetworkManager because of user friendlyness.
* [ ] CUPS
* [X] Sound
* [X] Web browsing
* [X] The usual code tools
* [ ] USB3.0, should work out of the box (How to test ?)
* [X] Main filesystems should be optimized for SSDs
* [X] Powersaving should be optimal-ish
* [ ] Optimus, at least disabling the Nvidia card for powersaving
* [X] Webcam
* [ ] Full (not incremental) backup solution for `/home`
* [X] VirtualBox
* [X] Music player (Clementine)
* [X] LibreOffice
* [ ] Latex

## Medium priority
* Hard drive protection: Modern hard drives should be able to resist a shock

## Features we don't give a fuck about
* Fingerprint reader

# Partition scheme
We will keep the disks running using MBR.
The main limitation in my use case is that we cannot use more than 4 partitions per disk, but even that could be worked around by using logical partitions.

Lessons learnt from previous install:

1. 25G for `/` is not enough.
2. No need for swap.
3. 100G for windows seems ok.
4. Don't put pacman cache in tmpfs, but instead empty it manually.
5. Having a few extra partitions for testing is nice.

Apparently being block aligned is important.
Just align it on 1024kb.
Apparently GParted will do this for us.

##Disk 1 (SSD, 160G):

1. Windows partition, 80G, NTFS
2. `/boot`, 200M, ext4
3. `/` on the rest, ext4

##Disk 2 (HDD, 500GO):

*For now this is only theoretical, as I don't have the 500GB disk yet.
I am currently using an 80G drive as my home device, with only one ext4 partition.
I will update this section with the migration informations when I do it.*

The second disk has 2 available partitions for general purpose use, and allow some flexibility.
It can be used to install an alternative OS for testing, or to provide more disk space to windows to install a few games.
`/home` is formatted in ext4, because I don't see any need for something else.

1. Part A, 80G
2. Part B, 80G
4. `/home`, the rest, ext4

# Installation notes
_Always do a full backup before. Don't forget that your SSH keys are not visible in your file explorer_

## Bootloader
I decided to keep Syslinux, because it worked pretty well, but I had a few issues during the install;
my drive was not recognized by my BIOS as bootable.
After a few reinstall to try various GPT/MBR settings, I noticed that the boot mode was to prefer UEFI boot.
After changing it to prefer BIOS boot, it worked perfectly.
I wonder how it worked before.

## User and sudo
This was trivial, simply use the wiki.

## Optimizing for SSDs
For now I did not do much, simply replaced the `relatime` option with `noatime` on my SSD filesystems to reduce write wear.
I also added the `discard` option to handle TRIM properly.
Apparently replacing the IO scheduler could help improve latency but I haven't done it yet.
Once I get my real drive I should put a few directories on it apparently.

## Yaourt and AUR
I chose not to install Yaourt via the archlinuxfr mirror, because I don't like to add a mirror just for a single software.
This choice made it slightly more difficult to install yaourt using the PKGBUILD directly, but is much cleaner.
Yaourt will now be update from source along other software.


# Programming environment
* Vim. Actually this is the first package I installed because doing everything in nano feels so horrible. Also replaced it with `gvim` to have graphical clipboard support.
* Git
* Mercurial
* `base-devel`
* `the_silver_searcher`
* Vagrant + VirtualBox
* Screen

Don't forget that to run VirtualBox, you need to put the following in `/etc/modules-load.d/virtualbox.conf`:

```
# VirtualBox modules
vboxdrv
vboxnetadp
vboxnetflt
vboxpci
```

# Keyboard
For virtual consoles, `/etc/vconsole.conf`:
```
KEYMAP=fr_CH
```

# Trackpoint (TBD)
Put the following in `/etc/X11/xorg.conf.d/20-thinkpad.conf`
```
Section "InputClass"
        Identifier      "Trackpoint Wheel Emulation"
        MatchProduct    "TPPS/2 IBM TrackPoint|DualPoint Stick|Synaptics Inc. Composite TouchPad / TrackPoint|ThinkPad USB Keyboard with TrackPoint|USB Trackpoint pointing device|Composite TouchPad / TrackPoint"
        MatchDevicePath "/dev/input/event*"
        Option          "EmulateWheel"          "true"
        Option          "EmulateWheelButton"    "2"
        Option          "Emulate3Buttons"       "false"
        Option          "XAxisMapping"          "6 7"
        Option          "YAxisMapping"          "4 5"
EndSection
```

# Wifi
Currently I am using NetworkManager, which is pretty well integrated into GNOME.
I have currently no way of configuring it from XMonad, which will need to be fixed soon.
I also still have to install `dnsmasq` to handle connection sharing.

# Powersaving
`powertop` is pretty useful for analysis.
Install it with `pacman -S powertop`.

## Disabling Bluetooth
Put the following in `/etc/modprobe/modprobe.conf` :
```
# disable bluetooth to save power
blacklist btusb
blacklist bluetooth
```

## Kernel parameters
Put the following in `/etc/sysctl.d/powersaving.conf` :

```
# Non maskable watchdog creates a lot of interrupts, disable it
kernel.nmi_watchdog = 0

# Reduces disk access
vm.dirty_writeback_centisecs = 6000

# Seems to be a bit magic but recommended
vm.laptop_mode = 5
```


## Laptop mode
We can install laptop mode tools by doing `yaourt -S laptop-mode-tools`.
We then enable it by doing `systemctl enable laptop-mode`.
Do not forget about blacklisting usb autosuspend on a few devices.
We should really do it on the mouse and keyboard, otherwise it just drives you insane.

To do this, first use `lsusb` to find the device ID.
For example for the WASD keyboard the relevant line is :

```
Bus 001 Device 006: ID 04d9:0169 Holtek Semiconductor, Inc.
```

and the device ID here is `04d9:0169`.
Then put this in `/etc/laptop-mode/conf.d/runtime-pm.conf` :

```
AUTOSUSPEND_USBID_BLACKLIST="04d9:0169"
```

Multiple blacklisted IDs are separated by spaces.
Finaly run `systemctl restart latop-mode`.


## PCI Power management
`/etc/udev/rules.d/pci_powersave.rules` :
```
ACTION=="add", SUBSYSTEM=="pci", ATTR{power/control}="auto"
```


# CUPS
Needs testing @EPFL

# Sound
First we need to configure the sound card driver to use the correct model.
Put the following in `/etc/modprobe/modprobe.conf` :
```
```

The rest will be done using ALSA (included in the kernel) and PulseAudio.
We can simply do it by installing `pulseaudio`.

To read MP3 in clementine I installed almost all gstreamers plugin to finally have it work with the base and ugly plugins. But just install them all.



# Web browsing
`pacman -S chromium firefox irssi`

# Network tools

`yaourt -S transmission ssh mosh nmap`

# Hard drive protection sensor
https://wiki.archlinux.org/index.php/Hard_Drive_Active_Protection_System

Looks a little bit outdate (rc.conf what?)

Apparently the current trend is to protect it via making uber resistant hard drives.

I should ask Joseph if he did anything to protect his drives.

# X11

## Optimus
Optimus support so far is simply disabling the Nvidia card.
I don't use it a lot (it may change if I do some OpenGL).

We simply install `bbswitch` and then put the following in `/etc/modprobe/modprobe.conf` :

```
# disable NVidia card at boot
options bbswitch load_state=0 unload_state=0
```

# Webcam
Worked out of the box.

# Backup solution
Something in the spirit of :
```sh
rsync -aAXv /home/antoine/ /path/to/backup/
```

It also needs to handle different backup locations, because one backup disk will stay at home and the second will stay at EPFL.

Also, don't forget to exclude `~/.cache`, `~/.config` and maybe other (hidden) folders.

Should the script be unit tested ? ;)


# Desktop environment
## Gnome 3
Very easy, just do `pacman -S gnome gnome-extra`.
I decided to use GDM because it is very well integrated and lightweight enough.
It is already in `gnome-extra`, so you only need to run `systemctl enable gdm.service`.

## XMonad
```sh
yaourt -S xmonad xmonad-contrib xmonad-gnome3 dmenu
```
The config files are hosted in Git so they will be easy to retrieve.
Just don't forget to run `xmonad --recompile` before starting the xmonad session.

# Media Transfer Protocol
The media transfer protocol is a USB device class used by my Nexus 7 to access its memory.
It was pretty easy to install : `pacman -S gvfs-mtp libmtp`.

# Non DKMS modules
Those modules are the one that needs to be updated after each major kernel update.
* `bbswitch`
* `vboxdrv`
