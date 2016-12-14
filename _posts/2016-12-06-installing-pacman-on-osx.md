---
layout: post
title: Installing the Pacman package manager on OSX
---


# Steps

*Note:* All those commands have been tested in Bash only.
Using them in other shells might require adjustement.

You will have to install developers tools (XCode) before proceeding with this procedure.

My plan of action is the following:

1. Build a local version of pacman, along with its dependencies.
    We don't install it globally because I want all the system-wide files to be managed by the package manager.
2. Create a global pacman database.
3. Build a global pacman and install it using the tools built in (1).
4. Install a dummy package providing the tools built in OSX.
5. Install my packages and crack a beer open to celebrate.

## Build a local version of libarchive

libarchive is used by pacman to create and extract various archive format.
To build it, use the following commands.

```bash
wget http://www.libarchive.org/downloads/libarchive-3.2.2.tar.gz
tar -xf libarchive-3.2.2.tar.gz
pushd libarchive-3.2.2/
./configure --prefix=`cd ..; pwd`
make -j
make install
popd
```


## Build a local version of pacman
Now we need to create a version of pacman build against our local version of libarchive.
To speed things up a little bit we won't build man pages.
As before, **do not run make install**.

```bash
wget https://sources.archlinux.org/other/pacman/pacman-5.0.1.tar.gz
tar -xf pacman-5.0.1.tar.gz
pushd pacman-5.0.1
./configure CFLAGS="-I `cd ..; pwd`/include" \
            LIBARCHIVE_CFLAGS="-I `cd ..; pwd`/include" \
            LIBARCHIVE_LIBS="-L`cd ..; pwd`/lib -larchive" \
            --disable-doc \
            --prefix=`cd ..; pwd` \
            --localstatedir=/usr/local/pacman/var

make -j
make install
popd
```

## Create Pacman database directory

This step will create a place for pacman to put its package database.

```bash
sudo mkdir -p /usr/local/pacman/var/lib/pacman

# Create the pacman database
sudo ./bin/pacman -Q

# Add the temporary pacman install to our path
export PATH=$PATH:`cd bin; pwd`
```

## Use pacman to install libarchive

In this step we will use pacman to cleanly build and install libarchive.
All pacman packages will be stored in `/usr/local/pacman`

First we need to create a `PKGBUILD` for libarchive.
We cannot use the same as Linux's because we need to adapt a few paths.

```bash
pushd packages/libarchive

# Build the libarchive package
../../bin/makepkg

# Install it
sudo ../../pacman-5.0.1/src/pacman/pacman \
    --config ../../pacman_bootstrap.conf \
    -U libarchive-*-x86_64.pkg.tar.gz
popd
```

## Use pacman to install itself

```bash
pushd packages/pacman
env PACMAN=../../pacman-4.2.1/src/pacman/pacman ../../pacman-4.2.1/scripts/makepkg --config makepkg.conf.osx
sudo ../../pacman-4.2.1/src/pacman/pacman --config ../../pacman_bootstrap_config.conf -U libarchive-3.1.2-8-x86_64.pkg.tar.gz
popd
```
