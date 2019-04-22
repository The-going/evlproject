---
title: "Building EVL"
weight: 20
---

Building EVL is a two-step process, which may happen in any order:

- a Linux kernel image featuring the EVL core is built.

- the EVL library (aka _libevl_), basic utilities and test programs
  are generated.

## Prerequisites

We need:

- a GCC toolchain for the target CPU architecture.

- the UAPI headers from the target Linux kernel fit with the EVL
  core. In other words, we need access to the contents of
  `include/uapi/asm/` and `include/uapi/evl/` from a source kernel
  tree which contains the EVL core code.

{{% notice warning %}}
libevl relies on thread-local storage support (TLS), which might be
broken in some obsolete toolchains.
{{% /notice %}}

## Building the core {#building-evl-core}

The kernel source tree which includes the latest EVL core is
maintained at git://git.evenless.org/linux-evl. The development branch
is _evl/master_.

Once your favorite kernel configuration tool is brought up, you should
see the EVL configuration block somewhere inside the **General setup**
menu. This configuration block looks like this:

![Alt text](/images/core_xconfig.png "EVL core configuration")

Enabling `CONFIG_EVL` should be enough to get you started, the default
values for other EVL settings are safe to use. You should make sure to
have `CONFIG_EVL_LATMUS` and `CONFIG_EVL_HECTIC` enabled too; those
are drivers required for running the `latmus` and `hectic` utilities
available with `libevl`, which measure latency and validate the
context switching sanity.

