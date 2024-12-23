---
title: "Running x86_64 Software on RISC-V Using Box64"
date: 2022-09-19T18:44:12+02:00
tags:
  - mangopi-mq-pro
  - risc-v
  - box64
draft: false
description: In this post I will give an overview on how a ran some software compiled for x86_64 on my RISC-V SBC, a MangoPi MQ-Pro. We will be using an awesome piece of software called Box64.
---
In this post I will give an overview on how a ran some software compiled for x86_64 on my RISC-V SBC, a [MangoPi MQ-Pro](https://mangopi.cc/mangopi_mqpro).
We will be using an awesome piece of software called [Box64](https://github.com/ptitSeb/box64/).
Basically this software runs x86_64 binaries through an emulator, but with a twist; it uses the native versions of some "system" libraries, like libc, libm, SDL, and OpenGL, like described in the README of the project:

```
Since Box64 uses the native versions of some "system" libraries, like libc,
libm, SDL, and OpenGL, it's easy to integrate and use with most applications,
and performance can be surprisingly high in many cases.
```

I have already tried the software on my RaspberryPi, both on an aarch64 and armv7 kernel (for the armv7 kernel, I used [Box86](https://github.com/ptitSeb/box86), which is for 32bit applications), but I saw Box64 was not limited to the ARM architecture. Other architectures like PowerPC 64 LE, LoongArch and, the one we are interested in, RISC-V, are also supported. So I decided to take it for a spin on my MangoPi MQ-Pro.

## Prerequisites

You need to install a few build-dependencies on the Linux distribution you're running on your RISC-V board, namely `cmake`, `git` and `python`.
I used the awesome [Arch Linux build](https://github.com/sehraf/riscv-arch-image-builder) from [sehraf](https://github.com/sehraf).
On that distribution I installed the dependencies with pacman like so:
```
sudo pacman -S base-devel cmake git python3
```
Another nice-to have is binfmt_misc kernel support. This allows x86_64 to be started directly from the shell, like you would execute any other binary.
More information can be found [here](https://docs.kernel.org/admin-guide/binfmt-misc.html).
Support for this feature can be checked by executing:
```
cat /proc/sys/fs/binfmt_misc/status
```
This should output `enabled` if this feature is supported.
If the system complains about a directory which does not exist, support is not enabled.
Sheraf's Arch Linux build, which I referenced earlier, has support for binfmt_misc.
Support for this is not an absolute necessity however, you can also ran x86_64 binaries by executing `box64 $BINARY`.

## Compiling

Compiling the software is pretty easy when all the build dependencies are installed.
It's just a case of cloning the repo, compiling and installing it, and as a last step restarting the binfmt systemd-service (so that systemd-binfmt is aware that x86_64 binaries can be executed via Box64).
Like I said in the prerequisites section, binfmt-support is optional.
The compile/install steps like described in the [compiling docs](https://github.com/ptitSeb/box64/blob/main/docs/COMPILE.md#for-risc-v) of Box64:
```
git clone https://github.com/ptitSeb/box64
cd box64
mkdir build; cd build; cmake .. -DRV64=1 -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j`nproc`
sudo make install

sudo systemctl restart systemd-binfmt
```

## Simple test

To take box64 for a spin for the first time, we will write a very simple C-program on a x86_64 machine, and run the binary on our RISC-V machine.
For this test you will need a C-compiler (I will use gcc) and make sure x86_64 is the target.
Our C-program looks like this:
``` hello.c
#include<stdio.h>

int main()
{
  printf("Hello, world!\n");
  return 0;
}

```

We can then compile this program with gcc using:
```
gcc -Wall hello.c -o hello
```

Executing this program on a x86_64, Linux machine should not be a problem:
```
./hello
Hello, world!
```

Now we will transfer the program to our RISC-V machine. You can use tools like `scp` for this when your RISC-V machine is on the same network.
Also, make sure the binary is executable: `chmod +x hello`.
Then you should be able to just run the binary on your RISC-V machine:
```
$ ./hello 
Box64 v0.1.9 21c56e7 built on Sep 16 2022 17:30:02
Using default BOX64_LD_LIBRARY_PATH: ./:lib/:lib64/:x86_64/:bin64/:libs64/
Using default BOX64_PATH: ./:bin/
Counted 22 Env var
Looking for ./hello
Rename process to "hello"
Using native(wrapped) libc.so.6
Using native(wrapped) ld-linux-x86-64.so.2
Using native(wrapped) libpthread.so.0
Using native(wrapped) librt.so.1
Hello, world!
```
If you don't have binfmt-support, you can still run the program like this:
```
box64 ./hello
```
And you should see the same output.

And voila, we have run our first x86_64-binary on a RISC-V machine!
If you want, you can also hide the log messages and Box64 banner, so no one would know you're actually running a x86_64 binary:
```
$ BOX64_NOBANNER=1 BOX64_LOG=0 ./hello 
Hello, world!
```

## Harder test

As a harder test I wanted to try out [Multi Theft Auto](https://multitheftauto.com/).
This is an Open Source server for an all time favorite game of mine; GTA: San Andreas.
I already successfully ran this software via Box64 on a Raspberry Pi, so it was a good candidate to test on my RISC-V board.
To install it, I just followed the [instructions](https://wiki.multitheftauto.com/wiki/Installing_and_Running_MTASA_Server_on_GNU_Linux#Installation_64_bit) posted on their website.
There does appear to be an issue with the ncurses library that's being used by MTA in combination with Box64.
But when running the software without ncurses with the `-n` argument, the software eventually starts without any issues:
```
$ BOX64_NOBANNER=1 BOX64_LOG=0 ./mta-server64 -n
Warning, function myw_wprintw not found
MTA:BLUE Server for MTA:SA

==================================================================
= Multi Theft Auto: San Andreas v1.5.9 [64 bit]
==================================================================
= Server name      : RISC-test
= Server IP address: auto
= Server port      : 22003
= 
= Log file         : ..o_linux_x64/mods/deathmatch/logs/server.log
= Maximum players  : 32
= HTTP port        : 22005
= Voice Chat       : Disabled
= Bandwidth saving : Medium
==================================================================
[16:56:48] Resources: 194 loaded, 0 failed
[16:56:49] Server password set to '******'
[16:56:49] Starting resources...
[16:57:14] Server minclientversion is now 1.5.9-9.21261.0
[16:58:10] Gamemode 'play' started.
[16:58:10] Querying MTA master server... success! (Auto detected IP:94.110.54.172)
[16:58:10] Authorized serial account protection is DISABLED. See http://mtasa.com/authserial
[16:58:10] WARNING: <owner_email_address> not set
[16:58:10] Server started and is ready to accept connections!
[16:58:10] To stop the server, type 'shutdown' or press Ctrl-C
[16:58:10] Type 'help' for a list of commands.
```

## Going further

The two tests in this article were very basic.
With Box64 on RISC-V a world of opportunities opens up; running software written for Windows using Wine, running GUI applications...
But those are things for another day.
