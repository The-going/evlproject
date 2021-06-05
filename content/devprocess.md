# EVL development process

The EVL project works on three software components:

- The [Dovetail]({{ relref "dovetail/_index.md" >}}) interface, which
  introduces a high-priority execution stage into the linux kernel
  logic, on which a functionally-independent companion software core
  may receive interrupts and run threads.

- A compact and SMP-scalable real-time core - aka the [EVL core]({{
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
  running in user-space and the [EVL core]({{ relref "core/_index.md"
  >}}), along with a few other basic services, [utilities]({{< relref
  "core/commands.md" >}}) and [sanity tests]({{< relref
  "core/testing.md" >}}).

The following aspects are addressed in the way the code is currently
maintained:

- We want to track the latest mainline kernel code as closely as it
  makes sense, while dealing with the issue of developing out-of-tree
  code from the standpoint of the mainline kernel.

- The Dovetail code must be accessible to other projects for their own
  use, based on releases of the reference (mainline) kernel.

- Maintaining the EVL core (and Dovetail) over several kernel releases
  should remain a tractable problem for a project with limited
  resources so far.
  
## EVL core development and releases

Currently, up to three kernel branches are maintained in parallel into
the [linux-evl](https://git.xenomai.org/xenomai4/linux-evl.git) GIT
repository:

- the EVL core development proper takes place into the **evl/master**
  branch (which includes the Dovetail code for that matter). This
  branch contains the bleeding edge EVL core implementation, tracking
  the development tip of [linux
  mainline](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/)
  as closely as it makes sense.

- in addition, a release branch tracking the [latest longterm mainline
  kernel release](https://kernel.org/releases.html) (aka LTS) from the
  [stable
  tree](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git)
  is maintained, with the EVL core implementation on top.

- finally, the **dovetail/master** branch is the reference code for
  the [Dovetail interface]({{ relref "dovetail/_index.md" >}}) on top
  of the latest [mainline kernel
  code](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/). Therefore,
  it is a basic subset of the `evl/master` branch, which does _not_
  include the commits related to the EVL core.

> e.g., from the [linux-evl](https://git.xenomai.org/xenomai4/linux-evl.git) repository:
```
$ git branch
  dovetail/master /* dovetail only, tracking mainline master */
  evl/master      /* EVL core on top of mainline master */
  evl/v5.4        /* EVL core on top of stable LTS, e.g. v5.4.24 */

$ git tag -l 'v5.4-evl[0-9]*'
v5.4-evl1
v5.4-evl2
v5.4.26-evl1
...
```

### EVL core development process

1. the EVL core is gradually rebased on [linux
mainline](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/)
into the `evl/master` branch, as the mainline kernel progresses
through -rc releases. This means that `evl/master` may be rebased when
the first upstream candidate release (i.e. -rc1) is out at the
earliest, never during the merge window. From that point, the EVL
development goes on into `evl/master` during the rest of the current
upstream development cycle, which is about 6-8 weeks. During this
period, some commits to `evl/master` may be backported to the LTS
release branch. Because developing the EVL core may entail adding
features to or fixing the Dovetail interface in the process, some
commits from `evl/master` may be cherry-picked into `dovetail/master`
on-the-fly.

2. once the [mainline
kernel](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/)
reaches a final **vX.Y.0** release, `evl/master` is rebased on it. The
EVL core development keeps going on that rebased branch for a while,
until the next upstream kernel cycle issues the release candidate we
are interested in.

3. eventually, time has come to rebase the EVL core on a candidate
release available for the next kernel development cycle, closing the
previous cycle for EVL, which triggers the following actions:

      1. the current HEAD of `evl/master` is tagged in order to point
      at the most recent EVL core commit for the closed kernel cycle,
      which represents the tip of the EVL core based on the **.0**
      upstream release (_vX.Y-evl1_ ).
     
      2. `evl/master` is rebased on the selected candidate release,
      which is usually -rc1. A later candidate could be picked
      instead, although not later than -rc3 under normal
      circumstances.

      3. the current HEAD of `dovetail/master` is tagged in order to
      point at the most recent Dovetail commit for the closed kernel
      cycle, which represents the tip of Dovetail based on the **.0**
      upstream release (_vX.Y-dovetail_). There is no serial number on
      these tags because a single kernel release is tracked for
      Dovetail strictly speaking at any point in time.

      4. eventually, `dovetail/master` is rebased on the same upstream
      candidate release than `evl/master` was rebased on at step 2.
      Therefore, you may expect `evl/master` and `dovetail/master` to
      always be based on the same upstream baseline.

At this point, the next EVL development cycle starts anew from step 1.

![Alt text](/images/evl-commit-stack.png "linux-evl branches")

### LTS releases of the EVL core {#lts-core-release}

Over time, arbitrary commits from the current LTS branch may be
tagged, marking an "official" EVL core release.  These tags look like
_vX.Y[.Z]-evl\<serial\>_, named after the mainline release they are
based on. The serial number progresses as subsequent releases are
issued based on the same mainline LTS branch; this number is therefore
reset to 1 whenever the mainline LTS release changes.

Each time a new mainline LTS release is selected upstream, a new LTS
release branch for the EVL core is created. The older LTS branch is
closed, and the new one receives the next EVL updates.

![Alt text](/images/evl-release-tags.png "EVL core release tags")

## libevl development and releases {#libevl-release}

The strictly linear development workflow of libevl is simpler in
comparision to the EVL core. There is a single **master** branch in
the [libevl GIT tree](https://git.xenomai.org/xenomai4/libevl.git/) where
the development takes place. Over time, "official" libevl releases are
tagged from arbitrary commits into this branch. These tags look like
_r\<serial\>_, with the serial number progressing indefinitely as
subsequent releases are issued.

![Alt text](/images/libevl-release-tags.png "libevl release tags")

A new [libevl](https://git.xenomai.org/xenomai4/libevl.git) release may be
tagged whenever any of the following happens:

- the [ABI]({{< relref "core/under-the-hood/abi.md" >}}) has changed
  in **evl/master** in a way which is not backward-compatible with the
  latest libevl release. In other words, EVL_ABI_PREREQ in some
  release of
  [libevl](https://git.xenomai.org/xenomai4/libevl/-/blob/d12db5d2688ca3aa06a738a924171ef5fe85c6ab/include/evl/evl.h#L25)
  is not in the range defined by
  [\[EVL_ABI_BASE..EVL_ABI_LEVEL\]](https://git.xenomai.org/xenomai4/linux-evl/-/blob/37f57d73123c3b05b9b4f11d5cd3aa2768010dee/include/uapi/evl/control.h#L14)
  anymore.

- the API libevl implements has changed, usually due to the addition
  of new services or changes to the signature of existing
  routine(s). This should happen rarely, since libevl is only meant to
  provide a small set of basic services exported by the EVL core. The
  API version implemented by libevl is an integer, which is assigned
  to the
  [\_\_EVL\_\_](https://git.xenomai.org/xenomai4/libevl/-/blob/d12db5d2688ca3aa06a738a924171ef5fe85c6ab/include/evl/evl.h#L23)
  C macro-definition. This information can also be retrieved at
  runtime by calling the [evl_get_version()]({{ relref
  "core/user-api/misc/_index.html#evl_get_version" >}})
  routine.

- each time `evl/master` reaches a **.0** kernel release, if commits
  are pending on top of the latest libevl release.

In any case, there is no strict relationship between a given [EVL core
release]({{< relref "#lts-core-release" >}}) tag and a [libevl
release]({{< relref "#libevl-release" >}}) tag. A particular libevl
release might be usable with multiple subsequent EVL core releases and
conversely, provided the [ABI requirements]({{ relref
"core/under-the-hood/abi.md" >}}) are met.

{{% notice tip %}}
If you need features from
[libevl](https://git.xenomai.org/xenomai4/libevl.git/) which have not been
released yet, then your best and only option is to build this library
from the `master` branch directly.
{{% /notice %}}

---

{{<lastmodified>}}
