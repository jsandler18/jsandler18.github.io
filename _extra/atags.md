---
layout: page
title:  Atags
---

The **Atags** is a list of information about certain aspects of the hardware.  This list is created by the bootloader before our kernel is loaded.  The bootloader places
it at address 0x100, and also passes that address to the kernel through register r2.  If you look at the function signature of kernel_main,
`void kernel_main(uint32_t r0, uint32_t r1, uint32_t atags)`, you can see that the atags pointer is the third argument.

The Atags can tell us how large the memory is, where the bootloader put a ramdisk, what is the serial number of the board, and the command line passed to the kernel via
`cmdline.txt`

An Atag consists of a size (in 4 byte words), a tag identifier, and tag specific information.  The list of Atags always starts with the `CORE` tag, with identifier `0x54410001`, and ends
with a `NONE` tag, with identifier `0`.  The tags are concatenated together, so the next tag in the list can be found by adding the number of bytes specified by the size
to the current Atag's pointer.

See [this page for a complete list of tags](http://www.simtec.co.uk/products/SWLINUX/files/booting_article.html#appendix_tag_reference)
