---
layout: page
title:  An in-depth Explanation - interrupt_vector.S
---
## interrupt_vector.S source
Recall the source code for interrupt_vector.S:
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

## Defining New Stacks
There are a few constants defined at the top of the file.  They all define locations and sizes for sections of memory.
```
.equ KERNEL_HEAP_START, (0x100000) // Heap starts at 1 MB
.equ KERNEL_HEAP_SIZE, (1024*1024) // Kernel heap is 1 MB large
```
When we did [dynamic memory](/tutorial/dyn-mem.html), we defined exactly these values as the beginning and length of the kernel heap for `kmalloc`.  Since we can't `#include` the header file with those constants, we redefine them here.
```
.equ KERNEL_STACK_SIZE, (1024 * 32)
.equ KERNEL_STACK_START, (KERNEL_HEAP_START + KERNEL_HEAP_SIZE + KERNEL_STACK_SIZE)
```
When we first got the [kernel to boot](/tutorial/boot.html), we said that the kernel stack should start at 0x8000 and grow down.  Now we are going to change that so that it starts 32 KB above the heap and grows towards it.  These constants are also defined in boot.S, and the original instruction to set the stack pointer is changed to use this new address.  We make this change so that we may easily declare a seconds stack on top of this one.
```
.equ IRQ_STACK_SIZE, (1024 * 32)
.equ IRQ_STACK_START, (KERNEL_STACK_START + IRQ_STACK_SIZE)
```
Here we define a second stack that starts 32 KB above the start of the kernel stack.  Why do we need a second stack? Each exception has its own processor mode, and each mode has its own private stack pointer.  When an exception occurs, the program switches to use the exception's stack pointer.  We need to set up a stack in order for this allocation to make any sense.

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
    irq_handler_abs_addr:                 .word (irq_handler + 0x8000)
```
This is because the `-fpic` flag to gcc, which means "Position Independent Code".  This means that if we were to write `ldr pc, irq_handler`, it would compile to `ldr pc, [pc, #offset_to_irq_handler]`.  It loads the address relative to its current position.  However, when we move the instuction to address 0, the handler would not be at the same relative offset.

To fix this, we need to put the absolute address of the handler into memory nearby, and load that address in relative to the current position.  This is what we did above.

In this solution, we added 0x8000 to the address of the handler.  We did this because all of the addresses are offsets from address 0, but remember that the kernel loads at address 0x8000.  This means that the absolute addresses of the handlers are 0x8000 above what they are compiled to be.  This is why we use PIC in the first place.  If you are working through the VM, you do not need to add 0x8000, as it does not actually load the kernel at offset 0x8000.

## Moving the Exception Vector Table
`move_exception_vector` is a function that will be called from `interrupts_init` in `interrupt.c`.  I will walk through this line by line.

```
    push    {r4, r5, r6, r7, r8, r9}
```
This part saves the registers above r3 that the function will be using.  Calling C functions expect these registers be saved for them, so we must save them and restore them later.
```
    ldr     r0, =exception_vector 
    add     r0, r0, #0x8000
```
Load the address of the exception vector into r0, and since this address is based on 0 and not on 0x8000, we must add 0x8000.
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
    mov     sp, #IRQ_STACK_START
    mov     r0, #(0x13 | 0x80 | 0x40)   // change back to supervisor mode
    msr     cpsr_c, r0
```
These instructions switches to the processor mode that the CPU will be in when an IRQ exception occurs.  It then loads the stack pointer up withthe top of our stack that we defined. Then it switches back to supervisor mode, which is what we started in.
```
    pop     {r4, r5, r6, r7, r8, r9}
    blx     lr
```
This restores the saved registers and returns to the caller.
