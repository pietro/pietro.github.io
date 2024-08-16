---
title: 'J-core Open Processor'
type: docs
---

## J-Core Open Processor

J-core is a clean-room open source processor and SOC design using the [SuperH](http://www.shared-ptr.com/sh_insns.html) instruction set, implemented in VHDL and available royalty and patent free under a BSD license.

The rest of this page explains how to compile and install a "bitstream" file to implement this processor in a cheap (about $50) FPGA board, then how to build Linux for that board and boot it to a shell prompt.

The steps are (roughly):

- Get an FPGA board (the cheapest option we've written a build target for is the Numato Mimas v2).
- Download a bitstream (or build one yourself from our VHDL source code).
- Flash the bitstream to onboard SPI flash (via USB).
- Download vmlinux (or build Linux yourself from source) and copy it to an sdcard.
- Boot the board connected to a serial terminal (also via USB) to a Linux shell prompt.

### What is this processor?

The [SuperH processor](https://en.wikipedia.org/wiki/SuperH)
is a Japanese design developed by Hitachi in the late 1990's. As a second
generation hybrid RISC design it was easier for compilers to generate good
code for than earlier RISC chips, and it recaptured much of the code density
of earlier CISC designs by using fixed length 16 bit instructions (with
32 bit register size and address space), using microcoding to allow
some instructions to perform multiple clock cycles of work. (Earlier pure
risc designs used one instruction per clock cycle even when that served
no purpose but to make the code bigger and exhaust the encoding space.)

Hitachi developed 4 generations of SuperH.
SH2 made it to the United states in the Sega Saturn game console, and
SH4 powered the Sega Dreamcast. They were also widely used in areas outside
the US cosumer market, such as the japanese automative industry.

But during the height of SuperH's development, the
[1997 asian economic crisis](https://en.wikipedia.org/wiki/1997_Asian_financial_crisis) caused Hitachi to tighten its belt, eventually
partnering with Mitsubishi to spin off its microprocessor division
into [a new company](http://www.hitachi.us/press/archive/10032002)
called "Renesas". This new company did not inherit the Hitachi
engineers who had designed SuperH, and Renesas' own
[attempts at further
development on SuperH](https://en.wikipedia.org/wiki/SuperH#SH-5) didn't even interest enough customers for the result
to go ito production. Eventually Renesas moved on to new designs it had
developed entirely in-house, and SuperH receded in importance to them...
until the patents expired.

Then Jeff Dionne (a hardware engineer who wandered into Linux long enough
to create the [uClinux](http://www.uclinux.org/) project back in
the 1990's, handing it off when he moved to Japan and went back to
hardware in 2003) created a new processor design compatible with the
SuperH instruction set, publicly releasing the first version under a BSD
liense in 2015. This new design is called j-core instead of superh because the
trademarks haven't expired.

The first j-core generation, j2, is compatible with the
[sh2](https://en.wikipedia.org/wiki/SuperH#SH-2) instruction set,
which means Linux and gcc and such required only minor tweaking to support
this processor. Current linux, gcc, binutils, and
musl-libc versions support j-core out of the box. (QEMU supports sh4,
which is backwards compatible with sh2 userspace but requires its own
kernel.)

J2 adds two backported sh3 barrel shift instructions (SHAD and SHLD)
to improve compiler efficiency, and a new cmpxchg
(mnemonic `CAS.L Rm, Rn, @R0` opcode `0010-nnnn-mmmm-0011`,
based on the IBM 360 instruction) for futexes and SMP. Support for these
is already upstream in vanilla Linux and gcc/binutils (linux
`make ARCH=sh j2_defconfig` and gcc `./configure
--target=sh2eb-linux-muslfdpic --with-cpu=mj2`).

In 2015 the j-core developers gave an [introductory presentation](talks/japan-2015.pdf) about it at Linuxcon Japan, which was
[covered by Linux Weekly News](https://lwn.net/Articles/647636/).
In 2016 the developers gave a j-core design walkthrough presentation at ELC
([slides](talks/ELC-2016.pdf), [video](https://youtu.be/GHORpXNRJiE)).

J2 is a nommu processor because sh2 (the processor in the Sega Saturn
game console) was, and the last sh2 patent expired in October 2014.
The sh4 processor (dreamcast) has an mmu, but the last sh4 patents don't
exire until 2016. (Update: we're probably implementing a
<href=http://lists.j-core.org/pipermail/j-core/2017-March/000558.html>simpler MMU design</a>
which will run the same userspace software but require kernel and QEMU
updates, which we'll submit upstream when ready.)

J-core's design is small and simple. As open source hardware it can be
manufactured cheaply (about 3 cents per processor) and audited for NSA
backdoors or
[vendor
backdoors](http://www.theregister.co.uk/2015/08/12/lenovo_firmware_nasty/) or
[exploitable firmware bugs](http://hothardware.com/news/researchers-discover-rootkit-exploit-in-intel-processors-that-dates-back-to-1997),
and allows systems built without hidden extra processors in things like
[storage devices](http://s3.eurecom.fr/~zaddach/docs/Recon14_HDD.pdf)
and [USB controllers](http://arstechnica.com/security/2014/07/this-thumbdrive-hacks-computers-badusb-exploit-makes-devices-turn-evil/)
easily repurposed into spyware.

The [j-core mailing
list](http://lists.j-core.org) is the best place for further information or to ask questions.

### Quick start on hardware

The theory is you flash a "bitstream" file into an FPGA board's onboard
SPI flash to configure the FPGA to act like a j2 processor. This
bitstream includes a small bootloader that attempts to load a file called
"vmlinux" from an sd card, providing a linux kernel with root filesystem
in initramfs using a serial console.

To do this, you need an FPGA board, microsd card, bitstream, vmlinux
file with bundled initramfs, an sdcard writer, and a computer with a
USB connection (to write the SPI flash and connect to the serial console;
we used a Linux laptop but macs work too).

#### 1) Get some hardware.

- **Numato**: The cheapest usable FPGA development board ($50 US) the j2 build system
currently targets is the
[Numato Mimas v2](http://numato.com/fpga-boards/xilinx/spartan6/mimas-v2-spartan-6-fpga-development-board-with-ddr-sdram.html)
(also available [on amazon](http://www.amazon.com/Numato-Mimas-Spartan-Development-Board/dp/B00RL7FCQW)).
It contains a Xlinux "Spartan 6" LX9 FPGA that can run a J2 at 50mhz,
64 megs of SDRAM, USB2 mini-B, and a micro-sd card slot.

  You will probably also need a USB mini-B cable (the kind playstation
  controllers use, not the kind android phones use), a
  [USB microsd
  card adapter](http://www.amazon.com/s/?keywords=usb+sd+card+adapter), and a blank microsd card.
  The Numato has a builtin USB serial
  converter, so its "serial port" is already USB. (This USB port can also power
  the board, and Numato provides a python script that writes
  bitstreams to the onboard SPI flash through it. Alas it's also hardwired
  to operate at 19200 bps (there's a
  [firmware
  update to 115200](http://langster1980.blogspot.com/2016/06/linux-on-mimas-v2.html) but numato
  [only provides a windows tool](https://community.numato.com/threads/how-download-a-firmware-from-linux.130/) to update it.)

The main downsides of the Numato board (other than the slow serial port)
are that it doesn't have ethernet, and it can't do SMP. (A single instance
of the processor with the cache disabled takes up about 60% of an LX9's
capacity.) So as an upgrade we're working on the
[Turtle Board](turtle).

(J-core's early development was done on an
[avnet
microboard](https://www.avnet.com/shop/us/p/kits-and-tools/development-kits/avnet-engineering-services-ade--1/aes-s6mb-lx9-g-3074457345628965461) but that's more expensive and doesn't have a built-in sdcard
reader, so needs an add-on board to boot Linux. You can find
[more
about that board here](https://tingcao.wordpress.com/category/lx9-microboard/). If you want to port j-core
to other FPGA boards, ask on the mailing list and we'll describe how or
write up more docs.)

The rest of this page describes using the Numato board.

#### 2) Get/install a bitstream.

The point of open hardware is that you can build a bitstream from the VHDL source code, but for your initial smoketesting you probably want to grab [a known working binary](/downloads/binaries/mimas_v2.bin) and install that first.

To build your own bitstream from VHDL source:

- [Install the Xilinx
bitstream compiler](bitcomp.html).
- [Install the sh2 bare metal compiler](http://landley.net/aboriginal/bin/cross-compiler-sh2elf.tar.gz) (to build the ROM bootloader).
It doesn't require a specific install location, you can extract it into
your home directory if you like.
- Download
the [latest
bitstream source.](https://j-core.org/downloads/source/jcore-source-latest.tar.gz)
- Enter xilinx context and add the cross-compiler-sh2elf/bin directory to
your $PATH so sh2elf-cc and friends are available, and cd into the bitstream
source directory.
- Fix the toolchain prefix with: `sed -i 's/sh2-elf-/sh2elf-/g' $(grep -rl sh2-elf- .)`
[TODO: check this in]
- Run `make mimas_v2`. (Other targets are available under targets/boards.)
- Your bitstream should wind up in `output/*/mimas_v2.bin`. [TODO: why the
date directory? That's not how package builds work, have output, overwrite
output when you rebuild. And make clean not deleting this? Really?]

The reason the bare metal compiler is different from the
<a href="http://landley.net/aboriginal/bin/cross-compiler-sh2eb.tar.gz">sh2
Linux compiler</a> (other than not containing a C library) is different
function prefixes. Since low level code like the ROM bootloader (which runs
when the processor starts up and loads vmlinux off the sdcard) is written
in assembly, it manually refers to prefixed function names. Although there
is a command line option to change the prefixes, the compiler contains library
code (such as libgcc.a) that has to match the calling conventions of the
rest of the code.

#### 3) Flash the bitstream to the board.

Numato
<a href="https://github.com/numato/samplecode/raw/master/FPGA/MimasV2/tools/configuration/python/MimasV2Config.py">provides</a>
a GPL-licensed python3 tool to flash bitstreams onto their board.
[TODO: port to python 2]

To use it:
- Nobody ever has python 3 installed, so:
`apt-get install python3 python3-serial`
- Flip the black switch on the board (between the VGA and USB ports)
towards the USB side. This is the "flash" position.
- Connect the board to your Linux box with a USB mini-B cable. (The
kind playstation controlers use, not the kind android phones use.)
- `sudo python3 MimasV2Config.py /dev/ttyACM0 mimas_v2.bin`
- Flip the switch back towards the VGA side. This is the "boot" position.

The above assumes the Numato serial port shows up as `/dev/ttyACM0`,
which is almost always the case.

The above assumes the Numato serial port shows up as `/dev/ttyACM0`,
which is almost always the case.

Note: **Ubuntu 14.04** decided that any serial device plugged into
a post-2014 computer MUST be a modem (a type of hardware used with telephone
land lines back in the 20th century), and have a hotplug daemon send
random AT commands at any new serial device, which confuses the Numato
firmware loader. If you are not particpating in the Great Modem Revival,
you need to `sudo service modemmanager stop`.
See [here](http://community.numato.com/threads/solved-mimas-v2-programming-in-linux.15/page-2#post-186) for details.

#### 4) Hook up a serial console.

Numato's serial port implementation only connected data send and receive
lines, meaning it doesn't provide hardware flow control. This confuses terminal
programs that expect RTS and CTS (let alone DTR or DSR). We can use the stty
tool to tell Linux not to care, then use a simple terminal program that
won't try to fiddle with this itself.

Since the `/dev/ttyACM0` device goes away each time you unplug and
replug the USB cable (which conveniently power cycles the board),
we can combine these two commands into a single command line in the command
history. (So after you power cycle the board, cursor up and hit enter to
reconnect the serial terminal.)

```bash
stty -F /dev/ttyACM0 -crtscts && ~/busybox-x86_64 microcom -s 115200 /dev/ttyACM0
```

Even without a vmlinux image on the sdcard (or without an sdcard), a
successfully flashed processor should run its bootloader out of the same
flash, which says "CPU tests passed" and announces
its revision and build date before complaining it can't load vmlinux from the
sdcard. If it doesn't do this, you probably forgot to put the flash/run
switch back in the right position. (You have to power cycle the board
after doing that.)

#### 5) Install vmlinux on the sdcard.

Again, you can build linux source but for our first go at it, grab
a [vmlinux image](/downloads/linux/vmlinux)
from [Aboriginal Linux](http://landley.net/aboriginal/about.html)'s sh2eb target.
(The "linux" file in the system-image tarballs is built for the numato board running j2.)

Stick the sdcard in the board, reboot, reconnect the serial console,
and login at the prompt (user root, password admin). Congratulations,
you have a Linux shell prompt on j2.

To build vmlinux from source:

Download [Aboriginal Linux](http://landley.net/aboriginal/about.html) and run `./build.sh sh2eb`.

If you want to build this by hand, the vanilla 4.7 kernel should support
this board, as should
[Rich Felker's musl-libc toolchain](https://github.com/richfelker/musl-cross-make).
You'll need to boot from an initramfs linked into the kernel image until we get the kernel sdcard driver updated.
