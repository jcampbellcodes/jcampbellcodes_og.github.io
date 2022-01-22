---
layout: post
title: ANATOMY OF A BARE-METAL SYNTH, PART 4
date: 2022-01-01 04:01:00
description: The fourth in a five part series about the components of a simple bare metal synth using the Daisy platform. This part is about the audio codec.
---

How are all these floats we are supplying the audio callback being turned into sound?
Yes, “digital to analog conversion”
But how is that process enabled by our hardware?

Generating audio
Just like in native audio software with plugin frameworks like JUCE, the heart of your audio processing happens in the “audio callback”
You are given a buffer of floats representing audio samples
You write floats to the output buffer
Where do those floats come from, where do they go? (Where do they come from Cotton Eyed Joe?)
To understand that, we need to know how our audio callback fits into the larger picture 

Let’s start with the audio that we want to plug into via the pins on the Daisy.
What does that digital audio look like?

It is yet another communication protocol called I2S

I2S
Another communication protocol!
This one is digital audio data
You can’t just plug it into a speaker
Looks like this:
Show chart
Not analog yet
Just a bunch of discrete floats
We have to convert it to an analog signal first
We convert to analog audio that we can plug into a speaker
Internally by default, the conversion from I2S to analog audio is set up with SAI1 and goes through an AKM codec (digital to analog converter), and comes out on two pins
This is the easiest way to configure the audio
You could also use SAI2 to get the digital I2S signal and hook it up to your own DAC
This sparkfun one is nice
This is a good concept to get familiar with because a lot of boards aren’t going to have a good codec right on them, so you’ll want to connect the I2S signal to your own codec in those cases


We register the callback as an interrupt and then the hardware calls them in a manner consistent with its configuration.
Daisy audio callback:
We write our audio callback
Pass it to daisy configuration
Daisy passes it to audio handler, SAI, serial audio interface…
Audio handler passes it to DMA handler
Now we have reached the HAL, which in many cases is the lowest level you should have to care about
In this case, we ultimately pass our callback to the Serial Audio Interface, which requests the audio data via an interrupt through its DMA interface


SAI
The Serial Audio Interface peripheral on the STM32 boards has a DMA interface and exposes audio data on a variety of pins
DMA
Direct memory access
Makes sense for audio; 
CPU initializes a DMA request, receives an interrupt when the DMA’s buffer is full
That’s when your audio is processed
These pins are how we read the data from our audio callback out as a digital I2S signal

How does SAI get the audio?

Talk about DMA
Talk about setting the audio interrupt
Talk about the form that the SAI exposes the audio
I2S serial communication protocol!

Then how do we convert the I2S digital audio to analog audio?

The audio codec!
