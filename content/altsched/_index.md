---
title: "Alternate Task Control"
date: 2018-06-27T19:01:42+02:00
weight: 2
draft: false
---

Dovetail promotes the idea that a *dual kernel* system should keep the
functional overlap between the kernel and the real-time core
minimal. To this end, a real-time thread should be merely seen as a
regular task with additional scheduling capabilities guaranteeing very
low response times. To support such idea, Dovetail enables kthreads
and regular user tasks to run alternatively in the out-of-band
execution context introduced by the interrupt pipeline (aka *head*
stage), or the common in-band kernel context for GPOS operations (aka
*root* stage).

As a result, real-time core applications in user-space benefit from
the common Linux programming model - including virtual memory
protection -, and still have access to the regular Linux services when
carrying out non time-critical work.