{{% notice tip %}}
If you are unfamiliar with building kernels, [this
document](https://kernelnewbies.org/KernelBuild) may help.
{{% /notice %}}

## Building libevl {#building-libevl}

### Build command

The generic command for building libevl is:

```
$ make [-C $SRCDIR] [ARCH=$cpu_arch] [CROSS_COMPILE=$toolchain] UAPI=$uapi_dir [OTHER_BUILD_VARS] [goal...]
```

> Main build variables

| Variable   |  Description
| --------   |    -------
| $SRCDIR    |  Path to this source tree
| $cpu_arch  |   CPU architecture you build for ('arm', 'arm64')
| $toolchain |  Optional prefix of the binutils filename (e.g. 'aarch64-linux-gnu-', 'arm-linux-gnueabihf-')

> Other build variables

| Variable      |  Description   |  Default
| --------      |    -------     |  -------
| D={0\|1}      |  Disable or enable debug build, i.e. -g -O0 vs -O2    | 0
| O=$output_dir |  Generate binary output files into $output_dir        | .
| V={0\|1}      |  Set build verbosity level, 0 is terse                | 0
| DESTDIR=$install_dir | Install library and binaries into $install_dir | /usr/evl

> Make goals

| Goal    |     Action
| ---     |     ---
| all     |     generate all binaries (library, utilities and tests)
| clean   |     remove the build files
| install |     do all, copying the generated binaries to $DESTDIR in the process

### Examples

Let's say the library source code is located at ~/git/libevl, and the
kernel sources featuring the EVL core is located at
~/git/linux-evl. Building and installing the EVL library and utilities
directly to a staging directory at /nfsroot/\<machine\>/usr/evl would
amount to:

> Building from a (temporary) build directory

```
$ mkdir /tmp/build-imx6q && cd /tmp/build-imx6q
$ make -C ~/git/libevl O=$PWD ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- UAPI=~/git/linux-evl DESTDIR=/nfsroot/imx6q/usr/evl install
```

or,

> Building directly into the EVL library source tree

```
$ mkdir /tmp/build-hikey
$ cd ~/git/libevl
$ make O=/tmp/build-hikey ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- UAPI=~/git/linux-evl DESTDIR=/nfsroot/hikey/usr/evl install
```

## Testing the installation

EVL comes with a series of tests you can run to make sure the core is performing correctly on your target system.

### latmus: the litmus test for latency

Without any argument, the program called `latmus` runs a 1Khz sampling
loop, collecting the min, max and average latency values obtained for
an EVL thread running in user-space.

```
# /usr/evl/bin/latmus
```

### hectic: hammering the EVL context switching machinery

By default, the `hectic` program runs a truckload of EVL threads both
in user and kernel spaces, for exercising the scheduler of the
autonomous core. In addition, this test can specifically stress the
floating-point management code to make sure the FPU is shared
flawlessly between out-of-band and in-band thread contexts.

```
# /usr/evl/bin/hectic -s 200
== Testing FPU check routines...
== FPU check routines: OK.
== Threads: switcher_ufps0-0 rtk0-1 rtk0-2 rtup0-3 rtup0-4 rtup_ufpp0-5 rtup_ufpp0-6 rtus0-7 rtus0-8 rtus_ufps0-9 rtus_ufps0-10 rtuo0-11 rtuo0-12 rtuo_ufpp0-13 rtuo_ufpp0-14 rtuo_ufps0-15 rtuo_ufps0-16 rtuo_ufpp_ufps0-17 rtuo_ufpp_ufps0-18 fpu_stress_ufps0-19 switcher_ufps1-0 rtk1-1 rtk1-2 rtup1-3 rtup1-4 rtup_ufpp1-5 rtup_ufpp1-6 rtus1-7 rtus1-8 rtus_ufps1-9 rtus_ufps1-10 rtuo1-11 rtuo1-12 rtuo_ufpp1-13 rtuo_ufpp1-14 rtuo_ufps1-15 rtuo_ufps1-16 rtuo_ufpp_ufps1-17 rtuo_ufpp_ufps1-18 fpu_stress_ufps1-19 switcher_ufps2-0 rtk2-1 rtk2-2 rtup2-3 rtup2-4 rtup_ufpp2-5 rtup_ufpp2-6 rtus2-7 rtus2-8 rtus_ufps2-9 rtus_ufps2-10 rtuo2-11 rtuo2-12 rtuo_ufpp2-13 rtuo_ufpp2-14 rtuo_ufps2-15 rtuo_ufps2-16 rtuo_ufpp_ufps2-17 rtuo_ufpp_ufps2-18 fpu_stress_ufps2-19 switcher_ufps3-0 rtk3-1 rtk3-2 rtup3-3 rtup3-4 rtup_ufpp3-5 rtup_ufpp3-6 rtus3-7 rtus3-8 rtus_ufps3-9 rtus_ufps3-10 rtuo3-11 rtuo3-12 rtuo_ufpp3-13 rtuo_ufpp3-14 rtuo_ufps3-15 rtuo_ufps3-16 rtuo_ufpp_ufps3-17 rtuo_ufpp_ufps3-18 fpu_stress_ufps3-19
RTT|  00:00:01
RTH|---------cpu|ctx switches|-------total
RTD|           0|         568|         568
RTD|           3|         853|         853
RTD|           2|         739|         739
RTD|           1|         796|         796
RTD|           0|         627|        1195
RTD|           2|        1258|        1997
RTD|           3|        1197|        2050
RTD|           1|        1311|        2107
RTD|           0|         627|        1822
RTD|           2|        1250|        3247
RTD|           3|        1254|        3304
RTD|           1|        1254|        3361
RTD|           2|        1254|        4501
RTD|           1|        1254|        4615
RTD|           0|         684|        2506
RTD|           3|        1311|        4615
RTD|           3|        1256|        5871
RTD|           2|        1311|        5812
RTD|           0|         684|        3190
RTD|           1|        1311|        5926
...
```

### Unit testing

A series of unit testing programs is produced in `$prefix/tests` as
part of building `libevl`. You should run each of them to make sure
everything is fine.

> Running the EVL unit tests
```
# for f in /usr/evl/tests/*; do echo $f && $f; done
/usr/evl/tests/basic-xbuf
thread tfd=7
xfd=8
write->oob_read: 4
write->oob_read: 2
write->oob_read: 1
write->oob_read: 1
oob_read[0]<-write: 1 => 0x41
oob_read[1]<-write: 1 => 0x42
oob_read[2]<-write: 1 => 0x43
oob_read[3]<-write: 1 => 0x44
oob_read[4]<-write: 1 => 0x45
oob_read[5]<-write: 1 => 0x46
oob_read[6]<-write: 1 => 0x47
oob_read[7]<-write: 1 => 0x48
dup(6) => 9
dup2(6, 9) => 9
peer reading from fd=6
oob_write->read: 2
oob_write->read: 2
oob_write->read: 2
inband[0] => 01
inband[1] => 23
inband[2] => 45
/usr/evl/tests/clock-timer-periodic
/usr/evl/tests/clone-fork-exec
thread has efd=7
exec() ok for pid 1653
/usr/evl/tests/detach-self
thread efd=7
detach ret=0
thread efd=7
detach ret=0
/usr/evl/tests/duplicate-element
/usr/evl/tests/logger-stdout
efd=7
logfd=8
stdout relay!
oob_write=14, errno=0
/usr/evl/tests/mapfd
file proxy has efd=5
...
```
