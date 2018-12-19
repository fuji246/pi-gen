# piem

_Tool to generate Raspbian image used for network emulator_

This image simplifies the network emulator setup on Raspberry Pi 3, by just installing the image without extra setup. If you are using old Raspberry Pi which doesn't have wifi module built in, you can either using an USB wifi adapter or using an usb-ethernet adapter and switch to use [ethernet bridge](#switch-to-wired-bridge).

## Installation

You can get the pre-build image [here](https://github.com/fuji246/pi-gen/releases/download/piem-v0.1/image_Piem-lite-v0.1.zip), and download [Etcher](https://etcher.io/) and install the image on the micro sdcard.

Once startup, it already setup wifi in AP mode and bridge between wifi (wlan0) and ethernet (eth0) ports. You can plug in the ethernet port to your local network, you can then ping `piemulator.local` to get the ip address of the emulator, or you can just use the domain name for ssh access, like `ssh pi@piemulator.local`.

And then connect your testing devices via wifi:

```
Wifi SSID: `piem`, password: `piemulator`
```

Then you can emulate the network condition of those test devices, both uplink and downlink.

## Network emulation

There is a emulator script [emulator.py](/stage2/02-net-tweaks/files/emulator.py) built in.

```
$ sudo emulator.py -h
emulator.py [-h] [--dryrun] {config,list,init,uninit,add,remove} ...

positional arguments:
  {config,list,init,uninit,add,remove}

optional arguments:
  -h, --help            show this help message and exit
  --dryrun              Dry run
```

Normally you won't need to config it since it already configured in the pre-build image, and will reconfigure when using the [bridge\_switch.sh](/stage2/02-net-tweaks/files/bridge_switch.sh) below.

```
$ sudo emulator.py config -h
loads config from /etc/piem/config.json
{
    # ingress of the bridge network connected with testing devices need add network impairment
    "ingress": "wlan0",

    # egress of the bridge network connected to private network
    "egress": "eth0",

    # need one ifb device for each emulation rule (identified by ip filter and uplink/downlink direction)
    "numifbs": 64
}

usage: emulator.py config [-h] [--ingress INGRESS] [--egress EGRESS]
                          [--numifbs NUMIFBS]

optional arguments:
  -h, --help            show this help message and exit
  --ingress INGRESS, -i INGRESS
                        ingress network interface
  --egress EGRESS, -e EGRESS
                        egress network interface
  --numifbs NUMIFBS, -n NUMIFBS
                        number of ifb devices
```

You can list the configuration and rules set:

```
$ sudo emulator.py list
loads config from /etc/piem/config.json
{
    "ingress": "wlan0",
    "egress": "eth0",
    "numifbs": 64
}

loads rules from /etc/piem/piem.rules
[
    {
        "qdelay": 100,
        "loss": 3,
        "direction": "downlink",
        "bw": 8000,
        "handle": 2,
        "delay": 10,
        "burst": null,
        "emfilter": "192.168.1.5"
    }
]
```

And normally you won't need init/uninit it either because they will be called automatically when system start/stop, besides using the python script, you can also use systemd:

```
$ sudo systemctl start piem
$ sudo systemctl stop piem
$ sudo systemctl restart piem
```

**stop the service or uninit will remove all the rules added**.

You can add an network emulation rule:

```
$ sudo emulator.py add -h
usage: emulator.py add [-h] [--bw BW] [--loss LOSS] [--burst BURST]
                       [--delay DELAY] [--qdelay QDELAY] --filter FILTER
                       --direction {uplink,downlink}

optional arguments:
  -h, --help            show this help message and exit
  --bw BW, -b BW        rate limit in kbps
  --loss LOSS, -l LOSS  loss ratio in percentage, 5 is 5%
  --burst BURST         burst length in packets
  --delay DELAY, -d DELAY
                        delay in ms
  --qdelay QDELAY, -q QDELAY
                        maxinum queuing delay in ms
  --filter FILTER, -f FILTER
                        src(uplink) or dst(downlink) ip filter
  --direction {uplink,downlink}, -c {uplink,downlink}
```

You can try add delay and use ping on the testing device (192.168.1.5 below) to validate that:

```
$ sudo emulator.py add -f 192.168.1.5 -c downlink -l 3

loads rules from /etc/piem/piem.rules
[]

save rules to /etc/piem/piem.rules
[
    {
        "qdelay": 100,
        "loss": 3,
        "direction": "downlink",
        "bw": 8000,
        "handle": 2,
        "delay": 10,
        "burst": null,
        "emfilter": "192.168.1.5"
    }
]


        ip link set dev ifb0 up
        tc filter add dev eth0 parent ffff: protocol ip prio 1 u32 match ip dst 192.168.1.5 flowid 1:1 action mirred egress redirect dev ifb0
        tc qdisc add dev ifb0 root handle 1: netem loss random 3% delay 10ms
        tc class add dev wlan0 parent 1:1 classid 1:2 htb rate 8000kbit
        tc filter add dev wlan0 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.1.5 flowid 1:2
        tc qdisc add dev wlan0 parent 1:2 bfifo limit 100000
```

To remove the rule, you need at least specify the ip filter, you can specify direction, if not it will try remove rules for both uplink and downlink:

```
$ sudo emulator.py remove -h

usage: emulator.py remove [-h] --filter FILTER [--direction {uplink,downlink}]

optional arguments:
  -h, --help            show this help message and exit
  --filter FILTER, -f FILTER
                        src(uplink) or dst(downlink) ip filter
  --direction {uplink,downlink}, -c {uplink,downlink}
```

## Switch to wired bridge

If you perfer wired connection, you can switch to two ethernet (another one using usb-ethernet port) using the script [bridge\_switch.sh](/stage2/02-net-tweaks/files/bridge_switch.sh) built in.

```
$ bridge_switch.sh -h
Usage: bridge_switch.sh {wlan|eth}
```

After plugin your usb-ethernet port, then `bridge_switch.sh eth`, you should be able to see "eth1" in `ifconfig` output now, and now the bridge is setup between "eth0" and "eth1".

You can use `emulator.py list` to make sure the ingress are set to "eth1" instead of "wlan0".

If you want to connect multiple devices to eth1, you can use network switch(DO NOT USE router, that will break the emulator settings because the ip address changes when using NAT).

## Packet loss burst model

Currently Simple Gilbert Model is used for burst loss, and for more intuitive way to set the parameters, burst length (consecutive loss) and packet loss ratio is used. See "3.2.4 Consecutive losses: GI model with 2 parameters" in the paper ["Definition of a general and intuitive loss model for packet networks and its implementation in the Netem module in the Linux kernel"](http://netgroup.uniroma2.it/TR/TR-loss-netem.pdf) for more details.

## Dependencies

pi-gen runs on Debian based operating systems. Currently it is only supported on
either Debian Stretch or Ubuntu Xenial and is known to have issues building on
earlier releases of these systems.

To install the required dependencies for pi-gen you should run:

```bash
apt-get install quilt parted realpath qemu-user-static debootstrap zerofree pxz zip \
dosfstools bsdtar libcap2-bin grep rsync xz-utils file git curl
```

The file `depends` contains a list of tools needed.  The format of this
package is `<tool>[:<debian-package>]`.


## Config

Upon execution, `build.sh` will source the file `config` in the current
working directory.  This bash shell fragment is intended to set needed
environment variables.

The following environment variables are supported:

 * `IMG_NAME` **required** (Default: unset)

   The name of the image to build with the current stage directories.  Setting
   `IMG_NAME=Raspbian` is logical for an unmodified RPi-Distro/pi-gen build,
   but you should use something else for a customized version.  Export files
   in stages may add suffixes to `IMG_NAME`.

 * `APT_PROXY` (Default: unset)

   If you require the use of an apt proxy, set it here.  This proxy setting
   will not be included in the image, making it safe to use an `apt-cacher` or
   similar package for development.

   If you have Docker installed, you can set up a local apt caching proxy to
   like speed up subsequent builds like this:

       docker-compose up -d
       echo 'APT_PROXY=http://172.17.0.1:3142' >> config

 * `BASE_DIR`  (Default: location of `build.sh`)

   **CAUTION**: Currently, changing this value will probably break build.sh

   Top-level directory for `pi-gen`.  Contains stage directories, build
   scripts, and by default both work and deployment directories.

 * `WORK_DIR`  (Default: `"$BASE_DIR/work"`)

   Directory in which `pi-gen` builds the target system.  This value can be
   changed if you have a suitably large, fast storage location for stages to
   be built and cached.  Note, `WORK_DIR` stores a complete copy of the target
   system for each build stage, amounting to tens of gigabytes in the case of
   Raspbian.
   
   **CAUTION**: If your working directory is on an NTFS partition you probably won't be able to build. Make sure this is a proper Linux filesystem.

 * `DEPLOY_DIR`  (Default: `"$BASE_DIR/deploy"`)

   Output directory for target system images and NOOBS bundles.

 * `USE_QEMU` (Default: `"0"`)

   Setting to '1' enables the QEMU mode - creating an image that can be mounted via QEMU for an emulated
   environment. These images include "-qemu" in the image file name.


A simple example for building Raspbian:

```bash
IMG_NAME='Raspbian'
```


## How the build process works

The following process is followed to build images:

 * Loop through all of the stage directories in alphanumeric order

 * Move on to the next directory if this stage directory contains a file called
   "SKIP"

 * Run the script ```prerun.sh``` which is generally just used to copy the build
   directory between stages.

 * In each stage directory loop through each subdirectory and then run each of the
   install scripts it contains, again in alphanumeric order. These need to be named
   with a two digit padded number at the beginning.
   There are a number of different files and directories which can be used to
   control different parts of the build process:

     - **00-run.sh** - A unix shell script. Needs to be made executable for it to run.

     - **00-run-chroot.sh** - A unix shell script which will be run in the chroot
       of the image build directory. Needs to be made executable for it to run.

     - **00-debconf** - Contents of this file are passed to debconf-set-selections
       to configure things like locale, etc.

     - **00-packages** - A list of packages to install. Can have more than one, space
       separated, per line.

     - **00-packages-nr** - As 00-packages, except these will be installed using
       the ```--no-install-recommends -y``` parameters to apt-get.

     - **00-patches** - A directory containing patch files to be applied, using quilt.
       If a file named 'EDIT' is present in the directory, the build process will
       be interrupted with a bash session, allowing an opportunity to create/revise
       the patches.

  * If the stage directory contains files called "EXPORT_NOOBS" or "EXPORT_IMAGE" then
    add this stage to a list of images to generate

  * Generate the images for any stages that have specified them

It is recommended to examine build.sh for finer details.


## Docker Build

```bash
vi config         # Edit your config file. See above.
./build-docker.sh
```

If everything goes well, your finished image will be in the `deploy/` folder.
You can then remove the build container with `docker rm -v pigen_work`

If something breaks along the line, you can edit the corresponding scripts, and
continue:

```bash
CONTINUE=1 ./build-docker.sh
```

After successful build, the build container is by default removed. This may be undesired when making incremental changes to a customized build. To prevent the build script from remove the container add

```bash
PRESERVE_CONTAINER=1 ./build-docker.sh
```

There is a possibility that even when running from a docker container, the
installation of `qemu-user-static` will silently fail when building the image
because `binfmt-support` _must be enabled on the underlying kernel_. An easy
fix is to ensure `binfmt-support` is installed on the host machine before
starting the `./build-docker.sh` script (or using your own docker build
solution).


## Stage Anatomy

### Raspbian Stage Overview

The build of Raspbian is divided up into several stages for logical clarity
and modularity.  This causes some initial complexity, but it simplifies
maintenance and allows for more easy customization.

 - **Stage 0** - bootstrap.  The primary purpose of this stage is to create a
   usable filesystem.  This is accomplished largely through the use of
   `debootstrap`, which creates a minimal filesystem suitable for use as a
   base.tgz on Debian systems.  This stage also configures apt settings and
   installs `raspberrypi-bootloader` which is missed by debootstrap.  The
   minimal core is installed but not configured, and the system will not quite
   boot yet.

 - **Stage 1** - truly minimal system.  This stage makes the system bootable by
   installing system files like `/etc/fstab`, configures the bootloader, makes
   the network operable, and installs packages like raspi-config.  At this
   stage the system should boot to a local console from which you have the
   means to perform basic tasks needed to configure and install the system.
   This is as minimal as a system can possibly get, and its arguably not
   really usable yet in a traditional sense yet.  Still, if you want minimal,
   this is minimal and the rest you could reasonably do yourself as sysadmin.

 - **Stage 2** - lite system.  This stage produces the Raspbian-Lite image.  It
   installs some optimized memory functions, sets timezone and charmap
   defaults, installs fake-hwclock and ntp, wifi and bluetooth support,
   dphys-swapfile, and other basics for managing the hardware.  It also
   creates necessary groups and gives the pi user access to sudo and the
   standard console hardware permission groups.

   There are a few tools that may not make a whole lot of sense here for
   development purposes on a minimal system such as basic Python and Lua
   packages as well as the `build-essential` package.  They are lumped right
   in with more essential packages presently, though they need not be with
   pi-gen.  These are understandable for Raspbian's target audience, but if
   you were looking for something between truly minimal and Raspbian-Lite,
   here's where you start trimming.

 - **Stage 3** - desktop system.  Here's where you get the full desktop system
   with X11 and LXDE, web browsers, git for development, Raspbian custom UI
   enhancements, etc.  This is a base desktop system, with some development
   tools installed.

 - **Stage 4** - Raspbian system meant to fit on a 4GB card.  More development
   tools, an email client, learning tools like Scratch, specialized packages
   like sonic-pi, system documentation, office productivity, etc.  This is the
   stage that installs all of the things that make Raspbian friendly to new
   users.

 - **Stage 5** - The official Raspbian Desktop image. Right now only adds
   Mathematica.

### Stage specification

If you wish to build up to a specified stage (such as building up to stage 2
for a lite system), place an empty file named `SKIP` in each of the `./stage`
directories you wish not to include.

Then add an empty file named `SKIP_IMAGES` to `./stage4` (if building up to stage 2) or
to `./stage2` (if building a minimal system).

```bash
# Example for building a lite system
echo "IMG_NAME='Raspbian'" > config
touch ./stage3/SKIP ./stage4/SKIP ./stage5/SKIP
touch ./stage4/SKIP_IMAGES ./stage5/SKIP_IMAGES
sudo ./build.sh  # or ./build-docker.sh
```

If you wish to build further configurations upon (for example) the lite
system, you can also delete the contents of `./stage3` and `./stage4` and
replace with your own contents in the same format.


## Skipping stages to speed up development

If you're working on a specific stage the recommended development process is as
follows:

 * Add a file called SKIP_IMAGES into the directories containing EXPORT_* files
   (currently stage2, stage4 and stage5)
 * Add SKIP files to the stages you don't want to build. For example, if you're
   basing your image on the lite image you would add these to stages 3, 4 and 5.
 * Run build.sh to build all stages
 * Add SKIP files to the earlier successfully built stages
 * Modify the last stage
 * Rebuild just the last stage using ```sudo CLEAN=1 ./build.sh```
 * Once you're happy with the image you can remove the SKIP_IMAGES files and
   export your image to test

# Troubleshooting

## `binfmt_misc`

Linux is able execute binaries from other architectures, meaning that it should be
possible to make use of `pi-gen` on an x86_64 system, even though it will be running
ARM binaries. This requires support from the [`binfmt_misc`](https://en.wikipedia.org/wiki/Binfmt_misc)
kernel module.

You may see the following error:

```
update-binfmts: warning: Couldn't load the binfmt_misc module.
```

To resolve this, ensure that the following files are available (install them if necessary):

```
/lib/modules/$(uname -r)/kernel/fs/binfmt_misc.ko
/usr/bin/qemu-arm-static
```

You may also need to load the module by hand - run `modprobe binfmt_misc`.
