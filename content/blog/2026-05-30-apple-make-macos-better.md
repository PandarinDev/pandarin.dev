---
title: How Apple could macOS instantly better with these accesibility features
description: A list of actionable items for Apple on how they could make macOS a better operating system if they wanted to.
date: 2026-05-30
tags: macos, apple
permalink: how-apple-could-make-macos-better/
draft: true
---
# Preamble
I had my trusty old Thinkpad T15 replaced at work with an M5 Macbook Pro recently and I have some thoughts.
First of all, the hardware truly is excellent - nothing else quite like it on the market. You get desktop class performance CPU and GPU in a portable form factor with long battery life, a beautifully CNC'd unibody chassis, great display and speakers, Thunderbolt 5 ports with 120Gbps bandwidth and all.

On another note however, in true Apple fashion the software is the achilles heel of the device. There are so many small features that it would literally take them a couple of days to develop and test and would improve quality of life so much on macOS. Worst of it all, almost all of these missing features are related to accessibility, yet [Apple praises themselves](https://www.apple.com/accessibility/) on how accessible all of their devices are.

So I got thinking, why not write a list of these features? Not just any list mind you, listing out flaws and critiquing is easy. I want to write a list with actionable items that would be good enough to be ticket descriptions for each and every one of the problems that I will list.

Furthermore, I will link existing tools as reference that already implement some of these features just to prove that they can be implemented without any architectural changes to the OS. Many people point to these tools and say "just install X or Y to fix this" to which I say a firm "no". These should be built-in OS features, you should not have to download and install half a dozen third-party tools that hijack your input devices just to make the OS usable.

# Action Items
## 1. Mouse handling
As long as you use macOS with a touchpad everything is great. However, the moment you connect a mouse to your device the experience starts to fall apart.

### 1.1 Separate scrolling direction setting for touchpad vs. mouse
The very first thing that you will encounter is that there is no way to separate the scrolling direction of your touchpad vs. your mouse wheel. This means that if you prefer natural scrolling on touchpad (which is the sane default where you drag your fingers from bottom to top to scroll the page down) you will have to scroll your mouse wheel up to scroll the page down (which is NOT the default behavior on almost any platform).

The operating system lets you change the scrolling direction - however, you only get to choose a universal scrolling direction that applies to both your touch inputs and your scrollwheel inputs. This means that you get to choose if your touchpad use or your mousewheel use will be very awkward - you don't get to choose your ideal setup for them separately.

This is madness. This is available as a basic setting separately for touch vs. scrollwheel on virtually all other operating systems. There are at least a dozen utilities that fix this on macOS - the most popular one is probably [LinearMouse](https://github.com/linearmouse/linearmouse) or [Scroll-Reverser](https://github.com/pilotmoon/scroll-reverser), but [Karabiner-Elements](https://github.com/pqrs-org/Karabiner-Elements) can also do this.

**Solution:** macOS already has a "Mouse & Trackpad" screen in the System Settings application with separate "Trackpad Options" and "Mouse Options" submenus. Simply add scroll direction comboboxes there where the user can decide if they want natural or reversed scroll direction for the selected input type. Bonus points if you can set these settings per device (where the implementation would store the scroll direction for the device per HID vendor+product ID).

### 1.2 Mouse side button support
Most mice have side buttons that typically serve as "back" and "forward" buttons. They are very handy and enable much faster operation of programs such as file- or internet browsers. For whatever reason macOS does not support this natively however, despite most of the built-in programs having "back" and "forward" or "undo" and "redo" operations.

There are a bunch of different tools that implement this feature, I can once again point to [LinearMouse](https://github.com/linearmouse/linearmouse)'s "Enable universal back and forward" feature or to [SaneSideButtons](https://github.com/thealpa/SaneSideButtons).

**Solution**: macOS already has "back" and "forward" gestures implemented for touchpad - in fact, that is what the mentioned third-party devices are simulating when pressing side buttons. Simply add a system setting in the "Mouse" screen to enable the side button back and forward functionality.

## 2. Display support
Display support on macOS is weird. Everything is optimized for integer multipliers of 110 PPI (pixels per inch) displays. Apple themselves are targeting 220 PPI for their own displays, which is what they are calling "retina" - basically this is a good threshold where individual pixels are no longer visible for the naked eye, so everything is super sharp. But this requirement is not easy to fulfill and even in 2026 there are not many monitors with such high PPI, especially in your run of the mill office environment.

The problem here is the proper support for fractional scaling. Most modern monitors fall into this awkward range of 120-160 PPI where if you are below 110 PPI every UI element will be larger than intended and if you are above 110 PPI but not quite reaching 220 PPI all the display scaling options available natively to you will make UI elements either smaller or larger than intended depending on the scaling that you choose.

Here are a few common monitor size and resolution examples just to highlight what I mean:
- 1080p at 24": 92 PPI
    - UI elements will be larger than intended, eating up more of the already not too generious screen real-estate
- 1440p at 27": 109 PPI
    - Ideal choice, this is what I would recommend if you are buying non-Apple monitors
- 4K at 27": 163 PPI, at 32": 138 PPI
    - UI elements will be either smaller or larger than intended, depending on your scaling option of choice

At 27 inches this means that the display has to have a 5K resolution (so 5120*2880 pixels) in order to reach this magic PPI threshold.