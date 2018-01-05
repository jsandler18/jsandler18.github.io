---
layout: page
title:  The Mailbox Peripheral
---

The **Mailbox Peripheral** is a [peripheral](/extra/peripheral.html) that facilitates communication between the ARM CPU and the VideoCore GPU.  It starts at offset
`0xB880` and has three relevent registers.  The `Read` register is at offset `0x00` from the mailbox base, and facilitates reading messages from the GPU.  The low 4 bits
of the register denote which channel the message is from, and the high 28 bits are data.  The `Status` register is at offset `0x18` from the mailbox base.  Bit 30 of this
register can tell you whether the `Read` register is empty and bit 31 can tell you whether the `Write` register is full.  The `Write` Register is at offset `0x20`, and has
similar form to the `Read` register
Here is the layout of the mailbox peripheral:
```
       0       4       8      12      16      20      24      28      32
       +-------+-------------------------------------------------------+
0x00   |rd chn |                     read data                         |
       +-------+-------------------------------------------------------+
0x04   |                             Unused                            |
...    \\                                                             \\
0x14   |                             Unused                            |
       +-----------------------------------------------------------+-+-+
0x18   |      status reserved                                      |E|F|
       +-----------------------------------------------------------+-+-+
0x1C   |                             Unused                            |
       +-----------------------------------------------------------+-+-+
0x20   |wt chn |                    write data                         |
       +-----------------------------------------------------------+-+-+

```

A **Channel** is a number that tells you and the GPU what the information being sent through the mailbox means.  We will only be needing channel 1, [the framebuffer
channel](/extra/fb-channel.html), and channel 8, [the property channel](/extra/prop-channel.html).
