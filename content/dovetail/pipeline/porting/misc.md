---
title: "Misc"
date: 2018-06-27T17:40:50+02:00
---

## `printk()` support

`printk()` may be called by out-of-band code safely, without encurring
extra latency. The output is conveyed like NMI-originated output,
which involves some delay until the in-band code resumes, and the
console driver(s) can handle it.
    
## Tracing

Tracepoints can be traversed by out-of-band code safely. Dynamic
tracing is available to a kernel running the pipelined interrupt
model too.
