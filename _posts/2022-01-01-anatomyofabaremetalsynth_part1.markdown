---
layout: post
title: ANATOMY OF A BARE-METAL SYNTH, PART 1
date: 2022-01-01 04:01:00
description: The first in a five part series about the components of a simple bare metal synth using the Daisy platform. This part is about the MIDI input circuitry.
---

Exposition of embedded system concepts
Breadth over depth
Throw jargon around
Make further explorations more approachable
Anatomy of a digital synthesizer 
Daisy platform
Bare metal C


The goal of this talk is to tie together some of the concepts and terms you will see floating around when you start to get into embedded audio. Give you something to latch on to if you are interested in making things like guitar pedals, digital synthesizers, instruments, etc. 

Not meant to teach you everything you need to know

Introduce you to some of the jargon, with the hope that it will be less intimidating the next time around if you start to tinker with this stuff

Intro (this part!)
Types of embedded software development
Anatomy of a bare metal synth
MIDI circuitry
UART driver
Software MIDI parsing
Software writing audio data
Decoding I2S audio
Amplifying resulting analog audio

Before getting into details, this is where we are headed.
First the intro (this part). We’re doing it!
Then I’m going to talk about embedded development a little bit just to put the main section into context. Different types of embedded development and how to learn more about electronics, analog circuitry, prototyping, etc.
Then I will show you a simple synth I made with Daisy and go over all the main sections one by one.
First show you how the MIDI input gets into the synth
Go over how Daisy consumes MIDI using a communication protocol known as UART
Talk about how the UART is parsed as MIDI data
Go over how the audio codec eats up the digital audio that your program writes in response to the MIDI data
Talk about how the codec converts I2S digital audio into analog audio (line level)
Finally, how the line level audio can be amplified with an analog circuit to be loud enough for unpowered speakers or headphones
Then will quickly review and show the finished prototype!
So let’s start!

------

What are we going to talk about?
How to hook up a bare metal synth on a breadboard!
Hopefully simple enough that it can be a starting point
Specific enough that it is practical
Getting midi into a device, playing with it, and getting audio out
Basic embedded programming concepts 
“Bare metal programming”
Through the lens of a bare-metal synth running on the Daisy platform
Communication protocols
UART, I2S, MIDI
Digital to analog audio conversion
GPIOs and multiplexing
Interrupts

Knowledge of analog electronics useful
Not 100% required for this talk
If that’s not you yet, check out:
Make: Electronics by Charles Platt
Practical projects/labs
Easy to read
Learn soldering, tools, prototyping
Has audio projects!
Sparkfun also has great resources
For analog + digital electronics learning


So to start: if you aren’t into analog circuits yet, this section will offer some breadcrumbs.

Not totally required for this talk, but some knowledge of analog electronics will be helpful

If you don’t know any analog electronics, but are interested, I highly recommend working through Make: Electronics. It’s full of teaching concepts through labs, approachable but practical, learn about common patterns like RC circuits, timers, voltage dividers, how to solder. Also has audio projects!

And then especially for digital concepts, Sparkfun has lot of good resources. Meant to sell their products, but I like their products :)

--------



Aside: Breadboarding and Analog Electronics
Check out Make: electronics
You’ll get set up with a workstation, components, tools, etc.
You’ll see schematics, breadboards, and analog circuits in this talk
Sort of awkwardly out of scope
Recommend Make: Electronics if you do not have analog experience yet; teaches a lot of important concepts 
through easy projects and will get you comfortable with tools, schematics, and tinkering.

For example, you might see some schematics. These are diagrams that show electrical connections and component types, but not necessarily the exact layout of the circuit.

Here’s one of those; they look like this. This is a headphone amplifier from an Instructable article (linked). I used this LM386 schematic in this synth for the headphone amp. This circuit is also covered in Make: Electronics!

Then here is the corresponding prototype implementation of this circuit, using a tool known as a “solderless breadboard”. Can just plug in all the components and make sure they work before you have to solder anything.

Before getting into the synth, I want to go over the different types of embedded programming and where our synth fits in. We mentioned this before, but the “type” of programming used in this synth is called “bare metal” programming. This just means that we are writing a program that talks directly to hardware peripherals. Those peripherals are exposed to your C or C++ code via pointers; when you read from those pointers, you are actually reading data that was put there by a peripheral. When you write to the registered pointed to, you are actually sending data over hardware to those peripherals.
The big delimiter is the absence of a true operating system. These systems can be a lot more straightforward to reason about (still not “easy” necessarily).
Elecia White has a book called Making Embedded Systems that is a great first read to get into bare metal development in C.

On the other end of the embedded spectrum, we have embedded Linux. Very powerful way to develop, and makes a lot of sense, especially if you are developing software that also needs to run on native platforms. (easier to port). The OS overhead is non-trivial and there is a lot of history to learn about, both with Linux itself and the complicated build systems such as Yocto and Buildroot to create an embedded Linux image on your board. You also seldom run into interfacing with hardware “directly”; your application runs in user space generally, whereas kernel space programs are dealing with hardware.

