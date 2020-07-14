---
layout: post
title: DEVELOPING AUDIO PROGRAMS FOR EMBEDDED LINUX, PART 2
date: 2020-7-1 04:01:00
description: The second in a two part series about cross-compiling an ALSA program for an ARM board and hooking up an external I2S DAC.
---

In the previous post, we covered how to write a dead-simple ALSA driver and cross compile it for an ARM embedded device.
What comes next is the deep end -- we need to figure out how to configure I2S audio to come out on pins from the Beaglebone,
hook those pins up to our external DAC and audio amplifier, deploy our cross compiled ALSA program to our Beaglebone, and
run the thing!

This article, even moreso than the last, is going to have a lot of tangents. Communication protocols (I2S, I2C, SPI, UART), 
Linux device trees, and ALSA configuration are all deep topics on their own. In many instances I'm going to link to
more in-depth articles, so hopefully this post can act as a jumping board for you to get more deep on this stuff. Let's start!

# Set up the Beaglebone Black

Before anything, let's quickly make sure your Beaglebone Black is functional.

The [official docs](https://beagleboard.org/getting-started) on how to set up the Beaglebone should get you to 
a place where you can run the code in this article. Make sure you can `ssh` into your board after following those
instructions by invoking this command on your host, to which the Beaglebone is USB connected:
```
ssh debian@beaglebone.local # default password is temppwd
```

Also, as stated in the Beaglebone docs, always make sure you power down the board before unplugging it! Otherwise
you may end up needing to start the bring up process all over.

## What's in a circuit?

Before deep diving into anything, here is a high level overview of our circuit and data flow:
- First, load our ALSA program (`booper`) onto the Beaglebone
- This program connects to the default audio device, which is `plughw:Black,0` (you can find the available audio devices using `aplay -L`)
- We read audio output from this device using the McASP enabled pins, which output the audio as an I2S data stream (this data is also being sent to the HDMI output, but we don't use that)
- We connect these pins and power to the corresponding inputs of the I2S Digitial-to-Analog Converter (DAC) breakout board
- The breakout board also includes an amplifier to bring the analog audio output to a reasonable amplitude before running it into the speaker

Or put another way, we are going to ask our Beaglebone to direct the audio from our application to a particular audio device, which we will configure to output the
audio as I2S, so that it can talk to our breakout board that speakers I2S. Before getting into any of that configuration, what is I2S?

## Communications Protocols are Key

One thing I've found from learning embedded systems is that communication protocols are a central topic for figuring out how to build things.
After all, an embedded system is really just a hodgepodge of isolated (usually specialized) components talking to one another -- so how
do they do it? They just send binary electrical signals to one another and agree on what they are supposed to mean!

Different[communication protocols](https://learn.sparkfun.com/tutorials/serial-communication) have different tradeoffs -- 
some require a bunch of wires and clock signals and might be more reliable or faster than others but require a ton of pin real estate, 
while some might only require a single wire but are slower. Understanding and considering these tradeoffs is a key skill, so I would 
recommend getting familiar with all the major communication protocols. The big ones are [UART](https://learn.sparkfun.com/tutorials/serial-communication),
[I2C](https://learn.sparkfun.com/tutorials/i2c/all), [SPI](https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi/all), and the 
one focused on in this article, [I2S](https://hackaday.com/2019/04/18/all-you-need-to-know-about-i2s/). I2S has persevered for a long time
as an audio protocol because it is simple and its tradeoffs are specially designed to work with high bandwidth (44kHz) stereo audio signals.


- [linux dai](https://www.kernel.org/doc/html/v4.10/sound/soc/dai.html)

## What is I2S?

- I chose I2S (Inter-IC Sound) protocol over USB or SPI mainly because it is yet another technology I have been meaning to learn and it leaves open 
a USB port to be used by another input device (planning to use it for USB MIDI in a future project). That 
said, it turns out I2S output on the Beaglebone Black is actually a little wonky. Unless you use an
external clock, you are limited to the 48kHz family of sample rates ([see here](https://hifiduino.wordpress.com/2014/03/10/beaglebone-black-for-audio/)). 
This was fine for my purposes, but it is an unfortunate drawback. The I2S pins are mainly used for the HDMI output, which 
[this article](https://www.raspberry-pi-geek.com/Archive/2013/02/HDMI-and-the-BeagleBone-Black-Multimedia-Environment) sheds some
helpful light upon.

- describe the signals and what they do


## How do I get I2S audio out of my Beaglebone?
- in this case, we are more concerned with understanding Beaglebone's processor, the AM335x
- don't be afraid of technical specs, pinouts, etc. They are dry and generally not beginner friendly,
- but most of the time they will be your only life raft. So adapt to them!
- A good way to learn about a board's quirks is to mess with various serial protocols on 
- the boards. Also! don't be afraid of computer specs. Similar to math equations, it's not 
- immediately natural to convert these to "english" in your mind. But it's important to fight the
- urge to skip over part numbers when reading, because that helps you make connections. Ie the Beaglebone
- uses the TI AM335x microcontroller -- you will find a lot of docs about this chip that aren't beaglebone specific
- and you need to be able to grok these without relying on the article talking specifically about the beaglebone

- so we know we want i2s out -- how do we get there? this is where you start to do some archaeology...
- knowing the terms to search is how you bootstrap this
- first identify how you get the i2s pins out
- knowing the Beaglebone processor, you could google "AM335x audio interface" and learn about the TI McASP
- personally I dug around the Beaglebone device tree files (described next) and checked out the HDMI one -- anything that didn't seem related to the LCD pins I assumed was audio!
- From there, I checked out this pinout <image here> and saw that these pins could be muxed as mcasp pins

- what is mcasp? Multi Channel Audio Serial Port
- Checkout [beaglebone pinout](https://microcontrollerslab.com/beaglebone-black-pinout-pin-configuration-features-applications/#MCASP_Pins)
- or the [reference manual](https://github.com/beagleboard/beaglebone-black/wiki/System-Reference-Manual)

The pins on the beaglebone can be used for multiple things -- we have to manually configurethese pins to be in "i2s mode". 
This is called "multiplexing", or "muxing".
- We need a way to tell the beaglebone we want those pins to be i2s (multiplex them)
- enable and configure the McASP device as our i2s output
- set up the mcasp device as a sound card that we can access via ALSA

- talk about "multiplexing". These pins can be configured as GPIOs, or output SPI, or output I2S audio when configured as McASP pins
  There are almost 69 I/O pins the device but the rest of them can also be used for other predefined functions.
BBB Description	    Header pin      	Description
MCASP0_AXR2        	P9_28           	Bitstream (audio data)
MCASP0_FSX         	P9_29           	Left/Right clock
MCASP0_ACLKX       	P9_31           	Bit clock
GND                 P9_01               Ground
VC3V3               P9_03               Power

on modern embedded linux, all this is done using "device trees".

### Enabling I2S Output from Beaglebone

TODO: 
- overview of device trees and device tree overlays, why we want them
- concede that the "common" bone dt has the mcasp pins enabled by default -- that's why it works
- users are really not meant to be writing their own device tree overlays if they can help it
  - it can be interesting and useful to understand them though. Here is a great raspberry pi reference (https://www.raspberrypi.org/documentation/configuration/device-tree.md).
  - note that a bunch of the stuff in that document is rpi specific, but it still explains the syntax and motivations for device trees in an approachable way
- for learning purposes, and in case you wanted to get this stuff up and running in other contexts (especially on TI MCUs with McASP modules), include the common overlay
  - with only the sound stuff and comment the heck out of it

Mention [Device Tree Overlays](http://www.ofitselfso.com/BeagleNotes/Beaglebone_Black_And_Device_Tree_Overlays.php), how to export etc
- check out bone common, see that mcasp module is enabled by default
- muxing pins is usually more straightforward on a bare metal device
- you can also mux the pins using bash
- device tree overlays are the current "real way" to do it on linux [bbb pin muxing](http://www.ofitselfso.com/BeagleNotes/BeagleboneBlackPinMuxModes.php)
- two things the fragments in device tree overlays can do: enable devices, or mux pins
- so since we want to enable the mcasp device and mux the relevant mcasp pins for I2S output, we need those two fragments
- (so we can't do device enable and pinmux in one fragment?) 
- learning how to write device tree overlays is a great skill if you are looking to design a cape or get an embedded linux job
  - however, you can probably get the functionality you need by checking out the *many* available devicetree overlays that set up the McASP pins
- I have found even reading a bunch of DTS files is way more approachable than trying to bumble through writing one on your own based on a spec
- what is ocp? (On-Chip-Peripherals (or OCP Devices), provided by the AM335x CPU SoC)

- check out this [audiocape](https://github.com/beagleboard/bb.org-overlays/blob/master/src/arm/BB-BONE-AUDI-02-00A0.dts) and the [BELA dts](https://github.com/beagleboard/bb.org-overlays/blob/master/src/arm/BB-BELA-B2.dts)

- how to: free space for those pins, mux them, and enable the device to use those pins

Beaglebone as of this writing enables I2S pins by default since they are used by the HDMI functionality
Show the video on device trees and where in the hdmi device tree the I2S stuff is specified

DAC, speaker, beaglebone pins



If you can get a logic analyzer, here is where you could verify the signals onthose pins!

## How is ALSA configured in general?

- oof
- sound field in device tree overlay
- asound.conf ?


# Hooking up the Components
- explain or link the purposes of the pins of the sparkfun board -- add to the previous table of mcasp pins
- how to determine which pins of the beaglebone to connect to the i2s breakout
- why is amplifier needed
- how to hook up speakers
- going to need to solder header pins

- do an alsa speaker test

# Make a sound!

- install your program with scp
- might have to chmod +x it
- run an you should hear the siren!


# scratch for dto
/opt/source/dtb-4.19-ti/src/arm/am335x-boneblack-audio.dts
/opt/source/bb.org-overlays/src/arm/cape-univ-audio-00A0.dts
/opt/source/dtb-4.14-ti/src/arm/am335x-boneblack-audio.dts
/usr/lib/ti/pru-software-support-package/pru_cape/pru_fw/PRU_Audio/PRU_audio.c
pulseaudio
/usr/include/linux/usb/audio.h
/usr/include/arm-linux-gnueabihf/libavutil/audio_fifo.h


Bone common

```
// SPDX-License-Identifier: GPL-2.0-only
/*
 * Copyright (C) 2012 Texas Instruments Incorporated - http://www.ti.com/
 */

#include <dt-bindings/display/tda998x.h>
#include <dt-bindings/interrupt-controller/irq.h>

&ldo3_reg {
	regulator-min-microvolt = <1800000>;
	regulator-max-microvolt = <1800000>;
	regulator-always-on;
};

&mmc1 {
	vmmc-supply = <&vmmcsd_fixed>;
};

&mmc2 {
	vmmc-supply = <&vmmcsd_fixed>;
	pinctrl-names = "default";
	pinctrl-0 = <&emmc_pins>;
	bus-width = <8>;
	status = "okay";
	non-removable;
};

&am33xx_pinmux {
	nxp_hdmi_bonelt_pins: nxp_hdmi_bonelt_pins {
		pinctrl-single,pins = <
			AM33XX_PADCONF(AM335X_PIN_XDMA_EVENT_INTR0, PIN_OUTPUT_PULLDOWN, MUX_MODE3)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA0, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA1, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA2, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA3, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA4, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA5, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA6, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA7, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA8, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA9, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA10, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA11, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA12, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA13, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA14, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_DATA15, PIN_OUTPUT, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_VSYNC, PIN_OUTPUT_PULLDOWN, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_HSYNC, PIN_OUTPUT_PULLDOWN, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_PCLK, PIN_OUTPUT_PULLDOWN, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_LCD_AC_BIAS_EN, PIN_OUTPUT_PULLDOWN, MUX_MODE0)
		>;
	};

	nxp_hdmi_bonelt_off_pins: nxp_hdmi_bonelt_off_pins {
		pinctrl-single,pins = <
			AM33XX_PADCONF(AM335X_PIN_XDMA_EVENT_INTR0, PIN_OUTPUT_PULLDOWN, MUX_MODE3)
		>;
	};

	mcasp0_pins: mcasp0_pins {
		pinctrl-single,pins = <
			AM33XX_PADCONF(AM335X_PIN_MCASP0_AHCLKX, PIN_INPUT_PULLUP, MUX_MODE0) /* mcasp0_ahcklx.mcasp0_ahclkx */
			AM33XX_PADCONF(AM335X_PIN_MCASP0_AHCLKR, PIN_OUTPUT_PULLDOWN, MUX_MODE2) /* mcasp0_ahclkr.mcasp0_axr2*/
			AM33XX_PADCONF(AM335X_PIN_MCASP0_FSX, PIN_OUTPUT_PULLUP, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_MCASP0_ACLKX, PIN_OUTPUT_PULLDOWN, MUX_MODE0)
			AM33XX_PADCONF(AM335X_PIN_GPMC_A11, PIN_OUTPUT_PULLDOWN, MUX_MODE7) /* gpmc_a11.GPIO1_27 */
		>;
	};
};

&lcdc {
	status = "okay";

	/* If you want to get 24 bit RGB and 16 BGR mode instead of
	 * current 16 bit RGB and 24 BGR modes, set the propety
	 * below to "crossed" and uncomment the video-ports -property
	 * in tda19988 node.
	 */
	blue-and-red-wiring = "straight";

	port {
		lcdc_0: endpoint@0 {
			remote-endpoint = <&hdmi_0>;
		};
	};
};

&i2c0 {
	tda19988: tda19988@70 {
		compatible = "nxp,tda998x";
		reg = <0x70>;
		nxp,calib-gpios = <&gpio1 25 0>;
		interrupts-extended = <&gpio1 25 IRQ_TYPE_LEVEL_LOW>;

		pinctrl-names = "default", "off";
		pinctrl-0 = <&nxp_hdmi_bonelt_pins>;
		pinctrl-1 = <&nxp_hdmi_bonelt_off_pins>;

		/* Convert 24bit BGR to RGB, e.g. cross red and blue wiring */
		/* video-ports = <0x234501>; */

		#sound-dai-cells = <0>;
		audio-ports = <	TDA998x_I2S	0x03>;

		ports {
			port@0 {
				hdmi_0: endpoint@0 {
					remote-endpoint = <&lcdc_0>;
				};
			};
		};
	};
};

&rtc {
	system-power-controller;
};

&mcasp0	{
	#sound-dai-cells = <0>;
	pinctrl-names = "default";
	pinctrl-0 = <&mcasp0_pins>;
	status = "okay";
	op-mode = <0>;	/* MCASP_IIS_MODE */
	tdm-slots = <2>;
	serial-dir = <	/* 0: INACTIVE, 1: TX, 2: RX */
			0 0 1 0
		>;
	tx-num-evt = <32>;
	rx-num-evt = <32>;
};

/ {
	memory@80000000 {
		device_type = "memory";
		reg = <0x80000000 0x20000000>; /* 512 MB */
	};

	clk_mcasp0_fixed: clk_mcasp0_fixed {
		#clock-cells = <0>;
		compatible = "fixed-clock";
		clock-frequency = <24576000>;
	};

	clk_mcasp0: clk_mcasp0 {
		#clock-cells = <0>;
		compatible = "gpio-gate-clock";
		clocks = <&clk_mcasp0_fixed>;
		enable-gpios = <&gpio1 27 0>; /* BeagleBone Black Clk enable on GPIO1_27 */
	};

	sound {
		compatible = "simple-audio-card";
		simple-audio-card,name = "TI BeagleBone Black";
		simple-audio-card,format = "i2s";
		simple-audio-card,bitclock-master = <&dailink0_master>;
		simple-audio-card,frame-master = <&dailink0_master>;

		dailink0_master: simple-audio-card,cpu {
			sound-dai = <&mcasp0>;
			clocks = <&clk_mcasp0>;
		};

		simple-audio-card,codec {
			sound-dai = <&tda19988>;
		};
	};
};
```