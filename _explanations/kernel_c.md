---
layout: page
title:  An in-depth Explanation - kernel.c
---

## kernel.c source
Recall the code for kernel.c
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
}

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



## Peripheral Specification and Basic Read and Write
If you are unfamiliar with memory mapped IO on the Raspberry Pi, I reccomend you read [Memory Mapped IO, Peripherals, and Registers](/extra/peripheral.html) before
continuing.

``` c
static inline void mmio_write(uint32_t reg, uint32_t data)
{
    *(volatile uint32_t*)reg = data;
}

static inline uint32_t mmio_read(uint32_t reg)
{
    return *(volatile uint32_t*)reg;
}

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
```
`mmio_write` and `mmio_read` both take as input a register, which is an absolute address that is going to look something like `0x20000000 + peripheral base + register offset`.  Write takes a 4 byte word to write to the register, while read returns whatever 4 byte word was in the register.  `delay` is just a function that busy loops for a while.  It is a very imprecise way of giving the hardware some time to respond to any writes we may have made.

The enum defines the peripheral offset of the GPIO and the UART hardware systems, as well as some of their registers.  Don't get caught up worrying about what each register is, as I am going to explain them as they are used.

## Setting up the Hardware
``` c
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

```
This is the `uart_init` function.  It sets up the UART hardware for use.  It consists of just setting come configuration flags in various registers.

<br>

``` c
    mmio_write(UART0_CR, 0x00000000);
```
This line disables all aspects of the UART hardware.  UART0_CR is the UART's Control Register.

<br>
``` c
    mmio_write(GPPUD, 0x00000000);
    delay(150);

    mmio_write(GPPUDCLK0, (1 << 14) | (1 << 15));
    delay(150);

    mmio_write(GPPUDCLK0, 0x00000000);
```
These lines disable pins 14 and 15 of the GPIO.  Writing 0 to GPPUD marks that pins should be disabled.  Writing `(1 << 14) | (1 << 15)` to GPPUDCLK0 marks which pins
should be disabled, and writing 0 to GPPUDCLK0 makes the whole thing take effect.  Since we aren't using the GPIO pins, this part isn't really that important

<br>
``` c
    mmio_write(UART0_ICR, 0x7FF);
```
This line sets all flags in the Interrupt Clear Register.  This has the effect of clearing all pending interrupts from the UART hardware.  What interrupts actually are and how they work have their own section, so I won't talk about that here.

<br>
``` c
    mmio_write(UART0_IBRD, 1);
    mmio_write(UART0_FBRD, 40);
```
This sets the *baud rate* of the connection.  This is essentially the bits-per-second that can go across the serial port.  This code is trying to get a baud rate of 115200.  To set the baud rate, we must calcuate a *baud rate divisor* and plug that into some registers.  The divisor is calcluated by `UART_CLOCK_SPEED/(16 *
         DESIRED_BAUD)`. The integer part of this calculation goes into IBRD the register of the UART. This calculation likely does not yield an integer, and in our case,
it winds up being about 1.67.  This means we store 1 into IBRD and then we must
also must calculate a *fractional baud rate divisor* from the fractional part of the previous calculation using this formula `(.67 * 64) + .5`.
This winds up being about 40, so we set the FBRD register to 40.

<br>
``` c
    mmio_write(UART0_LCRH, (1 << 4) | (1 << 5) | (1 << 6));
```
This writes bit 4, 5, and 6 to the Line control register.  Setting bit 4 means that the UART hardware will hold data in an 8 item deep FIFO, instead of a 1 item deep
register.  Setting 5 and 6 to 1 means that data sent or received will have 8-bit long words.

<br>
``` c
    mmio_write(UART0_IMSC, (1 << 1) | (1 << 4) | (1 << 5) | (1 << 6) |
            (1 << 7) | (1 << 8) | (1 << 9) | (1 << 10));
```
This disables all interrupts from the UART by writing a one to the relevent bits of the Interrupt Mask Set Clear register.

<br>
``` c
    mmio_write(UART0_CR, (1 << 0) | (1 << 8) | (1 << 9));
```
This writes bits 0, 8, and 9 to the control register.  Bit 0 enables the UART hardware, bit 8 enables the ability to receive data, and bit 9 enables the ability to
transmit data.

## Reading and Writing Text
``` c
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
```
This code enables reading and writing characters to and from the UART.  FR is the flags register, and it tells us whether the read FIFO has any data for us to read, and
whether the write FIFO can accept any data.  DR is the data register, which is where data is both read from and written to.
the `uart_puts` function just wraps up putc in a loop so we can write whole strings.

## The Heart of the Kernel
``` c
void kernel_main(uint32_t r0, uint32_t r1, uint32_t atags)
{
    // Declare as unused
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
This is the main function for our kernel.  This is where control is transfered to from boot.S.  All it does is call the `init_uart` function, print "Hello, kernel World!",
and print out any character you type.  This is where we will add calls to many other initialization functions.

The arguments to this function look a bit weird.  Normally, the `main` function of C code looks something like `int main(int argc, char ** argv)`, but our `kernel_main` looks nothing like that.  In ARM, the convention is that the first three parameters of a function are passed through registers r0, r1 and r2.  When the bootloader loads our kernel, it also places some information about the hardware and the command line used to run the kernel in memory.  This information is called *atags*, and a pointer to the atags is placed in r2 just before boot.S runs.  So for our `kernel_main`, r0 and r1 are parameters to the function simply by convention, but we don't care about those.  r2 contains the atags pointer, so the third argument to `kernel_main` is the atags pointer.

