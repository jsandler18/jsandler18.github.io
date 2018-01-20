---
layout: page
title:  Part 07 - Interrupts
---
The next most useful thing we could do is run additional processes.  In order to do this, we first must set up exceptions and interrupts.  If you are
unfamiliar with these concepts, see [this page](/extra/interrupts.html).

If you want to download the code for this part and play with it for yourself, [see my git
repo](https://github.com/jsandler18/raspi-kernel/tree/cbab3ed75055e0c74bde36c5752e71575f10958d)

## Setting up the Exception Vector Table
Before we can handle IRQs, we must set up exception handlers, and we must set up the [exception vector table](/extra/interrupts.html) to jump to those handlers.

Exception handlers are functions, but they cannot be normal functions, as exception handlers need more advanced prologue and epilogue code than a normal function.  Our
compiler can handle this for us by using an attribute.  Here is what the IRQ handler signature looks like:
``` c
void __attribute__ ((interrupt ("SWI"))) software_interrupt_handler(void);
```
Where `"SWI"` would be replaced with `"IRQ"` for the interrupt handler, `"FIQ"` for the FIQ handler, `"ABORT"` for the Reset, Data Abort, and Prefetch Abort handlers, and
`"UNDEF"` for the Undefined Instruction handler.  We want to have handlers defined for all exceptions, even if we don't really know what to do with them yet.  For now
lets declare them in `src/kernel/interrupt.c` as dummy functions as follows:
``` c
void irq_handler(void) {
    printf("IRQ HANDLER\n");
    while(1);
}

void __attribute__ ((interrupt ("ABORT"))) reset_handler(void) {
    printf("RESET HANDLER\n");
    while(1);
}
void __attribute__ ((interrupt ("ABORT"))) prefetch_abort_handler(void) {
    printf("PREFETCH ABORT HANDLER\n");
    while(1);
}
void __attribute__ ((interrupt ("ABORT"))) data_abort_handler(void) {
    printf("DATA ABORT HANDLER\n");
    while(1);
}
void __attribute__ ((interrupt ("UNDEF"))) undefined_instruction_handler(void) {
    printf("UNDEFINED INSTRUCTION HANDLER\n");
    while(1);
}
void __attribute__ ((interrupt ("SWI"))) software_interrupt_handler(void) {
    printf("SWI HANDLER\n");
    while(1);
}
void __attribute__ ((interrupt ("FIQ"))) fast_irq_handler(void) {
    printf("FIQ HANDLER\n");
    while(1);
}
```
This is the quick and easy way to define exception handlers that we don't really care about yet.  We didn't include an attribute on the IRQ handler because we want to have a custom handler in assembly that will call irq_handler.

Now that we have defined the handlers, we can set up the exception table.  This is done entirely in assembly to avoid gcc "optimizing" our access of memory address 0. Here is the code:

**interrupt_vector.S**
```
.section ".text"

.global move_exception_vector

exception_vector:
    ldr pc, reset_handler_abs_addr
    ldr pc, undefined_instruction_handler_abs_addr
    ldr pc, software_interrupt_handler_abs_addr
    ldr pc, prefetch_abort_handler_abs_addr
    ldr pc, data_abort_handler_abs_addr
    nop                                         // This one is reserved
    ldr pc, irq_handler_abs_addr
    ldr pc, fast_irq_handler_abs_addr

reset_handler_abs_addr:                 .word reset_handler
undefined_instruction_handler_abs_addr: .word undefined_instruction_handler
software_interrupt_handler_abs_addr:    .word software_interrupt_handler
prefetch_abort_handler_abs_addr:        .word prefetch_abort_handler
data_abort_handler_abs_addr:            .word data_abort_handler
irq_handler_abs_addr:                   .word irq_handler_asm_wrapper
fast_irq_handler_abs_addr:              .word fast_irq_handler

move_exception_vector:
    push    {r4, r5, r6, r7, r8, r9}
    ldr     r1, =exception_vector
    mov     r1, #0x0000
    ldmia   r0!,{r2, r3, r4, r5, r6, r7, r8, r9}
    stmia   r1!,{r2, r3, r4, r5, r6, r7, r8, r9}
    ldmia   r0!,{r2, r3, r4, r5, r6, r7, r8}
    stmia   r1!,{r2, r3, r4, r5, r6, r7, r8}
    pop     {r4, r5, r6, r7, r8, r9}
    blx     lr

irq_handler_asm_wrapper:
    sub     lr, lr, #4
    srsdb   sp!, #0x13
    cpsid   if, #0x13
    push    {r0-r3, r12, lr}
    and     r1, sp, #4
    sub     sp, sp, r1
    push    {r1}
    bl      irq_handler
    pop     {r1}
    add     sp, sp, r1
    pop     {r0-r3, r12, lr}
    rfeia   sp!
```
There is too much to explain about this code here, so it is explained in detail on [this page](/explanations/interrupt_vector_S.html)

## Handling IRQs
Now that we have set up the IRQ exception handler, we need to add the ability to find out which IRQ was triggered, and the ability to handle that IRQ. To do this, we are going to need to read and write to the [IRQ peripheral](/extra/interrupts.html).  We can easily acomplish this using a C struct:
``` c
typedef struct {
    uint32_t irq_basic_pending;
    uint32_t irq_gpu_pending1;
    uint32_t irq_gpu_pending2;
    uint32_t fiq_control;
    uint32_t irq_gpu_enable1;
    uint32_t irq_gpu_enable2;
    uint32_t irq_basic_enable;
    uint32_t irq_gpu_disable1;
    uint32_t irq_gpu_disable2;
} interrupt_registers_t;
```

For each possible IRQ, we are going to want to have a specialized handler function.  Instead of writing each specialized handler inside of `interrupts.c`, we are going to expose an interface to register interrupt handlers from outside of `interrupts.c`  First, we define the type of an interrupt handler:
``` c
typedef void (*interrupt_handler_f)(void);
```
This is a pointer to a function which has no parameters and no return value.  We can then store an array of these function pointers in `interrupt.c`.  Since there are three 4-byte words representing possible interrupts, but the last 3 bytes of the basic interrupts are repeats of others, there are 72 different interrupts that could be handled, so we declare an array of 72 handlers.
``` c
static interrupt_handler_f handlers[72];
```
The Raspberry Pi documentation says that the interrupt pending flag cannot be cleared using the interrupt peripheral.  It must be cleared using whatever hardware peripheral that triggered the interrupt.  For this reason, we will also want users to register a specialized clearer function for that particular interrupt:
``` c
typedef void (*interrupt_clearer_f)(void);
...
static interrupt_clearer_f clearers[72];
```

To register a handler, we use the following code:
```c
void register_irq_handler(irq_number_t irq_num, interrupt_handler_f handler, interrupt_clearer_f clearer) {
    uint32_t irq_pos;
    if (IRQ_IS_BASIC(irq_num)) {
        irq_pos = irq_num - 64;
        handlers[irq_num] = handler;
        clearers[irq_num] = clearer;
        interrupt_regs->irq_basic_enable |= (1 << irq_pos);
    }
    else if (IRQ_IS_GPU2(irq_num)) {
        irq_pos = irq_num - 32;
        handlers[irq_num] = handler;
        clearers[irq_num] = clearer;
        interrupt_regs->irq_gpu_enable2 |= (1 << irq_pos);
    }
    else if (IRQ_IS_GPU1(irq_num)) {
        irq_pos = irq_num;
        handlers[irq_num] = handler;
        clearers[irq_num] = clearer;
        interrupt_regs->irq_gpu_enable1 |= (1 << irq_pos);
    }
}
```
where `IRQ_IS_BASIC`, `IRQ_IS_GPU1`, and `IRQ_IS_GPU2` are macros that check whether the irq number is in that range, and `irq_number_t` is an enum that names each interrupt number.

In order to check which IRQs have been triggered and execute the handler, we must check the enabled bits of the IRQ peripheral, and execute the correct handler accordingly:
```c
void irq_handler(void) {
    int j;
    for (j = 0; j < NUM_IRQS; j++) {
        // If the interrupt is pending and there is a handler, run the handler
        if (IRQ_IS_PENDING(interrupt_regs, j)  && (handlers[j] != 0)) {
            clearers[j]();
            ENABLE_INTERRUPTS();
            handlers[j]();
            DISABLE_INTERRUPTS();
            return;
        }
    }
}
```
This code simply iterates over all irq pending flags.  If an interrupt is pending and there is a corresponding handler, the clearer is called, then interrupts are enabled so to allow for "nesting" of interrupts, the handler is called and interrupts are disabled to finish off the interrupt.

## Initializing Interrupts
We now have all the necessary infrastructure in place to have working interrupts.  All we need to do is intialize it.  Here is the code to do so:
``` c
static interrupt_registers_t * interrupt_regs;

static interrupt_handler_f handlers[72];
static interrupt_clearer_f clearers[72];

extern void move_exception_vector(void);

void interrupts_init(void) {
   interrupt_regs = (interrupt_registers_t *)INTERRUPTS_PENDING;
    bzero(handlers, sizeof(interrupt_handler_f) * NUM_IRQS);
    bzero(clearers, sizeof(interrupt_clearer_f) * NUM_IRQS);
    interrupt_regs->irq_basic_disable = 0xffffffff; // disable all interrupts
    interrupt_regs->irq_gpu_disable1 = 0xffffffff;
    interrupt_regs->irq_gpu_disable2 = 0xffffffff;
    move_exception_vector();
    ENABLE_INTERRUPTS();
}
```
This first initializes the interrupt register struct.  Then it zeros out all handlers.  Then it disables all interrupts, as we want to enable them as they are registered. Then we call `move_exception_vector` which is defined in `interrupt_vector.S` to copy the exception vector table to address 0.  Finally we enable interrups.

`ENABLE_INTERRUPTS` and `DISABLE_INTERRUPTS` (not called here) are inline functions that will be called frequently as we write more functionality. Here is the code:
``` c
__inline__ int INTERRUPTS_ENABLED(void) {
    int res;
    __asm__ __volatile__("mrs %[res], CPSR": [res] "=r" (res)::);
    return ((res >> 7) & 1) == 0;
}
__inline__ void ENABLE_INTERRUPTS(void) {
    if (!INTERRUPTS_ENABLED()) {
        __asm__ __volatile__("cpsie i");
    }
}
```
`INTERRUPTS_ENABLED` loads the Current Program Status Register, or `CPSR` into a register.  It then checks bit 7.  If bit 7 is clear, then interrupts are enabled.  `ENABLE_INTERRUPTS` executes the `cps` (Change Processor State) instruction with the `ie` (Interrupts Enable).  The argument `i` means to enable IRQs as opposed to `f` for FIQs.  `DISABLE_INTERRUPTS` works exactly like `ENABLE_INTERRUPTS`, except the suffix is `id` (Interrupts Disable).

## The System Timer

Interrupts are useless unless we have something to actually interrupt us!  We are going to set up the [system timer peripheral](/extra/sys-time.html), as this will be useful in setting up processes.

We declare a C struct to map out the system timer peripheral:
``` c
typedef struct {
    uint8_t timer0_matched: 1;
    uint8_t timer1_matched: 1;
    uint8_t timer2_matched: 1;
    uint8_t timer3_matched: 1;
    uint32_t reserved: 28;
} timer_control_reg_t;

typedef struct {
    timer_control_reg_t control;
    uint32_t counter_low;
    uint32_t counter_high;
    uint32_t timer0;
    uint32_t timer1;
    uint32_t timer2;
    uint32_t timer3;
} timer_registers_t;
```

Next we define an interrupt handler and a clearer.  We don't need it to do anything advanced right now.  We just want it to be obvious that it works:
``` c
static timer_registers_t * timer_regs;

static void timer_irq_handler(void) {
    printf("timeout :)\n");
    timer_set(3000000);
}

static void timer_irq_clearer(void) {
    timer_regs->control.timer1_matched = 1;
}
```
All we do here is print something and reset the timer.

All that is left is to register this function with the interrupt system:
``` c
void timer_init(void) {
    timer_regs = (timer_registers_t *) SYSTEM_TIMER_BASE;
    register_irq_handler(SYSTEM_TIMER_1, timer_irq_handler, timer_irq_clearer);
}
```

Now in `kernel_main`, we can add:
``` c
...
mem_init();
interrupts_init();
timer_init();
timer_set(1000000); // 1 second
puts("Hello, kernel World!\n");
...
```
to see our timer interupt in action.

Next, we are going to look at getting another process to run.

**Previous**:
[Part 6 - Printing to a Real Screen](/tutorial/hdmi.html)

**Next**:
[Part 8 - Processes](/tutorial/process.html)

\*\*\***To those following along using a VM**

At the time of writing this, QEMU's implementation of the Raspberry Pi Model 2 does not have the system timer implemented.  Therefore, this part and any subsequent part that relies on it will not work on the VM.
