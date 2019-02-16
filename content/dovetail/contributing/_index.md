---
title: "Contributing to Dovetail"
menuTitle: "Contributing"
date: 2018-07-02T15:54:45+02:00
weight: 25
---

## Submitting patches

Working on Dovetail should be customary Linux kernel development
happening out-of-tree at the present time, so the same rules apply when
[submitting patches](https://www.kernel.org/doc/html/latest/process/submitting-patches.html),
except that:

- your code should be based on the tip of the Dovetail _master_ branch
  at the time of the submission (see below for the location of our GIT
  tree).
      
- you should send all contributions and communicate via the [Xenomai
  mailing list](mailto:xenomai@xenomai.org), not the LKML.

## GIT trees

You can clone the Dovetail code base indifferently from those URLs:

- git://lab.xenomai.org/linux-dovetail.git
- https://lab.xenomai.org/linux-dovetail.git

In addition, a _cgit_ server is running on that tree at
[https://lab.xenomai.org/linux-dovetail.git](https://lab.xenomai.org/linux-dovetail.git/).

## Organization and workflow

Our basic goal is to keep the Dovetail development as close as
possible to the tip of the [mainline
kernel](git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git).

{{%notice note %}}
We are not in the business of maintaining vendor ports, or long-term
maintenance releases of Dovetail, although anyone would be welcome to
take on such task. However, we are very much in the business of
_lowering the engineering cost of maintaining a current dual kernel
interface for the Linux kernel_, by aiming at a lean and mean
implementation.
{{% /notice %}}

To this end, the following process is implemented:

1. Development takes place in the _master_ branch. This branch is
always based on an official release milestone of the [mainline
kernel](git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git). This
may be:

    - a base stable release (e.g. v4.18)
    - a release candidate (e.g. v4.19-rc1)

2. When the mainline kernel reaches the next release milestone we are
interested in, a snapshot of our _master_ branch is taken, named after
the upstream release it was based on. Snapshot branches are named
`snapshot/<upstream-release>-dtl` in our GIT tree.

3. Once a snapshot is taken, some of the Dovetail-related commits in
the _master_ branch may be rearranged (moved and/or squashed), so as
to present Dovetail as a clean-cut, incremental set of changes on top
of the upstream kernel release, where logically bound modifications
are grouped. On the other hand, individual commits are preserved in
snapshots which are frozen branches.

4. Our _master_ branch is eventually rebased on the next upstream
release we are interested in.

{{%notice note %}}
At times, we may convert some of the older snapshot branches to plain
tags if/when having too many branches hanging around becomes
impractical.
{{% /notice %}}
