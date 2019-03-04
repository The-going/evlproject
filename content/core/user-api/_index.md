---
title: "The EVL library"
menuTitle: "Application interface"
weight: 5
pre: "&rsaquo; "
---

The EVL interface library in user-space - aka `libevl` - is fairly
compact, with roughly 50 routines, which is less than half of
Xenomai's [POSIX
interface](https://xenomai.org/documentation/xenomai-3/html/xeno3prm/group__cobalt__api.html/).
This API gives access to the real-time services implemented by the EVL
core in kernel space, nothing more.

- Do not expect a full-blown _glibc_-like library with a truckload of
services and utilities, this is mostly a set of system call
wrappers.

- Do not expect a POSIX-compliant interface, this is purely an ad hoc
one, closely matching the semantics of the EVL core.

Some may only need the basic features readily available from `libevl`
in their applications, others may want to implement high level service
libraries based on those features, YMMV.
