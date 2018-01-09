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
Model 1 and Model 2.  Since I am developing for a Model 2 VM and Model 1 hardware, I will walk through both.

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
So now make will choose the correct implementation depending on whether `RASPI_MODEL=1` is passed.

### Initializing the Framebuffer on the Model 1
In order to get a framebuffer on the model 1, we need to use the [framebuffer mailbox channel](/extra/fb-channel.html).  This channel's only purpose is to initialize a framebuffer.  The details of how to use the mailbox to get a framebuffer are covered [on this page](/extra/fb-channel.html), so here we will focus on the implementation.

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
In order to get a framebuffer on the model 2 and up, we need to use the [property mailbox channel](/extra/prop-channel.html). This channel has purposes other than getting a framebuffer, so the code for using it is much more abstract than for the framebuffer channel.  As before, the details of how to use the property channel to get a framebuffer are discussed on [this page](/extra/prop-channel), so here we will talk about the implementation.

Since the property interface is more abstract, the definitions of tags and implementation of sending and receiving messages is done in `include/kernel/mailbox.h` and `src/kernel/mailbox.c`.  We define an interface `send_message(property_message_tag_t * tags)`, which takes a null terminated array of message tags, packs them into the proper format, sends the message, gets the response, and writes each tag's response back into the given array.

`property_message_tag_t` and its related definitions are shown below:
``` c
typedef enum {
    NULL_TAG = 0,
    FB_ALLOCATE_BUFFER = 0x00040001,
    FB_RELESE_BUFFER = 0x00048001,
    FB_GET_PHYSICAL_DIMENSIONS = 0x00040003,
    FB_SET_PHYSICAL_DIMENSIONS = 0x00048003,
    FB_GET_VIRTUAL_DIMENSIONS = 0x00040004,
    FB_SET_VIRTUAL_DIMENSIONS = 0x00048004,
    FB_GET_BITS_PER_PIXEL = 0x00040005,
    FB_SET_BITS_PER_PIXEL = 0x00048005,
    FB_GET_BYTES_PER_ROW = 0x00040008
} property_tag_t;

typedef struct {
    void * fb_addr;
    uint32_t fb_size;
} fb_allocate_res_t;

typedef struct {
    uint32_t width;
    uint32_t height;
} fb_screen_size_t;

/*
 * The value buffer can be any one of these types
 */
typedef union {
    uint32_t fb_allocate_align;
    fb_allocate_res_t fb_allocate_res;
    fb_screen_size_t fb_screen_size;
    uint32_t fb_bits_per_pixel;
    uint32_t fb_bytes_per_row;
} value_buffer_t;

/*
 * A message_buffer can contain any number of these
 */
typedef struct {
    property_tag_t proptag;
    value_buffer_t value_buffer;
} property_message_tag_t;
```

The code to pack an array of these tags into the proper format is just a straightforward implementation of the format described [here](/extra/prop-channel.html).  The `get_value_buffer_len` function simply hardcodes the various value buffer sizes for each defined tag.

