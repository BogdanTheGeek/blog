+++
title = 'MicroPPPID'
date = '2025-05-15T17:24:53+01:00'
author = 'Bogdan Ionescu'
description = 'MicroPython Programable PID for Temperature Control'
tags = ['programming', 'pottery', 'micropython', 'embedded']
cover = '/images/micropppid_home.png'
+++

# TLDR;
You can find the project repository [here](https://github.com/BogdanTheGeek/MicroPPPID).

# Motivation
I recently took up pottery as my [new favourite hobby](/thoughts/pottery-is-great) and I needed a better way to control the firing of my pots.

For my first few pots, I just re-purposed my *very* cheap metal casting forge. I built this out of eight 1" thick fire bricks, some fire cement, an electric stove top "burner" and a cheap PID controller.

![old kiln](/images/old_kiln.jpeg)

The cheap PID controller was *horrible*. Firstly, it could only go up to 999C, which meant that I could really glaze anything.
The lack of programmable temperature curves (aka, reflow profiles) also meant that I had to baby the kiln for the couple of hours or so that it took to fire a pot.

The bigger problem was that when I actually tried to reach 999C, it overheated the coils and burned them up.
After that *fun* experience, I changed to kiln rated coils, but at that point, I didn't trust this controller, so I started looking for something better.

# Build or Buy?

I did look an off the shelf solution, but I wasn't very impressed. Anything that looked half decent was >£100 and still lacked some of the features I wanted.

I also had a look at existing projects like [PyKiln](https://github.com/RinthLabs/PyKiln) and [PIDKiln](https://github.com/Saur0o0n/PIDKiln), but after 2 hours of messing with different versions of Arduino IDE, library versions and node.js dependencies, I gave up. I am sure some of these projects have worked perfectly for some people, but they were way too fragile for me.
I didn't want to waste any more time fixing someone else's code.

This is where most writers would insert some foreshadowing for comedic effect. Something like:
> I could write something better in a day! How hard can it be?

[Insert picture of Sponge Bob transition]

Well, in reality it wasn't very hard at all.

# Requirements
 - Web UI: Why fiddle with buttons and small screens?
 - Programmable temperature curves
 - Maximum duty cycle limit: I don't want to blow up my coils again
 - Enduring: I want this code to still work in 10 years
 - Quick to develop: I want to get back to playing with mud.

# Design
Even though I spend most of my time programming in C, for this project I went with MicroPython.
I have used MicroPython[^1] for a lot of automation jigs and quick proofs of concept for many years now. I always carry an ESP32-S2 and a Raspberry Pi Pico in my bag as an EDC(Every Day Carry) microcontroller. All I need to have some code running is a text editor and a USB Port.

[^1]: I am using MicroPython and CircuitPython interchangeably here, both have their strengths.

Another reason why I knew I wanted to go with mpy was its incredible portability. I wanted people to not depend on the specific microcontroller I used. There is no reason why this project couldn't run on anything from an ESP8266 to a ESP32-P4 to a Raspberry Pi Pico W to whatever might come out in the next decade.

However, the best feature of MicroPython is that its just Python. I can run the same code on both the final microcontroller target as well as my machine while I develop everything. There is no waiting for compilation, flashing, or messing with Makefiles (even though I did add one for convenience).

All I had to do to enable this was wrap all of the MicroPython specific library imports(which were very few) in a `try` block. If the import fails, I just declare a stub interface.
```python
try:
    from machine import Pin
except ImportError:
    # For testing on a non-MicroPython environment
    class Pin:
        OUT = 0
        IN = 1
        val = 0

        def __init__(self, pin, mode):
            self.pin = pin
            pass

        def value(self, val=None):
            if val is None:
                return self.val
            else:
                self.val = val
                print(f"Pin {self.pin} set to {val}")

```

There are so many other benefits for using such a *loose* language. I can overload objects, cast add new properties to objects at runtime and de-serialise JSON very quickly.

# Dependencies

In order to make sure that this code will still work in a few years, all libraries used have been *vendored*. No git submodules, no external package managers or build steps.[^2]

The only external dependency I have is Plotly.js because its way too big, but I am looking at replacing it soon.

[^2]: I lied somewhat, there are some optional build steps.

# Cool Features
Development went so smoothly, that I ended up implementing a lot more cool features than I expected.

![home page](/images/micropppid_home.png)

I used JSON for the settings and temperature profiles to allow easy editing. The settings page and settings object in the code are built dynamically based on the structure of document, so it should be backwards and forwards compatible.

![settings](/images/micropppid_settings.png)

All pins are configurable at runtime!.[^3]

[^3]: It does require a power cycle for now.

I used WebSockets to send sensor data, controller state and well as commands to the controller. This also allowed me to send all the internal logs to the application in real time.

Over the air updates, as well as changing any other files on the controller is done with a simple drag and drop.

Speaking of dragging, the temperature profiles are also edited by just dragging the rows around, adding new ones, deleting some etc.

Did I mention that all of this is done in *pure* JavaScript and HTML?

I wish I could say that there are no build steps, but I wanted to keep size down, so all .py files are compiled to .mpy bytecode and all other assets(except for .json files) are minified and gzip-ed.

# Reflection
When I started using python, I hate it! Broken dependencies, half a dozen different packaging tools, no types, horrible performance etc.
I still can't say I like it, I would never choose to use it anywhere where performance or efficiency matters. But I have to admit, It did feel good to work on this project.

I can only imagine that this is how people felt when C came about. If you spend any significant amount of time writing assembly, you will immediately appreciate what C does for you.

Every time I wrote `jsonData.get('something', 23)'` I could visualise the half dozen lines of C required to do the same. Every time I had to read or write to a file, dozes of lines of C, not to mention all the faffing about with `partitions.csv` and `idf.py menuconfig`. It felt liberating, but I can't say it was all rainbows and sunshine, I do live in England after all.

A couple of things were a bit frustrating. The fact that I can't easily create static objects (singletons) without some voodoo overloading of `__new__` was quite annoying. In the same vane, having to pass around classes and callbacks around just to access certain functions was way less convenient that just having some global state in C.

Lastly, I'm not sure how I feel about `asyncio`. It's an interesting pattern, but I feel like it's not as robust or flexible as pre-emptive multitasking. On a few occasions I managed to block the event loop.

Overall, it was a fun project and I am glad I can now get back to my pottery.
