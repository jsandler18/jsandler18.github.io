---
layout: page
title:  An in-depth Explanation - interrupt_vector.S
---
## interrupt_vector.S source
Recall the source code for interrupt_vector.S:
```
.section ".text"

.equ KERNEL_STACK_SIZE, 4096
.equ IRQ_STACK_SIZE, 4096

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

reset_handler_abs_addr:                 .word reset_handler
undefined_instruction_handler_abs_addr: .word undefined_instruction_handler
software_interrupt_handler_abs_addr:    .word software_interrupt_handler
prefetch_abort_handler_abs_addr:        .word prefetch_abort_handler
data_abort_handler_abs_addr:            .word data_abort_handler
irq_handler_abs_addr:                   .word irq_handler
fast_irq_handler_abs_addr:              .word fast_irq_handler

move_exception_vector:
    push    {r4, r5, r6, r7, r8, r9}
    ldr     r0, =exception_vector 
    mov     r1, #0x0000
    ldmia   r0!,{r2, r3, r4, r5, r6, r7, r8, r9}
    stmia   r1!,{r2, r3, r4, r5, r6, r7, r8, r9}
    ldmia   r0!,{r2, r3, r4, r5, r6, r7, r8}
    stmia   r1!,{r2, r3, r4, r5, r6, r7, r8}
    mov     r0, #(0x12 | 0x80 | 0x40)   // Change to irq mode and set the irq stack pointer
    msr     cpsr_c, r0
    mov     r4, #(KERNEL_STACK_SIZE + IRQ_STACK_SIZE)
    ldr     sp, =__end
    add     sp, sp, r4
    mov     r0, #(0x13 | 0x80 | 0x40)   // change back to supervisor mode
    msr     cpsr_c, r0
    pop     {r4, r5, r6, r7, r8, r9}
    blx     lr
```

## Defining New Stacks
There are a couple constants defined at the top of the file:
```
.equ KERNEL_STACK_SIZE, 4096
.equ IRQ_STACK_SIZE, 4096
```
When we first got the [kernel to boot](/tutorial/boot.html), we said that the kernel stack should start at 0x8000 and grow down.  Now we are going to change that so that it starts 32 KB above the end of the kernel image and grows towards it.  These constants are also defined in boot.S, and the original instruction to set the stack pointer is changed to use this new address.  In order to accomidate the stack, we move the page metadata that used to start after the kernel image to start after the stack.  We make this change so that we may easily declare a seconds stack on top of this one.

The second stack that starts 32 KB above the start of the kernel stack.  Why do we need a second stack? Each exception has its own processor mode, and each mode has its own private stack pointer.  When an exception occurs, the program switches to use the exception's stack pointer.  We need to set up a stack in order for this allocation to make any sense.

## Writing the Exception Vector Table
Our strategy for this is to write out each branch instruction by hand and copy the instructions from the .text section to address 0 at runtime.

The first thing to notice is that our branch instructions are not
```
    ldr pc, irq_handler
```
but
```
    ldr pc, irq_handler_abs_addr
    ...
    irq_handler_abs_addr:                 .word irq_handler
```
This is because the `-fpic` flag to gcc, which means "Position Independent Code".  This means that if we were to write `ldr pc, irq_handler`, it would compile to `ldr pc, [pc, #offset_to_irq_handler]`.  It loads the address relative to its current position.  However, when we move the instuction to address 0, the handler would not be at the same relative offset.

To fix this, we need to put the absolute address of the handler into memory nearby, and load that address in relative to the current position.  This is what we did above.

## Moving the Exception Vector Table
`move_exception_vector` is a function that will be called from `interrupts_init` in `interrupt.c`.  I will walk through this line by line.

```
    push    {r4, r5, r6, r7, r8, r9}
```
This part saves the registers above r3 that the function will be using.  Calling C functions expect these registers be saved for them, so we must save them and restore them later.
```
    ldr     r0, =exception_vector 
```
Load the address of the exception vector into r0.
```
    mov     r1, #0x0000
```
r1 is the destination address.
```
    ldmia   r0!,{r2, r3, r4, r5, r6, r7, r8, r9}
    stmia   r1!,{r2, r3, r4, r5, r6, r7, r8, r9}
```
`ldm` loads the data in r0 into registers r2-r9. The `ia` suffix means that after storing to each register, incriment the address so the next register gets the next word of memory.  The `!` means to store the address after the last word of memory copied back into r0, so we can start from there again later. This loads the 8 exception words from their starting location into registers.  The next instruction does the same thing, but in reverse, writing the register values to address 0.
```
    ldmia   r0!,{r2, r3, r4, r5, r6, r7, r8}
    stmia   r1!,{r2, r3, r4, r5, r6, r7, r8}
```
This repeats the above, but this copies the absolute addresses to sit above the exception instructions.
```
    mov     r0, #(0x12 | 0x80 | 0x40)   // Change to irq mode and set the irq stack pointer
    msr     cpsr_c, r0
    mov     r4, #(KERNEL_STACK_SIZE + IRQ_STACK_SIZE)
    ldr     sp, =__end
    add     sp, sp, r4
    mov     r0, #(0x13 | 0x80 | 0x40)   // change back to supervisor mode
    msr     cpsr_c, r0
```
These instructions switches to the processor mode that the CPU will be in when an IRQ exception occurs.  It then loads the stack pointer up with the end of the kernel image plus the stack size, so the stack starts at the top of this reserved block of memory and grows down. Then it switches back to supervisor mode, which is what we started in.
```
    pop     {r4, r5, r6, r7, r8, r9}
    blx     lr
```
This restores the saved registers and returns to the caller.
