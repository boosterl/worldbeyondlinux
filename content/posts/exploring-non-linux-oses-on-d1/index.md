---
title: "Exploring non-Linux Based Operating Systems on Allwinner D1"
date: 2022-10-31T16:28:12+01:00
tags:
  - mangopi-mq-pro
  - risc-v
  - freebsd
  - xv6
draft: false
description: As of writing this article, there are a few choices of operating systems to choose from for the Allwinner D1. In the future, more choices will probably become available. In this article we will take a closer look into FreeBSD and xv6 for the Allwinner D1. I will use a MangoPi MQ-Pro to test these operating systems.
---
# Introduction

As of writing this article, there are a few choices of operating systems to choose from for the Allwinner D1.
In the future, more choices will probably become available.
In this article we will take a closer look into [FreeBSD](https://github.com/freebsd-d1/freebsd-d1) and [xv6](https://github.com/michaelengel/xv6-d1) for the Allwinner D1.
I will use a [MangoPi MQ-Pro](https://mangopi.org/mangopi_mqpro) to test these operating systems.

# FreeBSD

For many people, FreeBSD won't need an introduction.
FreeBSD is a free and open-source Unix-like operating system first released in 1993.
To create an image ready to flash on an SD card, access to a FreeBSD (virtual) machine is recommended.
On this machine, the [readme](https://github.com/freebsd-d1/freebsd-d1/blob/main/README.md) of the project can be followed.
This will result in a `freebsd-d1.img`-img file in the working directory.
This image can then be flashed on an SD card in the usual fashion; `dd if=freebsd-d1.img of=/dev/<SD card location>`.
When inserting the image, and starting your board, a serial connection will also be necessary, because as far as I can tell, and at the moment of writing this article, no HDMI-support is present.
I used a Raspberry Pi to open a serial console, more information on how I did this, can be found in [this](/posts/how-to-use-a-pi-instead-of-a-usb-console-cable) article.
After staring up the board with an open console, you should be greeted with a FreeBSD login screen:
```
FreeBSD/riscv (freebsd-d1) (rcons)

login: root
Sep 20 16:09:29 freebsd-d1 login[1101]: ROOT LOGIN (root) ON rcons
Last login: Tue Sep 20 16:09:23 on rcons
FreeBSD 14.0-CURRENT #1 n256327-be9d9f3eb4c3: Tue Sep 20 18:00:54 CEST 2022     root@freebsd-riscv-builder:/root/freebsd-d1/freebsd/obj/riscv.riscv64/sys/GENERIC

Welcome to FreeBSD!

Release Notes, Errata: https://www.FreeBSD.org/releases/
Security Advisories:   https://www.FreeBSD.org/security/
FreeBSD Handbook:      https://www.FreeBSD.org/handbook/
FreeBSD FAQ:           https://www.FreeBSD.org/faq/
Questions List:        https://www.FreeBSD.org/lists/questions/
FreeBSD Forums:        https://forums.FreeBSD.org/

Documents installed with the system are in the /usr/local/share/doc/freebsd/
directory, or can be installed later with:  pkg install en-freebsd-doc
For other languages, replace "en" with a language code like de or fr.

Show the version of FreeBSD installed:  freebsd-version ; uname -a
Please include that output and any error messages when posting questions.
Introduction to manual pages:  man man
FreeBSD directory layout:      man hier

To change this login announcement, see motd(5).
root@freebsd-d1:~ # uname -a
FreeBSD freebsd-d1 14.0-CURRENT FreeBSD 14.0-CURRENT #1 n256327-be9d9f3eb4c3: Tue Sep 20 18:00:54 CEST 2022     root@freebsd-riscv-builder:/root/freebsd-d1/freebsd/obj/riscv.riscv64/sys/GENERIC riscv
root@freebsd-d1:~ # file /usr/bin/vi
/usr/bin/vi: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), dynamically linked, interpreter /libexec/ld-elf.so.1, for FreeBSD 14.0 (1400062), FreeBSD-style, stripped
root@freebsd-d1:~ #
```
From this point on, people with more FreeBSD experience than me could jump in and start hacking, setting up a network stack etc.
The built-in WiFi adapter does not work out of the box, a USB to ethernet adapter which I had lying around, also didn't work.
But I'm sure FreeBSD experts could quickly create a setup for their needs.

# xv6

I can't introduce xv6 better than the developers do on their [GitHub](https://github.com/michaelengel/xv6-d1) repository:
```
xv6 is a re-implementation of Dennis Ritchie's and Ken Thompson's Unix
Version 6 (v6).  xv6 loosely follows the structure and style of v6,
but is implemented for a modern RISC-V multiprocessor using ANSI C.
```

Getting xv6 up and running is a case of following the [guide](https://github.com/michaelengel/xv6-d1/blob/main/README_D1.txt) in the root of the repository.
You do also need to have [xfel](https://github.com/xboot/xfel) on your computer.
This is a tool which can be used to communicate with Allwinner based devices in [FEL mode](https://linux-sunxi.org/FEL).
Starting up your Allwinner D1 based board in FEL mode can be as easy as powering the board without an SD card inserted, as long as there is no other storage (SPI flash, eMMC etc.) available.
On my MangoPi MQ-Pro this method definitely works, since I don't have any SPI soldered to the board.
You can read more about possible ways to trigger FEL mode [here](https://linux-sunxi.org/FEL#Triggering_FEL_mode).
To communicate with FEL, you must connect your board with your computer via the USB OTG connector on the board.
You can verify whether the board is connected, and in FEL mode using `lsusb` and `dmesg`.

After triggering FEL mode, and having a serial console available to your D1 board, and having a xv6-image compiled, the image can be started like so:
```
$ xfel ddr d1
Initial ddr controller succeeded
$ xfel write 0x40000000 ../xv6-d1/kernel/kernel.bin
100% [================================================] 1.006 MB, 397.628 KB/s
$ xfel exec 0x40000000
```

The `xfel write` command is slightly different than the command in the xv6 repository, but this is because of a change in the `xfel`-tool.

There is one caveat, the serial console needs an extra argument to display the xv6-shell correctly.
For picocom, the argument looks like this:
```
picocom -b 115200 /dev/serial0  --imap lfcrlf
```
This maps linefeed to carriage return + linefeed.
Otherwise, the serial output could quickly become a mess.

On the serial console you should be greeted with a very basic shell into the xv6 operating system:
```
xv6 kernel is booting

init: starting sh
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2059
cat            2 3 23392
echo           2 4 22232
forktest       2 5 13000
grep           2 6 26544
init           2 7 23048
kill           2 8 22152
ln             2 9 22032
ls             2 10 25576
mkdir          2 11 22288
rm             2 12 22280
sh             2 13 40296
stressfs       2 14 23264
usertests      2 15 147600
grind          2 16 36776
wc             2 17 24384
zombie         2 18 21536
console        3 19 0
$ echo Hello, world! > hi   
$ cat hi
Hello, world!
$ grep world hi
Hello, world!
$ grep xv6 README
xv6 is a re-implementation of Dennis Ritchie's and Ken Thompson's Unix
Version 6 (v6).  xv6 loosely follows the structure and style of v6,
xv6 is inspired by John Lions's Commentary on UNIX 6th Edition (Peer
The code in the files that constitute xv6 is
(kaashoek,rtm@mit.edu). The main purpose of xv6 is as a teaching
$ wc README
45 295 2059 README
$ wc -l README
wc: cannot open -l
$ wc hi
1 2 14 hi
$ mkdir hello
$ echo Hello, from a dir > hello/hi_again
$ ls hello    
.              1 20 48
..             1 1 1024
hi_again       2 21 18
$ cat hello/hi_again
Hello, from a dir
$ rm hello/hi_again
$ rm hello
```

As you can see, some very basic, but fundamental Unix tools are available, and work!
