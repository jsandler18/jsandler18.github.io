---
layout: page
title:  Interrupts and Exceptions
---

## Exceptions 
An **Exception** is an event that is triggered when something exceptional occurs during normal program execution.  Examples of such exceptional occurrences include hardware devices
presenting new data to the CPU, user code asking to perform a privileged action, and a bad instruction was encountered.

On the Raspberry Pi, when an exception occurs, a specific address is loaded into the program counter register, branching execution to this point.  At this location, the
kernel developer needs to write branch instructions to routines that handle the exceptions.  This set of addresses, also known as the **Vector Table**, starts at address
0.  Below is a table that describes each exception

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

