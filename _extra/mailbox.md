---
layout: page
title:  The Mailbox Peripheral
---

The **Mailbox Peripheral** is a [peripheral](/extra/peripheral.html) that facilitates communication between the ARM CPU and the VideoCore GPU.  It starts at offset
`0xB880` and has three relevent registers.  The `Read` register is at offset `0x00` from the mailbox base, and facilitates reading messages from the GPU.  The low 4 bits
of the register denote which channel the message is from, and the high 28 bits are data.  The `Status` register is at offset `0x18` from the mailbox base.  Bit 30 of this
register can tell you whether the `Read` register is empty and bit 31 can tell you whether the `Write` register is full.  The `Write` Register is at offset `0x20`, and has
similar form to the `Read` register
Here is the layout of the mailbox peripheral, if written out as a C struct:
``` c
struct mailbox {
    uint8_t read_channel : 4;  // ARM is little endian, so low bytes come first
    uint32_t read_data : 28;
    uint32_t unused1;
    uint32_t unused2;
    uint32_t unused3;
    uint32_t unused4;
    uint32_t unused5;
    uint32_t status_reseved : 30;
    uint8_t status_read_empty : 1;
    uint8_t status_write_full : 1;
    uint32_t unused6;
    uint8_t write_channel : 4;
    uint32_t write_data : 28;
}

```
