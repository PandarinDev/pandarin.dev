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

The problem here is that if you are running a monitor that has lower than 110 PPI or higher than 110 PPI but lower than 220 PI you will have all UI elements either be smaller or larger than intended, depending on which scaling option you select in display settings. And there are not that many scaling options to choose from to begin with.

There is also another problem with Apple's display handling handling however, that is arguably even worse, so let's start with that one.

### 2.1 Add control over temporal dithering and PWM
This is not very well documented but Apple uses temporal dithering (FRC) to emulate support for wider color spaces. While this is usually not visible on high pixel density screens, the moment you hook up your Mac to a lower resolution external screen and look at anything dark the effect immediately becomes visible. To add insult to injury, since Macs output 10 bit color most panels will apply their own dithering on top resulting in double dithering which can be very annoying depending on your monitor and your sensitivity to this effect.

Dithering can be disabled using [Stillcolor](https://github.com/aiaf/Stillcolor) or [BetterDisplay](https://github.com/waydabber/BetterDisplay).

In addition PWM is also present on some of Apple's own displays to control backlight. PWM, if you are not familiar with it, is a technique where the display backlight gets turned on and off rapidly at a high frequency to emulate lower brightness levels. The problem with this is that there are many people who are sentitive to this effect and they get headaches and eye strain just by looking at a display that uses PWM for extended periods.

Since PWM is typically only used for lower brightness levels tools such as [BetterDisplay](https://github.com/waydabber/BetterDisplay) can be used to enforce minimum brightness levels of the display to avoid this issue. Another alternative of course is to just always leave your display at a higher brightness level and disable auto-brightness, but that unnecessarily eats into your device's battery life.

**Solution:** Add explicit control over both dithering and minimum display brightness levels. Bonus points if the system auto-detects its own type of Apple panel used and based on that it automatically figures out the minimum brightness which below it should never go - in that case the option could be also just to "disable PWM" and the system figures out the brightness levels automatically.

### 2.2 Add better fractional scaling support
As eluded to in the beginning of the display section Apple targets this magical number of integer multipliers of 110 PPI and offers you to choose from a few different scaling options. However, if the scaling option that you choose is not an integer multiplier you will run into fractional scaling, which is implemented on macOS in a bit of a weird way. Basically if you decide to do fractional scaling Apple will render into a framebuffer that has the resolution of the next integer multiplier of the 110 PPI target (e.g. will render everything in 5K) and then downscale the resulting framebuffer back to your native resolution (e.g. 4K). This is basically a supersampling technique which is a weird thing to do for fractional scaling of UI elements.

The problem with this approach is two-fold:
- It wastes GPU power by rendering things at an unnecessarily high resolution.
- It introduces sampling artifacts when scaling the framebuffer back to the lower target resolution (typically some amount of blurriness)

**Solution:** Now this is the first point where I'll admit there is no easy fix for this. Almost every platform is plagued by fractional scaling issues (Windows has its own share of blurry scaled UI elements) - but some platforms such as recent KDE version on Linux are really good at this. I'm just sure it would take significant amounts of work to make progress on this, and its easier to lean back and say "just by one of our 5K displays" - but then I don't think you get to claim that your platform is super accessible.

## 3. Better keyboard customization
If you are using a popular keyboard layout such as US or UK, good for you. If you are not, guess what? Apple will make your life unnecessarily difficult. But even if you are fortunate enough to be using e.g. the US keyboard layout it is still not trivial to switch between Mac and Windows. Command is placed where normally "Alt" would be and for most of the things you need to press Command instead of Control. If you want to type any special character you'll have to press Option, except Option is not on the same location as AltGr on a regular keyboard. And so, and so on.

Apple already has built-in solutions for some of these problems, but they are good enough and would need so little extra work to be great.

### 3.1 More granular modifier key modification
In the System Settings it is already possible to remap the special keys to other special keys. This is great - it allows me to change the Globe key to act as Control and now I have Control on the bottom left of my keyboard, where it would normally be anyways. I can remap Command to Option and now I have Option at the exact same location where AltGr would be. Hey, wait a minute, now I also swapped Option and Command on the left side of my keyboard as well - that's not what I want.

Now something like [Karabiner-Elements](https://github.com/pqrs-org/Karabiner-Elements) can already do this for you, but there is even a built-in CLI tool called `hidutil` that can do exactly this. For example to swap the right Option and Command keys (and only those, not the ones on the left side) you can do:
```
hidutil property --set '{"UserKeyMapping":[{"HIDKeyboardModifierMappingSrc":0x7000000E6,"HIDKeyboardModifierMappingDst":0x7000000E7},{"HIDKeyboardModifierMappingSrc":0x7000000E7,"HIDKeyboardModifierMappingDst":0x7000000E6}]}'
```
and this will persist until the next reboot. You could just save this to a script and run it everytime your system boots, but why can't we have this as part of the UI?!

**Solution:** Inside the existing modifier key remapping window allow separate remapping for left and right Options and Command keys.

### 3.2 Better custom keyboard layout support
Some of the more niche keyboard layouts have really, really bad support on macOS. I am Hungarian and I have used a Hungarian keyboard layout all my life - both for typing out regular text and for programming -, so I will use that as an example, but everything I say here applies to custom keyboard layouts as a whole.

Programming is a great example why Apple's more niche keyboard layouts are bad. Hungarian keyboard layout requires keyboard modifiers (AltGr on Windows, Option on Mac) for almost all of the special characters such as `@`, `$`, `&`, `|`, `/`, `\`, `<`, `>`, `;` and more. All of these special characters are very frequently used in programming.

There are multiple problems around this:
- Apple's own Hungarian keyboard layout places all of these special characters in different locations than the standard Hungarian keyboard layout
- Apple does not show the location of special characters on their keyboard

Now problem #1 is bad enough already, as it means that if you are switching between PC and Mac your muscle memory is completely ruined, but problem #2 is even worse I'd argue since now if your password contains special characters good look finding them on your keyboard unless you can touch type Apple's own weird layout - special characters are not shown anywhere on the keyboard itself.

All of these issues can be solved by installing custom keyboard layouts, however, support for this is not as good as it could be. First of all why do I need to download modified keyboard layouts such as [Hungarian_Win](https://github.com/zaki/mac-hun-keyboard) in the first place to get back the normal Hungarian keyboard layout? Why is this not built into the OS or at least why don't we have a built-in way to remap keys?

Second, when you install a custom keyboard layout you are not allowed to have it as your only input source. You have to have at least one built-in keyboard layout in your input sources, the settings simply will not let you have the custom layout as your only option. I understand why this is the case but it would be much more elegant if the system instead shown a warning dialog to the user that having only a custom keyboard layout set might leave you unable to type certain characters.

**Solution:** Have alternative built-in keyboard layout for more niche languages such as Hungarian where the special characters are on the same location as on regular keyboards. Also allow the selection of only custom keyboard layouts system wide. Bonus points if there is a built-in way to edit keyboard layouts as currently this is only possible to do (at least visually) through third-party tools such as [Ukelele](https://github.com/sillsdev/Ukelele).

# Conclusion
And with that it is a wrap. I could list a bunch more things that irk me about macOS (such as the lack of persistent mapping of network drives) but these are all the complaints I have at the moment about accessibility. As you can see all of these are possible to fix using third-party tools already and most of them are pretty simple feature requests too.

Apple could really implement any (or even all) of these in a single sprint if they wanted to. The question is do they want to? Do they care? I'm not sure to be honest, but I'm hopeful.