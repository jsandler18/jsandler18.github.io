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
