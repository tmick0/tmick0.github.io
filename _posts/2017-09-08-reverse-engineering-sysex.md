---
layout: post
status: publish
published: true
title: Reverse engineering the Alesis V-series SysEx protocol.
author: Travis Mick
date: 2017-09-08
image: /images/Screenshot-from-2017-09-08-22-30-10.png
---
I recently got back into music production and decided to order myself a MIDI controller. I got a few recommendations for the the [Alesis V25](https://www.amazon.com/Alesis-V25-Keyboard-Controller-Buttons/dp/B00IWWBSD6/?th=1), so I went ahead and ordered it. However, I was less than pleased to find that its configuration software wouldn't run on Linux, even under Wine. Of course, this prompted me to reverse engineer the protocol that lets the software talk to the keyboard.

<!-- more -->

## Overview of MIDI

Musical Instrument Digital Interface (MIDI) is a long-lived standard for interactions both among musical instruments and between instruments and software. The Alesis V25 is a MIDI controller, which is essentially just a keyboard that lets you send notes to a Digital Audio Workstation (DAW) such as [Bitwig](https://www.bitwig.com/) (which is just my personal favorite).

MIDI messages are always three bytes, and can look something like this (in hexadecimal):

```
09 30 6a
```

The first byte is called the "status byte," and the second two are simply called "data bytes."

The meaning of the particular bytes in the above message are as follows:

| `09` | Channel 0, note on |
| `30` | Note 48 (C#4)      |
| `6a` | Velocity 106       |

In addition to note messages, there are also Control Change (CC) messages which indicate that a knob was turned or a button was pressed. However, these standard messages aren't going to be very important for the adventure we're going on today. Reconfiguring MIDI devices uses an extended protocol called SysEx.

## Introduction to SysEx

System Exclusive (SysEx) is an extension to MIDI that lets instrument manufacturers define custom commands for their devices. The specification is quite simple: it only requires that SysEx messages start with a two byte header, and end with a one-byte trailer:

```
f0 xx ... f7
```

Here, `f0` is the start byte, `xx` is a placeholder for a manufacturer ID, and `f7` is the end byte. The bytes in between can be anything, as long as their high bits aren't set (i.e., they are less than or equal to `7f`). Their meaning is interpreted on a manufacturer-by-manufacturer basis.

## Overview of the Alesis V25 Editor

Alesis' editor lets you do things such as reassign the control numbers on the keyboard's knobs and buttons. It doesn't do much -- it mostly just supports reading the configuration from the keyboard, editing it, and writing it back. The interface looks something like this:

![screenshot of the official alesis configuration application](/images/Screenshot-from-2017-09-08-22-14-58.png)

Since the software doesn't work on Linux, I had it running in a virtual Windows machine under Virtualbox, with USB forwarding enabled for the device.

## Intercepting SysEx Messages

We want to see what this software is doing in the backend to update the MIDI device's configuration. To do that, we'll need to snoop on the SysEx messages that it exchanges with the device.

It turns out that the popular network-capture software [Wireshark](https://www.wireshark.org) supports snooping on USB devices. That's exactly what we'll need to do today.

First, I had to enable the `usbmon` kernel module:

```
sudo modprobe usbmon
```

Then, I started up Wireshark and started listening for packets. The display was initially quite noisy due to other USB devices (mostly my mouse). I found the bus and device IDs via `lsusb` and filtered on those.

![screenshot of captured communication in wireshark](/images/Screenshot-from-2017-09-08-22-26-48.png)

In this particular capture I've simply queried for the controller's current configuration and received a reply. The 80-byte packet is the query, and the 128-byte packets are chunks of the reply. The 64-byte messages in between appear to be acknowledgements occurring at the USB protocol level.

We can dissect one of the packets to see what's going on:

![dissected sysex packet](/images/Screenshot-from-2017-09-08-22-30-10.png)

We can see that the SysEx message has been divided up into 3-byte segments in accordance with the traditional MIDI specification. At the USB level, each 3-byte segment is given a header to indicate which device it is from and how many of the SysEx bytes are meaningful. The details of that aren't important, but the result is that we are able to pull out the actual SysEx message for the query:

```
f0 00 00 0e 00 41 62 00 5d f7
```

We can identify the SysEx start and end bytes, and conveniently the manufacturer ID is `00`. A bit of research has driven me toward the conclusion that because Alesis hasn't officially registered a MIDI manufacturer ID, it uses 00 with a two-byte extension of `00 0e`. I haven't deciphered all of this message, however have been able to determine that the byte `62` is the important part; it indicates that this packet is a query for the current configuration.

## Decoding the Reply

To determine which part of the SysEx messages do what, I had to play around with the GUI for a while, changing options to see how they affected the data going over the wire. This was a tedious process, however in the end it only took about three hours.

The reply is quite long, and is split over several USB packets, however I've copied and annotated the final SysEx message here:

<table>
<thead>
<tr>
    <th>Raw bytes</th>
    <th>Section</th>
    <th>Interpretation</th>
</tr>
</thead>
<tbody>
<tr>
    <td><code>f0</code></td>
    <td>SysEx start byte</td>
    <td></td>
</tr>
<tr>
    <td><code>00 00 0e</code></td>
    <td>Alesis manufacturer ID</td>
    <td></td>
</tr>
<tr>
    <td><code>00 41 63 00 5d</code></td>
    <td>Some sort of header</td>
    <td>The 3rd byte indicates message type:
        <ul>
            <li><code>61</code>: Set configuration</li>
            <li><code>62</code>: Query configuration</li>
            <li><code>63</code>: Current configuration (reply)</li>
        </ul>
    </td>
</tr>
<tr>
    <td><code>0c 02 00 00</code></td>
    <td>Keys configuration</td>
    <td>
        <ol>
            <li>Base note</li>
            <li>Current octave</li>
            <li>Channel</li>
            <li>Velocity curve</li>
        </ol>
    </td>
</tr>
<tr>
    <td><code>00</code></td>
    <td>Pitch wheel configuration</td>
    <td>
        <ol>
            <li>Channel</li>
        </ol>
    </td>
</tr>
<tr>
    <td><code>00 01 00 7f</code></td>
    <td>Mod wheel configuration</td>
    <td>
        <ol>
            <li>Channel</li>
            <li>CC number</li>
            <li>Minimum value</li>
            <li>Maximum value</li>
        </ol>
    </td>
    </tr>
<tr>
    <td><code>40 00 7f 00</code></td>
    <td>Sustain pedal configuration</td>
    <td>
        <ol>
            <li>CC number</li>
            <li>Minimum value</li>
            <li>Maximum value</li>
            <li>Channel</li>
        </ol>
    </td>
</tr>
<tr>
    <td>
        <code>00 14 00 7f 00</code><br/>
        <code> 00 15 00 7f 00</code><br/>
        <code> 00 16 00 7f 00</code><br/>
        <code> 00 17 00 7f 00</code>
    </td>
    <td>Knobs configuration</td>
    <td>
        <ol>
            <li>Operation mode
                <ul>
                    <li><code>00</code>: CC</li>
                    <li><code>01</code>: Aftertouch</li>
                </ul>
            </li>
            <li>CC number</li>
            <li>Minimum value</li>
            <li>Maximum value</li>
            <li>Channel</li>
        </ol>
    </td>
</tr>
<tr>
    <td>
        <code>00 31 00 00 09</code><br/>
        <code>00 20 00 00 09</code><br/>
        <code>00 2a 00 00 09</code><br/>
        <code>00 2e 00 00 09</code><br/>
        <code>00 24 00 00 09</code><br/>
        <code>00 25 00 00 09</code><br/>
        <code>00 26 00 00 09</code><br/>
        <code>00 27 00 00 09</code>
    </td>
    <td>Pads configuration</td>
    <td>
        <ol>
            <li>Operation mode
                <ul>
                    <li><code>00</code>: Note</li>
                    <li><code>01</code>: Toggle CC</li>
                    <li><code>02</code>: Momentary CC</li>
                </ul>
            </li>
            <li>Note / CC number</li>
            <li>Fixed (?) / Minimum CC value</li>
            <li>Velocity curve / Maximum CC value</li>
            <li>Channel</li>
        </ol>
    </td>
</tr>
<tr>
    <td>
        <code>00 30 7f 00 00</code><br/>
        <code>00 31 7f 00 00</code><br/>
        <code>00 32 7f 00 00</code><br/>
        <code>00 33 7f 00 00</code>
    </td>
    <td>Buttons configuration</td>
    <td>
        <ol>
            <li>Operation mode
                <ul>
                    <li><code>00</code>: Toggle</li>
                    <li><code>01</code>: Momentary</li>
                </ul>
            </li>
            <li>CC number</li>
            <li>On value</li>
            <li>Off value</li>
            <li>Channel</li>
        </ol>
    </td>
</tr>
<tr>
    <td><code>f7</code></td>
    <td>SysEx end byte</td>
    <td></td>
</tr>
</tbody>
</table>

Note that I've neglected exploring some of the options provided by the GUI, such as assigning the buttons to Program Change events. I also have no idea what the "Fixed" field does for the drum pads, as the word "Fixed" is the only description offered by the GUI. I suspect it enables fixed velocity, but I haven't bothered to check.

## Next Steps

Now that I've decoded the SysEx protocol, I plan to make a tool to enable editing the controller's configuration under Linux. I may go as far as writing a controller script to enable modifying it from within Bitwig. Check for updates here to see what's to come.

