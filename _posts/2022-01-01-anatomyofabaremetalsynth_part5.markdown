---
layout: post
title: ANATOMY OF A BARE-METAL SYNTH, PART 5
date: 2022-01-01 04:01:00
description: A post covering the audio codec on the Daisy synth.
---

This is the fifth part in a series about a bare metal synth. See [the previous post](/2021/12/31/anatomyofabaremetalsynth_part4.html) and more for info needed here.

We will be covering this portion of the flow diagram:

<figure>
  <img class="col center" src="/img/bare_metal/flow_diagram_highlights/4_SAI_DMA_driver.png">
  <figcaption>SAI DMA Driver on Daisy</figcaption>
</figure>

In other words, the audio samples (`float`s) that we generated in our `AudioCallback` part of our application -- where do they go?

In short, SAI.

# Serial Audio Interface
<br>

What does the SAI do?
- also known as an I2S interface
- in our application, handles taking our output signal as floats representing analog voltages (NOTE: that's what digital audio is) and exposing that audio on a series of pins that output I2S protocol

# Inter-IC Sound (I2S)
<br>

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


# How Daisy Configures the SAI
<br>

- Seed configures the SAI handle that lives in libDaisy
- Initializes SAI peripheral in SaiHandle::Impl::Init(const SaiHandle::Config& config)
- Multiplexes GPIOs to output I2S: SaiHandle::Impl::InitPins()
- Initializes DMA:  SaiHandle::Impl::InitDma(PeripheralBlock block)

```
 * Use SAI Handle like this:
 * 
 *  SaiHandle::Config sai_config;
 *  sai_config.periph          = SaiHandle::Config::Peripheral::SAI_1;
 *  sai_config.sr              = SaiHandle::Config::SampleRate::SAI_48KHZ;
 *  sai_config.bit_depth       = SaiHandle::Config::BitDepth::SAI_24BIT;
 *  sai_config.a_sync          = SaiHandle::Config::Sync::MASTER;
 *  sai_config.b_sync          = SaiHandle::Config::Sync::SLAVE;
 *  sai_config.a_dir           = SaiHandle::Config::Direction::RECEIVE;
 *  sai_config.b_dir           = SaiHandle::Config::Direction::TRANSMIT;
 *  sai_config.pin_config.fs   = {DSY_GPIOE, 4};
 *  sai_config.pin_config.mclk = {DSY_GPIOE, 2};
 *  sai_config.pin_config.sck  = {DSY_GPIOE, 5};
 *  sai_config.pin_config.sa   = {DSY_GPIOE, 6};
 *  sai_config.pin_config.sb   = {DSY_GPIOE, 3};
 *  // Then Initialize
 *  SaiHandle sai;
 *  sai.Init(sai_config);
```

# Transporting I2S using DMA
<br>

- user calls "start" on the seed object, which starts the DMA requests
    - initiates DMA request: `SaiHandle::Impl::StartDmaTransfer`. Sets up audio io buffers that will come from and go to SAI

```
 * Callback functions will be called once per half of the buffer. In the above example, 
 * the callback function would be called once for every 48 samples.
 * 
 *  // Now you can use it:
 *  sai.StartDma(. . .);
```

- We get an interrupt from audio hardware when it wants audio samples (one for full receipt and one for half receipt), `extern "C" void HAL_SAI_RxCpltCallback(SAI_HandleTypeDef* hsai)`. this is when the user audio callback is invoked.

# Digital Audio Exposed on GPIOs
<br>


# Next Up
<br>

The next and final post in this series will go over how we convert the digital I2S audio to analog audio and amplify it to line level.








---------------
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
