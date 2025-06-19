+++
title = 'J-Link RTT for the Masses'
date = '2025-06-01T14:47:43+01:00'
author = 'Bogdan Ionescu'
description = 'How to do bidirectional logging over SWD on ARM.'
tags = ['programming', 'arm', 'tools', 'electronics']
cover = 'images/monty-python-peasants.jpg'
+++

# TLDR;
You can use semihosting on any ARM CPU to send and receive data in a few lines of code.

Click [here](https://github.com/BogdanTheGeek/minimal-py32/tree/main/src) to see an example implementation on a 10p "disposable" microcontroller.

# Context
There are many ways to debug embedded projects.
For high speed stuff, you might toggle a pin at the beginning and end of a subroutine.
On a more advanced project, you might have structured logging to a file-system.
In 99% of cases however, most people resort to either firing up a debugger or logging to a console (aka. `printf` debugging).

The first thing I do whenever I start working with a new microcontroller platform is get some sort of logs working. In the past, I tended to setup a UART peripheral and overload `stdio.h::write`[^1].

[^1]: If your SDK provider doesn't tag `write` with `__attribute__(weak)`, stop using them, they don't deserve your business. You can work around it using `-Wl,--wrap=write` linker flag.

Here is an example of what that looks like
```c
int _write(int fd, const char *buf, int size)
{
    (void)fd;
    UART_Transmit(buf, size);
    return size;
}
```

Fancier implementation may use a ring buffer, or have multiple destinations, but this is how it usually looks.

## Great, so what's the problem?
Well, for one, I don't want to waste a pin on my microcontroller.
It's also quite annoying to have to dig for a USB-UART adaptor.
There's also the issue of sending data to the microcontroller, It's quite nice to be able to paste some commands into the terminal to control the device while it's running.
This would not only necessitate another precious GPIO, but also a lot of faffing about with ring buffers, DMA, or interrupts. All just to do what [wozmon](https://www.sbprojects.net/projects/apple1/wozmon.php) could do in 254 bytes of 6502 machine code.
Finally, I might need that UART for something else.

## SWO!!! I hear you shouting!
NO!
All ARM chips have a dedicated debugger pin to output data.
It's basically just UART (although it can also do manchester encoding), but most of the same drawbacks apply; it requires a GPIO pin and a debug probe that supports it. I just don't bother.

# Just throw money at the problem
I have used SEGGER's J-Link probes for years. Though they are very good, they are also very expensive and have quite strange terms of use. However, they have one *killer feature*.

## RTT, my sweet prince
[Real Time Transfer](https://www.segger.com/products/debug-probes/j-link/technology/about-real-time-transfer/) is one of those things that I didn't know I needed it until I saw it, and now I could never live without it.
The selling point is that you just include their library, and you get non-blocking bidirectional communication with your microcontroller over the SWD lines (SWDIO and SWCLK) at an incredible bandwidth.

The way this works is actually pretty cool, so I will take a moment to explain how it works before I show my alternative.
All ARM Cortex CPUs allow the debugger probe to read the contents of any address that the CPU can access without halting it. Take a second to appreciate how cool that is.
When you add the RTT library to your code, it creates a couple of ring buffers in memory with a header that can be easily found like `0xF0CACCIA` or `0x1234ABCD`.
What the J-Link Viewer software does, is scan the contents of RAM for that header, then it can just read from and write to the ring buffer structure.

This is such a powerful technique. Tools like [VisualGDB](https://visualgdb.com/) and [IAR](https://www.iar.com/embedded-development-tools/iar-embedded-workbench) do the same thing to plot the contents of variables at runtime.

Ok, time to spoil the fun. Even though I have praised RTT, there are a few issues, and they are big enough to make it not want to use it.

First, it only works with J-Link probes ($$$).
Second, it has a noticeable memory footprint. Not bad, but we can do better.
Lastly, it doesn't have a permissive licence. I cannot use RTT with any other debugger, even if I reimplement it from scratch.

## Are you done yapping?
Almost. If you made it so far, have a 🍪, you deserve it.

I have no idea why ARM has not come up with a standard way of logging with the same technique. Maybe they want SEGGER to not go out of business (I don't want that either, they are good engineers).

They have however came up with an alternative solution, not as good, but at least it's widely supported.

# Semihosting to the rescue
[Semihosting](https://developer.arm.com/documentation/101470/1900/Controlling-Target-Execution/Using-semihosting-to-access-resources-on-the-host-computer?lang=en) is a very fancy name, for a very simple concept. What if we could do syscalls, but the debug probe was the kernel.

Semihosting is a very powerful mechanism, not quite as powerful as direct memory access, but it still holds it own. It boils down to putting your "syscall" arguments in some registers and calling `asm("bkpt 0xAB")`.

This is it:
```c
int SEMIHOST_SysCall(int reason, void *arg)
{
    int value;
    __asm volatile(
        "mov r0, %[rsn] \n" // place semihost operation code into R0
        "mov r1, %[arg] \n" // R1 points to the argument array
        "bkpt 0xAB      \n" // call debugger
        "mov %[val], r0 \n" // debugger has stored result code in R0

        : [val] "=r"(value)                 // outputs
        : [rsn] "r"(reason), [arg] "r"(arg) // inputs
        : "r0", "r1", "r2", "memory");
    return value; // return result code, stored in R0
}
```

It supports ~~too~~ many syscalls, from reading files from the host, to getting UTC time.
For our application however, we only care about 2 `SYS_WRITE` and `SYS_READC`.

Here is how we use them:
```c
int getchar(void)
{
    return SEMIHOST_SysCall(SYS_READC, NULL);
}

int _write(int fd, const char *buf, int size)
{
    if (fd == SEMIHOST_STDOUT || fd == SEMIHOST_STDERR)
    {
        int32_t args[3] = {fd, (int32_t)buf, size};
        return SEMIHOST_SysCall(SYS_WRITE, &args[0]);
    }
    return 0; // Unsupported file descriptor
}
```

To see the output, just enable semihosting on your debug probe. For `pyocd`, just pass the `-S` flag. For `openocd`, add `monitor arm semihosting enable` to the run commands.

There is one more thing that we need do. If we don't have a debugger connected, the breakpoint instruction is going to throw and exception. We need to modify our hard fault handler to ignore the exception.
I'm not going to bore you with how you have to do it, but it will require a bit of assembly. [Here](https://github.com/BogdanTheGeek/minimal-py32/blob/529c0b3962d8cad490d942f291cb6f0fa50734cf/src/semihost.c#L49) is how I implemented it, based on Erich Styger's [example](https://mcuoneclipse.com/2023/03/09/using-semihosting-the-direct-way/).
Erich also goes into some performance metrics.

## Nothing is ever free
The only negative with this approach is that we do have to interrupt the CPU to signal to the debugger that we have some data to be read, and wait for it to read it.
This only happens when the MCU decides, so I don't consider it to be any worse than something like blocking UART.

Even though semihosting is wildly supported by most debug probes, not all GUI debuggers seem to handle it so gracefully. The Cortex-debug VSCode extension doesn't seem to like it at the time of writing. I don't personally use it, and you can just disable semihosting with a compilation flag, so this is not a deal breaker to me.


