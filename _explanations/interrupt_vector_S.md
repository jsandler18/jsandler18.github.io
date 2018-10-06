---
layout: page
title:  An in-depth Explanation - interrupt_vector.S
---
## interrupt_vector.S source
Recall the source code for interrupt_vector.S:
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
    ldr     r0, =exception_vector
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
    pop     {r4, r5, r6, r7, r8, r9}
    blx     lr
```
This restores the saved registers and returns to the caller.

## A Custom Handler Wrapper
Exceptions cannot just jump to normal functions.  There are certain things that must be taken care of before and after the normal function is executed.  By using the `__attribute__((interrupt("FIQ")))`, we can embed a default version of this special prologue and epilogue directly in the function.  However, it is very minimal, and for our IRQ handler, we need a bit of customization.  Let's break down the code:

```
    sub     lr, lr, #4
```
Due to a quirk with how arm does exceptions, we have to adjust the return address to be one instruction back.

```
    srsdb   sp!, #0x13
```
When an IRQ exception occurs, the processor switches from whatever mode it was in to IRQ mode.  IRQ mode has its own stack and its own versions of a few registers, including `sp` and `cpsr`, that are separate from the normal registers.  It will make life easier in the long run if we change from IRQ mode back to supervisor mode, which is what we started with.

This instruction stores `lr`(the return address), and `spsr` (the general `cpsr` register that is shadowed by the IRQ mode's own version) to the stack of mode 0x13 (supervisor mode), and then uses that mode's stack pointer.

```
    cpsid   if, #0x13
```
This instruction switches to supervisor mode with interrupts disabled.

```
    push    {r0-r3, r12, lr}
```
This saves all of the caller-save registers.  Any function we call will save the registers `r4-r11` for us and restore them for us, so we don't notice the difference.  We have to save `r0-r3`, `r12`, and `lr`.  Normal functions usually just accept that these registers are clobbered and may not save them.  Since we are interrupting some other code that did not consent to calling a function, we must preserve all of the registers so it does not notice something happened when control is returned to it.

```
    and     r1, sp, #4
    sub     sp, sp, r1
```
According to the ARM documentation, the stack must be 8 byte aligned when calling functions, though the exception handler may leave us with a stack that is not 8 byte aligned.  This corrects that.

```
    bl      irq_handler
```
This calls our c code handler

```
    add     sp, sp, r1
```
This restores the stack alignment

```
    pop     {r0-r3, r12, lr}
```
This restores the caller-save registers

```
    rfeia   sp!
```
This restores the stored `cpsr` and returns to the address in the stored `lr`
