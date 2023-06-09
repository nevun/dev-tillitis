---
title: Tools & libraries
weight: 2
---

# Tools & libraries

## Introduction

To build you can either use our OCI images or use native tools on your
dev box.

If want your device applications not to change, which as you know also
means changing that application's CDI as explained in [the
introduction](intro/), it might be better to use the OCI images. At
the very least you want to be sure that the versions of the compiler
and other tools you use stays the same. Perhaps pin those packages if
you don't want to use containers?

## Host toolchain

To create applications you need at least `clang`, `llvm`, `lld`,
`golang` packages installed. Version 15 or later of LLVM/Clang is
required (with riscv32 support and the Zmmul extension,
`-march=rv32iczmmul`). Packages on Ubuntu 22.10 (Kinetic) are known to
work.

On Ubuntu, you can install the required packages with the following
command:

```
$ sudo apt install build-essential clang lld llvm bison flex libreadline-dev \
                 gawk tcl-dev libffi-dev git mercurial graphviz \
                 xdot pkg-config python3 libftdi-dev \
                 python3-dev libeigen3-dev \
                 libboost-dev libboost-filesystem-dev \
                 libboost-thread-dev libboost-program-options-dev \
                 libboost-iostreams-dev cmake libusb-1.0-0-dev \
                 ninja-build libglib2.0-dev libpixman-1-dev \
                 golang clang-format
```

## Toolchain container `tkey-builder`

We provide a container image which has all the above packages and
tools already installed for use with Podman or Docker.

This assumes a working rootless podman. On Ubuntu 22.10, running

```
$ sudo apt install podman rootlesskit slirp4netns
```

should be enough to get you a working Podman setup.

You can use the following command to fetch the image:

```
$ podman pull ghcr.io/tillitis/tkey-builder:2
```

**Note well**: This image is really large (~ 2 GiB) because it also
contains all the tools necessary to build the FPGA bitstream and the
firmware.

## QEMU

Tillitis provides a TKey emulator based on QEMU.

The easiest way to run the TKey emulator is to use our OCI image (~120
MiB):

```
ghcr.io/tillitis/tkey-qemu-tk1-23.03.1:latest
```

We provide a script in
[tillitis-key1-apps](https://github.com/tillitis/tillitis-key1-apps)
that assumes a working rootless Podman setup and `socat` installed. It
currently only works on a Linux system (specifically, it does not work
when containers are run in Podman's virtual machine, which is required
on MacOS and Windows). On Ubuntu 22.10, running `apt install podman
rootlesskit slirp4netns socat` should be enough. Then you can just run 
the script like:

```
./contrib/run-tkey-qemu
```

This will let you run client apps with `--port ./tkey-qemu-pty` and it
will find the running emulator.

### QEMU on macOS

Note that on macOS you need to add `--speed 9600` on the client apps
when you use the QEMU pty.

### Building QEMU

If you want to build QEMU yourself, go to the `tk1` branch in our
[qemu repository](https://github.com/tillitis/qemu) to fetch the
emulator and then build it, or execute the following commands:

```
$ git clone -b tk1 https://github.com/tillitis/qemu
$ mkdir qemu/build
$ cd qemu/build
$ ../configure --target-list=riscv32-softmmu --disable-werror
$ make -j $(nproc)
```

(Built with warnings-as-errors disabled, see [this
issue](https://github.com/tillitis/qemu/issues/3).)

Then execute the following commands to fetch and build the firmware:

```
$ git clone https://github.com/tillitis/tillitis-key1
$ cd tillitis-key1/hw/application_fpga
$ make firmware.elf
```

Then execute the following commands to run the emulator, setting the
built firmware with the `-bios` flag:

```
$ /path/to/qemu/build/qemu-system-riscv32 -nographic -M tk1,fifo=chrid -bios firmware.elf \
  -chardev pty,id=chrid
```

In the output from QEMU it tells you which serial port it's using, for
instance `/dev/pts/1`. This is what you need to use as `--port` when
This is what you need to set with `--port` when running a client
application.

## Device libraries

Libraries for development of TKey device apps are available in:

https://github.com/tillitis/tkey-libs

Build the tkey-libs first, typically just:

```
$ git clone https://github.com/tillitis/tkey-libs.git
$ cd tkey-libs
$ make
```

or

```
$ make podman
```

if you have Podman installed.

## Client libraries

We provide two Go packages to help in developing client applications.
What we call "client" is the computer or mobile device you insert your
TKey into.

- `github.com/tillitis/tkeyclient`: Contains functions to connect to,
  load and start a device application on the TKey. - [Go
  doc](https://pkg.go.dev/github.com/tillitis/tkeyclient).
- `github.com/tillitis/tkeysign`: Contains functions to communicate
  with the `signer` device app, an ed25519 signing oracle. [Go
  doc](https://pkg.go.dev/github.com/tillitis/tkeysign).

## Our TKey Client and Device Apps

We provide some client and device apps in the
[tillitis-key1-apps](https://github.com/tillitis/tillitis-key1-apps)
GitHub repository.

First clone and build the device libraries [as explained above](#device-libraries).

Execute the following command to clone the repository:

```
$ git clone https://github.com/tillitis/tillitis-key1-apps.git
$ cd tillitis-key1-apps
```

Again you have the choice of building with host tools or an OCI image.

### Building with host tools

Execute the following command to build all TKey client and device
applications:

```
$ make
```

If you cloned and built the tkey-libs somewhere else than in a
directory called `tkey-libs` next to `tillitis-key1-apps` you need to
provide the path relative to `tillitis-key1-apps/apps`, for instance:

```
$ make LIBDIR=../../tkey-libs-main
```

If your available `objcopy` is anything other than the default
`llvm-objcopy`, then define `OBJCOPY` to whatever they're called on
your system.


TKey device applications can run both on the real hardware TKey and in
the QEMU emulator. In both cases, the client application (for example
`tkey-ssh-agent`) talks to the device app over a serial port, virtual
or real. There is a separate section below that explains how to run
device apps in QEMU.

### Building with `tkey-builder`

To build everything in the apps repo:

```
$ git clone https://github.com/tillitis/tillitis-key1-apps
$ cd tillitis-key1-apps
$ make podman
```

Or use podman directly if you haven't got `make` installed:

```
$ podman run --rm --mount type=bind,source=.,target=/src --mount type=bind,source=../tkey-libs,target=/tkey-libs -w /src -it ghcr.io/tillitis/tkey-builder:1 make -j
```

