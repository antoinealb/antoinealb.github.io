I recently decided I wanted to reinstall my Arch Linux system.
I have been running Arch on my Lenovo T420s for a few years now, and it has been working flawlessly.
However, I started working a lot with virtual machines, I was slowly running out of disk space.
Since I wanted to document my installation for future reference, and I thought it may be useful to someone, I decided to make a blog post about it.

## Why reinstall ? You could have migrated instead!
I wanted to split my data between a spinning hard drive for my data and virtual machines and a SSD for my host OS.
I could probably have migrated the data to the new drive, changed partition size and changed config just to make it work;
It would have been very complex and error prone.
Remember the first point of the Arch Way:

> Simplicity is absolutely the principal objective behind Arch development.

# Pre-install planning
Before wiping the hard drive, I took a few days to note what I was really using on my existing installation.
I researched those topics, mostly using the ArchWiki, to make sure the install would be painless.
After a few days of research, here is the list of what should work on my system in the end :

* Desktop environment. I am currently using GNOME3 with XMonad.
* Wifi. Wpa supplicant is not a lot of fun: I want NetworkManager to work.
* CUPS. At home I have a printer that I know is not working with CUPS on Linux, but at least at the university it should.
* Sound multiplexing between different applications.
* Web browsing.
* Usual programming environment.
* TRIM support for the SSD.
* Powersaving should be optimal-ish.
* My laptop has two different graphic cards. I should be able to disable the Nvidia one to avoid eating all the power.
* The webcam should work, even if I use it twice a year.
* VirtualBox and Vagrant.
* Music player. I decided to go with Clementine (a fork of Amarok 2.X) since a friend advised it to me.
* LibreOffice
* Latex
* Dual boot with Windows. I am not entierly sure this is really needed. Maybe I will just try working with more virtual machines.

I decided not to implement the use of the fingerprint reader for two reasons.
First it is a bit of a pain to support for Linux, and is prone to breaking.
Then because fingerprint are not a correct way to authenticate yourself on a system.
Since you leave them on almost everything you touch, they do not provide additional security.

Don't forget to do a full backup of your home partition before reinstalling.
I almost forgot to copy my SSH keys, which would have left me locked out of my servers.

# Installation
For the installation part of the process, I highly suggest reading Arch's beginner guide.
It is one of the most complete installation walkthrough I saw and it explains your different options very well.
I won't go into details that are in there, and focus on what is specific to my system.

## Partionning
After booting the installer USB key, the first step before installing is the partionning of the system.
I decided to stay on an MBR partition scheme, even if my system supports EFI and GPT, because I did not need more than 4 partitions per disk.

From my previous installation of Arch Linux, I learn a few things about partition sizing:

* 25G for `/` is not enough, at least for my usage.
* Swap partition is useless.
    Do a swapfile if you really need suspend or swap ram.
* 80G for a Windows partition seems enough.
* I used to put the pacman cache in a tmpfs that would get wiped at each boot.
    This is a bad idea;
    most problems are only seen after a reboot, when you cannot downgrade your packages.
* Having an empty partition or two that you can use to temporarily install a Linux/BSD to is useful.
    This problem can be mitigated by using virtual machines, but for sustained use they are less comfortable.

For SSDs it is recommended aligning your partitions on 1024kb blocks, but I think partition tools will do it for you anyways.

Finally I went with the following partition scheme for my main drive, which is a 160G Intel SSD:

1. Windows partition, 80G, NTFS
2. `/boot`, 200M, ext4
3. `/` on the leftover space (about 70G), ext4

My second disk is only made of `/home` right now, but I plan to put other testing partitions on it when I get a bigger drive.

## Bootloader installation
I decided to keep Syslinux as on my previous install.
I like it because it is quite powerful and way easier to setup than GRUB2.
However, it only supports MBR and BIOS booting;
if you use EFI you'll have to use another bootloader.

The installation was quite difficult, and it took me quite a while to understand what was wrong:
The BIOS would just loop on the boot device selection screen.
I tried redoing the installation and the formatting a few times, thinking it had something to do with the MBR.
This was by far the most time consuming part of the reinstall.
After a few tries, I realized my BIOS was set to try EFI boot first.
After changing it to BIOS emulation, it worked perfectly.
This is a bit weird, as it worked flawlessly and the settings haven't changed since.
I did not investigate it further and went on with my install.

## Basic setup: User, vim and sudo
After my first succesful boot, the first thing I did was installing a non priviledged user.
Doing your work as root is dangerous: even if you are not the target of hackers, any mistake can be a disaster.
I installed `sudo` and setup it to accept my user as temporary superuser, which was super easy thanks to the wiki.
The most important trick here is to never edit the config files directly but instead to use `visudo`.
Not doing it can result in you being locked out of your computer.

I also directly installed Vim, as I wanted to be able to edit my config files in my usual editor.
Since I store my vimrc and other configs on Github I just had to clone them to the appropriate location.


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
