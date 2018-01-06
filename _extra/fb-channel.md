---
layout: page
title:  The Framebuffer Mailbox Channel
---
## Framebuffer Channel Overview
The **Framebuffer Channel** is [mailbox channel](/extra/mailbox.html) 1, and is the "old" way of asking the GPU for a framebuffer.  This method is used for Raspberry Pi
Model 1 only.  For Model 2 and up, you can get a framebuffer through the [property channel](/extra/prop-channel.html).

## The Framebuffer Initialization Message
The only message you can send over this channel is a pointer to an initialization structure, and the only message you receive is a success/failure message.  The
initialization structure looks like this:
``` c
typedef struct {
    uint32_t width;
    uint32_t height;
    uint32_t vwidth;
    uint32_t vheight;
    uint32_t pitch;
    uint32_t depth;
    uint32_t ignorex;
    uint32_t ignorey;
    void * pointer;;
    uint32_t size;
} fb_init_t;
```

In this struct, you initialize the desired width, height, virtual width, virtual height and color depth.  Width and height are measured in pixels, and for this tutorial, we will always use 640x480.  Virtual width and virtual height should just be set to the width and the height.  [Depth](/extra/framebuffer.html) should be 24.

The values for [pitch](/extra/framebuffer.html), pointer and size will be filled in by the GPU.  The pointer will bethe framebuffer.

## Sending the Initialization message
To prepare this message to be sent over the mailbox, three things must be done:
1. Ensure the initialization structure is 16 byte aligned, so that the address only takes up the high 28 bits
2. Add 0x40000000 to the address of the initialization structure
3. Bitwise or this value with 1, so that channel 1 takes up the low 4 bits

Once these are done, this value can be sent to the GPU through the mailbox.  The GPU will send back the same address through the mailbox on success, and 0 on error.

You might have noticed that step 2 is a bit weird.  Why are we adding 0x40000000 to our address?  It's becaues the GPU has the RAM mapped to begin at physical address 0x40000000 (if L2 caching is enabled, which it is by default.  If it is disabled, it actually begins at 0xC0000000).  When the GPU accesses `0x40000000 + our_address` in physical memory, it is actually accessing `our_address` in RAM.  The GPU uses the physical addresses below 0x40000000 for [MMIO](/extra/peripheral.html) with hardware devices.
