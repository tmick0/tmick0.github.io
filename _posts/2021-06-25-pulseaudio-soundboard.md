---
layout: post
status: publish
published: true
title: Pulseaudio configuration for soundboard routing
author: Travis Mick
date: 2021-06-25
---

Pulseaudio has been the default audio engine in Ubuntu for several years now, and
fortunately it has a very flexible module system that lets you route and mix audio.
Today, we'll be looking at how we can set it up to mix mic and soundboard
inputs into a virtual source.

<!-- more -->

I recently wrote a [simple soundboard app](https://github.com/tmick0/pymidisoundboard)
in Python that uses a MIDI controller to trigger clips, and relied on the fact that
gstreamer would default to Pulseaudio output to later allow me to configure routing
and mixing without having to do anything specific in the code.

The main system-wide Pulseaudio configuration lives in `/etc/pulse/default.pa`; since
I'm the only user on my system I decided to just throw everything in there, but
[it is also possible to set this up per-user](http://manpages.ubuntu.com/manpages/bionic/man5/default.pa.5.html).

The relevant lines in my `default.pa` are:

```
load-module module-null-sink sink_name=soundboard_mix sink_properties=device.description=SoundboardMix
load-module module-combine-sink sink_name=soundboard_router slaves=alsa_output.pci-0000_00_1f.3.analog-stereo,soundboard_mix sink_properties=device.description="SoundboardRouter"
load-module module-loopback sink=soundboard_mix source=mic_ec
```

Respectively, these lines will

1. Create a virtual output called `soundboard_mix` where we will combine the soundboard and mic sources together; Pulseaudio will also automatically create a monitor called `soundboard_mix.monitor` that we can use as a source.
2. Create another virtual output called `soundboard_router` that we can use to route the soundboard to both our main output (in my case, `alsa_output.pci-0000_00_1f.3.analog-stereo`) as well as `soundboard_mix`.
3. Route our mic (in my case, `mic_ec`, since I have an echo cancellation module configured) into the `soundboard_mix`.

To find the names for the mic source and main output, use `pactl list sources` and `pactl list sinks`, respectively.

Now, all we have to do is make sure the soundboard application and whatever app in which we want to use our audio source are passed environment variables to choose the correct source and sink. Pulseaudio respects
variables `PULSE_SOURCE` and `PULSE_SINK`, so in my case I can launch my soundboard app with `PULSE_SINK=soundboard_router` and launch the app in which I want to use it with `PULSE_SOURCE=soundboard_mix.monitor`
and everything will function as desired.

As an added bonus, here's the line I'm using to create my echo-cancelled mic source:

```
load-module module-echo-cancel source_name="mic_ec" use_master_format=1 aec_method="webrtc" source_master="alsa_input.pci-0000_00_1f.3.analog-stereo" sink_master="alsa_output.pci-0000_00_1f.3.analog-stereo" aec_args="analog_gain_control=0 digital_gain_control=0 mobile=1 routing_mode=quiet-earpiece-or-headset comfort_noise=0 intelligibility_enhancer=1 high_pass_filter=0"
```

Change `alsa_input.pci-0000_00_1f.3.analog-stereo` to the correct input as listed by `pactl list sources`. I have a lot of extra options supplied here, which
are [documented nicely here](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Modules/#module-echo-cancel).

Also, feel free to check out [my soundboard app](https://github.com/tmick0/pymidisoundboard) if you're interested in something very simple to trigger clips.
It features a pyqt5 GUI for configuration and automatic MIDI note detection for setting triggers.

![soundboard](/images/soundboard.png)
