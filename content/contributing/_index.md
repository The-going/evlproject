---
menuTitle: "Contributing"
title: "Contributing to Xenomai 4"
weight: 30
pre: "&#8226; "
---

We obviously welcome contribution to this effort, small or large. If
you are interested in contributing to Xenomai 4 here are some helpful
hints on how to get started. Start small, then progress as you get
your feet wet.  There is no such thing as a minor contribution, there
is no shame in making mistakes.  A contributor who submits a
perfectible one-liner surely advances the project further than any
smart lurker. Things you will need:

- to [build EVL]({{< relref "core/build-steps.md" >}}) from source for
  your platform of choice, unless you precisely want to port EVL to
  that platform.

- access to the [Xenomai mailing list](https://xenomai.org/mailman/listinfo/xenomai/)

You can contribute to [Dovetail]({{< relref "dovetail/_index.md" >}}),
the [EVL core]({{< relref "core/_index.md" >}}), [libevl]({{< relref
"core/user-api/_index.md" >}}), and/or the [documentation
effort](https://git.xenomai.org/xenomai4/website), there is a lot to
be done in all areas.

### Submitting patches

Xenomai 4 follows the [kernel coding
style](https://www.kernel.org/doc/html/latest/process/coding-style.html)
for [Dovetail](https://git.xenomai.org/linux-dovetail.git), the [EVL
core](https://git.xenomai.org/xenomai4/linux-evl.git), and
[libevl](https://git.xenomai.org/xenomai4/libevl.git) too.

Please send individual patch or series to the [Xenomai mailing
list](https://xenomai.org/mailman/listinfo/xenomai/) for review.

### Getting the basics right

Understanding the two key concepts of [interrupt pipelining]({{<
relref "dovetail/pipeline/_index.md" >}}) and [alternate
scheduling]({{< relref "dovetail/altsched.md" >}}) implemented by
Dovetail is a must. This is what enables the kernel to hand over high
priority events and tasks to a companion core such as EVL (and any
other Dovetail client for that purpose) to get them processed with
bounded, ultra-low latency.

### Porting EVL to a new platform

This basically means porting [Dovetail]({{< relref
"dovetail/_index.md" >}}) to that platform, in the sense that most
machine and architecture-specific code we may need involves running
the [interrupt pipeline]({{< relref "dovetail/pipeline/_index.md" >}})
the EVL core depends on. Luckily, a [Dovetail porting guide]({{<
relref "dovetail/porting/_index.md" >}}) is available to help you in
this task, explaining the fundamentals of a port, and what changes
should be applied to the vanilla mainline kernel.

### Digging into the EVL core internals

First, the documentation on this
[website](https://git.xenomai.org/xenomai4/website) tends to
explicitly refer to particular pieces of code via Permalinks to GIT
commits when discussing a matter. This should give you direct pointers
to the implementation in many occasions. For instance, you can find
some of these in the discussion about the [alternate scheduling]({{<
relref "dovetail/altsched.md" >}}) technique [Dovetail]({{< relref
"dovetail/_index.md" >}}) implements and which the [EVL core]({{<
relref "core/_index.md" >}}) uses, or in the discussion about
[out-of-band GPIO]({{< relref "core/oob-drivers/gpio.md" >}})
services.

Second, there is a work-in-progress section called ["Under the
hood"]({{< relref "core/under-the-hood/_index.md" >}}).

Last but not least, there is always the [mailing
list](https://xenomai.org/mailman/listinfo/xenomai/) to ask for
implementation details.

### Use the rules of thumb

This [document]({{< relref "dovetail/rulesofthumb.md" >}}) gives you a
few but important hints, tips and tricks when developing in a dual
kernel environment based on [Dovetail]({{< relref "dovetail/_index.md"
>}}).

---

{{<lastmodified>}}