Here is the code:
```
static uint32_t get_value_buffer_len(property_message_tag_t * tag) {
    switch(tag->proptag) {
        case FB_ALLOCATE_BUFFER: 
        case FB_GET_PHYSICAL_DIMENSIONS:
        case FB_SET_PHYSICAL_DIMENSIONS:
        case FB_GET_VIRTUAL_DIMENSIONS:
        case FB_SET_VIRTUAL_DIMENSIONS:
            return 8;
        case FB_GET_BITS_PER_PIXEL:
        case FB_SET_BITS_PER_PIXEL:
        case FB_GET_BYTES_PER_ROW:
            return 4;
        case FB_RELESE_BUFFER:
        default:
            return 0;
    }
}

int send_messages(property_message_tag_t * tags) {
    property_message_buffer_t * msg;
    mail_message_t mail;
    uint32_t bufsize = 0, i, len, bufpos;
   
    // Calculate the sizes of each tag
    for (i = 0; tags[i].proptag != NULL_TAG; i++) {
        bufsize += get_value_buffer_len(&tags[i]) + 3*sizeof(uint32_t);          
    }

    // Add the buffer size, buffer request/response code and buffer end tag sizes
    bufsize += 3*sizeof(uint32_t); 

    // buffer size must be 16 byte aligned
    bufsize += (bufsize % 16) ? 16 - (bufsize % 16) : 0;

    // kmalloc returns a 16 byte aligned address
    msg = kmalloc(bufsize);
    if (!msg)
        return -1;

    msg->size = bufsize;
    msg->req_res_code = REQUEST;

    // Copy the messages into the buffer
    for (i = 0, bufpos = 0; tags[i].proptag != NULL_TAG; i++) {
        len = get_value_buffer_len(&tags[i]);
        msg->tags[bufpos++] = tags[i].proptag;
        msg->tags[bufpos++] = len;
        msg->tags[bufpos++] = 0;
        memcpy(msg->tags+bufpos, &tags[i].value_buffer, len);
        bufpos += len/4;
    }

    msg->tags[bufpos] = 0;

    // Send the message
    mail.data = ((uint32_t)msg) >>4;
    
    mailbox_send(mail, PROPERTY_CHANNEL);
    mail = mailbox_read(PROPERTY_CHANNEL);


    if (msg->req_res_code == REQUEST) {
        kfree(msg);
        return 1;
    }
    // Check the response code
    if (msg->req_res_code == RESPONSE_ERROR) {
        kfree(msg);
        return 2;
    }

    
    // Copy the tags back into the array
    for (i = 0, bufpos = 0; tags[i].proptag != NULL_TAG; i++) {
        len = get_value_buffer_len(&tags[i]);
        bufpos += 3; //skip over the tag bookkepping info
        memcpy(&tags[i].value_buffer, msg->tags+bufpos,len);
        bufpos += len/4;
    }

    kfree(msg);
    return 0;
}
```

Now that we have this code, all `src/kernel/model2/framebuffer.c` needs to do is create an array of these tags, pass them to `send_message`, and put the resulting information in the global info struct, `fbinfo`. 

Here is the code:
``` c
int framebuffer_init(void) {
    property_message_tag_t tags[5];


    tags[0].proptag = FB_SET_PHYSICAL_DIMENSIONS;
    tags[0].value_buffer.fb_screen_size.width = 640;
    tags[0].value_buffer.fb_screen_size.height = 480;
    tags[1].proptag = FB_SET_VIRTUAL_DIMENSIONS;
    tags[1].value_buffer.fb_screen_size.width = 640;
    tags[1].value_buffer.fb_screen_size.height = 480;
    tags[2].proptag = FB_SET_BITS_PER_PIXEL;
    tags[2].value_buffer.fb_bits_per_pixel = COLORDEPTH;
    tags[3].proptag = NULL_TAG;


    // Send over the initialization
    if (send_messages(tags) != 0) {
        return -1;
    }

    fbinfo.width = tags[0].value_buffer.fb_screen_size.width;
    fbinfo.height = tags[0].value_buffer.fb_screen_size.height;
    fbinfo.chars_width = fbinfo.width / CHAR_WIDTH;
    fbinfo.chars_height = fbinfo.height / CHAR_HEIGHT;
    fbinfo.chars_x = 0;
    fbinfo.chars_y = 0;
    fbinfo.pitch = fbinfo.width*BYTES_PER_PIXEL;

    // request a framebuffer
    tags[0].proptag = FB_ALLOCATE_BUFFER;
    tags[0].value_buffer.fb_screen_size.width = 0;
    tags[0].value_buffer.fb_screen_size.height = 0;
    tags[0].value_buffer.fb_allocate_align = 16;
    tags[1].proptag = NULL_TAG;


    if (send_messages(tags) != 0) {
        return -1;
    }

    fbinfo.buf = tags[0].value_buffer.fb_allocate_res.fb_addr;
    fbinfo.buf_size = tags[0].value_buffer.fb_allocate_res.fb_size;

    return 0;
}
```

