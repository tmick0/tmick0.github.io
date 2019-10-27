---
layout: post
status: publish
published: true
title: RIOT OS ported to TI Tiva C Connected Launchpad
author: Travis Mick
date: 2015-01-31
---

My current project is porting [RIOT OS](https://github.com/RIOT-OS/RIOT) to the [EK-TM4C1294XL](http://www.ti.com/tool/ek-tm4c1294xl) evaluation board. RIOT is an embedded operating system aimed at the Internet of Things, developed primarily by Free University of Berlin. The EK-TM4C1294XL is a pretty powerful board, featuring an ARM Cortex M4 MCU and built-in Ethernet MAC. So far, I have implemented only the most basic support for the CPU - just timers and UART. However, I'm currently working on the Ethernet drivers (almost done) and my next focus will be drivers for an XBee add-in.

<!-- more -->

This port of RIOT (and the boards) will be deployed in a Smart Grid project being worked on by myself and the others in the Network Systems and Optimization Lab at NMSU.

The port is forked from the 2014.12 release of RIOT.

My repository is here: [https://github.com/nsol-nmsu/RIOT](https://github.com/nsol-nmsu/RIOT)

