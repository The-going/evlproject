---
title: "Contributing to Dovetail"
menuTitle: "Contributing"
date: 2018-07-02T15:54:45+02:00
weight: 3
draft: false
---

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

With this in mind, simple rules apply:

1. Development takes place in the _master_ branch. This branch is
routinely rebased on the [mainline
kernel's](git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git)
_master_ branch, as the upstream kernel progresses. However, the
linear history of changes to the _master_ branch implementing Dovetail
is (from now on) kept.

2. From time to time, a snapshot branch will be made of the current
state of the _master_ branch, usually when the mainline code it is
based on reaches a release milestone (possibly _-rc_ ones). Some of
the Dovetail-related changes in this branch may be squashed into a
general feature commit, so as to present Dovetail as a clean-cut,
incremental set of changes on top of the _upstream_ kernel
release. Those branches are named after the reference _upstream_
release, appearing as _squashed/<release>_ in our tree. At times, we
may convert some of the older snapshot branches to plain tags if/when
having too many branches hanging around becomes impractical.

## Submitting changes

Please post any patch you would like to submit for integration to
the [Xenomai mainling list](mailto:xenomai@xenomai.org).