## Rendering Characters
Now that we have a framebuffer, we want to be able to draw to it easily.  We are going to define `src/kernel/gpu.c` and `include/kernel/gpu.h` to define and implement drawing to the screen.  We are going to expose three functions: `gpu_init(void)`, `write_pixel(void write_pixel(uint32_t x, uint32_t y, const pixel_t * pixel)`, and `gpu_putc(char c)`.

`gpu_init` simply initializes the screen by calling `framebuffer_init()` and blacking out the screen.

`write_pixel` colors the pixel at the given coordinates. Here is the definition of `pixel_t` and the implementation of `write_pixel`:
```c
typedef struct pixel {
    uint8_t red;
    uint8_t green;
    uint8_t blue;
} pixel_t;
...

void write_pixel(uint32_t x, uint32_t y, const pixel_t * pix) {
    uint8_t * location = fbinfo.buf + y*fbinfo.pitch + x*BYTES_PER_PIXEL;
    memcpy(location, pix, BYTES_PER_PIXEL);
}
```
In order to draw a character, we need bitmaps of ever single character we want to be able to render.  Which set of bitmaps you choose determines the font of the characters.  [Here is the bitmaps I used](https://raw.githubusercontent.com/jsandler18/raspi-kernel/df09417ad6b1b0b6f849cc0575ab890921fa0634/include/kernel/chars_pixels.h). It is a 2D array, with 128 entries of 8 entry arrays of u8 integers, creating 8x8 character bitmaps.  This can easily be swapped out for another font, if you so desire.

If you look at my bitmap, I have done something strange.  Instead of having the font be a global array, it is a static array inside a function.  This is a workaround to an issue where a global array was not included when compiled for real hardware. Unfortunately, I am still not sure why this was the case, but this strange way of accessing the bitmaps works around it well enough.

Now that we have character bitmaps, we can implement `gpu_putc`. Here is the code:
``` c
void gpu_putc(char c) {
    static const pixel_t WHITE = {0xff, 0xff, 0xff};
    static const pixel_t BLACK = {0x00, 0x00, 0x00};
    uint8_t w,h;
    uint8_t mask;
    const uint8_t * bmp = font(c);
    uint32_t i, num_rows = fbinfo.height/CHAR_HEIGHT;

    // shift everything up one row
    if (fbinfo.chars_y >= num_rows) {
        // Copy a whole character row into the one above it
        for (i = 0; i < num_rows-1; i++)
            memcpy(fbinfo.buf + fbinfo.pitch*i*CHAR_HEIGHT, fbinfo.buf + fbinfo.pitch*(i+1)*CHAR_HEIGHT, fbinfo.pitch * CHAR_HEIGHT);
        // zero out the last row
        bzero(fbinfo.buf + fbinfo.pitch*i*CHAR_HEIGHT,fbinfo.pitch * CHAR_HEIGHT);
        fbinfo.chars_y--;
    }

    if (c == '\n') {
        fbinfo.chars_x = 0;
        fbinfo.chars_y++;
        return;
    }

    for(w = 0; w < CHAR_WIDTH; w++) {
        for(h = 0; h < CHAR_HEIGHT; h++) {
            mask = 1 << (w);
            if (bmp[h] & mask)
                write_pixel(fbinfo.chars_x*CHAR_WIDTH + w, fbinfo.chars_y*CHAR_HEIGHT + h, &WHITE);
            else
                write_pixel(fbinfo.chars_x*CHAR_WIDTH + w, fbinfo.chars_y*CHAR_HEIGHT + h, &BLACK);
        }
    }

    fbinfo.chars_x++;
    if (fbinfo.chars_x > fbinfo.chars_width) {
        fbinfo.chars_x = 0;
        fbinfo.chars_y++;
    }
}
```

Now, all we need to do is replace `uart_putc` with `gpu_putc` in our implementation of `putc`, and now we can print to a real screen.
Here was what it should look like:
![My First Screen](/first_screen.jpg)

To find out how to load your kernel on to real hardware, [see this page](/extra/hardware.html)

The next obvious step is to get keyboard input, but before we can do that, we need to set up a system called interrupts.

**Previous**:
[Part 4 - Dynamic Memory](/tutorial/dyn-mem.html)

**Next**:
[Part 6 - Interrupts](/tutorial/interrupts.html)
