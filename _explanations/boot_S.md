---
layout: page
title:  An in-depth Explanation - boot.S
---


## boot.S source
Recall the code for boot.S
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
Let's step through this line by line

## Setting up the C Runtime Environment

```
.section ".text.boot"

.globl _start
```
These are notes to the linker.  The first is about where this code belongs in the compiled binary.  In a little bit, we are going to specify where that is.  The second specifies that \_start is a name that should be visible from outside of the assembly file

<br>
```
_start:
    mov sp, #0x8000
```
This is the first instruction of our kernel.  It says that our C stack should start at address 0x8000 and grow downwards.  Why 0x8000?
Well when the hardware loads our kernel in to memory, it does not load it into address 0, but to address 0x8000.  Since our kernel runs from 0x8000 and up, our stack can safely run from 0x8000 and down without clobbering our kernel.

<br>
```
    ldr r4, =__bss_start
    ldr r9, =__bss_end
```
This loads the addresses of the start end end of the BSS section into registers.  BSS is where C global variables that are not initialized at compile time are stored.  The C runtime requires that uninitialized global variables are zero, so we must zero out this entire section ourselves.  The symbols `__bss_start` and `__bss_end` are going to be defined later in when we work with the linker, so don't worry about where they come from for now.

<br>
```
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
```
This code is what zeros out the BSS section.  First it loads 0 into four consecutive registers.  Then it checks whether the address stored in r4 is less than the one in r9.  If it is, then it executes `stmia r4!, {r5-r8}`.  The `stm` instruction stores the second operand into the address contained in the first.  The `ia` suffix on the instruction means *increment after*, or increment the address in r4 to the address after the last address written by the instruction.  The `!` means store that address back in r4, as opposed to throwing it out.  The `{r5-r8}` operand means that `stm` should store the values in the consecutive registers r5,r6,r7,r8 (so 16 bytes) into r4.  So overall, the instruction stores 16 bytes of zeros into the address in r4, then increments that address by 16 bytes.  This loops until r4 is greater than or equal to r9, and the whole BSS section is zeroed out.

<br>
```
    ldr r3, =kernel_main
    blx r3

halt:
    wfe
    b halt
```
This loads the address of the C function called `kernel_main` into a register and jumps to that location.  When the C function returns, it enters the `halt` procedure where it loops forever doing nothing.
