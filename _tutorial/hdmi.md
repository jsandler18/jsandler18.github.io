---
layout: page
title:  Part 5 - Printing to a Real Screen
---
As of now, our kernel can do IO through its serial port.  On real hardware, we expect output to be printed on a real screen, usually through HDMI, not through some
specialty serial connection.  In this part, we will loook at how to achieve this.

If you want to download the code and play with it yourself, [see my git repo](https://github.com/jsandler18/raspi-kernel/tree/df09417ad6b1b0b6f849cc0575ab890921fa0634).
