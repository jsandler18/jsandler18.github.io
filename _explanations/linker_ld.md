---
layout: page
title:  An in-depth Explanation - linker.ld
---

## linker.ld source
Recall the source code for linker.ld
```
ENTRY(_start)
 
SECTIONS
{
    /* Starts at LOADER_ADDR. */
    . = 08000;
    __start = .;
    __text_start = .;
    .text :
    {
        KEEP(*(.text.boot))
        *(.text)
    }
    . = ALIGN(4096); /* align to page size */
    __text_end = .;
 
    __rodata_start = .;
    .rodata :
    {
        *(.rodata)
    }
    . = ALIGN(4096); /* align to page size */
    __rodata_end = .;
 
    __data_start = .;
    .data :
    {
        *(.data)
    }
    . = ALIGN(4096); /* align to page size */
    __data_end = .;
 
    __bss_start = .;
    .bss :
    {
        bss = .;
        *(.bss)
    }
    . = ALIGN(4096); /* align to page size */
    __bss_end = .;
    __end = .;
}
```

Let's step through this line by line.

## Some Notes About Linking
- In a linker script, `.` means *current address*.  You can assign the current address and also assign things to the current address.

- Review of the sections of a C program.
  - `.text` is where executable code goes. 
  - `.rodata` is *read only data*; it is where global constants are placed.  
  - `.data` is where global variables that are initialized at compile time are placed. 
  - `.bss` is where uninitialized global variables are placed.


## Putting the sections in the right places

```
ENTRY(_start)
```
This declares that the symbol `_start` from boot.S is the entry point to our code.

<br>
```
    . = 08000;
    __start = .;
    __text_start = .;
    .text :
    {
        KEEP(*(.text.boot))
        *(.text)
    }
    . = ALIGN(4096); /* align to page size */
    __text_end = .;
```
This sets the symbols __start, and __text_start to be 0x8000.  It then declares the .text section to start right after that.  The first part of the .text section is .text.boot, where the code from boot.S resides.  KEEP signifies that the linker should not try to optimize out the code in .text.boot, even though it is not referenced anywhere.  The second part of the .text section is all .text sections from all other objects, in any order.  Then declare __text_end to be the next largest address divisible by 4096 after all of the .text is put in.  This rounding to the nearest 4096 is called *page alignment*, and it becomes important when we start working with memory.

<br>
```
    __rodata_start = .;
    .rodata :
    {
        *(.rodata)
    }
    . = ALIGN(4096); /* align to page size */
    __rodata_end = .;
 
    __data_start = .;
    .data :
    {
        *(.data)
    }
    . = ALIGN(4096); /* align to page size */
    __data_end = .;
 
    __bss_start = .;
    .bss :
    {
        bss = .;
        *(.bss)
    }
    . = ALIGN(4096); /* align to page size */
    __bss_end = .;
    __end = .;
```
Similarly, we declare __rodata_start to be the same as __text_end, then we declare the .rodata section, which consists of all the .rodata sections from all the object files, then declare the __rodata_end to be the next page aligned address after the .rodata.  We then repeat this for the .data and .bss sections.
