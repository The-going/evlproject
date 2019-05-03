---
title: "FAQ"
weight: 25
pre: "&#9656; "
---

### Who is behind the EVL project?

A single developer at the moment, the guy who started [this
stuff](https://xenomai.org) nearly two decades ago, then [that
one](https://lwn.net/Articles/1743/) too, which eventually became
[this](https://lwn.net/Articles/140374/). Hopefully, at some point, I
will get the dual kernel thing right eventually.

### Why the EVL project?

EVL is a way to experiment freely with dual kernel technology going
back to the drawing board for the most part, which can be pretty damn
fun.

In the long run, I'm hoping that the work which takes place in EVL
will help in improving [Xenomai](https://xenomai.org) which I
contribute to as well. For instance, substituting the venerable
[_I-pipe_](https://gitlab.denx.de/Xenomai/xenomai/wikis/Dovetail/)
with [Dovetail]({{% relref "dovetail/_index.md" %}}) as Xenomai's
dual kernel interface is a valuable goal.

### Where should the discussion on EVL take place?

You can register on the [EVL mailing
list](https://evenless.org/mailman/listinfo/evl/) to discuss
EVL-related topics including Dovetail and the real-time EVL core.

### How do EVL and Xenomai differ?

In several ways implementation-wise, starting from the interface which
connects the Linux kernel to the autonomous core: EVL is developing
[Dovetail]({{% relref "dovetail/_index.md" %}}), Xenomai relies on its
ancestor, the [interrupt pipeline](https://git.xenomai.org/ipipe-arm).

Although the EVL core inherits some basic code from Xenomai's [Cobalt
core](https://git.xenomai.org/Xenomai/xenomai), both implementations
are already quite different, and poised to diverge even more over
time:

- the EVL core only implements a handful of basic features in kernel
  space known as [_elements_]({{% relref
  "core/_index.md#evl-core-elements" %}}), which should be sufficient
  to provide high-level API services from user-space. At the opposite,
  Xenomai's Cobalt core implements a POSIX interface directly from
  kernel space which amounts to 100+ system calls.

- another major difference is the lack of specific driver model in the
  EVL core, which fits in the regular Linux model. Xenomai relies on
  the [RTDM]
  (https://xenomai.org/documentation/xenomai-3/html/xeno3prm/group__rtdm.html)
  layer instead, which is entirely separate.

- high scalability with large multi-core systems is a strong goal of
  the EVL core, and the current single-lock model Cobalt exhibits
  won't fit that bill. For this reason, their respective scheduler
  infrastructures are already diverging.

EVL and Xenomai differ in purpose too.  On the one hand,
[Xenomai](https://xenomai.org) aims at a comprehensive real-time
framework offering multiple APIs, RTOS emulators and various protocol
stacks, based on a POSIX-compliant core and the [I-pipe dual kernel
interface](https://gitlab.denx.de/Xenomai/xenomai/wikis/Getting_The_I_Pipe_Patch),
tracking [LTS kernel
releases](https://www.kernel.org/category/releases.html).  Xenomai
APIs can even run in single kernel configurations such as
[PREEMPT_RT](https://wiki.linuxfoundation.org/realtime/rtl/blog).

On the other hand, EVL is about enabling any autonomous software core
to run along the most recent mainline kernel release, according to
[clear interface rules]({{% relref "dovetail/_index.md" %}}) defined
by the latter. In other words, _EVL is all about, and only about dual
kernel technology_. To showcase this, EVL comes with a compact
[real-time core]({{%relref "core/_index.md" %}}) delivering basic
services via a [small library]({{% relref "core/user-api/_index.md"
%}}) implementing its ad hoc API, for anyone to play with and build
on.
