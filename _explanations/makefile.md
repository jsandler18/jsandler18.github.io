---
layout: page
title:  An in-depth Explanation - Makefile
---

## Makefile source
Recall the source for the makefile

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

Lets step through this line by line

## Setting the Variables

```
CC = ./gcc-arm-none-eabi-6-2017-q2-update/bin/arm-none-eabi-gcc
```
This sets the compiler that we will be using to the cross compiler we downloaded when [setting up our development environment](/tutorial/dev-env.html)

<br>
```
ifeq ($(RASPI_MODEL),1)
    CPU = arm1176jzf-s
    DIRECTIVES = -D MODEL_1
else
    CPU = cortex-a7
endif
```
There are differences in hardware between raspberry pi models.  Since I am developing for a Model 2 VM and a real Model 1, I make the Model 2 stuff default, and have to trigger the Model 1 stuff manually.  I trigger this by passing `RASPI_MODEL=1` as a parameter to `make`.
In addition to setting the appropriate CPU, it also adds defines the preproccessor directive `MODEL_1`.  Any C code that depends on differing hardware, like the [peripheral base address](/extra/peripheral.html), can determine which hardware to compile for using `#ifdef MODEL_1` as a preprocessor directive.

<br>
```
CFLAGS= -mcpu=$(CPU) -fpic -ffreestanding $(DIRECTIVES)
CSRCFLAGS= -O2 -Wall -Wextra
LFLAGS= -ffreestanding -O2 -nostdlib
```
These are the just the same compiler and linker flags that [we used before](/tutorial/boot.html#compiling-and-running), but extracted into variables 

<br>
```
KER_SRC = ../src/kernel
KER_HEAD = ../include
COMMON_SRC = ../src/common
```
These lines specify the directories where the kernel source files, header files, and common source files reside, respectively.  They start with `../` since the Makefile will be running out of the build directory.

<br>
```
OBJ_DIR = objects
```
Similarly, this line specifies the location of a folder to store all binary object files.  This directory may not exist, and you should not create it.  It will be created by the make file when it is needed.

<br>
```
KERSOURCES = $(wildcard $(KER_SRC)/*.c)
COMMONSOURCES = $(wildcard $(COMMON_SRC)/*.c)
ASMSOURCES = $(wildcard $(KER_SRC)/*.S)
...
HEADERS = $(wildcard $(KER_HEAD)/*.h)
```
This gets a list of all the kernel source files, common source files, assembly source files, and header files, respectively.  `wildcard` is a make command that enumerates all files that match the pattern.  So `$(wildcard $(KER_SRC)/*.c)` means enumerate all files in the `KER_SRC` directory that ends with a `.c` extension.

<br>
```
OBJECTS = $(patsubst $(KER_SRC)/%.c, $(OBJ_DIR)/%.o, $(KERSOURCES))
OBJECTS += $(patsubst $(COMMON_SRC)/%.c, $(OBJ_DIR)/%.o, $(COMMONSOURCES))
OBJECTS += $(patsubst $(KER_SRC)/%.S, $(OBJ_DIR)/%.o, $(ASMSOURCES))
```
This takes the lists of kernel source files, common source files, and assembly source files, and adds them all to a list of compiled object files that will be created.  It does this using `patsubst`, which matches each element in the list passed as the third argument to the pattern in the first argument.  It discards anything matched and keeps anything covered by the `%`.  It then makes what remains match the pattern described in the second argument. For example, `$(patsubst $(KER_SRC)/%.c, $(OBJ_DIR)/%.o, $(KERSOURCES))` changes all files that look like `../src/kernel/file.c` to be `objects/file.o`.

<br>
```
IMG_NAME=kernel.img
```
This variable is the name of the kernel binary

## The Make Targets
Makefiles work through a series of targets.  Each target has a series of dependencies and a series of commands to execute. If any dependencies do not exist or are newer than the target, the commands are executed. A target is executed by running `make target`.  Our makefile has the following targets:

```
build: $(OBJECTS) $(HEADERS)
	echo $(OBJECTS)
	$(CC) -T linker.ld -o $(IMG_NAME) $(LFLAGS) $(OBJECTS)
```
This is the main target that compiles the kernel.  It can be invoked by running `make build`.  It depends on all of the object files for the respective source code files, and all the header files. This means that all the object files must be compiled before this can execute.  Its only command is to link all of the objects together into the final kernel binary.

<br>
```
$(OBJ_DIR)/%.o: $(KER_SRC)/%.c
	mkdir -p $(@D)
	$(CC) $(CFLAGS) -I$(KER_SRC) -I$(KER_HEAD) -c $< -o $@ $(CSRCFLAGS)
```
This target is any object file that depends on a kernel source file.  It creates the objects directory if it does not exist, then compiles the source file into the object file.  `-I` allows source files to access include files by `#include <kernel/header.h>` instead of `#include <../../include/kernel/header/h>`. `$<` is the name of the first dependency, so the c source file.  `$@` is the name of the target file itself.

There are two more targets that look very similar, but one is for kernel assembly files instead of kernel source files, and one is for common source files.

<br>
```
clean:
	rm -rf $(OBJ_DIR)
	rm $(IMG_NAME)
```
This target removes all of the compiled binaries so that no stale file is accidentally used when building

<br>
```
run: build
    qemu-system-arm -m 256 -M raspi2 -serial stdio -kernel kernel.img
```
This ensures that the kernel is compiled, and then runs the VM with that compiled kernel

This reduces the build and test process down to
```
make build
make run
```
