---
title: The State of the Amiga 500 in 2018
description: "(Hint: It's still the best home computer ever made)"
date: 2018-10-18 14:46:30
updated: 2019-02-03 11:17:26
tags:
    - Commodore
    - Hardware
    - Music
    - Demoscene
---

![Image](IMG_20180805_212408568.jpg)

It's here! The machine I fell in love with as a teenager, though it was horribly misunderstood here in the United States. I found this clean, recapped Amiga 500 for the single purpose of installing a [Vampire accelerator](https://www.apollo-accelerators.com) and turning it into a [music tracking and sampling beast](https://en.wikipedia.org/wiki/OctaMED). If you aren't familiar with the Vampire line of Amiga accelerators, they are based on the _Apollo Core_, which in turn is based on a CPU that is code-compatible with the Motorola 68k CPUs available in the 1990s, but roughly three to four times faster. The added benefits include jacked-up RAM and video capabilities as well (though it's possible my tracking software won't be able to take advantage of next-gen video). The [M68k](https://en.wikipedia.org/wiki/Motorola_68000_series) holds a special place in my heart, independent of its position as the Amiga 500 CPU, as it was the architecture that I first deep-dove into during my university days.

<!-- markdownlint-disable no-space-in-emphasis -->
Presuming not all my Amiga software can take advantage of next-gen video and thus, the Vampire accelerator's HDMI connector, one of the first problems out of the gate is that this is not an IBM PC, and VGA was not a cross-architecture standard between device manufacturers in the days of the Amiga 500. The Amiga 500 was designed with a proprietary RGB D-Sub connector to an Amiga video monitor. It took me long enough to find a {% post_link witness-the-c64-mssiah 'C64 video monitor in working condition' %}, so I wasn't _particularly_ interested in tracking down a working one for the Amiga.
<!-- markdownlint-enable no-space-in-emphasis -->

Fortunately, many schematics exist for wiring the VGA pins to a DB23 connector. I used [this one](https://www.ikod.se/rgb-to-vga), though I couldn't locate a DB23 connector. I ended up purchasing a DB25 and using a Dremel to slice off the extra two pins for a snug fit in the Amiga's RGB port.

The pins were wired correctly, but then it was a matter of finding a VGA monitor that still supports 15 kHz analog RGB video signals. This is a video mode that is extinct from all modern computer monitors, and was already nearly extinct by the time flat panel (LCD) monitors became common. One option was to locate a CRT that supported it. The great irony is, from a price perspective, I found compatible CRTs to be _more expensive_ than the compatible LCD that I chose, due to the recent high-demand for CRTs to recreate a retro-gaming experience (I've probably junked a comfortable retirement's worth of CRTs in my lifetime).

![Image](IMG_20181018_121504100.jpg)

[This wiki](http://15khz.wikidot.com) helped me choose the NEC Multisync LCD1970NX. A flat panel was important due to space considerations in the eventual home studio build-out plan.

A MIDI connector, so that I could remotely start and stop the Amiga's tracker in sync with Logic Pro X (on a separate MacBook Pro), was far less of a challenge to locate as plenty of parallel port-compatible DB25 MIDI I/O connectors are available on eBay or in various Amiga enthusiasts' online shops. The same goes for a DB15 to USB connector for a modern mouse (I'm not exactly in love with the Amiga "tank mouse"). Be warned though, I had a lot of trouble with a wireless mouse behaving as expected. I ended up using a USB wired mouse with laser tracking.

![Image](IMG_20190202_201324_559.jpg)

_Update 2019-02-03_: The Vampire accelerator arrived! I plan to post a step-by-step of my experience installing it, a more "modern" version of AmigaOS, and wire up my tracker software to the rest of the studio.
