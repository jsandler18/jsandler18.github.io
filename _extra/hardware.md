---
layout: page
title:  Loading the Kernel onto Real Hardware
---

To load the kernel onto real hardware, you need to do the following:
1. Ensure that you have an SD card that can boot a full operating system like Raspbian
2. Rip out the code from the elf file to form a raw binary file.  You can do this by adding `OBJCOPY = ./gcc-arm-none-eabi-X-XXXX-XX-update/bin/arm-none-eabi-objcopy` to the top of the makefile, and adding `$(OBJCOPY) $(IMG_NAME).elf -O binary $(IMG_NAME).img` to the build target.
3. Mount your SD card onto the computer you develop on
4. There should be a file called `kernel.img` if you have a Model 1, and `kernel7.img` for a Model 2 or 3.  Rename this to something else.
5. Copy the kernel raw binary to the SD card and name it `kernel.img` on the model 1, and `kernel7.img` on the model 2 and 3.
6. Safely eject the SD card and boot the Raspberry Pi
