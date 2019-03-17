---
title: "Alternate Task Control"
date: 2018-06-27T19:01:42+02:00
weight: 15
pre: "&rsaquo; "
---

Dovetail promotes the idea that a *dual kernel* system should keep the
functional overlap between the kernel and the autonomous core
minimal. To this end, a thread from such core should be merely seen as
a regular task with additional scheduling capabilities guaranteeing
very low response times. To support such idea, Dovetail enables
kthreads and regular user tasks to run alternatively in the
out-of-band execution context introduced by the interrupt pipeline
(aka *oob* stage), or the common in-band kernel context for GPOS
operations (aka *in-band* stage).

As a result, autonomous core applications in user-space benefit from
the common Linux programming model - including virtual memory
protection -, and still have access to the main kernel services when
carrying out non time-critical work.

![Alt text](/images/wip.png?height=250px&width=420px "Not there yet")
