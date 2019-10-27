---
layout: post
status: publish
published: true
title: The fruits of some recent Arduino mischief.
author: Travis Mick
date: 2016-12-21
---
I recently consulted on a project involving embedded devices. Like most early-stage embedded endeavors, it currently consists of an Arduino and a bunch of off-the-shelf peripherals. During the project, I developed two small libraries (unrelated to the main focus of the project) which I'm open-sourcing today.

<!-- more -->

## Library 1: A generic median filter implementation.

The first library I'm releasing is a simple, generic median filter. Median filters are useful for smoothing noise out of data, but unfortunately I couldn't find a good existing library and was forced to write my own. Because I had to apply median filters to several types of data, I implemented it as a template class.

Note that while I developed the library for the Arduino use case, it is actually just plain ol' C++ and will work literally anywhere.

Github link: [https://github.com/tmick0/generic_median](https://github.com/tmick0/generic_median)

## Library 2: A simple driver for a poorly-documented Chinese RFID module.

Next up, we have a barebones driver for DFRobot's ID01 UHF RFID reader. The sample code released by the vendor is a joke, and the manual is even worse. To put the module to use, I had to reverse-engineer the frame format for tag IDs. I don't think anyone else should ever have to go through this, so I'm publishing the library here.

Github link: [https://github.com/tmick0/dfrobot_rfid](https://github.com/tmick0/dfrobot_rfid)

Refer to the READMEs on the Github repos for more information about both libraries.

