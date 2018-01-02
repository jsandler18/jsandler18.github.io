---
layout: page
title:  Memory Mapped IO, Peripherals, and Registers
---

**Memory Mapped IO** or *MMIO* is the process of interacting with hardware devices by by reading from and writing to predefined memory addresses.  All interactions with hardware on the Raspberry Pi occur using MMIO.

A **Peripheral** is a hardware device with a specific address in memory that it writes data to and/or reads data from.  All peripherals can be described by an offset from
the **Peripheral Base Address**, which starts at 0x20000000 on the Raspberry Pi model 1, and at 0x0x3F000000 on the models 2 and 3.


A **Register** is a 4 byte piece of memory through that a peripheral can read from or write to.  These registers are at predefined offsets from the peripheral's base address.  For example, it is quite common for one at least one register to be a control register, where each bit in the register corresponds to a certain behavior that the hardware should have.  Another common register is a write register, where anything written in it gets sent off to the hardware.

Figuring out where all the peripherals are, what registers they have, and how to use them can mostly be found in [the BCM2835 ARM peripheral manual](https://www.raspberrypi.org/app/uploads/2012/02/BCM2835-ARM-Peripherals.pdf).  The BCM2835 is the name of the chipset the Raspberry Pi model 1 uses, and most of the information is good for the model 2 and 3.  This document is not easy to parse, and it is missing quite a bit of information, but it is a good starting point, as well as proof that I am not pulling all these addresses out of thin air.
