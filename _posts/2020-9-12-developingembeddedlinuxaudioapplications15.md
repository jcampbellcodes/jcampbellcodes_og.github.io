---
layout: post
title: DEVELOPING AUDIO PROGRAMS FOR EMBEDDED LINUX, PART 1.5
date: 2020-9-12 04:01:00
description: Better building. Expands on the first article by using a different board and a more robust build process.
---

After writing the first article in this mini embedded audio series, I got a fair bit into the "part 2" article
that's going to be about configuring ALSA and device drivers to output I2S audio before I got
quite bogged down by the limits of my knowledge on this stuff. (Turns out it's a deep rabbit hole!)

Yes, I got things "working" on my end by blurting out some I2S bloops from my Beaglebone out a speaker, 
but most of it was by tinkering around and following docs online that I didn't fully understand. And while I 
definitely recommend this learning-by-doing type of exploration, I wanted to get a deeper understanding of "why" my
steps worked before I published anything.

So I read a [book](https://www.packtpub.com/product/mastering-embedded-linux-programming/9781784392536)! (And after
reading, I definitely recommend it. Was great glue for my fractured understanding about a lot of this stuff.) 
And after reading that book, I thought I'd revisit the content of my first article and go through a "better" way
to develop embedded Linux (audio) applications, closer to how things are done in ~The Industry~.

And finally, to cast a wider net, I'm going to take a break from the Beaglebone connected over SSH (if you want Beaglebone specific steps, check out that book above!) and instead use the Raspberry Pi 4 over UART using our very own custom
Linux system and toolchain made using Buildroot. Let's start!

# Materials
<br>
- [Raspberry Pi](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/) (I used the rpi4 model B, but should work with all of them)
- [USB to TTL Serial Cable](https://www.sparkfun.com/products/12977) for talking UART
- Host machine. I used a Mac running latest Ubuntu via VirtualBox.
- SD Card (at least 2GB, more is better) and SD Card Reader

<figure>
  <img class="col center" src="/img/rpi/serial_cable.jpg">
  <figcaption>Serial Cable with Pi</figcaption>
</figure>

<figure>
  <img class="col center" src="/img/rpi/sdcard.jpg">
  <figcaption>If you don't have an SD card reader, check out one of these little buddies!</figcaption>
</figure>

# What will we be building?
<br>

Similar to the last article, we will be cross-compiling the little ALSA toy program from the [last article](2020/07/01/developingembeddedlinuxaudioapplications.html), but this time we will also be building our own cross compile toolchain,
root filesystem, kernel image, and bootloader as well. Last time we used Debian's multiarch tools and relied on the "out of the box" Debian image provided by Beaglebone.org, but this approach
has several advantages over that:
- Much more flexibility in which OS your host is running
- Easier to reason about what components are needed for your board -- it can easily be regenerated fresh
- Easier to specify which versions of components to use (compiler, target libs, kernel version)

Note that this article will just output audio via the headphone jack rather than via I2S pins.

# Get and Configure Buildroot
<br>

All of the components needed to create a custom embedded Linux can be created by hand -- the [cross compiler](https://crosstool-ng.github.io/),
[root filesystem](https://tldp.org/HOWTO/Bootdisk-HOWTO/buildroot.html), [bootloader](https://github.com/u-boot/u-boot),
and [Linux kernel](https://www.cyberciti.biz/tips/compiling-linux-kernel-26.html). In fact, creating all of these
by hand is a good exercise! But in practice, developers tend to use a build system to generate all these components
in an automated, easily reproducible way, and spit out an image that you can easily flash onto your board. Two of 
the most common tools for this are [Buildroot](https://buildroot.org/) and [Yocto](https://www.yoctoproject.org/).
Both are great tools; without going into too much detail, Buildroot is generally less fuss than Yocto to get a Linux
system working quickly and easily, while Yocto is more flexible but has a bit more of a learning curve.

We'll be using Buildroot in this article, and you can clone it like so:

{% highlight bash %}
git clone git://git.buildroot.net/buildroot
cd buildroot
{% endhighlight %}

> Tip: If you have issues with building later, you might want to consider checking out a particular stable release branch and see if that resolves the issue.

Then before anything else, make sure you download all the dependencies as enumerated in the [Buildroot docs](https://buildroot.org/downloads/manual/manual.html#requirement-mandatory). 

Those docs have a lot of good information about what the various parts of the repo are for and how to use Buildroot, definitely
worth at least a skim. Things you'll want to look out for are about the toolchain, various make commands, and when you need to clean the repo.

# Configure for Raspberry Pi 4 64-bit
<br>

Now that you've got the Buildroot repo, you can begin configuring it to build images that are compatible with your board.
You could in theory do this all by hand, but luckily most boards (including both the Rpi and BeagleBone) have "default 
configurations", or `defconfig`s that can be used as a starting point, upon which you can make further customizations.

To see all the available defconfigs that folks have made in the mainline, run:

{% highlight bash %}
make list-defconfigs
{% endhighlight %}

The one we will be using is the config for the Rpi4 in 64-bit mode, `raspberrypi4_64_defconfig`.

> Will mention more later, but note that these configs are generally the least amount of configuration possible
> in order to get a working image on your hardware. Every feature/library you want to add (audio, SSH, UART, python)
> must be done explicitly, described in the next steps.

From here you can activate that default configuration:

{% highlight bash %}
make raspberrypi4_64_defconfig
{% endhighlight %}

A good area to check out when starting a new project is the board-specific readme, found under the `board` subdir of the
repo. For Raspberry Pi, you can find that at `board/raspberrypi4-64/readme.txt`.

## Buildroot menuconfig
<br>

Configuring Buildroot is done with a tools known as `Kconfig` (invoked using `make menuconfig` on the command line). This is a 
useful interface to get familiar with because many open source tools for Linux systems use this same tool for writing
configuration files, such as the Linux kernel itself, U-Boot, and Crosstool-ng.

In order to open the menuconfig interface, run:
{% highlight bash %}
make menuconfig
{% endhighlight %}

> As a side note, you can customize indivudual projects that Buildroot builds separately such as the kernel, U-Boot, Barebox, etc. Run `make help` to see which ones you can choose from.

# Customize Config to Work With ALSA
<br>

The first round of configuration to do is to open `menuconfig` and make sure that the ALSA utils and libraries that we need 
will be included on the target device.

> Some basics first: to enable or disable an option in Kconfig, press `y` or `n`, respectively. To open a search box to look for 
> a particular option, hit `/`.

When you run `make menuconfig`, you will end up on this screen.

<figure>
  <img class="col center" src="/img/rpi/menuconfig/1_target_packages.png">
  <figcaption>Initial Kconfig Screen</figcaption>
</figure>

First we will go look at "Target packages", which is the section where you can decide which 
applications and libraries ("packages") that Buildroot will download and build for installation
on your target board and in some cases in your cross-compilation toolchain.

First let's go look at audio applications.

<figure>
  <img class="col center" src="/img/rpi/menuconfig/2_audio_apps.png">
  <figcaption>Under Target Packages, see Audio Apps</figcaption>
</figure>

Under here we see alsa-utils is already selected -- but if you inspect further...

<figure>
  <img class="col center" src="/img/rpi/menuconfig/3_alsa_utils.png">
  <figcaption>ALSA Utils Looks To Be Selected...</figcaption>
</figure>

Many of the utils are actually unselected by default. We won't need all of these, but 
in development, some of them will be useful (especially `aplay` and `speaker-test`).

<figure>
  <img class="col center" src="/img/rpi/menuconfig/4_needs_wchar_support.png">
  <figcaption>For amixer, we need to enable wchar support in the toolchain!</figcaption>
</figure>

Also, you can see that `amixer` requires another option to be selected! Let's assume we want that
application (could be helpful). First, go back to the main screen by hitting selecting `Exit`
with the left/right keys. `Exit` is more like a "back" button, until you get to the main screen;
at that point, `Exit` will prompt a "Save?" dialog.

<figure>
  <img class="col center" src="/img/rpi/menuconfig/5_toolchain_options.png">
  <figcaption>Go back to the main screen, to Toolchain</figcaption>
</figure>

So go back to the main screen and see the toolchain options.

<figure>
  <img class="col center" src="/img/rpi/menuconfig/6_enable_wchar.png">
  <figcaption>From there you can enable wchar support</figcaption>
</figure>

This is where you can enable WCHAR support in the toolchain with `y`.

<figure>
  <img class="col center" src="/img/rpi/menuconfig/7_alsa_utils.png">
  <figcaption>Go back to alsa-utils, enable these and/or others if desired</figcaption>
</figure>

Now navigate back to where you were and select whichever utils you want.

<figure>
  <img class="col center" src="/img/rpi/menuconfig/8_target_libraries.png">
  <figcaption>Go up a level and find "Libraries"</figcaption>
</figure>

Go up one level -- we just looked at applications, but "Libraries" may be more
important. These are the libraries that will be both in your toolchain and on the 
target device.

<figure>
  <img class="col center" src="/img/rpi/menuconfig/9_audio_libs.png">
  <figcaption>Go under Audio/Sound</figcaption>
</figure>

Go under Audio/Sound...

<figure>
  <img class="col center" src="/img/rpi/menuconfig/10_alsa_lib.png">
  <figcaption>Check out alsa-lib</figcaption>
</figure>

And alsa-lib...

<figure>
  <img class="col center" src="/img/rpi/menuconfig/11_all_there.png">
  <figcaption>Everything is enabled anyway!</figcaption>
</figure>

You'll see that it's all already enabled! This is because we selected a bunch of ALSA applications
that depend on these libraries anyway. If you wanted to forego the ALSA command line applications
and only include the libraries that your ALSA app will need, you could just come here straight from
the beginning.

This little walkthrough was meant to illustrate the workflow for menuconfig/Kconfig. Some options
depend on other options and you may find yourself going around a bit to make everything right.

When you are ready to save, `Exit` to the main page and say "yes" to saving. Now your `.config` file
in the Buildroot directory, which was generated initially by `raspberrypi4_64_defconfig`, includes
your local customizations. Before we start the build, there is one more series of customizations to make
in order to get audio to work on the Pi.

Before getting into those customizations, there is a question we should answer...

# How does an embedded Linux system start up?
<br>

There are [other](https://en.wikipedia.org/wiki/Linux_startup_process), [better](https://buildroot.org/downloads/manual/manual.html#_init_system) resources that explain in general how embedded systems bootstrap themselves. For our purposes, it may
suffice that a special program known as a "bootloader" is compiled for the board and executed, which loads the Linux kernel and device tree. The kernel then initializes itself, mounts our root filesystem, and runs an `init` program. 

This `init` program is the hook where you can add initialization scripts, such as which kernel modules to load and which background daemons to run (either standard ones or applications that you have written). Buildroot offers several different `init` programs, such as BusyBox, SysV, and systemd, which all have different mechanisms for how you add your own custom logic. We will be using the default in Buildroot, which is the Busybox `init`.

# Board Specific Build Scripts
<br>

- explain the inittab and S35audio (using busybox init) for importing modules
- explain changes to config.txt in order to enable audio by default
- explain the ALSA config and how to set the headphones as the default
- explain how to add a program to run once on startup
- check out the docs in the rpi directory


# Initial Build
<br>

- run `make` to build: will build your toolchain, kernel, bootloader, and rootfs
  - possible you may just need to run `make sdk` here
  - how to troubleshoot: rebuild clean, takes a long time but buildroot will fail in weird ways (like things not being on your machine that you expect)
  - I thought "make toolchain" would just make the toolchain -- but turned out my target packages, like alsa, werent present!
  - unfortunately, when in doubt, do a `make clean all && make` and go make lunch, three times

- explain the output dir - what's in there?


# Cross Compile ALSA Toy
<br>

- add `export PATH=$(pwd)/output/host/usr/bin:$PATH` -- this allows your session to find the cross compiler first.
  - in our case, `aarch64-linux- `
- compile alsa-sweep with `aarch64-linux-gcc`


# Deploy Program Using Root Filesystem Overlay
- add a local "rootfs_overlay" dir to the config, pointing to `/usr/local/bin/alsa-sweep` (or something)
- copy the binary to rootfs overlay
- do `make` again in buildroot -- this time will be much quicker, rootfs_overlay changes are one of the few things you can modify incrementally

- add this program to be run once on startup when you turn on the board

# Put the Image On Your Board
<br>

- use "balena etcher" to etch your image onto an SD card
  - it's free and beaglebone recommends it, so its cool
  - you can do this with the command line as well, but I don't want to be responsible for accidentally formatting anyone's hard drive!


# How To Connect To the Board
<br>

- explain how to connect to the board via serial

<figure>
  <a href="https://www.raspberrypi.org/documentation/usage/gpio/"><img class="col center" src="/img/rpi/pinout.png"></a>
  <figcaption>Raspberry Pi Pinout (from official docs)</figcaption>
</figure>

<figure>
  <img class="col center" src="/img/rpi/plugging_in.jpg">
  <figcaption>Hook up ground to ground, cable Tx to Pi Rx, and cable Rx to Pi Tx</figcaption>
</figure>

# Run the bloops!
<br>

Connect your headphones
`./alsa-sweep`


# Acknowledgements

Thanks Archie3D from TheAudioProgrammer Discord for the tips on enabling the bcm2835 audio kernel module with the 
BusyBox init scripts as well as a simple ALSA config file.


