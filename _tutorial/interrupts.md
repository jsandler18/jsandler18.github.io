---
layout: page
title:  Part 6 - Interrupts
---
The next most usefule thing we could do is get input from a real USB keyboard.  In order to do this, we first must set up exceptions and interrupts.  If you are
unfamiliar with these concepts, see [this page](/extra/interrupts.html).

If you want to download the code for this part and play with it for yourself, [see my git
repo](https://github.com/jsandler18/raspi-kernel/tree/c7abe6f5ed05c9ed8c44e6a7f30982fdc3a47676)

## Setting up the Exception Vector Table
Before we can handle IRQs, we must set up exception handlers, and we must set up the [exception vector table](/extra/interrupts.html) to jump to those handlers.

Exception handlers are functions, but they cannot be normal functions, as exception handlers need more advanced prologue and epilogue code than a normal function.  Our
compiler can handle this for us by using an attribute.  Here is what the IRQ handler signature looks like:
``` c
void __attribute__ ((interrupt ("IRQ"))) irq_handler(void);
```
Where `"IRQ"` would be replaced with `"FIQ"` for the FIQ handler, `"SWI"` for the SWI handler, `"ABORT"` for the Reset, Data Abort, and Prefetch Abort handlers, and
`"UNDEF"` for the Undefined Instruction handler.  We want to have handlers defined for all exceptions, even if we don't really know what to do with them yet.  For now
lets declare them in `src/kernel/interrupt.c` as dummy functions as follows:
``` c
void __attribute__ ((interrupt ("IRQ"))) irq_handler(void) {
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
Now that we have defined the handlers, we can set up the exception table.  This is done entirely in assembly to avoid gcc "optimizing" our access of memory address 0. Here is the code:

**interrupt_vector.S**
```
.section ".text"

.equ KERNEL_HEAP_START, (0x100000) // Heap starts at 1 MB
.equ KERNEL_HEAP_SIZE, (1024*1024) // Kernel heap is 1 MB large
.equ KERNEL_STACK_SIZE, (1024 * 32)
.equ KERNEL_STACK_START, (KERNEL_HEAP_START + KERNEL_HEAP_SIZE + KERNEL_STACK_SIZE)
.equ IRQ_STACK_SIZE, (1024 * 32)
.equ IRQ_STACK_START, (KERNEL_STACK_START + IRQ_STACK_SIZE)

.global move_exception_vector
.global exception_vector

exception_vector:
    ldr pc, reset_handler_abs_addr
    ldr pc, undefined_instruction_handler_abs_addr
    ldr pc, software_interrupt_handler_abs_addr
    ldr pc, prefetch_abort_handler_abs_addr
    ldr pc, data_abort_handler_abs_addr
    nop                                         // This one is reserved
    ldr pc, irq_handler_abs_addr
    ldr pc, fast_irq_handler_abs_addr

reset_handler_abs_addr:                 .word (reset_handler + 0x8000)
undefined_instruction_handler_abs_addr: .word (undefined_instruction_handler + 0x8000)
software_interrupt_handler_abs_addr:    .word (software_interrupt_handler + 0x8000)
prefetch_abort_handler_abs_addr:        .word (prefetch_abort_handler + 0x8000)
data_abort_handler_abs_addr:            .word (data_abort_handler + 0x8000)
irq_handler_abs_addr:                   .word (irq_handler + 0x8000)
fast_irq_handler_abs_addr:              .word (fast_irq_handler + 0x8000)

move_exception_vector:
    push    {r4, r5, r6, r7, r8, r9}
    ldr     r0, =exception_vector 
    add     r0, r0, #0x8000
    mov     r1, #0x0000
    ldmia   r0!,{r2, r3, r4, r5, r6, r7, r8, r9}
    stmia   r1!,{r2, r3, r4, r5, r6, r7, r8, r9}
    ldmia   r0!,{r2, r3, r4, r5, r6, r7, r8}
    stmia   r1!,{r2, r3, r4, r5, r6, r7, r8}
    mov     r0, #(0x12 | 0x80 | 0x40)   // Change to irq mode and set the irq stack pointer
    msr     cpsr_c, r0
    mov     sp, #IRQ_STACK_START
    mov     r0, #(0x13 | 0x80 | 0x40)   // change back to supervisor mode
    msr     cpsr_c, r0
    pop     {r4, r5, r6, r7, r8, r9}
    blx     lr
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
This is a pointer to a function which has no parameters and no return value.  We can then store an array of these function pointers in `interrupt.c`.  Since there are three 4 byte words representing interrupts, we will have one array for each word:
``` c
static interrupt_handler_f gpu1_handlers[32];
static interrupt_handler_f gpu2_handlers[32];
static interrupt_handler_f basic_handlers[32];
```
To register a handler, we use the following code:
```c
void register_irq_handler(irq_number_t irq_num, interrupt_handler_f handler) {
    uint32_t irq_pos;
    if (IRQ_IS_BASIC(irq_num)) {
        irq_pos = irq_num - 64;
        basic_handlers[irq_pos] = handler;
        interrupt_regs->irq_basic_enable |= (1 << irq_pos);
    }
    else if (IRQ_IS_GPU2(irq_num)) {
        irq_pos = irq_num - 32;
        gpu2_handlers[irq_pos] = handler;
        interrupt_regs->irq_gpu_enable2 |= (1 << irq_pos);
    }
    else if (IRQ_IS_GPU1(irq_num)) {
        irq_pos = irq_num;
        gpu1_handlers[irq_pos] = handler;
        interrupt_regs->irq_gpu_enable1 |= (1 << irq_pos);
    }
}
```
where `IRQ_IS_BASIC`, `IRQ_IS_GPU1`, and `IRQ_IS_GPU2` are macros that check whether the irq number is in that range, and `irq_number_t` is an enum that names each interrupt number.

In order to check which IRQs have been triggered and execute the handler, we must check the enabled bits of the IRQ peripheral, and execute the correct handler accordingly:
```c
void __attribute__ ((interrupt ("IRQ"))) irq_handler(void) {
    int j;
    for (j = 0; j < 7; j++) {
        // If the interrupt is pending and there is a handler, run the handler
        if (((1 << j) & interrupt_regs->irq_basic_pending) && (basic_handlers[j] != 0)) {
            // the pending bit is cleared by the handler
            basic_handlers[j]();
        }

    }
    for (j = 0; j < 32; j++) {
        if (((1 << j) & interrupt_regs->irq_gpu_pending1) && (gpu1_handlers[j] != 0)) {
            gpu1_handlers[j]();
        }

    }
    for (j = 0; j < 32; j++) {
        if (((1 << j) & interrupt_regs->irq_gpu_pending2) && (gpu2_handlers[j] != 0)) {
            gpu2_handlers[j]();
        }

    }
}
```
This code simply iterates over all irq pending flags.  If an interrupt is pending and there is a corresponding handler, then the handler will be called.

## Initializing Interrupts
We now have all the necessary infrastructure in place to have working interrupts.  All we need to do is intialize it.  Here is the code to do so:
``` c
static interrupt_registers_t * interrupt_regs;

static interrupt_handler_f gpu1_handlers[32];
static interrupt_handler_f gpu2_handlers[32];
static interrupt_handler_f basic_handlers[32];

extern void move_exception_vector(void);

void interrupts_init(void) {
    interrupt_regs = (interrupt_registers_t *)INTERRUPTS_PENDING;
    bzero(gpu1_handlers, sizeof(interrupt_handler_f) * 32);
    bzero(gpu2_handlers, sizeof(interrupt_handler_f) * 32);
    bzero(basic_handlers, sizeof(interrupt_handler_f) * 32);
    interrupt_regs->irq_basic_disable = 0xffffffff; // disable all basic interrupts
    move_exception_vector();
    ENABLE_INTERRUPTS();
}
```
This first initializes the interrupt register struct.  Then it zeros out all handlers.  Then it disables all ARM Specific interrupts, as we don't need any of those right now.  We don't disable any GPU interrupts as those are poorly documented and I don't want to disable anything important for the GPU.  Then we call `move_exception_vector` which is defined in `interrupt_vector.S` to copy the exception vector table to address 0.  Finally we enable interrups.

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
Interrupts are useless unless we have something to actually interrupt us!  The ultimate goal is to have the USB controller interrupt us, but the USB controller is hard to set up, so instead let's set up the [system timer peripheral](/extra/sys-time.html).

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

Next we define an interrupt handler.  We don't need it to do anything advanced right now.  We just want it to be obvious that it works:
``` c
static timer_registers_t * timer_regs;

static void timer_irq_handler(void) {
    timer_regs->control.timer1_matched = 0; // Writing to this clears the pending bit in the irq
    printf("timeout :)\n");
}
```
All we do here is clear the interrupt pending flag and print something.

All that is left is to register this function with the interrupt system:
``` c
void timer_init(void) {
    timer_regs = (timer_registers_t *) SYSTEM_TIMER_BASE;
    register_irq_handler(SYSTEM_TIMER_1, timer_irq_handler);
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

Next, we are going to look at how to get the USB keyboard to work.

**Previous**:
[Part 5 - Printing to a Real Screen](/tutorial/hdmi.html)

**Next**:
[Part 7 - Typing on a Real Keyboard](/tutorial/keyboard.html)
