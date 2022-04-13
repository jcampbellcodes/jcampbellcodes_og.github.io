---
layout: post
title: ANATOMY OF A BARE-METAL SYNTH, PART 4
date: 2022-01-01 04:01:00
description: A post about the application portion of the Daisy synth.
---

This is the fourth part in a series about a bare metal synth. It assumes knowledge from the last post about pin multiplexing and the Daisy UART driver from [the previous post](/2021/12/31/anatomyofabaremetalsynth_part3.html).

# Daisy Synth Code
<br>

We've now reached the part of the Daisy diagram where we get to do something interesting with the
UART MIDI signal described in the last few posts.

<figure>
  <img class="col center" src="/img/bare_metal/flow_diagram_highlights/3_UART_driver.png">
  <figcaption>UART Driver on Daisy Synth</figcaption>
</figure>

We can use the Daisy MIDI library to parse the UART signal as MIDI note-on and note-off messages
and generate audio samples to send to the audio hardware based on those messages.

This synth combines a few Daisy examples (specifically PolyOsc and PluckEcho) with some edits to give us polyphony, a nice reverb, and an ADSR envelope / filter on each voice's oscillator.

{% highlight cpp %}
{% endhighlight %}

# Program Components
<br>

There are several high level components that make up the program:

- `DaisySeed seed`: The interface that libDaisy exposes to talk to the hardware on the Daisy Seed
- `MidiHandler midi`: The interface to the UART hardware on the Daisy that listens for UART messages and translates them to MIDI that we can use in our program
- `VoiceManager<24> voice_handler`: A manager of all the voices active in our program. Each voice has an oscillator, and the manager handles passing our MIDI messages to those voices and generating audio that represents all active voices
- `ReverbSc verb`: A Sean Costello reverb module from DaisySP that we run the `voice_handler` audio through to mix with reverb

# Initilization
<br>

- talk about the init code, how it relates to what we saw before. save audio callback for later

{% highlight cpp %}
int main(void)
{
    // initialize seed hardware and daisysp modules
    seed.Configure();
    seed.Init();
    float sample_rate = seed.AudioSampleRate();
    midi.Init(MidiHandler::INPUT_MODE_UART1, MidiHandler::OUTPUT_MODE_NONE);

    verb.Init(sample_rate);
    verb.SetFeedback(0.95f);
    verb.SetLpFreq(5000.0f);

    voice_handler.Init(sample_rate);

    // start callback
    seed.StartAudio(AudioCallback);
    midi.StartReceive();

    while(1) 
    {
        midi.Listen();
        while(midi.HasEvents())
        {
            HandleMidiMessage(midi.PopEvent());
        }
    }
}
{% endhighlight %}

# MIDI Handling
<br>

- go into the midi handler; how do we parse midi?
    - we just care about note on's and note off's
    - tell the VoiceHandler that either a note on or a note off came in, and it will react with the appropriate audio to play when its process method is called

{% highlight cpp %}
void HandleMidiMessage(MidiEvent m)
{
    switch(m.type)
    {
        case NoteOn:
        {
            NoteOnEvent p = m.AsNoteOn();
            // Note Off can come in as Note On w/ 0 Velocity
            if(p.velocity == 0.f)
            {
                voice_handler.OnNoteOff(p.note, p.velocity);
            }
            else
            {
                voice_handler.OnNoteOn(p.note, p.velocity);
            }
        }
        break;
        case NoteOff:
        {
            NoteOnEvent p = m.AsNoteOn();
            voice_handler.OnNoteOff(p.note, p.velocity);
        }
        break;
        default: break;
    }
}
{% endhighlight %}

# Voice Handler
<br>

- the synthesizer is made up of "voices"
    - each voice has an oscillator to produce audio
    - also includes an envelope and filter to shape the sound
- to make the example more interesting, we are using a class called VoiceManager to enable polyphony in our synth up to the max number of specified voices. (we set it as 24)

{% highlight cpp %}
{% endhighlight %}

# Audio Callback
<br>

{% highlight cpp %}
static void AudioCallback(float **inBuffer, float **outBuffer, size_t inNumSamples)
{
    // Assign Output Buffers
    float *out_left = outBuffer[0];
    float *out_right = outBuffer[1];
    float dry = 0.0f, send = 0.0f, wetl = 0.0f, wetr = 0.0f; // Effects Vars
    for(size_t sample = 0; sample < inNumSamples; sample++)
    {
        // get dry sample from the state of the voices
        dry  = voice_handler.Process() * 0.5f; 
        // run an attenuated dry signal through the reverb
        send = dry * 0.45f;
        verb.Process(send, send, &wetl, &wetr);
        // sum the dry oscillator and processed reverb signal
        out_left[sample]  = dry + wetl;
        out_right[sample] = dry + wetr;
    }
}
{% endhighlight %}

- audio callback
    - essentially registering an interrupt that the audio driver calls when it needs more audio samples
    - the callback gives us output buffers that the audio driver will send to the audio hardware, we are in charge of filling that audio output buffer
    - we call the "process" on our voice handler; based on the state it is in from the midi messages, this method gives us one sample of our dry signal
    - daisy also has some great processing libraries; here we pass the dry signal through a Sean Costello reverb processor just because we can and it sounds nice
    - then finally we sum the dry signal with the wet reverb signal and put it in the audio output buffer that is being sent to the audio hardware

- next: now that we have taken a look at the code, the next post will be a more in depth look at how the daisy libraries set up the audio callback and where the audio goes after our AudioCallback has been called






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

