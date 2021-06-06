# Xenomai 4 development process

The Xenomai 4 project works on three software components:

- The [Dovetail]({{< relref "dovetail/_index.md" >}}) interface, which
  introduces a high-priority execution stage into the linux kernel
  logic, on which a functionally-independent companion software core
  may receive interrupts and run threads.

- A compact and SMP-scalable real-time core - aka the [EVL core]({{<
  relref "core/_index.md" >}}) - leveraging Dovetail's capabilities,
  which is intended to be the reference implementation for other
  Dovetail-based companion cores. As such, the Dovetail code base
  progresses as much as the EVL core runs on the most recent kernel
  releases and exercises this interface, uncovering issues. The EVL
  core lives in kernel space, as an optional component of the linux
  kernel.

- A library named [libevl]({{< relref "core/user-api/_index.md" >}}),
  which implements the [system call interface]({{< relref
  "core/user-api/function_index/_index.md" >}}) between applications
  running in user-space and the [EVL core]({{< relref "core/_index.md"
  >}}), along with a few other basic services, [utilities]({{< relref
  "core/commands.md" >}}) and [sanity tests]({{< relref
  "core/testing.md" >}}).

The following aspects are addressed in the way the code is currently
maintained:

- We want to track the latest mainline kernel code as closely as it
  makes sense, while dealing with the issue of developing out-of-tree
  code from the standpoint of the mainline kernel.

- The [Dovetail]({{< relref "dovetail/_index.md" >}}) code must be
  accessible to other projects for their own use, based on releases of
  the reference (mainline) kernel.

- Maintaining the EVL core (and Dovetail) over multiple kernel
  releases should remain a tractable problem.

## EVL core development and releases

The EVL core is maintained in the following GIT repository:

  * git@git.xenomai.org:Xenomai/xenomai4/linux-evl.git
  * https://git.xenomai.org/xenomai4/linux-evl.git

This repository tracks the
[Dovetail](https://git.xenomai.org/linux-dovetail.git) kernel tree,
which in turn tracks the [mainline
kernel](git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git). Details
of the current policy are as follows:

- since the EVL core is always based on a Dovetail port to a mainline
  kernel release, every EVL branch is initially based on a
  [Dovetail](https://git.xenomai.org/linux-dovetail.git) branch. For
  instance
  [v5.10.y-evl-rebase](https://git.xenomai.org/xenomai4/linux-evl/-/tree/v5.10.y-evl-rebase)
  in the [EVL tree](https://git.xenomai.org/xenomai4/linux-evl.git)
  originates from
  [v5.10.y-dovetail-rebase](https://git.xenomai.org/linux-dovetail/-/tree/v5.10.y-dovetail-rebase)
  in the [Dovetail tree](https://git.xenomai.org/linux-dovetail.git).

- the content of `*-rebase` branches may be rebased without notice, so
  that Dovetail- and EVL-related commits always appears on top of the
  vanilla mainline code. All EVL branches are currently rebased, we
  have no merge branch yet.

- EVL release tags are of the form:

  - `v<kversion>-evl<serial>-rebase` if added to a rebase branch.
  - `v<kversion>-evl<serial>` if added to a merge branch.

## libevl development and releases {#libevl-release}

The strictly linear development workflow of [libevl]({{< relref
"core/user-api/_index.md" >}}) is simpler in comparision to the EVL
core. There is a single
[master](https://git.xenomai.org/xenomai4/libevl/-/tree/master) branch
in the [libevl GIT tree](https://git.xenomai.org/xenomai4/libevl.git/)
where the development takes place. Over time, `libevl` releases are
tagged from arbitrary commits into this branch. These tags look like
_r\<serial\>_, with the serial number progressing indefinitely as
subsequent releases are issued.

![Alt text](/images/libevl-release-tags.png "libevl release tags")

A new [libevl](https://git.xenomai.org/xenomai4/libevl.git) release may be
tagged whenever any of the following happens:

- the [ABI]({{< relref "core/under-the-hood/abi.md" >}}) exported by
  the EVL core has changed in a way which is not backward-compatible
  with the latest `libevl` release. In other words,
  [EVL_ABI_PREREQ](https://git.xenomai.org/xenomai4/libevl/-/blob/d12db5d2688ca3aa06a738a924171ef5fe85c6ab/include/evl/evl.h#L25)
  in some release of
  [libevl](https://git.xenomai.org/xenomai4/libevl/-/blob/d12db5d2688ca3aa06a738a924171ef5fe85c6ab/include/evl/evl.h#L25)
  is not in the range defined by
  [\[EVL_ABI_BASE..EVL_ABI_LEVEL\]](https://git.xenomai.org/xenomai4/linux-evl/-/blob/37f57d73123c3b05b9b4f11d5cd3aa2768010dee/include/uapi/evl/control.h#L14)
  anymore.

- the API `libevl` implements has changed, usually due to the addition
  of new services or changes to the signature of existing
  routine(s). This should happen rarely, since `libevl` is only meant to
  provide a small set of basic services exported by the EVL core. The
  API version implemented by `libevl` is an integer, which is assigned
  to the
  [\_\_EVL\_\_](https://git.xenomai.org/xenomai4/libevl/-/blob/d12db5d2688ca3aa06a738a924171ef5fe85c6ab/include/evl/evl.h#L23)
  C macro-definition. This information can also be retrieved at
  runtime by calling the [evl_get_version()]({{< relref
  "core/user-api/misc/_index.md#evl_get_version" >}})
  routine.

In any case, there is no strict relationship between a given [EVL core
release]({{< relref "#lts-core-release" >}}) tag and a [libevl
release]({{< relref "#libevl-release" >}}) tag. A particular `libevl`
release might be usable with multiple subsequent EVL core releases and
conversely, provided the [ABI requirements]({{< relref
"core/under-the-hood/abi.md" >}}) are met.

---

{{<lastmodified>}}
