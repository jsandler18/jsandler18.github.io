---
layout: page
title:  The Property Mailbox Channel
---

## Property Channel Overview
The **Property Channel** is [mailbox channel](/extra/mailbox.html) 8, and is the "new" way of asking the GPU for a framebuffer.  This method is used for the Raspberry Pi
Model 2 and up.  For Model 1, you can get a framebuffer through the [framebuffer channel](/extra/fb-channel.html).

The property channel is a way of getting and setting data about various hardware devices, of which the framebuffer is just one.

## Property Channel Messages
Messages have a fairly complex and poorly documented structure.  A message must be a 16 byte aligned buffer of 4 byte words.

A Message starts with a 4 byte size of the message, including the 4 bytes for the size itself.

The size is followed by 4 byte a *request/response code*.  When sending a message, this value must be 0.  When receiving a message, this part will either be 0x80000000 for success and 0x80000001 for an error.

Following the request/response code comes a concatenated list of *tags*.  Tags are simultaneously commands and buffers for responses to those commands. There are many tags, but we will only talk about those directly relevent to us.

The very last tag must be an end tag, which is just 4 bytes of 0.

Finally, the message must be padded so that the size of the whole buffer is 16 byte aligned.

Here is a map of the message buffer:

```
       0       4       8      12      16      20      24      28      32
       +---------------------------------------------------------------+
0x00   |                         Buffer Size                           |
       +---------------------------------------------------------------+
0x04   |                   Request/Response Code                       |
       +---------------------------------------------------------------+
0x08   |                             Tags                              |
...    \\                                                             \\
0xXX   |                             Tags                              |
       +---------------------------------------------------------------+
0xXX+4 |                           End Tag (0)                         |
       +---------------------------------------------------------------+
0xXX+8 |                           Padding                             |
...    \\                                                             \\
0xXX+16|                           Padding                             |
       +---------------------------------------------------------------+
```

## Message Tags
A Tag is a specific command to get or set certain data about the hardware.

A Tag starts with a 4 byte tag id, which identifies which command this tag wants to run.  In general, Tags have form `0x000XYZZZ`, where `X` identifies which hardware
device you are accessing, `Y` identifies which type of command it is (0 = get, 4 = test, 8 = set), and `ZZZ` identifies the specific command.

Next, the tag stores the size of its value buffer in bytes.  The buffer size should be the maximum of the size of the parameters for the command, and the size of the results.

Then, the tag has its own request/response code.  When sending a message, the value should be 0.  When receiving a message, the value should be 0x80000000 + length of the
result.

Next is the value buffer, where parameters and results are stored.  Though the message buffer that contains the value buffer is a u32 array, the value buffer is
a u8 array.

Finally, we need padding to make the tag 4 byte aligned.

Here is a map of a tag:
```
       0       4       8      12      16      20      24      28      32
       +---------------------------------------------------------------+
0x00   |                         Tag Identity                          |
       +---------------------------------------------------------------+
0x04   |                       Value Buffer Size                       |
       +---------------------------------------------------------------+
0x08   |                     Request/Response Code                     |
       +---------------------------------------------------------------+
0x0C   |                          Value Buffer                         |
...    \\                                                             \\
0xXX   | Value Buffer  |                    Padding                    |
       +---------------------------------------------------------------+

```
