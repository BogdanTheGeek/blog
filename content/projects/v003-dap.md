+++
title = "World's Cheapest ARM Debugger is Actually RISC-V"
date = '2025-10-19T15:07:31+01:00'
author = 'Bogdan Ionescu'
description = 'Debugging a free ARM MCU with a 10¢ RISC-V probe'
tags = ['programming', 'arm', 'risc-v', 'ch32v003', 'tools', 'electronics']
cover = 'images/v003-dap.jpeg'
+++

# Background
Continuing my work with [arm debugging]({{< ref "/insights/jlink-rtt-for-the-masses.md" >}}) on free microcontrollers recovered from [disposable vapes]({{< ref "/projects/vapeserver.md" >}}), I felt like using a $5 raspberry pi pico to program and debug these micros was a bit too extravagant, too bourgeoisie.
A working man's microcontroller deserves a blue collar debugger to match. I have been using the 10¢ ch32v003 RISC-V microcontroller for a few years now and I though it would be a perfect fit for this project.
It also helped that a member of the electronics community I am part of had a bounty out for this exact project.

# I Missed Having Fun
My day job focuses on safety, reliability, and security. It's enjoyable in it's own way, but it does make me more reluctant to work on embedded projects in my free time. I end up spending time trying to find the documentation or trying to understand some one else's mental model of how a library should work, instead of trying to understand how the hardware works.
I find hardware much more interesting than software. I missed the days of programming [PICs](https://www.microchip.com/en-us/products/microcontrollers/8-bit-mcus/pic-mcus) where all you needed was a compiler and a Technical Reference Manual.

Well, it seems like I am not the only one who feels this way, but unlike me, [cnlohr](https://github.com/cnlohr/) decided to do something about it. A few years ago he created [ch32fun](https://github.com/cnlohr/ch32fun) to make working with the ch32v003(and more) really fun. He recently did a presentation on [how this project came about](https://www.youtube.com/watch?v=T_Lw4m3O6fY).
I have been using it for more than a dozen projects so far and it really been a lot of *fun*.

The ch32v003 is pretty famous by now as a *really* cheap microcontroller, as low as 10¢ in bulk, which alone has made it my micro of choice for quick projects.
However, it's real party trick is a Software Low Speed USB stack that cnlohr wrote called [rv003usb](https://github.com/cnlohr/rv003usb). This opens up the door for *so* many more projects, including this one.

# What am I supposed to do with my ARMs?
I've talked about j-link before, it's widely known as a "universal" programmer. A lot of engineers love them because they remove the need for vendor specific debugging probes like st-link or renesas e2 lite[^1]. They "just work" with almost everything. I think they do provide a lot of value, however, you can get almost all the same features for pretty much free.

[^1]: I hate the fact that these manufacturers keep advertising their own systems when a universal system already exist. A system that allows them to extend it for their own needs too.

All ARM Cortex CPUs have a common debugging interface called CoreSight DAP(Debug Access Probe) which means that they can all be controlled with a probe running a compatible firmware like [CMSIS-DAP](https://github.com/ARM-software/CMSIS-DAP), [DAPLink](https://github.com/ARMmbed/DAPLink), or [free-dap](https://github.com/ataradov/free-dap).

![CMSIS-DAP Architecture](https://github.com/ARM-software/CMSIS-DAP/blob/main/Documentation/Doxygen/src/images/CMSIS_DAP_INTERFACE.png?raw=true)

All of these projects support multiple programming interfaces, but I will only focus on SWD(Serial Wire Debug) because it's my favourite.

# RISC Assessment
So we know that the v003 can do USB, but there's a catch. It can only do Low Speed USB. This comes with a few limitation, the biggest one of which is that we cannot use any bulk endpoints. This wouldn't really be a problem, if not for the moronic insistence of kernel developers to not support interrupt endpoints for most USB device classes.[^3]

[^3]: I know it's not in the spec, but their are literally checking for it and choosing not to support it.

In order to to assess the viability of this project, we need to make sure that CMSIS-DAP can use interrupt endpoints. Thankfully, whoever wrote the CMSIS-DAP V1 spec was a top notch engineer and allowed for interrupt endpoints. If I ever find out who they are, I owe them a pint.

After a bit of googling, I also found that someone already [ported DAPLink](https://github.com/andrepan/ch32v-DAPLink) to other WCH chips with built in USB, so I decided to start there. 


# Bits and Bobs
ARM SWD is a surprisingly easy to understand protocol. I ended up having to understand how it works while I was troubleshooting some issues and really appreciated it's simplicity. It leaves pretty much all the complex logic to the host, which means that the probe firmware can be very simple. I won't go into the inner working of SWD, because [this great article](https://qcentlabs.com/posts/swd_banger/) does a much better job that I could. I will however show a trace of the first successful packet sent over SWD by my code.

![](images/swd_trace.png)

# The Grind
It took me only a couple of hours to port the code to ch32fun and rv003usb and figure out the correct USB descriptors. I then spent a bit more time figuring out the DAPlink state machine and ring buffers. Within a couple of days I got `pyocd list -p` to reliably show my probe.

```
  #   Probe/Board                   Unique ID   Target
--------------------------------------------------------
  0   CNLohr RV003 CMSIS-DAP V1.3   1234        n/a
```

There we go, Bob's your uncle, job's a good 'un! Right? DAPlink makes it so easy to port to other devices, so it's pretty embarrassing that it took me 2 more days to figure out why I couldn't talk to the target device.

In theory, all you have to do is implement some of the inline functions in `DAP_config.h` and you are done, easy peasy:
```c

/** SWDIO/TMS I/O pin: Set Output to High.
Set the SWDIO/TMS DAP hardware I/O pin to high level.
*/
__STATIC_FORCEINLINE void PIN_SWDIO_TMS_SET(void)
{
    funDigitalWrite(SWDIO_PIN, FUN_HIGH);
}

/** SWDIO/TMS I/O pin: Set Output to Low.
Set the SWDIO/TMS DAP hardware I/O pin to low level.
*/
__STATIC_FORCEINLINE void PIN_SWDIO_TMS_CLR(void)
{
    funDigitalWrite(SWDIO_PIN, FUN_LOW);
}

/** SWDIO I/O pin: Get Input (used in SWD mode only).
\return Current status of the SWDIO DAP hardware I/O pin.
*/
__STATIC_FORCEINLINE uint32_t PIN_SWDIO_IN(void)
{
    return funDigitalRead(SWDIO_PIN) ? 1U : 0U;
}
```

To troubleshoot this, I ended up writing the initialisation sequence manually by calling the `SW*_Sequence` functions.

```c
    extern void SWJ_Sequence(uint32_t bitCount, const uint8_t *data);
    extern uint8_t SWD_Transfer(uint32_t request, uint32_t * data);

    static uint8_t buffer[64];
    memset(buffer, 0xFF, sizeof(buffer));

    uint8_t jtagswd[2] = {0x9E, 0xE7};
    uint8_t nil[2] = {0x00, 0x00};
    uint32_t idcode = DAP_TRANSFER_RnW;
    LOGI(TAG, "SWD Test icode 0x%x", idcode);

    uint32_t val = 0;
    PORT_SWD_SETUP();
    while (1)
    {
        SWJ_Sequence(50, buffer);
        Delay_Us(500);
        SWJ_Sequence(16, jtagswd);
        Delay_Us(500);
        SWJ_Sequence(50, buffer);
        Delay_Us(500);
        SWJ_Sequence(12, nil);
        Delay_Us(500);
        uint8_t ack = SWD_Transfer(idcode, &val);
        LOGI(TAG, "ack=%02X val=%08X", ack, val);
        Delay_Ms(2000);
    }
```
This is one of the reasons I really love working with C. I don't have to hope the author made the function public, or instantiate a class, or befriend the compiler to call a function. Just `extern` it and the linker will sort it out. Don't get me wrong, I have my issues with C, but just like all my friends in high school, when it's time to party, they are a lot of fun.

Anyway, looking at the traces, I finally narrowed it down to `PIN_SWDIO_OUT`, and as it tends to always happen, the more time you spend chasing a bug, the simpler the issues turns out to be:


```diff
/** SWDIO I/O pin: Set Output (used in SWD mode only).
\param bit Output value for the SWDIO DAP hardware I/O pin.
*/
__STATIC_FORCEINLINE void PIN_SWDIO_OUT(uint32_t bit)
{
-    funDigitalWrite(SWDIO_PIN, bit);
+    funDigitalWrite(SWDIO_PIN, bit & 1);
}
```
I didn't realise that the function expected me to check the least significant bit, a comment would have been nice, but on a positive note, I now know a lot more about how DAPlink works.

# Success! (almost)
After tweaking the delay functions a bit, I managed to fully debug a py32v002b with my debugger using pyOCD...on macOS that is.
When I tried linux and MS-DOS++, pyOCD decided to reenact a scene from Little Britain.

![computer says no!](images/pyocd_says_no.jpeg)

With the help of my mate [clever](https://github.com/cleverca22), who is *very* clever, we traced down the issue to pyOCD using libhid on macOS, but libusb for other architectures. I describe the issue better [here](https://github.com/pyocd/pyOCD/issues/1825), but as far as I am concerned, this is an issue with pyOCD, not my probe. We also tested [openocd](https://github.com/openocd-org/openocd) and after compiling from source[^4], it worked perfectly on his rp2040 on linux.

[^4]: This is exactly the reason why I've moved to pyOCD, I have always had to compile openocd from source to get anything to work.

I am honestly amazed that macOS, for the first time ever, "just worked". Hopefully it will still work a couple year from now.

# Final Thoughts
This project really was *fun*! I learned a lot about debugging on ARM, SWD, HID, and USB. I wonder if there is a record for most 3 letter acronyms in a CV, I might start tracking it when reviewing candidates from now on.

I haven't written any articles about my other RISC-V endeavours like my [embedded risc-v dynamic loader](https://github.com/BogdanTheGeek/rv32-dynamic-loading) or my [Cheap Crappy Constant Current Portable Programmable Power Supply](https://github.com/BogdanTheGeek/CCCCPPPS), but I absolutely love working with risc-v WCH micros, not because of WCH, but the community.

Finally, for anyone interested, the firmware is available [here](https://github.com/BogdanTheGeek/ch32v003-daplink).
