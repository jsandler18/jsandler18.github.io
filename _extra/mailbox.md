---
layout: page
title:  The Mailbox Peripheral
---

## Mailbox Overview
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

## Reading from the Mailbox
To read from the mailbox, you must follow these steps:
1. In a loop, read the status register to make sure the the read register is not empty. 
2. Read the contents of the read register.
3. Check the channel of the read message.  If it is not what you were looking for, do something responsible with it (or discard it)
4. If the channel is correct, get the data.

Here is the code for read:
``` c
mail_message_t mailbox_read(int channel) {
    mail_status_t stat;
    mail_message_t res;

    // Make sure that the message is from the right channel
    do {
        // Make sure there is mail to recieve
        do {
            stat = *MAIL0_STATUS;
        } while (stat.empty);

        // Get the message
        res = *MAIL0_READ;
    } while (res.channel != channel);

    return res;
}
```
Where `mail_message_t` and `mail_status_t` are defined as follows:
``` c
typedef struct {
    uint8_t channel: 4;
    uint32_t data: 28;
} mail_message_t;

typedef struct {
    uint32_t reserved: 30;
    uint8_t empty: 1;
    uint8_t full:1;
} mail_status_t;
```

## Writing to the Mailbox
Writing has a similar process to reading:
1. In a loop, check the status register to see if the write register is not full
2. Write the data to the write register

Here is the code:
``` c
void mailbox_send(mail_message_t msg, int channel) {
    mail_status_t stat;
    msg.channel = channel;
    

    // Make sure you can send mail
    do {
        stat = *MAIL0_STATUS;
    } while (stat.full);

    // send the message
    *MAIL0_WRITE = msg;
}
```

