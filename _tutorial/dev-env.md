---
layout: page
title:  Part 1 - Setting up the Development Environment
date:   2017-12-29 11:07:26 -0500
---

Before we can dive in to creating an operating system, first we have to set up our development environment.  For many tutorials, this can be difficult, as often you will
have mysterious errors that the tutorial does not mention, you are left stuck before you even begin.  Here I am going to describe the steps that I went through to get my
dev environment set up.  I am, however, going to assume that you are working in a linux environment.

# The Compiler

Most likely, your computer is running on an Intel or AMD processor that uses the x86_64 architecture.  The raspberry pi processor, however, uses the ARM architecture.
This difference means that we cannot use regular gcc that is likely already installed on your machine.  We need to download a *cross compiler*, a compiler build for
x86_64 that compiles for ARM.  The compiler we want is called `gcc-arm-none-eabi`, which can download this from [the ARM developer website](https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads).

If you are on linux, create a new directory for this project, and copy the downloaded file there. Then extract the the file using this command:
```
tar -xf tar -xf gcc-arm-none-eabi-X-XXXX-XX-update-linux.tar.bz2
```
where the Xs may vary depending on when you download the file. The compiler should be located in `gcc-arm-none-eabi-X-XXXX-XX-update/bin/arm-none-eabi-gcc`

# The Virtual Machine

This part is not strictly necessary, but it will make things like debugging and quickly testing things much easier.  We are going to use QEMU to run the virtual machine.
Newer versions of QEMU can emulate the rasperry pi hardware, but it is unlikely that your package manager of choice has this version.  We need version no lower than
2.11.0 of the QEMU ARM VM.  We can build the latest version of QEMU from the source code by doing the following:
```
wget https://download.qemu.org/qemu-2.11.0.tar.xz
tar xvJf qemu-2.11.0.tar.xz
cd qemu-2.11.0
./configure --target-list=arm-softmmu,arm-linux-user
make -j 2
sudo make install
```
We can check if this was successful by doing `qemu-system-arm --version`

Now that we have the environment set up, we can now [build a kernel](/tutorial/boot.html)!
