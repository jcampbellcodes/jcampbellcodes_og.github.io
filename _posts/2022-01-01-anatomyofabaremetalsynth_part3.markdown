---
layout: post
title: ANATOMY OF A BARE-METAL SYNTH, PART 3
date: 2022-01-01 04:01:00
description: A post about the UART driver in a Daisy synth.
---

go back to the previous parts if you haven't read them

# Receiving UART Data on the Daisy
<br>

How do we connect MIDI to daisy? The uart!

At a high level, we want to find a “pin” (connector) on the Daisy which can accept a UART signal, so that our program can read the MIDI data. So we need to look at the available pins on the Daisy and try to find a UART one.

<figure>
  <img class="col center" src="/img/bare_metal/DaisyPinoutRev4@4x.png">
  <figcaption>MIDI input to circuit</figcaption>
</figure>

This diagram is known as a “pinout”.
It shows the Daisy, and lays out what each pin can be used for.
We need to find the one that we can accept UART…
Note: You’ll see the letters “rx” and “tx” a lot in embedded
They stand for: Rx==Receive(input) Tx==Transmit (output)

So we want to find a UART Rx pin, because we are transmitting MIDI and want the board to Receive it on a UART pin
(On this pinout you’ll see USART and UART; for this scenario, the difference doesn’t matter)
The USART is a special peripheral that can deal with regular UART signals with start and stop bits or be put into “sync” mode, where a internal clock is used and no start/stop bits are needed in the signal

But many of the pins, including the USART pin, have more than function listed!
What does that mean? How do we tell Daisy that we want to send a UART signal to a particular pin without it thinking that we are sending an I2C signal?

# Multiplexing
<br>

To understand, let’s talk about multiplexing.
Those pins that have multiple functions next to them are called “General Purpose Input Output” pins, or GPIO.
It means that the pin can be assigned different functions (and possibly internally connected to different peripherals) depending on configuration in the software.

It would waste precious space to organize this board such that each of these possible functions have their own pins, which is why they are GPIOs in the first place.

So we need to find the GPIO we want, and “multiplex it” (sometimes calling “muxing”) in the code so that the pin we want is a UART Rx pin that we can send MIDI data to. 


# General Purpose Input/Output Pins
<br>

These pins are called General Purpose Input/Output (GPIO) pins and have multiple meanings, since it would be infeasible and inefficient to fit all the pins you need for all these functions on one board
Daisy pinout says “peripheral GPIO”; the pin we are looking at could connect to either the USART1 peripheral, or the I2C4 peripheral
For that reason, we have a concept called “multiplexing” where we specify which pins should be assigned to which function.
So we have to set up a UART MIDI Rx pin; turns out libDaisy does this for us, let’s see how!

General Purpose Input Output
Can be assigned various functions
Would waste space otherwise
Setting the pin function == “multiplexing”
Or muxing
Find a GPIO that can be UART Rx
Set the mode in the code


Once we enable the correct pin to be in “USART1” configuration, where does the data go?

Well an implication of muxing the pin to be in USART1 configuration means this pin now sends data to the USART1 STM32H7 peripheral.

# Direct Memory Access
<br>
The final conceptual part about this communication I want to talk about before digging into code is how the USART1 peripheral receives the data and uses the DMA (direct memory access) interface to allow the program to receive the MIDI data via the UART driver

DMA request
Set Destination (give me data) vs source (send this data to another peripheral)
How much data to receive
When we receive that data, we get an interrupt
That’s when we deal with the data (do some midi stuff)
When do we initiate the DMA request?
Hardware vs software trigger
see:
How to use this driver
In stm32h7xx_hal_uart.c

# Interrupts
<br>

What an interrupt is in general
How does an audio interrupt work

Interrupts
Similar to polling vs callbacks
You want some data
You can repeatedly ask for it from a peripheral (polling)
Or you can give the peripheral a function and say “call this when the data is ready”
The audio callback in this case is what is known as an “interrupt”
We actually have already dealt with an interrupt in this talk already:
The MIDI’s UART driver operates using DMA interrupts
High priority callback
IRQ calls ISR (service routine); the AudioCallback is the ISR
There are certain rules to keep in mind when writing ISR’s, mostly the same as the “usual suspects” for realtime safe code. No allocations, etc.

# Putting it into context
<br>

The next step to understanding this stuff is to see how the Daisy platform sets it all up!

We know generally that we need to find and configure a UART Rx pin to send our MIDI to. To understand the best way to do that on the Daisy (currently),
I want to dig into the MIDI handling code and show how they’ve set up the pin multiplexing and the UART driver, which allows your code to read the MIDI data that it might expect (not raw bytes, but parsed into “note on” events, etc that we might expect)

What are we looking for?
Which UART the libDaisy MIDI driver can use
Corresponding UART driver
How/when a DMA request is initiated
GPIO multiplexing
Which pins we connect to!
Where is the DMA receive interrupt

Turns out, we don’t actually have to do this ourselves with Daisy.
They handle the multiplexing; we just use their library and “turn MIDI on”
But let’s look into what they are doing a little bit to see which pin they use and how they enable it

# Next steps...

Now that we know how the UART driver is set up, let's look at the application code itself to see
how we process the MIDI in code and create audio.
