---
layout: post
title: ANATOMY OF A BARE-METAL SYNTH, PART 4
date: 2022-01-01 04:01:00
description: A post about the application portion of the Daisy synth.
---

*now show some code*
Let’s start from the code for the actual synth. Notice there aren’t any direct references to a UART driver in this application; just an Init call to the MIDI handler, set to UART mode. (as opposed to USB midi, etc).
Let’s look in there…
There is an init call to the UART handler...
Don’t worry about interrupts or the uart driver itself yet, we will get there…
Just looking for the multiplexing code
Check out the UART config; this UART handle currently is only set up to use USART1, which should work for what we want
See the GPIO initialization here, which sets the corresponding pins to be their USART1 configuration
TODO: Find mapping between those GPIO defines and the Daisy pinout



-----

libDaisy does this for us, let’s see how!
Let’s look at libDaisy to see what it expects
Go into repo, checkout midi.h
It references UART1! And calls init on it… hmmm..
Dig a little deeper, what is UartHandler…
Ok cool, its set currently to always use USART1 and we can see this is where the board would expect to see midi
Important to note: it sets the baud rate as well. MIDI is a non-standard baud rate.
Easy to set here; a pro of bare metal!
Pain in the butt in linux
So now we know how midi is getting in!


Now we have covered how MIDI is handled in the system -- next let’s go into how audio comes out of the thing

Now that we have a way to access MIDI data, our program looks a lot like a regular ol’ C++ audio program. There is an audio callback firing off concurrently with a “main thread” (will talk about that more later). The main thread is listening to MIDI messages, and writing to a shared structure in response to note on/note offs. The audio callback asks this structure for an audio sample, which the structure gives back based on the current state of its oscillators and envelopes.
There are concurrency concerns here I might ignore…
The audio sample is still just a series of floats. Next we are going to focus on...


Read by a program running on the CPU
Now you can access midi data; time to write a program
We set up the MIDI driver using libDaisy; this reads from the UART and interprets the UART specifically as MIDI messages that we can read
parses that raw UART data against the MIDI spec
Raw UART bytes parsed with MidiHandler::Parse(uint8_t byte)
Now we can use some little functions to write some audio to an output buffer

