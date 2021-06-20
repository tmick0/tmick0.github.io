---
layout: post
status: publish
published: true
title: Adapting a smart doorbell to an incompatible digital chime
author: Travis Mick
date: 2021-06-19
---
For a few months, I've been using an [Ubiquiti G4 Doorbell](https://store.ui.com/products/uvc-g4-doorbell),
but recently chose to upgrade my mechanical chimebox to a digital one. Though The Ubiquiti store lists the
[NuTone LA600WH](https://www.broan-nutone.com/en-us/product/doorbells/la600wh) as compatible, it turns out
there are some caveats which I managed to overcome by conjuring up an adapter from parts I had lying around.
This technique can likely be adapted to work with other doorbells or chimeboxes that don't play nice with each
other, and avoids spending money on a commercially-available adapter.

<!-- more -->

The main issue when using a smart doorbell with a digital chimebox seems to be that both want to
continuously draw power. A normal "dumb" doorbell will simply close a circuit to trigger a mechanical
chime. In the case of my digital chime, it required a diode in order to continuously receive power when
the button isn't being pressed. However, my smart doorbell also requires a special hack to receive power --
an adapter that Ubiquiti provides which sits between the doorbell and the chimebox, and triggers the chimebox
when the button is pressed. Unfortunately, these two hacks are inherently incompatible with one another.
Based on some searches, it seems that this is the case for a lot of doorbells and digital chimes.

There were two "easy" ways to get the doorbell working, in the case of the LA600WH. Since it supports completely
wireless operation, there is an option to install three D-cell batteries, however who wants to change batteries
when a wired installation is possible? The second hack would have been to configure a long enough ring duration
in the Unifi Protect app for the chime to respond; in this case, there was unfortunately a long delay that seemed
to be due to the chime "booting up."

It was at this point that I consulted Google: surely, someone has run into the same problem as me, right?
Well, yes. I managed to find [a how-to post in the Ubiquiti community](https://community.ui.com/questions/Successful-installation-of-G4-Doorbell-with-digital-chime/a02cba33-8899-4e53-8b26-dfdd71230bde)
explaining how to get the G4 to play nice with a digital chime. Unfortunately, it required [something called the VDBI](https://bcsideas.com/products/vdbi-video-door-button-interface)
to serve as an adapter. Not wanting to spend an additional $20 or wait for shipping, I turned to my parts shelf.

After searching a few bins and finding a relay and some diodes, it occurred to me that I could easily build
something that would allow the G4 to pretend to be a regular, mechanical doorbell. Doorbell circuits use low-voltage AC
(in my case, 18V); the fact that my chimebox came with a diode led me to believe that it wants half-wave power in order
to stay powered on but idle. Then, full wave power should cause it to ring. The adapter provided with the G4 simply seemed
to provide power to the doorbell continuously, while also closing the circuit intended to ring the chime when the button is
pressed. If I could exploit the "output" of the Ubiquiti adapter to trigger a relay (or a triac, probably, but I didn't have
one) that provides full-wave power, and use a diode to provide half-wave power continuously (as originally intended by the
manufacturer), the chimebox should work as it was meant to. Unfortunately, relays can typically only be triggered by DC (the
inductive properties of the internal coil don't play nice with AC) but this could be remedied with a bridge rectifier. An
integrated one would work, but I didn't have one, so I built one with discrete diodes. I considered omitting the relay and
just feeding whatever comes out of the Ubiquiti adapter directly to the chime, but not being sure what else is going on inside
there I didn't want to backfeed it from the constant supply.

The circuit I came up with looks like this:

![schematic](/images/doorbell-adapter-circuit.png)

The AC provided by the Ubiquiti adapter when the doorbell is pressed will be rectified in order to trigger the relay, causing
full-wave AC to be provided to the chime. When it isn't being triggered, the 1N4002 diode that was provided with the chime will
provide it half-wave current, allowing it to stay powered-on but idle. As I mentioned, I ended up building the rectifier out
of discrete diodes; for what it's worth, I used 1N4007s. It's important that you use a relay that will tolerate the voltage coming
out of your doorbell transformer; in the case of 18VAC fully rectified, this would be about 16.2VDC. In some cases, you may also 
need to add a smoothing capacitor; if the relay isn't staying latched as long as you would hope, try adding one. You might also
be able to get away with a half-bridge rectifier, but I didn't bother.

I wasn't originally intending to do this write-up, so unfortunately I didn't take any pictures prior the installation,
but what I built ended up looking like this (after being wrapped in Kapton tape).

![adapter](/images/doorbell-adapter-build.jpg)

Yes, I know that's a 5V relay, and I just told you not to do that. This is what I had around, but it hasn't caught fire yet and
I'll replace it eventually.

Hooked into the Ubiquiti adapter and the chime, everything ends up looking like this:

![schematic](/images/doorbell-adapter-full-schematic.png)

It even all fits nicely into the battery compartment of the chime:

![installation](/images/doorbell-adapter-install.jpg)

As mentioned in previous posts, I'm certainly not an electrical engineer, but fortunately I was able to summon enough
ability to solve this problem. Hopefully my solution can benefit others as well.
