---
layout: page
title:  The System Timer Peripheral
---

The **System Timer** is a hardware clock that can be used to keep time and generate interrupts after a certain time.  It is located at offset 0x3000 from the [peripheral
base](/extra/peripheral.html).

The system timer is a **Free Running Timer** that incriments a 64 bit counter every microsecond, starting as soon as the Pi boots up, and runs in the background for
as long as the Pi is on.

There are four **Compare Registers** that the timer compares the low 32 bits low the counter every tick.  If any compare register matches the counter, an [IRQ is
triggered](/extra/interrupts.html#interrupt-requests).  Each compare register has its own interrupt, interrupt numbers 0-3.  Compare registers 0 and 2 are used by the GPU
and probably shouldn't be messed with, but 1 and 3 are availible for our use.

There is also a control register.  The low four bits are flags that indicate whether or not an interrupt was triggered.  Clearing this bit clears the interrupt pending
flag for that timer.

Here is the layout of the timer peripheral:
```
       0       4       8      12      16      20      24      28      32
       +---------------------------------------------------------------+
0x00   |0|1|2|3|                       Reserved                        |
       +---------------------------------------------------------------+
0x04   |                          Counter Low 32                       |
       +---------------------------------------------------------------+
0x08   |                          Counter High 32                      |
       +---------------------------------------------------------------+
0x0C   |                         Compare Register 0                    |
       +---------------------------------------------------------------+
0x10   |                         Compare Register 1                    |
       +---------------------------------------------------------------+
0x14   |                         Compare Register 2                    |
       +---------------------------------------------------------------+
0x18   |                         Compare Register 3                    |
       +---------------------------------------------------------------+
```
