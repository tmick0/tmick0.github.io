---
layout: post
status: publish
published: true
title: A graphical tool for configuring Alesis V-Series MIDI controllers on Linux.
author: Travis Mick
date: 2017-11-12
image: /images/Screenshot-from-2017-11-12-13-47-14.png
---
In my [last post]({% post_url 2017-09-08-reverse-engineering-sysex %}), I explained how I reverse-engineered the Alesis SysEx protocol and detailed my findings. Now, two months later, I've finally decided to implement a tool for configuring these controllers on Linux.

![screenshot of the application](/images/Screenshot-from-2017-11-12-13-47-14.png)

<!-- more -->

The entirety of this work was done over the past two days, so it likely contains some hidden bugs, but as far as I can tell it is entirely usable.

Unfortunately, this is my first attempt at writing any kind of GUI, so it's not beautiful. But hey, it works.

You can [find the project on GitHub](https://github.com/tmick0/alesisvsysex). Contributions are welcome.
