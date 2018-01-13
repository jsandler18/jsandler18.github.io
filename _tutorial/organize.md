---
layout: page
title:  Part 3 - Organizing our Project
---

This section is more of an exercise is C project organization than OS Dev.  If that doesn't interest you, you should skip ahead to [Part 4 - Wrangling
Memory](/tutorial/wrangling-mem.html)

If you want to download the code and play with it yourself, [see my git repo](https://github.com/jsandler18/raspi-kernel/tree/700b97a63c5f77a1ced814fe70d8560cba3ffacc).

## Separating our files

Right now, we have our C files, our linker file, our compiled objects, and our compiled kernel all floating around in the same directory.  Before we
start making the kernel more complex, it would be a good idea to separate different kinds of files.

The way that we are going to do it is to separate C files, header files, and compilation files into separate directories: `src`, `include`, and `build` respectively.
`src` and `include` are going to be structured very similarly, so when I talk about one, assmue it holds for the other unless otherwise stated.

`src` and `include` are going to have subdirectories `kernel` and `common`, where `kernel` is for files that are exclusive to the kernel, and `common` is for files that
contain standard functionality that may not be exclusive to the kernel, such as `memcpy`

`build` is going to contain our linker file, and a makefile.  During compilation, the makefile will put all object files, as well as the compiled kernel binary in the
`build` directory.

Here is the code for the makefile:

**Makefile**
```
# Don't use normal gcc, use the arm cross compiler
CC = ./gcc-arm-none-eabi-6-2017-q2-update/bin/arm-none-eabi-gcc

# Set any constants based on the raspberry pi model.  Version 1 has some differences to 2 and 3
ifeq ($(RASPI_MODEL),1)
    CPU = arm1176jzf-s
    DIRECTIVES = -D MODEL_1
else
    CPU = cortex-a7
endif

CFLAGS= -mcpu=$(CPU) -fpic -ffreestanding $(DIRECTIVES)
CSRCFLAGS= -O2 -Wall -Wextra
LFLAGS= -ffreestanding -O2 -nostdlib

# Location of the files
KER_SRC = ../src/kernel
KER_HEAD = ../include
COMMON_SRC = ../src/common
OBJ_DIR = objects
KERSOURCES = $(wildcard $(KER_SRC)/*.c)
COMMONSOURCES = $(wildcard $(COMMON_SRC)/*.c)
ASMSOURCES = $(wildcard $(KER_SRC)/*.S)
OBJECTS = $(patsubst $(KER_SRC)/%.c, $(OBJ_DIR)/%.o, $(KERSOURCES))
OBJECTS += $(patsubst $(COMMON_SRC)/%.c, $(OBJ_DIR)/%.o, $(COMMONSOURCES))
OBJECTS += $(patsubst $(KER_SRC)/%.S, $(OBJ_DIR)/%.o, $(ASMSOURCES))
HEADERS = $(wildcard $(KER_HEAD)/*.h)

IMG_NAME=kernel.img


build: $(OBJECTS) $(HEADERS)
    echo $(OBJECTS)
    $(CC) -T linker.ld -o $(IMG_NAME) $(LFLAGS) $(OBJECTS)

$(OBJ_DIR)/%.o: $(KER_SRC)/%.c
    mkdir -p $(@D)
    $(CC) $(CFLAGS) -I$(KER_SRC) -I$(KER_HEAD) -c $< -o $@ $(CSRCFLAGS)

$(OBJ_DIR)/%.o: $(KER_SRC)/%.S
    mkdir -p $(@D)
    $(CC) $(CFLAGS) -I$(KER_SRC) -c $< -o $@

$(OBJ_DIR)/%.o: $(COMMON_SRC)/%.c
    mkdir -p $(@D)
    $(CC) $(CFLAGS) -I$(KER_SRC) -I$(KER_HEAD) -c $< -o $@ $(CSRCFLAGS)

clean:
    rm -rf $(OBJ_DIR)
    rm $(IMG_NAME)

run: build
    qemu-system-arm -m 256 -M raspi2 -serial stdio -kernel kernel.img
```

For a line-by-line explanation of this code, [see this page](/explanations/makefile.html)

This leaves our directory structure like this:
```
raspi-kernel/
L__ src/
    L__ common/
    L__ kernel/
        L__ kernel.c
        L__ boot.S
L__ include/
    L__ common/
    L__ kernel/
L__ build/
    L__ Makefile
    L__ linker.ld
```

## Refactoring kernel.c

Right now, kernel.c contains all of the C source code for the entire kernel.  It contains the logic for setting up UART, and for doing IO.  Let's factor that stuff out into logical files.

First, we are going to put that [that big enum](/explanations/kernel_c.html#peripheral-specification-and-basic-read-and-write) from kernel.c that describes the UART peripheral, and all of the function signatures related to UART into `include/kernel/uart.h`. Then we are going to move all of the uart code to `src/kernel/uart.c`. This should leave `kernel.c` with just `kernel_main` in it.

Now, we are going to write some library code.  We create the files `src/common/stdlib.c` and `src/common/stdio.c` and corresponding header files.

In `stdlib.c`, we define the functions `memcpy` and `bzero`, as these will come in handy later, and we define `itoa` (integer to ascii) to make debugging easier.

In `stdio.c`, we define `getc`, `putc`, `gets` and `puts` as general purpose IO functions.  We do this even though `uart.c` had `uart_putc` and `uart_puts` because later we are going to want to swap out `uart_putc` for a function that renders text to an actuall screen, and it will be easier to replace one call to `uart_putc` here than many possible places.

The implementation of these functions is not that important. If you really want to see them, [check the git
repo](https://github.com/jsandler18/raspi-kernel/tree/a40c043d7ec93fe904bc6bc83a25e08f36a37cfb).

Now our directory structure looks like this:
```
raspi-kernel/
L__ src/
    L__ common/
        L__ stdio.c
        L__ stdlib.c
    L__ kernel/
        L__ kernel.c
        L__ boot.S
        L__ uart.c
L__ include/
    L__ common/
        L__ stdio.h
        L__ stdlib.h
    L__ kernel/
        L__ uart.h
L__ build/
    L__ Makefile
    L__ linker.ld
```

Now that our project is organized sensibly, lets look at how to manage memory.

**Previous**:
[Part 2 - Getting Something to Boot](/tutorial/boot.html)

**Next**:
[Part 4 - Wrangling Memory](/tutorial/dyn-mem.html)
