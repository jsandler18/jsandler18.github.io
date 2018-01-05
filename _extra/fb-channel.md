---
layout: page
title:  The Framebuffer Mailbox Channel
---

The **Framebuffer Channel** is [mailbox channel](/extra/mailbox.html) 1, and is the "old" way of asking the GPU for a framebuffer.  This method is used for Raspberry Pi
Model 1 only.  For Model 2 and up, you can get a framebuffer through the [property channel](/extra/prop-channel.html).

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

The values for [pitch](/extra/framebuffer.html), pointer and size will be filled in by the GPU.
