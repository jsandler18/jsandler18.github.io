---
layout: page
title:  Interrupts and Exceptions
---

## Exceptions 
An **Exception** is an event that is triggered when something exceptional occurs during normal program execution.  Examples of such exceptional occurrences include hardware devices
presenting new data to the CPU, user code asking to perform a privileged action, and a bad instruction was encountered.

On the Raspberry Pi, when an exception occurs, a specific address is loaded into the program counter register, branching execution to this point.  At this location, the
kernel developer needs to write branch instructions to routines that handle the exceptions.  This set of addresses, also known as the **Vector Table**, starts at address 0.  Below is a table that describes each exception

|Address|Exception Name  |Exception Source|Action to take            |
|-------|----------------|----------------|--------------------------|
|0x00   |Reset|Hardware Reset  |Restart the Kernel        |
|0x04   |Undefined instruction |Attempted to execute a meaningless instruction|Kill the offending program|
|0x08   |Software Interrupt (SWI) |Software wants to execute a privileged operation| Perform the opertation and return to the caller|
|0x0C   |Prefetch Abort|Bad memory access of an instruction|Kill the offending program|
|0x10   |Data Abort|Bad memory access of data| Kill the offending program|
|0x14   |Reserved|  Reserved  | Reserved|
|0x18   |Interrupt Request (IRQ) |Hardware wants to make the CPU aware of something|Find out which hardware triggered the interrupt and take appropriate action|
|0x1C   |Fast Interrupt Request (FIQ)|One select hardware can do the above faster than all others|Find out which hardware triggered the interrupt and take appropriate action|

## Interrupt Requests
An **Interrupt Request** or **IRQ**  is a notification to the processor that something happened to that hardware that the processor should know about.  This can take many
forms, like a keypress or access of privileged memory or receiving a network packet.

In order to determine which hardware devices are allowed to trigger interrupts, and determine which device triggered an interrupt, we need to use the IRQ
[peripheral](/extra/peripheral.html), which starts at offset 0xB000 from the peripheral base address.  This peripheral has three types of registers: pending, enable, and disable.  Pending registers indicate
whether a given interrupt has been triggered.  These are used in order to determine which hardware device triggered the IRQ exception.  Enable registers enable certain
interrupts to be triggered by setting the appropriate bit.  Disable registers disable certain interrupts by setting the appropriate bit. 

The Raspberry Pi has 72 possible IRQs.  IRQs 0-63 are shared between the GPU and CPU, and 64-71 are specific to the CPU. The two most important IRQs for our purposes will be the system timer (IRQ number 1) and the USB controller (IRQ number 9).

Here is the layout of the IRQ peripheral:
```
       0       4       8      12      16      20      24      28      32
       +---------------------------------------------------------------+
0x00   |                 ARM Specific Pending (64-71)                  |
       +---------------------------------------------------------------+
0x04   |                ARM/GPU Shared Pending 1 (0-31)                |
       +---------------------------------------------------------------+
0x08   |               ARM/GPU Shared Pending 2 (32-63)                |
       +---------------------------------------------------------------+
0x0C   |                       Fast IRQ Control                        |
       +---------------------------------------------------------------+
0x10   |                ARM/GPU Shared Enable 1 (0-31)                 |
       +---------------------------------------------------------------+
0x14   |                ARM/GPU Shared Enable 2 (32-63)                |
       +---------------------------------------------------------------+
0x18   |                 ARM Specific Enable (64-72)                   |
       +---------------------------------------------------------------+
0x1C   |                ARM/GPU Shared Disable 1 (0-31)                |
       +---------------------------------------------------------------+
0x20   |                ARM/GPU Shared Disable 2 (32-63)               |
       +---------------------------------------------------------------+
0x24   |                 ARM Specific Disable (64-72)                  |
       +---------------------------------------------------------------+
```
There are a couple of unintuitive notes about this peripheral:
1. When you see an interrupt is pending, you should not clear that bit.  Each peripheral that can trigger an interrupt has its own mechanism for clearing the pending bit
   that should be done in the handler for that peripheral's IRQ.
2. Clearing a bit in an enable regiester is neither sufficient nor neccesary to disable an IRQ.  An IRQ should only be disabled by setting the correct bit in the disable
   register
