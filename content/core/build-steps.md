---
title: "Building EVL"
weight: 1
---

## Building EVL from source

{{% mixedgrid src="/images/overview-build-process.png" %}}

**The build process.** Building EVL from the source code is a two-step
process: we need to build a kernel enabling the EVL core, and the
library implementing the user API to this core - aka [libevl]({{<
relref "core/user-api/_index.md" >}}) - using the proper
toolchain. These steps may happen in any order. The output of this
process is:

- a Linux kernel image featuring [Dovetail]({{< relref
  "dovetail/_index.md" >}}) and the [EVL core]({{< relref
  "core/_index.md" >}}) on top of it.

- the `libevl.so` shared library<sup>*</sup> which enables
  applications to request services from the EVL core, along with a few
  [basic utilities]({{< relref "core/commands.md" >}}) and [test
  programs]({{< relref "core/testing.md" >}}).

  <sup>*</sup> The static archive `libevl.a` is generated as well.
 
{{% /mixedgrid %}}

### Getting the sources

EVL sources are maintained in [GIT](https://git-scm.com)
repositories. As a preliminary step, you may want to have a look at
the [EVL development process]({{< relref "devprocess.md" >}}), in
order to determine which GIT branches you may be interested in these
repositories:

- The kernel tree featuring the EVL core:

  * git@git.xenomai.org:Xenomai/xenomai4/linux-evl.git
  * https://git.xenomai.org/xenomai4/linux-evl.git

- The libevl tree which provides the user interface to the core:

  * git@git.xenomai.org:Xenomai/xenomai4/libevl.git
  * https://git.xenomai.org/xenomai4/libevl.git

### Other prerequisites {#building-evl-prereq}

In addition to the source code, we need:

- a GCC toolchain for the target CPU architecture.

- the UAPI headers from the target Linux kernel fit with the EVL
  core. Each UAPI file exports a set of definitions and interface
  types which are shared with `libevl.so` running in user-space, so
  that the latter can submit well-formed system calls to the
  former. In other words, to build `libevl.so`, we need access to the
  contents of `include/uapi/asm/` and `include/uapi/evl/` from a
  source kernel tree which contains the EVL core which is going to
  handle the system calls.

{{% notice warning %}}
libevl relies on thread-local storage support (TLS), which might be
broken in some obsolete (ARM) toolchains. Make sure to use a current one.
{{% /notice %}}

### Building the core {#building-evl-core}

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
list](https://xenomai.org/mailman/listinfo/xenomai/) with the relevant
information.
{{% /notice %}}

#### All core configuration options {#core-kconfig}

<div>
<style>
#kconfig {
       width: 100%;
}
#kconfig th {
       text-align: center;
}
#kconfig td {
       text-align: left;
}
#kconfig tr:nth-child(even) {
       background-color: #f2f2f2;
}
#kconfig td:nth-child(2) {
       text-align: center;
}
</style>

<table id="kconfig">
  <col width="10%">
  <col width="5%">
  <col width="85%">
  <tr>
    <th>Symbol name</th>
    <th>Default</th> 
    <th>Purpose</th> 
  </tr>
  <tr>
    <td><a href="/core/" target="_blank">CONFIG_EVL</a></td>
    <td>N</td>
    <td>Enable the EVL core</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/scheduling/#SCHED_QUOTA" target="_blank">CONFIG_EVL_SCHED_QUOTA</a></td>
    <td>N</td>
    <td>Enable the quota-based scheduling policy</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/scheduling/#SCHED_TP" target="_blank">CONFIG_EVL_SCHED_TP</a></td>
    <td>N</td>
    <td>Enable the time-partitioning scheduling policy</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/scheduling/#SCHED_TP" target="_blank">CONFIG_EVL_SCHED_TP_NR_PART</a></td>
    <td>N</td>
    <td>Number of time partitions for CONFIG_EVL_SCHED_TP</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_HIGH_PERCPU_CONCURRENCY</td>
    <td>N</td>
    <td>Optimizes the implementation for applications with many real-time threads running concurrently on any given CPU	core</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/#thread-stats" target="_blank">CONFIG_EVL_RUNSTATS</a></td>
    <td>Y</td>
    <td>Collect runtime statistics about threads</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_COREMEM_SIZE</td>
    <td>2048</td>
    <td>Size of the core memory heap (in kilobytes)</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/" target="_blank">CONFIG_EVL_NR_THREADS</a></td>
    <td>256</td>
    <td>Maximum number of EVL threads</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_NR_MONITORS</td>
    <td>512</td>
    <td>Maximum number of EVL monitors (i.e. mutexes + semaphores + flags + events)</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/clock/" target="_blank">CONFIG_EVL_NR_CLOCKS</a></td>
    <td>8</td>
    <td>Maximum number of EVL clocks</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/xbuf/" target="_blank">CONFIG_EVL_NR_XBUFS</a></td>
    <td>16</td>
    <td>Maximum number of EVL cross-buffers</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/proxy/" target="_blank">CONFIG_EVL_NR_PROXIES</a></td>
    <td>64</td>
    <td>Maximum number of EVL proxies</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/observable/" target="_blank">CONFIG_EVL_NR_OBSERVABLES</a></td>
    <td>64</td>
    <td>Maximum number of EVL observables (does not include threads)</td>
  </tr>
  <tr>
    <td><a href="/core/runtime-settings" target="_blank">CONFIG_EVL_LATENCY_USER</a></td>
    <td>0</td>
    <td>Pre-set core timer gravity value for user threads (0 means use pre-calibrated value)</td>
  </tr>
  <tr>
    <td><a href="/core/runtime-settings" target="_blank">CONFIG_EVL_LATENCY_KERNEL</a></td>
    <td>0</td>
    <td>Pre-set core timer gravity value for kernel threads (0 means use pre-calibrated value)</td>
  </tr>
  <tr>
    <td><a href="/core/runtime-settings" target="_blank">CONFIG_EVL_LATENCY_IRQ</a></td>
    <td>0</td>
    <td>Pre-set core timer gravity value for interrupt handlers (0 means use pre-calibrated value)</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_DEBUG</td>
    <td>N</td>
    <td>Enable debug features</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_DEBUG_CORE</td>
    <td>N</td>
    <td>Enable core debug assertions</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_DEBUG_CORE</td>
    <td>N</td>
    <td>Enable core debug assertions</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_DEBUG_MEMORY</td>
    <td>N</td>
    <td>Enable debug checks in core memory allocator.
        **This option adds a significant overhead affecting latency figures**</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/#health-monitoring" target="_blank">CONFIG_EVL_DEBUG_WOLI</a></td>
    <td>N</td>
    <td>Enable warn-on-lock-inconsistency checkpoints</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_WATCHDOG</td>
    <td>Y</td>
    <td>Enable watchdog timer</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_WATCHDOG_TIMEOUT</td>
    <td>4</td>
    <td>Watchdog timeout value (in seconds).</td>
  </tr>
  <tr>
    <td><a href="/core/oob-drivers/gpio" target="_blank">CONFIG_GPIOLIB_OOB</a></td>
    <td>n</td>
    <td>Enable support for out-of-band GPIO line handling requests.</td>
  </tr>
  <tr>
    <td><a href="/core/oob-drivers/spi" target="_blank">CONFIG_SPI_OOB, CONFIG_SPIDEV_OOB</a></td>
    <td>n</td>
    <td>Enable support for out-of-band SPI transfers.</td>
  </tr>
