---
layout: page
title:  Framebuffer, Pitch, and Depth
---

A **Framebuffer** is a piece of memory that is shared between the CPU and the GPU.  The CPU writes RGB pixels to the buffer, and the GPU renders it to whatever output
device you have connected.

The **Depth** of a framebuffer is the number of bits in every pixel.  For this tutorial, we will be exclusively using a depth of 24, meaning that each of the red, green
and blue values will have 8 bits, or 1 byte to them.

The **Pitch** of a framebuffer is simply the number of bytes that are in each row on screen.

We can calculate the **Pixels Per Row** by `pitch / (depth / 8)`,
or equivalently `(pitch * 8) / depth`.  

We can calculate the offset within the framebuffer of a pixel located at **Coordinates (x,y)** by `pitch * y + (depth / 8) * x`
