---
title: "Playing around with Gentoo on riscv32"
date: 2025-03-12T13:55:34+01:00
tags:
  - gentoo
  - podman
  - risc-v
draft: false
description: Since the first time I learned about the existence of RISC-V, I have been fascinated by it. I wanted to run operating systems on it, and try out software on the ISA.
---
Since the first time I learned about the existence of RISC-V, I have been
fascinated by it. I wanted to run operating systems on it, and try out software
on the ISA.

In the beginning this was made possible by emulation, and the last years I
treated myself with two physical pieces of RISC-V hardware. But these means
of playing around with RISC-V had a thing in common; it was using the 64-bit
version of the ISA (riscv64).

I also wanted to play around with the 32-bit variant of RISC-V. After looking
around a bit on the internet I found some references for a [Debian riscv32 port](https://wiki.debian.org/RISC-V/32).
But this is something I would have to do a lot of setup for myself, or use a
downstream rootfs. I wanted to use an official rootfs built by the distribution
maintainers themselves.

After searching some more I found out [Gentoo](https://www.gentoo.org/downloads/#riscv)
offers "stage 3" tarballs for riscv32!

Now, how do you actually get a shell in this rootfs? The answer for this can be
found in the ability for [Podman](https://podman.io/) to run images 
[built for foreign architectures](https://wiki.archlinux.org/title/Podman#Foreign_architectures).
It's pretty simple from this point to get thins running if systemd-binfmt is
installed and configured:
1. Download the Gentoo riscv32 stage 3 tarball from [this page](https://podman.io/).
I used the "musl-variant". But other variants should also work.
2. Import the tarball as a container image, and change the entrypoint to a more
powerful shell:
```bash
$ podman import --arch riscv32 --change ENTRYPOINT=/usr/bin/bash stage3-rv32_ilp32_musl-<$BUILD_DATE>.tar gentoo-riscv32
```
3. Actually run the image in a container:
```bash
$ podman run -it --rm --arch riscv32 gentoo-riscv32:latest 
```

That's it! You're now running riscv32 binaries, inside a Gentoo rootfs, using 
QEMU user mode emulation:
```
5468377bf688 / # arch
riscv32
5468377bf688 / # file /bin/bash
/bin/bash: ELF 32-bit LSB pie executable, UCB RISC-V, RVC, soft-float ABI, version 1 (SYSV), dynamically linked, interpreter /lib/ld-musl-riscv32-sf.so.1, stripped
```

As a bonus, let's try installing [fastfetch](https://github.com/fastfetch-cli/fastfetch):
```
5468377bf688 / # emerge --sync
5468377bf688 / # emerge --ask app-misc/fastfetch
```

This only took 66 minutes on my laptop with a Intel Core Ultra 7 155U... But it
works!
![fastfetch on riscv32](posts/play-with-gentoo-rv32/images/fastfetch.png)
