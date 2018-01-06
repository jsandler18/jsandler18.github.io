---
layout: page
title:  Part 5 - Printing to a Real Screen
---
As of now, our kernel can do IO through its serial port.  On real hardware, we expect output to be printed on a real screen, usually through HDMI, not through some
specialty serial connection.  In this part, we will loook at how to achieve this.

If you want to download the code and play with it yourself, [see my git repo](https://github.com/jsandler18/raspi-kernel/tree/df09417ad6b1b0b6f849cc0575ab890921fa0634).

## Getting a Framebuffer
If you are unfmilaiar with the concept framebuffers, I reccomend you read [Framebuffer, Pitch, and Depth](/extra/framebuffer.html) before continuing.

In order to draw anything to the screen, we need to get a framebuffer.  We can only get one by asking the GPU very nicely.  This process differs between the Raspberry Pi
Model 1 and Model 2.  Since I am developing for a Model 2 VM and Model 1 ahrdware, I will walk through both.

Both models get a framebuffer using the mailbox peripheral.  Read [The Mailbox Peripheral](/extra/mailbox.html) to see details on how the mailbox works and the interfaces
that it has availible.

For both methods, we will use a single header file, `include/kernel/framebuffer.h`, which will declare a struct to hold information about our framebuffer, a global
instance of that structure, and a function `framebuffer_init` that will initialize it. To build this, we will modify the makefile as follows:
```
...
ifeq ($(RASPI_MODEL),1)
    CPU = arm1176jzf-s
    DIRECTIVES = -D MODEL_1
    ARCHDIR = model1
else
    CPU = cortex-a7
    ARCHDIR = model2
endif
...
KERSOURCES += $(wildcard $(KER_SRC)/$(ARCHDIR)/*.c)
...
$(OBJ_DIR)/%.o: $(KER_SRC)/$(ARCHDIR)/%.c
    mkdir -p $(@D)
    $(CC) $(CFLAGS) -I$(KER_SRC) -I$(KER_HEAD) -c $< -o $@ $(CSRCFLAGS)
```
So now make will choose the correct implimentation depending on whether `RASPI_MODEL=1` is passed.

### Initializing the Framebuffer on the Model 1
In order to get a framebuffer on the model 1, we need to use the [framebuffer mailbox channel](/extra/fb-channel.html).  This channel's only purpose is to initialize a framebuffer.  The details of how to use the mailbox to get a framebuffer are covered [on this page](/extra/fb-channel.html), so here we will focus on the implimentation.

Once we get the framebuffer from the mailbox, we fill out that global info struct, `fbinfo`, with the width, height, and depth that that we used to initialize the framebuffer.  We also put in the [pitch](/extra/framebuffer.html), the pointer to the framebuffer, the framebuffer size, the width and height of the screen in characters, and the position of the character cursor. These last two fields will be important when we are rendering characters instead of just pixels.

Here is the code:
``` c
typedef struct {
    uint32_t width;
    uint32_t height;
    uint32_t vwidth;
    uint32_t vheight;
    uint32_t bytes;
    uint32_t depth;
    uint32_t ignorex;
    uint32_t ignorey;
    void * pointer;;
    uint32_t size;
} fb_init_t;

fb_init_t fbinit __attribute__((aligned(16)));

int framebuffer_init(void) {
    mail_message_t msg;

    fbinit.width = 640;
    fbinit.height = 480;
    fbinit.vwidth = fbinit.width;
    fbinit.vheight = fbinit.height;
    fbinit.depth = COLORDEPTH;

    msg.data = ((uint32_t)&fbinit + 0x40000000) >> 4;

    mailbox_send(msg, FRAMEBUFFER_CHANNEL);
    msg = mailbox_read(FRAMEBUFFER_CHANNEL);

    if (!msg.data)
        return -1;

    fbinfo.width = fbinit.width;
    fbinfo.height = fbinit.height;
    fbinfo.chars_width = fbinfo.width / CHAR_WIDTH; 
    fbinfo.chars_height = fbinfo.height / CHAR_HEIGHT;
    fbinfo.chars_x = 0;
    fbinfo.chars_y = 0;
    fbinfo.pitch = fbinit.bytes;
    fbinfo.buf = fbinit.pointer;
    fbinfo.buf_size = fbinit.size;

    return 0;
}
```

### Initializing the Framebuffer on the Model 2 and up
