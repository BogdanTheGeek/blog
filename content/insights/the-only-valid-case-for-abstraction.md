+++
title = 'Abstraction is for Artists'
date = '2025-06-13T10:49:30+01:00'
author = 'Bogdan Ionescu'
description = 'Why, How and Most Importantly When to Abstract.'
tags = ['programming']
cover = '/images/encapsulation.jpg'
draft = true
+++

# TLDR;
TBD

# Prelude
I'm not going to harp on C++ or OOP or make any claims about performance.
This is a purely practical discussion about how and when to add layers to your binary cake.

# The Promise of Simplicity
Humans have a finite context window in which we can operate.
To be productive in any capacity, we must split big problems into smaller ones and recursively decent into solving each one until there are no more problems to be solved.
This is the classic [divide and conquer](https://en.wikipedia.org/wiki/Divide-and-conquer_algorithm) algorithm.

Engineers are very good at this. Programmers are not.

When an Engineer[^1] is given a problem to solve, they write a list of all the sub-problems they can see immediately, and start approaching them one by one.
When a programmer is give a problem to solve, they write a list of all the sub-problems they can see immediately, can imagine possible, have heard about from someone on the [orange website](https://news.ycombinator.com/) and a couple more just for good measure, then they try to solve them all at once.

[^1]: I intentionally capitalise the word 'Engineer' to accentuate their importance.

I'm being hyperbolic, but it certainly feels that way sometimes.
Why does a UART API need to look like a POSIX socket?

Humans are notoriously bad at pattern matching. Well, we are actually very good at it, but also really lazy.

{{< figure src="/images/jesus-toast.jpg" alt="picture of Jesus on toast" title="This is not Jesus, this is toast." class="center" >}}

## Words are Hard
Abstraction is autological. The word itself is abstract.

Programming is all about abstraction. There is no such thing as variables, functions or pointers. Computers have no concept of them. All they know is that when they see some bytes, they should do something with them.

It's very easy to complain about bad abstractions, leaky abstractions, C++.
I want to talk about a special *kind* of abstraction, the useful kind, platform abstraction.

# The Most Useful Hammer, for the Perfect Nail
Testing code is hard, testing embedded code is harder.

I like to to write code *fast*. I'm not talking about typing speed, I'm talking about iteration cycle. I want to minimise the delay between making a change and seeing the effect.
Embedded toolkits are designed in juxtaposition to my goals. Most IDEs are slow, most makefiles are overly complex and most SDKs have too much bloat.

There are many ways to work around this:
 - run the code from RAM (avoids slow flashing)
 - incremental builds (linking is still slow)
 - don't use Eclipse

But the best way in my opinion is to not run the code on the device at all. Run it locally.

## MicroPython
Don't worry, we'll get back to real programming in C in just a sec, but I want to show a very useful technique for rapid prototyping.

I have used MicroPython for rapid prototyping for a few years now. It's so quick to just throw it on a microcontroller, connect it as a mass storage device, open the `main.py` file and go at it. No SDK, no IDE, no compilation.

All of those things are really nice on their own, but the real power comes when you are writing more complex apps and just copying the files over takes too long.

MicroPython is just Pyhton, so we can just run the code locally.
Anytime we need to do something MicroPython specific, we abstract it away, and throw it behind a try block. Here is an excerpt from a [relay](https://github.com/BogdanTheGeek/MicroPPPID/blob/main/relay.py) abstraction:
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
I have used this technique in my [MicroPPPID](https://github.com/BogdanTheGeek/MicroPPPID) Project, to rapidly develop a web based PID temperature controller for my pottery kiln. I will write about it at some point.

Importantly, this is done at the highest level you can. No point creating stubs for SPI, Bus, Pin, if you just need something that will look like the NeoPixel library.

Python is really good at this type of loose typing programming.

## The Real World
For real software development, we have to use C.

The problem at hand is that our embedded code depends on some low level functions to work.
Sure, we can write our libraries to not care about the implementation, but that can only take you so far.
At some point, you need to do something that interacts with the world.

Here are some of the hammers I have used to solved this problem.

### Function Pointers
If a piece of code needs to work on multiple platforms, and its main task is to interact with the world, defining an interface with some function pointers is one way to solve the problem.
```c
typedef struct
{
    int (*read)(void *buf, size_t size);
    void (*write)(void *buf, size_t size);
} Interface_t;
```
Now you can just populate the struct with the right function pointers and you are done.

### Weak Functions
Marking a function of your library with `__attribute__(weak)` will allow users to re-define in their own code. If they don't the linker will use the library implementation.
This has close to 0 overhead (you can't optimise the function call away), and it can work well for certain types of libraries.

### Macro Magic
If we need some platform specific code, we can just hide it behind a pre-processor macro.
```c
#if PLATFORM_ESP32
#include <driver/gpio.h>
#else
void gpio_set(int pin, bool enabled){ (void)pin; (void)enable; }
#endif
```

### Ports
When your code needs to run on a bunch of different machines, it might be useful to put all of the implementation specific code in a single file `port.c`.
Whenever you need to move the code to a new target, you just add a new `port.c` file that implements all the functions required by `port.h`.

```sh
platform
├── inc
│   └── port.j
├── esp32
│   └── port.c
└── renesas
    └── port.c
```

Projects like FreeRTOS and Micropython use this.
The only problem is that you might have not known that you wanted to support multiple platforms when you started the project.
It also means that there will be some compromise in the implementation. Not every microcontroller works the same way.
It can also be a pain to add new functionality, as you have to update every single port.

### Linking Errors
The linker is my favourite feature of C compilers.
As a novice, it has the scariest errors, but as you grow, you discover that they have a superpower.
If we don't compile the source of the platform specific code, but we still include the header file, we now get a nice list of all the function that we need to implement.

# Putting it All Together
My project structure usually looks like this:
```sh
project
├── firmware
├── tools
└── tests
```
The code destined to end up on the microcontroller lives in firmware.
Tools contains a bunch mostly utility scripts.
Tests is where all the magic happens.

All tests have their own folder. They look like this:
```sh
modbus
├── Makefile
├── modbus.c -> ../../firmware/lib/modbus/modbus.c
└── test.c
```

Of note is that in this example, `modbus.c` is just a symbolic link to the code in the firmware. That code is pure, there are no macros, function pointer, conditional compilation flags, nothing. That code was written with 1 intention only, to work on the platform it was destined to.

All header files are outside of each test and are just copies of the platform specific header files.
