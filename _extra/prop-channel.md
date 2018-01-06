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


## Getting a Framebuffer
The first step to getting a framebuffer is to set the screen size, virtual screen size, and [depth](/extra/framebuffer.html). The tag ids for these commands are
0x00048003, 0x00048004, and 0x00048005 respectively.

To set the screen size, you must pass a 4 byte width and a 4 byte height, and there is no result.  Therefore, the value buffer has size 8 for the screen size commands.

To set the depth, you must pass a 4 byte depth value, and there is no result.  Therefore, the value buffer has size 4.

We are going to set the screen size to 640x480, and the depth to 24.

If we put these together in a message, it should look something like this:
``` c
80,                             // The whole buffer is 80 bytes
0,                              // This is a request, so the request/response code is 0
0x00048003, 8, 0, 640, 480,     // This tag sets the screen size to 640x480
0x00048004, 8, 0, 640, 480,     // This tag sets the virtual screen size to 640x480
0x00048005, 4, 0, 24,           // This tag sets the depth to 24 bits
0,                              // This is the end tag
0, 0, 0                         // This pads the message to by 16 byte aligned
```
We then need to send this buffer through the mailbox.  To do that, we must ensure that our our buffer is located at a 16 byte aligned address, so only the high 28 bits
contain the address.  Then, we must bitwise or 8 with this address, setting the low 4 bits to the channel number.  Finally, we can send this through the mailbox using the
process described [here](/extra/mailbox.html). In order to verify that this worked, we must check the request/response code of our buffer.  If it is 0, then the message
did not go through properly, as the GPU couldn't overwrite this part with a response code.  If it is 0x80000001, then an error occured.  If it is 0x80000000, then it was
successful.

Now that we have set up the screen parameters, we can request a framebuffer. The tag id for this command is 0x00040001.  This command takes one 4 byte parameter, the
requested alignment of the framebuffer, and returns two 4 byte values, a pointer to the buffer and the buffer size.  Therefore, the value buffer has size 8. We will be
asking for a 16 byte aligned buffer.

The messge should look like this:
``` c
32,                         // The whole buffer is 32 bytes
0,                          // This is a request, so the request/response code is 0
0x00040001, 8, 0, 16, 0,    // This tag requests a 16 byte aligned framebuffer
0                           // This is the end tag
```

