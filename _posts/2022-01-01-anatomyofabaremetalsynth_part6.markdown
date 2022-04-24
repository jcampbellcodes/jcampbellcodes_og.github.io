---
layout: post
title: ANATOMY OF A BARE-METAL SYNTH, PART 6
date: 2022-1-1 04:01:00
description: The final part in the Daisy synth series, covering the (optional) audio amplification.
---

This is the final part in a series about a bare metal synth. See [the previous post](/2021/12/31/anatomyofabaremetalsynth_part5.html) and more for info needed here.

We last discussed how audio makes it from our program to the serial audio interface. Next up is this portion of the flow diagram, where the SAI exposes that audio as I2S data
available on a series of pins we can hook up to.

<figure>
  <img class="col center" src="/img/bare_metal/flow_diagram_highlights/5_I2S_codec.png">
  <figcaption>SAI DMA Audio to I2S Interface</figcaption>
</figure>

Note that Daisy abstracts some of this by default, by sending the SAI I2S signal to the on-board AK4556
audio codec, which then exposes the analog line-level audio output on pins 18 and 19 that we can hook up directly to a speaker. This means we don't strictly need to 
hook anything up manually to the I2S pins exposed by the SAI if we are ok with the conversion done by the AK4556, since those connections are also made internally for us for convenience. 

# Appendix: Digital Audio Exposed on GPIOs
<br>

But I think it is useful to briefly dig into this abstraction a bit by exploring what it would be like to hook up our own codec; just note that this next section isn't
strictly necessary to implement in your application, and we use the provided audio outputs in our actual synth as well to save breadboard real estate.

There are [I2S breakout boards](https://learn.sparkfun.com/tutorials/i2s-audio-breakout-hookup-guide) that you can plug into your breadboard. 

Here is what our circuit looks like if we hook up to one of these boards:
<figure>
  <img class="col center" src="/img/bare_metal/i2s_external_codec.jpg">
  <figcaption>Synth with External I2S Codec and Amplifier Breakout Board</figcaption>
</figure>

These accept input from
the pins that we multiplexed above, all of the I2S lines:
- `frame synchronization`: whether the current bit being read is the left or right channel
- `bit clock`: synchronization so we can tell which byte is which
- `data line`: the digital audio data

{% highlight cpp %}
sai_config.pin_config.fs   = {DSY_GPIOE, 4}; // frame synchronization
sai_config.pin_config.sck  = {DSY_GPIOE, 5}; // bit clock
sai_config.pin_config.sa   = {DSY_GPIOE, 6}; // data line
{% endhighlight %}

Those three pins come out of the Daisy here, pins 35, 34, and 33 (from top to bottom):
<figure>
  <img class="col center" src="/img/bare_metal/2_uartin_highlight.jpg">
  <figcaption>I2S Lines Coming out of the Daisy</figcaption>
</figure>

And we connect those three lines into the breakout board like so (see the Sparkfun link for more info on their input/output pins):
- SAI1 FS ("frame synchronization") -> LRCLK ("left right clock")
- SAI1 SCK ("serial clock") -> BCLK ("bit clock")
- SAI1 SA ("serial A") -> DIN ("data in")

<figure>
  <img class="col center" src="/img/bare_metal/3_i2s_out.jpg">
  <figcaption>Digital audio output sent to I2S codec</figcaption>
</figure>

The breakout board then sends the output of the DAC to an amplifier (whose gain can be controlled by other pins exposed as inputs) whose analog audio output
can be hooked up to headphones or a speaker.

# Appendix: Amplifying Analog Audio
<br>

The final piece I want to dig into is amplification of the analog audio that we get from the DAC output:

<figure>
  <img class="col center" src="/img/bare_metal/flow_diagram_highlights/6_codec_to_amp.png">
  <figcaption>Sending analog audio to amplifier</figcaption>
</figure>

The audio we receive from the DAC is line-level, so if we plug headphones directly into it, it will be much too quiet
to hear at a usable level.

This part is abstracted away from the I2S breakout board we used above, so actually I think it is useful to
go *back* to the default way of using the Daisy and hooking up to the default analog outputs from the AK4556 codec.

We can add a pair of headphone amplifiers for the left and right output signals to boost the line level
audio coming out of the AK4556 so that we can hook up unpowered speakers or headphones directly to this output.

<figure>
  <img class="col center" src="/img/bare_metal/overhead_synth_highlight.jpeg">
  <figcaption>Annotated Synth with Headphone Amps and Line Out Options</figcaption>
</figure>

The connection to the speaker in the above also marks the final part in the flow diagram: analog audio to amplifier to speaker!

<figure>
  <img class="col center" src="/img/bare_metal/flow_diagram_highlights/7_amp_to_speaker.png">
  <figcaption>Sending amplified audio to speaker</figcaption>
</figure>

The headphone amp is an LM386 design taken from this [helpful article](https://www.instructables.com/Tales-From-the-Chip-LM386-Audio-Amplifier/):
<figure>
  <img class="col center" src="/img/bare_metal/instructable_lm386.png">
  <figcaption><a href="https://www.instructables.com/Tales-From-the-Chip-LM386-Audio-Amplifier/"> LM386 amplifier tutorial</a></figcaption>
</figure>

And here is one of the amplifiers on its own, breadboarded out based on that schematic:

<figure>
  <img class="col center" src="/img/bare_metal/headphone_amp.jpg">
  <figcaption>Built headphone amplifier based on schematic</figcaption>
</figure>


# Wrapping Up
<br>

<figure>
  <img class="col center" src="/img/bare_metal/0_midiout.jpg">
  <figcaption>MIDI output from controller</figcaption>
</figure>
<figure>
  <img class="col center" src="/img/bare_metal/1_midiin.jpg">
  <figcaption>MIDI input to circuit</figcaption>
</figure>
<figure>
  <img class="col center" src="/img/bare_metal/2_uartin.jpg">
  <figcaption>Daisy processing MIDI input, generating I2S digital audio output</figcaption>
</figure>
<figure>
  <img class="col center" src="/img/bare_metal/3_i2s_out.jpg">
  <figcaption>Digital audio output sent to I2S codec</figcaption>
</figure>
<figure>
  <img class="col center" src="/img/bare_metal/4_amplified_analog.jpg">
  <figcaption>Analog audio output sent to speaker</figcaption>
</figure>
