---
title: "Rules Of Thumb"
date: 2018-06-30T19:02:50+02:00
weight: 4
draft: false
---

## Dealing with spinlocks {#spinlock-rule}

Converting regular kernel spinlocks (e.g. `spinlock_t`,
`raw_spin_lock_t`) to hard spinlocks should involve a careful review
of every section covered by such lock. Any such section would then
inherit the following requirements:

- no in-band kernel service should be called within the section,

- the section covered by the lock should be short enough to keep
  interrupt latency low.

TBD.
