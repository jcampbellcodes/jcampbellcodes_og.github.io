# Set up the Beaglebone Black

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
- Load the ALSA program onto the Beaglebone
- This program connects to the default audio device, which is `plughw:Black,0`
- We read audio output from this device using the SPI pins, which output the audio as an I2S data stream by default (this data is also being sent to the HDMI output, but we don't ue that)
- We connect these pins and power to the corresponding inputs of the I2S digitial-to-analog converter breakout board
- The breakout board also




- what is mcasp? Multi Channel Audio Serial Port
- Checkout [beaglebone pinout](https://microcontrollerslab.com/beaglebone-black-pinout-pin-configuration-features-applications/#MCASP_Pins)
- explain overall the parts of our circuit
- what is [I2S](https://hackaday.com/2019/04/18/all-you-need-to-know-about-i2s/)
- I chose I2S (Inter-IC Sound) protocol over USB or SPI mainly because it is yet another technology I have been meaning to learn and it leaves open 
a USB port to be used by another input device (planning to use it for USB MIDI in a future project). That 
said, it turns out I2S output on the Beaglebone Black is actually a little wonky. Unless you use an
external clock, you are limited to the 48kHz family of sample rates ([see here](https://hifiduino.wordpress.com/2014/03/10/beaglebone-black-for-audio/)). 
This was fine for my purposes, but it is an unfortunate drawback. The I2S pins are mainly used for the HDMI output, which 
[this article](https://www.raspberry-pi-geek.com/Archive/2013/02/HDMI-and-the-BeagleBone-Black-Multimedia-Environment) sheds some
helpful light upon.
- general embedded learning
- don't be afraid of technical specs, pinouts, etc. They are dry and generally not beginner friendly,
- but most of the time they will be your only life raft. So adapt to them!
- A good way to learn about a board's quirks is to mess with various serial protocols on 
- the boards. Also! don't be afraid of computer specs. Similar to math equations, it's not 
- immediately natural to convert these to "english" in your mind. But it's important to fight the
- urge to skip over part numbers when reading, because that helps you make connections. Ie the Beaglebone
- uses the TI AM335x microcontroller -- you will find a lot of docs about this chip that aren't beaglebone specific
- and you need to be able to grok these without relying on the article talking specifically about the beaglebone
- check out other serial protocols: 
- [serial communication, UART](https://learn.sparkfun.com/tutorials/serial-communication)
- [I2C](https://learn.sparkfun.com/tutorials/i2c/all) 
- [SPI](https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi/all)

# Hooking up the Components
- explain or link the purposes of the pins of the sparkfun board
- how to determine which pins of the beaglebone to connect to the i2s breakout
- why is amplifier needed
- how to hook up speakers
- going to need to solder header pins

## Enabling I2S Output from Beaglebone

Mention Device Tree Overlays, how to export etc
Beaglebone as of this writing enables I2S pins by default since they are used by the HDMI functionality
Show the video on device trees and where in the hdmi device tree the I2S stuff is specified

DAC, speaker, beaglebone pins


- talk about "multiplexing". These pins can be configured as GPIOs, or output SPI, or output I2S audio when configured as McASP pins
  There are almost 69 I/O pins the device but the rest of them can also be used for other predefined functions.
BBB Description	Header pin      	Description
SPI1_CS0        	P9_28           	Bitstream
SPI1_D0         	P9_29           	Left/Right clock
SPI1_SCLK       	P9_31           	Bit clock
GND                 P9_00 (check)
VC3V3               P9_03

MCASP0_FSX (Frame Sync) – Pin29
MCASP0_ACLKX (Clock Sync) – Pin25
MCASP0_AHCLKX (Data Out) – Pin31
MCASP0_AXR2 (Data In) – Pin28

- a note on device tree overlays

- do an alsa speaker test

# Make a sound!

- install your program with scp
- might have to chmod +x it
- run an you should hear the siren!