There are other “in-betweens”. One is RTOS programming (you may see “FreeRTOS”). This is entwined with “real time programming” which requires specialized concerns, but in short, an RTOS based program is generally mostly baremetal, but it has a scheduler that allows for deterministic execution of your code. Then there is FPGA programming, which is also bare metal programming, but different in that you are programming logic gates directly using a “hardware definition language” such as Verilog, rather than C.

Types of embedded programming
To put what we are talking about into context, these are the main “camps” of embedded programming -- we are only talking about one of them
Linux
Powerful: you have a small computer
More easily port existing applications that build with gcc
Overhead of an OS
Build systems are extremely complicated
Kernel space vs user space
If you squint and set kernel build variables in a specific way, you can use Linux as an RTOS. This is a point of contention in the industry.
“Bare metal”
Talking directly to memory mapped addresses of physical devices
In reality, you’ll use a HAL and vendor-supplied libraries much of the time to make driver development much easier than starting from the ground up
A loose term; I’m personally only familiar with “embedded C or C++ programming” but there are similar fields such as FPGA programming done in Verilog, which is a sort of “hardware programming”, programming at the logic gate level
RTOS (ie FreeRTOS)
Still sort
“Mini OS” with a real-time scheduler, but you still gotta write the drivers

ELECTROSMITH DAISY

Embedded platform for music
On-board AKM audio codec
ARM Cortex M7 processor
Low cost
Software libraries make great examples
Bare-metal
Powerful DSP library DaisySP
Hardware library libDaisy

This synth was made using the “daisy platform”, which includes both hardware and software.
Useful for this talk because in most cases, platforms like Teensy are more general purpose, and you need to find libraries meant for audio and sift through a lot of stuff that doesn’t apply to your usage.
This board has a built-in audio codec, so you don’t need to buy some expansion or connect an external one. You can if you want, and I do in this talk just because it is a little easier to see the peripherals that way, but in practice that is edge case-y.
Still forces us to look at how usual bare metal techniques work, like pin multiplexing and gpios, communication protocols, etc.

This is the high level diagram of the synth
Shows how data flows through it.
We will use this diagram to talk through all the parts

This part o the block diagram deals with routing MIDI from an external source (such as a MIDI controller) into the Daisy
We start with one of these 5 pin DIN connectors you’ll see on mostly older synths
Or more concretely...

You can see the MIDI output coming from a Microkorg here into this breadboard jack

This is the whole synth, and:
Here is the part that deals with MIDI
It’s not just a single wire though with the MIDI data…
What is all that stuff?

MIDI Circuitry

Defined by MIDI spec
Optoisolator
Avoid ground loop
RX Pin out
Connect to Daisy
Transport mechanism

Let’s briefly talk about that circuit, because it’s kind of cool.
This schematic is from notesandvolts, who has a tutorial about it. Go read it!
This circuit is defined by the MIDI spec. It’s more complicated than just connecting the MIDI output of a device into the MIDI input of another device.
The whole thing revolves around this component, the 6N138, which is an optoisolator. Its purpose in life is to 
Take an electrical signal (MIDI)
Flash an LED
Put it next to a component that converts light into electrical signals
Outputs the corresponding data
We connect that data to a pin on the daisy
Why do this?
Since the MIDI output device is not directly electrically connected to our circuit, it is just connected by this light system, we won’t get any ground loops
Ground loops are bad, they can cause buzzing/humming by connecting one device’s ground to another

So that’s the transport mechanism, or how we get MIDI from point A to point B… but that begs the question...


MIDI circuit
Before worrying about how to get the data into the daisy, let’s determine how to receive from a midi device via a circuit
The MIDI circuit itself is fairly simple 
but is also fun because it requires an optoisolator
Show diagram, note which is the “midi out” wire
The circuit is specified by the official midi spec
https://www.notesandvolts.com/2015/02/midi-and-arduino-build-midi-input.html

what is MIDI?

It’s a “serial communication protocol”!

But that begs the question...

Very important in embedded
Peripheral walkie talkies
Binary electrical signals
Both sides agree what it means
UART, SPI, I2S, I2C
Great practice
Write your own drivers

Serial communications protocols are a crucial concept in learning embedded development
Embedded systems are really made up of a collection of peripherals who are talking to one another
The transport mechanism they speak is a binary electrical signal (a square wave)
Both sides need to know what that signal actually means
These are various protocols, UART, SPI, I2S, I2C

One great way to get started in embedded is to try writing your own drivers
Get a breakout board like an I2C temperature sensor
Write an I2C driver to read the temperature
That sort of stuff

Here is what a communication protocol looks like (this is from the Phillips spec for the I2C protocol). The top line is a square wave that has actual data (temperature, motor control, new sample rate setting for an audio interface). The bottom line is a clock so the receiving side knows where each byte starts and stops.

Simpler mechanisms can be like UART, which has only a single line of data

No clock involved, just an agreement on each side of how many “bits per second” the data will be sent at
Called the “baud rate”

So back to MIDI…
MIDI itself is a serial communication protocol, but for all intents and purposes, it is a specialization of UART
There are many ways to transmit midi
But the spec only identifies that MIDI is a single signal

Can come in many forms
Bluetooth
TRS
MIDI 5-pin DIN
Just a UART signal
“Universal Asynchronous Receiver-Transmitter”
Baud rate of 31250
See Sparkfun MIDI tutorial
