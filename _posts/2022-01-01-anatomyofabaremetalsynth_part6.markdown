---
layout: post
title: ANATOMY OF A BARE-METAL SYNTH, PART 6
date: 2022-1-1 04:01:00
description: The final part in the Daisy synth series, covering the (optional) audio amplification.
---

This is the final part in a series about a bare metal synth. See [the previous post](/2021/12/31/anatomyofabaremetalsynth_part5.html) and more for info needed here.

# Digital to Analog Audio
<br>

<figure>
  <img class="col center" src="/img/bare_metal/flow_diagram_highlights/5_I2S_codec.png">
  <figcaption>Sending digital audio to codec</figcaption>
</figure>

- the pins on our daisy have I2S digital audio on them.
- using these pins, we can connect to a peripheral that converts I2S audio to a stereo analog signal
- the daisy also offers an internal connection from these pins to its onboard AKM audio codec
- but to make things more visual, here is how it would look to connect to an external I2S breakout board

<figure>
  <img class="col center" src="/img/bare_metal/i2s_external_codec.jpg">
  <figcaption>Synth With External I2S Codec</figcaption>
</figure>

# Amplifying Analog Audio
<br>

<figure>
  <img class="col center" src="/img/bare_metal/flow_diagram_highlights/6_codec_to_amp.png">
  <figcaption>Sending analog audio to amplifier</figcaption>
</figure>

So now we have line level audio out!
You could plug this into your audio interface or powered speakers at this point
What if you want to plug in headphones?
Last step, I promise

<figure>
  <img class="col center" src="/img/bare_metal/flow_diagram_highlights/7_amp_to_speaker.png">
  <figcaption>Sending amplified audio to speaker</figcaption>
</figure>

<figure>
  <img class="col center" src="/img/bare_metal/overhead_synth_midi.jpg">
  <figcaption>Synth Using Internal I2S Codec And External Headphone Amp</figcaption>
</figure>


This is where amplifiers come in
You’ve got your analog signal, but its too quiet to listen on regular old speakers, like this chonky one or your headphones
You can hook up this signal into this analog circuit with the LM386
Show instructable circuit
[LM386 amplifier tutorial](https://www.instructables.com/Tales-From-the-Chip-LM386-Audio-Amplifier/):
<figure>
  <img class="col center" src="/img/bare_metal/instructable_lm386.png">
  <figcaption><a href="https://www.instructables.com/Tales-From-the-Chip-LM386-Audio-Amplifier/"> LM386 amplifier tutorial</a></figcaption>
</figure>

<figure>
  <img class="col center" src="/img/bare_metal/headphone_amp.jpg">
  <figcaption>Built headphone amplifier based on schematic</figcaption>
</figure>

Now the audio signal is powerful enough to drive a speaker or your headphones!

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
