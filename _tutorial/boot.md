---
layout: page
title:  Part 2 - Getting Something to Boot
---

As with any new project, the best way to get started is to copy a bunch of code from somewhere and get something working, then go back and try to understand the code.  I
pulled this first batch of code from [the OSDev wiki](http://wiki.osdev.org/Raspberry_Pi_Bare_Bones), but I am going to post it here and explain each piece.

If you want to download the code and play with it yourself, [see my git repo](https://github.com/jsandler18/raspi-kernel/tree/1eddfe2c6b42be11f5f01500a285b406c14e5c41).

## boot.S - The kernel entry point

boot.S is going to be the first thing that the hardware executes in our kernel.  This must be done in assembly.  When the hardware loadsthe kernel, it does not set up a C
runtime environment.  It does not even know what the C runtime environment looks like!  This assembly code sets this up so that we can jump to C as soon as possible.
Here is the code:

**boot.S**
```
.section ".text.boot"

.global _start

_start:
    mov sp, #0x8000

    ldr r4, =__bss_start
    ldr r9, =__bss_end
    mov r5, #0
    mov r6, #0
    mov r7, #0
    mov r8, #0
    b       2f

1:
    stmia r4!, {r5-r8}

2:
    cmp r4, r9
    blo 1b

    ldr r3, =kernel_main
    blx r3

halt:
    wfe
    b halt
```

For a line-by-line explanation of this code, [see this page](/explanations/boot_S.html)


## kernel.c - The C code

This file contains the meat of our baby kernel.

The bulk of the code is for setting up the hardware for basic I/O. The I/O is done through the UART hardware, which allows us to send and recieve text data through the serial ports.  The only way to take advantage of this on actual hardware is to get a [USB to TTL serial cable](https://www.adafruit.com/product/954).  Since I don't have one of those cables, I am going interact with the kernel through the VM until we can get more sophisticated I/O like HDMI out and USB keyboard.

Aside from the hardware setup, there are a few helper functions to abstract away the hardware, and, of course, the main function.

Here is the code:

**kernel.c**
``` c
#include <stddef.h>
#include <stdint.h>

static inline void mmio_write(uint32_t reg, uint32_t data)
{
    *(volatile uint32_t*)reg = data;
}

static inline uint32_t mmio_read(uint32_t reg)
{
    return *(volatile uint32_t*)reg;
#}

// Loop <delay> times in a way that the compiler won't optimize away
static inline void delay(int32_t count)
{
    asm volatile("__delay_%=: subs %[count], %[count], #1; bne __delay_%=\n"
            : "=r"(count): [count]"0"(count) : "cc");
}

enum
{
    // The GPIO registers base address.
    GPIO_BASE = 0x3F200000, // for raspi2 & 3, 0x20200000 for raspi1

    GPPUD = (GPIO_BASE + 0x94),
    GPPUDCLK0 = (GPIO_BASE + 0x98),

    // The base address for UART.
    UART0_BASE = 0x3F201000, // for raspi2 & 3, 0x20201000 for raspi1

    UART0_DR     = (UART0_BASE + 0x00),
    UART0_RSRECR = (UART0_BASE + 0x04),
    UART0_FR     = (UART0_BASE + 0x18),
    UART0_ILPR   = (UART0_BASE + 0x20),
    UART0_IBRD   = (UART0_BASE + 0x24),
    UART0_FBRD   = (UART0_BASE + 0x28),
    UART0_LCRH   = (UART0_BASE + 0x2C),
    UART0_CR     = (UART0_BASE + 0x30),
    UART0_IFLS   = (UART0_BASE + 0x34),
    UART0_IMSC   = (UART0_BASE + 0x38),
    UART0_RIS    = (UART0_BASE + 0x3C),
    UART0_MIS    = (UART0_BASE + 0x40),
    UART0_ICR    = (UART0_BASE + 0x44),
    UART0_DMACR  = (UART0_BASE + 0x48),
    UART0_ITCR   = (UART0_BASE + 0x80),
    UART0_ITIP   = (UART0_BASE + 0x84),
    UART0_ITOP   = (UART0_BASE + 0x88),
    UART0_TDR    = (UART0_BASE + 0x8C),
};

void uart_init()
{
    mmio_write(UART0_CR, 0x00000000);

    mmio_write(GPPUD, 0x00000000);
    delay(150);

    mmio_write(GPPUDCLK0, (1 << 14) | (1 << 15));
    delay(150);

    mmio_write(GPPUDCLK0, 0x00000000);

    mmio_write(UART0_ICR, 0x7FF);

    mmio_write(UART0_IBRD, 1);
    mmio_write(UART0_FBRD, 40);

    mmio_write(UART0_LCRH, (1 << 4) | (1 << 5) | (1 << 6));

    mmio_write(UART0_IMSC, (1 << 1) | (1 << 4) | (1 << 5) | (1 << 6) |
            (1 << 7) | (1 << 8) | (1 << 9) | (1 << 10));

    mmio_write(UART0_CR, (1 << 0) | (1 << 8) | (1 << 9));
}

void uart_putc(unsigned char c)
{
    while ( mmio_read(UART0_FR) & (1 << 5) ) { }
    mmio_write(UART0_DR, c);
}

unsigned char uart_getc()
{
    while ( mmio_read(UART0_FR) & (1 << 4) ) { }
    return mmio_read(UART0_DR);
}

void uart_puts(const char* str)
{
    for (size_t i = 0; str[i] != '\0'; i ++)
        uart_putc((unsigned char)str[i]);
}

void kernel_main(uint32_t r0, uint32_t r1, uint32_t atags)
{
    (void) r0;
    (void) r1;
    (void) atags;

    uart_init();
    uart_puts("Hello, kernel World!\r\n");

    while (1) {
        uart_putc(uart_getc());
        uart_putc('\n');
    }
}
```

For a line-by-line explanation of this code, [see this page](/explanations/kernel_c.html)

## linker.ld - Tying the pieces together
For those unfamiliar with the C compilation process, there are, broadly speaking, three main steps.  The first is preproccessing, where all of your `#define` statements are expanded.  The second is compilation to object files, where the individual code files are converted to individual binaries called object files.  The third is linking, where these individual object files are tied together into a single executable.

By default, GCC links your program as if it were user level code.  We need to override the default, because our kernel is not an ordinary user program.  We do this with a linker script.  Here is the linker script we will be using:

**linker.ld**
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
For a line-by-line explanation of this code, [see this page](/explanations/linker_ld.html)


## Compiling and Running
To compile this code for the VM, we must run the following commands:
```
./gcc-arm-none-eabi-X-XXXX-XX-update/bin/arm-none-eabi-gcc -mcpu=cortex-a7 -fpic -ffreestanding -c boot.S -o boot.o
./gcc-arm-none-eabi-X-XXXX-XX-update/bin/arm-none-eabi-gcc -mcpu=cortex-a7 -fpic -ffreestanding -std=gnu99 -c kernel.c -o kernel.o -O2 -Wall -Wextra
./gcc-arm-none-eabi-X-XXXX-XX-update/bin/arm-none-eabi-gcc -T linker.ld -o myos.elf -ffreestanding -O2 -nostdlib boot.o kernel.o
```
The first two commands compile boot.S and kernel.c into object code, respectively.  The second links those object files into an executable elf file.

Lets take a look at those less used gcc options.  `-mcpu=cortex-a7` means that the target ARM cpu is the cortex-a7, which is what the raspberry pi model 2 has, and what our VM emulates.  `-fpic` means create position independent code.  This means that references to any function, variable, or symbol should be done relative to the current instruction, and not by an absolute address.  `-ffreestanding` means that gcc cannot depend on libc being availible at runtime, and that there may not be a `main` function as an entry point.  `-nostdlib` indicates to the linker that it should not try to link in libc, which it will do by default.

<br>
To run the code in the VM, execute this command:
```
qemu-system-arm -m 256 -M raspi2 -serial stdio -kernel myos.elf
```
This runs a VM that emulates the raspberry pi model 2 with 256 megabytes of memory.  It is set up to read/write data from/to your normal terminal as if it were connected to the raspberry pi through a serial connection.  It specifies our kernel elf file as the kernel to run in the VM.

After running this, you should see "Hello, kernel World!" in your normal terminal. If you type in your terminal, it should echo every character.

Now that we have a kernel that boots, we should organize our project.

**Previous**:
[Part 1 - Setting up the Development Environment](/tutorial/dev-env.html)

**Next**:
[Part 3 - Organizing our Project](/tutorial/organize.html).
