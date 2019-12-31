---
title: "Building EVL"
weight: 1
---

Building EVL is a two-step process, which may happen in any order:

- a Linux kernel image featuring the EVL core is built.

- the EVL library (aka _libevl_), basic utilities and test programs
  are generated.

## Prerequisites {#building-evl-prereq}

We need:

- a GCC toolchain for the target CPU architecture.

- the UAPI headers from the target Linux kernel fit with the EVL
  core. Each UAPI file exports a set of definitions and interface
  types which are shared with _libevl.so_ running in user-space, so
  that the latter can submit well-formed system calls to the
  former. In other words, to build _libevl.so_, we need access to the
  contents of `include/uapi/asm/` and `include/uapi/evl/` from a
  source kernel tree which contains the EVL core which is going to
  handle the system calls.

{{% notice warning %}}
libevl relies on thread-local storage support (TLS), which might be
broken in some obsolete toolchains.
{{% /notice %}}

## Building the core {#building-evl-core}

The kernel source tree which includes the latest EVL core is
maintained at git://git.evlproject.org/linux-evl. The development
branch is _evl/master_.

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
document](https://kernelnewbies.org/KernelBuild) may help. If you face
hurdles building directly into the kernel source tree as illustrated
in the document mentioned, you may want to check whether building
out-of-tree might work, since this is how Dovetail/EVL developers
usually rebuild kernels. If something goes wrong while building
in-tree or out-of-tree, please send a note to the [EVL mailing
list](https://evlproject.org/mailman/listinfo/evl/) with the relevant
information.
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

#### Cross-compiling EVL

Let's say the library source code is located at ~/git/libevl, and the
kernel sources featuring the EVL core is located at
~/git/linux-evl.

{{% notice note %}}
This is good practice to always generate the build output files to a
separate build directory using the O= directive on the _make_ command
line, not to clutter your source tree with those. Generating output to
a separate directory also creates convenience Makefiles on the fly in
the output tree, which you can use to run subsequent builds, without
having to mention the whole series of settings on the _make_ command
line again.
{{% /notice %}}

Cross-compiling EVL and installing the resulting library and utilities
to a staging directory located at /nfsroot/\<machine\>/usr/evl would
amount to this:

> Cross-compiling from a separate build directory

```
# First create a build directory the where output files should go
$ mkdir /tmp/build-imx6q && cd /tmp/build-imx6q
# Then start the build+install process
$ make -C ~/git/libevl O=$PWD ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- UAPI=~/git/linux-evl DESTDIR=/nfsroot/imx6q/usr/evl install
```

or,

> Cross-compiling from the EVL library source tree

```
$ mkdir /tmp/build-hikey
$ cd ~/git/libevl
$ make O=/tmp/build-hikey ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- UAPI=~/git/linux-evl DESTDIR=/nfsroot/hikey/usr/evl install
```

#### Native EVL build

Conversely, you may want to build EVL natively on the target system.
Installing the resulting library and utilities directly to their final
home located at e.g. /usr/evl can be done as follows:

> Building natively from a build directory

```
$ mkdir /tmp/build-native && cd /tmp/build-native
$ make -C ~/git/libevl O=$PWD UAPI=~/git/linux-evl DESTDIR=/usr/evl install
```

or,

> Building natively from the EVL library source tree

```
$ mkdir /tmp/build-native
$ cd ~/git/libevl
$ make O=/tmp/build-native UAPI=~/git/linux-evl DESTDIR=/usr/evl install
```

## Testing the installation

At this point, you really want to [test the EVL installation]({{<
relref "core/testing.md" >}}).

---

{{<lastmodified>}}
