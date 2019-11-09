---
title: "EVL project"
---

# The EVL Project

## Dual Kernel Rebooted

For certain types of applications, offloading a particular set of
time-critical tasks to an autonomous core hosted by the Linux kernel
may deliver the best possible performance when compared to forcing the
entire Linux kernel to meet the most stringent scheduling latency
requirements that only those tasks may have. EVL is a practical work
on finding the best possible integration of such a dedicated software
core into the mainline Linux kernel.

EVL re-imagines the venerable dual kernel design by introducing the
dual kernel support logic at the heart of Linux, which defines the set
of rules to host a guest core.  EVL also comes with a real-time core
showcasing this integration, which is conceived as a learning tool for
discovering the dual kernel technology as much as it aims at excellent
performance and usability.

The key technical issue is about introducing a new execution stage in
the mainline kernel logic which would represent high priority,
out-of-band activities separated from the common work running in-band.
This requires a few kernel subsystems to know intimately about the new
execution context, which in turn has the following upsides:

- the integration is simpler and cleaner, because we don't need
  sideways. The point is not about hiding the dual kernel interface
  from the main logic, but on the contrary to make it a customary
  interface of the main kernel.

- compared to the
  [I-pipe](https://git.xenomai.org/xenomai/wikis/Getting_The_I_Pipe_Patch)
  which is the interface currently used by several dual kernel systems
  such as [Xenomai](https://xenomai.org/), maintaining the new
  [Dovetail dual kernel interface]({{% relref "dovetail/_index.md"
  %}}) out-of-tree proved to be a much easier task already, without
  fundamental conflicts with upstream changes, only marginal
  adjustments so far.

## Pitching EVL

In a nutshell, the EVL project is about introducing a simple, scalable
and dependable dual kernel architecture for Linux, based on the
[Dovetail interface]({{% relref "dovetail/_index.md" %}}) for coupling
a task-specific software core to the main kernel. This interface is
showcased by a compact [real-time core]({{%relref "core/_index.md"
%}}) delivering basic services to applications via a [straightforward
API]({{% relref "core/user-api/_index.md" %}}). EVL starts from a
clean sheet, for the purpose of implementing a production-ready
real-time infrastructure, which can also serve as a starting point for
engineers to build their own flavour of software core running as part
of the Linux system. This ongoing work is composed of:

- the [Dovetail]({{% relref "dovetail/_index.md" %}}) interface, which
  introduces a high-priority execution stage into the main kernel
  logic, where a functionally-independent software core runs.

- a compact [real-time core]({{%relref "core/_index.md" %}}), which is
  deeply integrated into the main kernel. The EVL core delivers
  dependable low-latency services to applications which have to meet
  real-time requirements. Applications are developed using the common
  Linux programming model.

- an in-depth documentation which covers both Dovetail and the EVL
  core, with many cross-references between them, so that engineers can
  implement their software core of choice almost by example.

The result looks promising:

- Low engineering and maintenance costs. With a manageable code
  footprint (20 KLOC, which is not even half the size of the
  [Xenomai](https://xenomai.org/) core) and a well-integrated
  implementation, working on EVL only requires common kernel
  development knowledge.

- Low runtime cost. The EVL core has excellent real-time performances
  including on low-end, single-core hardware, leaving plenty of CPU
  cycles for running GPOS work concurrently.

- High scalability. From single core to high-end multi-core machines
  running time-critical workloads in parallel with low and bounded
  latency. EVL does not require CPU isolation for running tasks with
  demanding real-time response requirements, although [this may
  help]({{% relref "core/caveat/_index.md#caveat-isolcpus" %}}) in
  getting even lower latency figures.

- Low configuration. No runtime tweaks are required to ensure that the
  real-time behavior is not affected by other parts of the Linux
  system. Once enabled in the kernel, the EVL core is ready to
  deliver.

## Getting the sources

You need to clone two GIT repositories:

- [Dovetail](({{% relref "dovetail/_index.md" %}})) and the [EVL
core]({{% relref "core/_index.md" %}}) are maintained in separate
branches of the same GIT repository, tracking the [mainline
kernel](git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git):

  * git://git.evlproject.org/linux-evl.git
  * https://git.evlproject.org/linux-evl.git

- the second repository contains the source code of the [EVL
  library]({{% relref "core/user-api/_index.md" %}}), which is the
  user-space API to the EVL core:

  * git://git.evlproject.org/libevl.git
  * https://git.evlproject.org/libevl.git

The build recipe is available [there]({{% relref "core/build-steps.md"
%}}).

## Mailing list

You can register on the [EVL mailing
list](https://evlproject.org/mailman/listinfo/evl/) to discuss
EVL-related topics including Dovetail and the real-time EVL core.

## Licensing terms

SPDX license identifiers are used throughout the code to state the
licensing terms of each file clearly. This boils down to:

- [GPL-2.0](https://spdx.org/licenses/GPL-2.0.html) for the EVL core
  in kernel space.

- [GPL-2.0] (https://spdx.org/licenses/GPL-2.0.html) WITH
  [Linux-syscall-note](https://spdx.org/licenses/Linux-syscall-note.html)
  for the UAPI bits exported to user-space, so that `libevl` knows at
  build time about the ABI details of the system call interface
  implemented by the EVL core.

- [MIT](https://spdx.org/licenses/MIT.html) for all code from
  `libevl`, which implements the EVL system call wrappers, a few
  utilities and test programs.
