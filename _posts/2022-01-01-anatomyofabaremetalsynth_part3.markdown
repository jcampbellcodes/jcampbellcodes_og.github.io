---
layout: post
title: ANATOMY OF A BARE-METAL SYNTH, PART 3
date: 2022-01-01 04:01:00
description: A post about the UART driver in a Daisy synth.
---

This is the third part in a series about a bare metal synth. For an introduction to this series, see [the previous post](/2021/12/31/anatomyofabaremetalsynth_part2.html).

# Receiving UART Data on the Daisy
<br>

In the last post, we talked about what MIDI is (a UART signal) and the circuit involved with
preparing the MIDI signal to be sent to the Daisy. But how do we take the output from that
circuit and use it in our code?

The first step is to find where to connect the output to the Daisy. The connections to the
outside world on a microcontroller are called "pins", and we need to find a pin that will
accept our UART signal and feed it to the Daisy's UART driver.

<figure>
  <img class="col center" src="/img/bare_metal/DaisyPinoutRev4@4x.png">
  <figcaption>Daisy pinout</figcaption>
</figure>

This diagram above is known as a "pinout". It diagrams all of the available pins on the
Daisy and what they can be used for. 

---
**NOTE**

You'll see the letters "Rx" and "Tx" a lot in embedded documentation. These stand for
"receive" (input) and "transmit" (output), respectively.
---

Using this diagram, we want to find the UART Rx pin, because we are transmitting a MIDI
signal to be "received" by the Daisy. In this particular pinout, you'll see the pin we want (pin 15)
marked as "USART" (Universal Synchronous/Asynchronous Receiver/Transmitter). In this
case, the difference does not matter, because USARTs can handle regular UART signals as well,
like the one we are sending.

But you'll notice -- many of the pins, including the USART pin, have more than one function
listed. What does that mean? How do we tell the Daisy that we want to send a UART signal to pin 15,
rather than an I2C signal which is also listed for pin 15?

# Multiplexing
<br>

The term for assigning multiple potential uses to a signal pin is "multiplexing" (or sometimes
"muxing"). Multiplexing helps to save space. While it might be more straightforward to have a single
function exposed on every pin without having to mess around with configuration in the code, 
not every application is going to need every function from every pin. So multiplexing keeps
the space footprint of the board down.


### General Purpose Input/Output Pins
<br>

The multi-purpose pins themselves are called "General Purpose Input Output" pins, or
GPIOs. These pins can be assigned to any of the labeled functions and internal peripherals 
depending on configuration in the software. So we need to find a way to set up the pin 
configuration in our code such that pin 15 is in "USART Rx" mode, which makes it ready to
receive our MIDI signal. Luckily, this is handled in `libDaisy` for us.

### Daisy MIDI UART Config
<br>

`libDaisy` implements two wrappers we can look at for the particular case of UART MIDI handling.
The first is the `MidiHandler` class in midi.h/cpp. This class is how our application
code can interface with the MIDI hardware without worrying about hardware details. This class
also contains the second wrapper, `UartHandler` (uart.h/cpp), which abstracts away the details of 
dealing with the STM32 hardware abstraction library (HAL) that sets the registers that interact
directly with the UART peripheral. 
Diving into these two files can teach you a lot about how a UART driver is implemented.

This is how the `MidiHandler` configures the UART class, notably setting the baud rate to the
one specified by the MIDI spec (31250), initializing the USART_1 peripheral, and telling the
USART_1 peripheral to use `GPIO port B pin 7` as the "receive" pin and `GPIO port B pin 6`
as the "transmit" pin.

{% highlight c++ %}
UartHandler::Config config;
config.baudrate      = 31250;
config.periph        = UartHandler::Config::Peripheral::USART_1;
config.stopbits      = UartHandler::Config::StopBits::BITS_1;
config.parity        = UartHandler::Config::Parity::NONE;
config.mode          = UartHandler::Config::Mode::TX_RX;
config.wordlength    = UartHandler::Config::WordLength::BITS_8;
config.pin_config.rx = {DSY_GPIOB, 7};
config.pin_config.tx = {DSY_GPIOB, 6};
uart_.Init(config);
{% endhighlight %}

But wait, how did we choose those pins? On the pinout earlier, weren't the USART1 pins 14
and 15? Well, they were indeed, and this is one of the areas where you will be forced
to dive into some datasheets to figure out what's up. The pins exposed by the Daisy
translate to pins exposed by the STM32 microcontroller, and the mapping is not 1-1.

To find the mapping, you will need to look at the Daisy's datasheet, which lives [here](https://github.com/electro-smith/Hardware/blob/master/doc/daisy_seed/Daisy_Seed_datasheet_v1.0.4.pdf).
In there, you'll find a table called "pin functions". The leftmost column
shows the numbers on the pinout, so if we go to 14 and 15, we will see 
that these pins map to PB6 and PB7. 

<figure>
  <img class="col center" src="/img/bare_metal/pin_functions_gpiob.png">
  <figcaption>MIDI input to circuit</figcaption>
</figure>

---
**NOTE**
GPIO pins are grouped into ports, and `Px` is a common short hand for "port X", but you'll also see it as "GPIOx", as in this case, where PB6 is shorthand for "GPIO port B, pin 6".

---

So now the pins are initialized to be in "receive MIDI" mode. Now that these pins can receive data, how do we access that data from our code?


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
