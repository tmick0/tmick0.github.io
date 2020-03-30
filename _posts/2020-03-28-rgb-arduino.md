---
layout: post
status: publish
published: true
title: A highly customizable RGB controller implementation for Arduino
author: Travis Mick
date: 2020-03-28
image: /images/breadboard-rgb-controller-on.jpg
---

I recently decided that I needed to add some color to my workspace, but didn't want to just use any off-the-shelf RGB controller.
My first thought was to use an Arduino to control some RGB strips, but I didn't want to have to to open the Arduino IDE and modify
the firmware every time I wanted to change the program.

This desire eventually escalated to implementing a virtual machine on top of the Arduino with an application-specific instruction
set designed to easily manipulate LEDs.

<!-- more -->

# Controller build

Though I did my initial testing with an Arduino Uno, I chose to go with 16MHz Pro Micros (like [this one available from Sparkfun](https://www.sparkfun.com/products/12640)) for the actual
build in order to keep the footprint small. Other than that, the supporting circuitry is dependent on the application. I've deployed two controllers: one inside my PC, and one for backlight
behind my monitors. The setup I'm using for the PC is the simplest, so I'll explain that one first.

The controller inside my PC is driving an SK9822-based RGB strip, so it only needs a 5V supply. Given that it's inside a PC, there is easy access to 5V via a Molex connector from the PSU.
Therefore, I've used that to power both the Arduino and the RGB strip. The controller module was built by soldering two female header rows to a proto board to accept the Arduino, wiring
the Molex to the RAW and GND pins, and attaching a JST socket to the data, clock, and power pins. It ended up fitting nicely in the back of my case and I was able to secure it with a Velcro
strap.

![Controller mounted inside a PC case](/images/pc-rgb-controller.jpg)

(Sorry, I forgot to take a better photo before installing it.)

The controller for my monitor backlight is a bit more involved. For this application, wanted an on/off switch as well as a few potentiometer inputs to control the color of the strip.
It's running a WS2811 strip that takes 12V input, but I still needed a 5V reference and I didn't want to have to keep the Arduino connected over USB. Therefore, I included a voltage
regulator to step down the 12V from the strip's power supply and a MOSFET to handle the on/off capability (since I didn't expect the latching pushbutton switch I was using to be rated
for much current).

The result was still a reasonable size and fit nicely into a decently sized project box, with just the barrel jack for the 12V and a JST connector for the strip protruding from opposite sides.

Since the circuit was a bit involved for me (having little to no experience doing such things), I drew up a schematic first.

![Schematic](/images/led-control-circuit.png)

I then prototyped the design on a breadboard.

![Breadboard prototype](/images/breadboard-rgb-controller.jpg)

I tested it out and everything worked smoothly. I decided to leave out the decoupling capacitor depicted in the schematic as I found it wasn't necessary.

![Breadboard prototype driving an RGB strip](/images/breadboard-rgb-controller-on.jpg)

For the final build, I ended up placing the potentiometers and the power switch on their own boards, connecting them to the main board with some jumpers.

![Modular design in the case](/images/rgb-controller-modules.jpg)

Testing it out prior to closing up the case, everything still worked as intended.

![Final test](/images/rgb-final-test.jpg)

(Excuse the assorted debris on my workbench.)

For completeness, the final product:

![Desktop RGB controller](/images/desktop-rgb-controller.jpg)


# Virtual machine

As I mentioned, the RGB controller itself is implemented in a virtual machine. It's an 8 bit machine with 15 general purpose registers, and otherwise unremarkable except for a few 
special instructions for analog input, RGB output, and HSV-to-RGB color space conversion. I wrote an assembler for the machine in Python so I could write code with mnemonics instead of 
manually constructing bytecode.

The program memory of the VM persists in the EEPROM of the Arduino and is read into RAM at startup. It can be reprogrammed over a simple serial protocol.

The VM doesn't have function calls (though there is branching) or any form of data memory; instead, programs only operate via I/O and registers. This was my first time implementing
a virtual machine, and I wanted to keep it simple.

# Drivers

The VM uses three instructions to interact with RGB devices through a set of drivers that have been implemented in the firmware. Namely, there's an `init` instruction to activate a 
driver, a `write` instruction to store an RGB value to a buffer, and a `send` instruction to activate the buffered values.

I've implemented three drivers: a simple PWM-based driver for analog RGB, a driver for the WS281x family of RGB chips, and a driver for the APA102 family of RGB chips.

Of the three, the WS281x driver was the most challenging, as it involved bitbanging the control signal with precise timings. Writing simple C or C++ code would not have met 
the timing constraints, thus necessitating the use of inline assembly. I found a few implementations online, but they were all designed to have a hardcoded pin number known at
compile time. Since I wanted my driver to have the flexibility to choose an arbitrary pin, or even run multiple strips simultaneously, I needed to relearn AVR assembly and modify
the code to use a non-constant value. Other than that, everything went pretty smoothly.

The APA102 driver was much easier to implement, since it uses a clock signal and thus does not rely on precise timings.

# Outcome

After about a week of working on this for a couple of hours every day, I've finally got a colorful battlestation.

![RGB workspace](/images/rgb-outcome.jpg)

The source code for this project is [available on Github](https://github.com/tmick0/arduino-rgbctrl).