</table>

#### Enabling 32-bit support in a 64-bit kernel (`CONFIG_COMPAT`) {#enable-kernel-compat-mode}

Starting from [EVL ABI]({{< relref "core/under-the-hood/abi.md" >}})
20 in the v5.6 series, the EVL core generally allows 32-bit
applications to issue system calls to a 64-bit kernel when both the 32
and 64-bit CPU architectures are supported, such as ARM (aka Aarch32)
code running over an arm64 (Aarch64) kernel. For arm64, you need to
turn on `CONFIG_COMPAT` and `CONFIG_COMPAT_VDSO` in the kernel
configuration. To be allowed to change the latter, the
`CROSS_COMPILE_COMPAT` environment variable should be set to the
prefix of the 32-bit ARMv7 toolchain which should be used to compile
the vDSO (yes, this is quite convoluted). For instance:

```
$ make <your-make-args> ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_COMPAT=arm-linux-gnueabihf- (x|g|menu)config
```

{{% notice tip %}}
For instance, if you plan to run EVL over any of the [Raspberry
PI](https://raspberrypi.org) 64-bit computers, you may find useful to
use the PI-centric 32-bit Linux distributions readily available such
as [Raspbian](https://www.raspberrypi.org/downloads/raspbian/). To do
so, make sure to enable `CONFIG_COMPAT` and `CONFIG_COMPAT_VDSO` for
your EVL-enabled kernel, building the 32-bit vDSO alongside as
mentioned earlier.
{{% /notice %}}

### Building libevl {#building-libevl}

The generic command for building libevl is:

```
$ make [-C $SRCDIR] [ARCH=$cpu_arch] [CROSS_COMPILE=$toolchain] UAPI=$uapi_dir [OTHER_BUILD_VARS] [goal...]
```

> Main build variables

| Variable   |  Description
| --------   |    -------
| $SRCDIR    |  Path to this source tree
| $cpu_arch  |   CPU architecture you build for ('arm', 'arm64', 'x86')
| $toolchain |  Optional prefix of the binutils filename (e.g. 'arm-linux-gnueabihf-', 'aarch64-linux-gnu-')

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
| install |     do all, copying the generated system binaries to $DESTDIR in the process
| install_all | install, copying all the generated binaries including the tidbits

#### Cross-compiling EVL

Let's say the library source code is located at ~/git/libevl, and the
kernel sources featuring the EVL core is located at
~/git/linux-evl.

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

{{% notice note %}}
This is good practice to always generate the build output files to a
separate build directory using the O= directive on the _make_ command
line, not to clutter your source tree with those. Generating output to
a separate directory also creates convenience Makefiles on the fly in
the output tree, which you can use to run subsequent builds without
having to mention the whole series of variables and options on the
_make_ command line again.
{{% /notice %}}

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

### Testing the installation

At this point, you really want to [test the EVL installation]({{<
relref "core/testing.md" >}}).

---

{{<lastmodified>}}
