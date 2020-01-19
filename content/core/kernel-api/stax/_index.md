---
title: "Stage exclusion lock"
weight: 50
---

The STAge eXclusion lock (aka _stax_) serializes in-band vs
out-of-band thread activities for accessing an arbitrary
resource. Such lock can be nested so that multiple threads which run
on the same execution stage may 'own' the stax guarding the resource,
excluding any access from the converse stage until the last thread
drops the innermost lock. In other words, at any point in time, the
resource guarded by a stax is either owned by out-of-band threads
exclusively, or by in-band threads exclusively, or by no thread at
all.  A stax is definitely not designed for guaranteeing bounded low
latency to out-of-band threads: if the application allows in-band
threads to compete for the stax, the out-of-band work serialized on
the same object may be delayed at some point by
definition.

Nevertheless, since we may assume that there will be no stage
concurrency when accessing the resource guarded by a stax, as a
consequence we may rely on stage-specific serializers such as mutexes
(either regular [in-band
ones](https://kernel.readthedocs.io/en/sphinx-samples/kernel-locking.html#mutex-api-reference)
or [EVL]({{< relref "core/kernel-api/mutex/_index.md" >}})) to enforce
mutual exclusion among them while they own the stax, without having to
care further about threads running on the converse stage. This solves
a tricky issue about sharing the implementation of a common driver
between in-band and out-of-band users in a safe way.

![Alt text](/images/wip.png "To be continued")

---

{{< proto evl_init_stax >}}
int evl_init_stax(struct evl_stax *stax)
{{< /proto >}}

---

{{< proto evl_destroy_stax >}}
int evl_destroy_stax(struct evl_stax *stax)
{{< /proto >}}

---

{{< proto evl_lock_stax >}}
int evl_lock_stax(struct evl_stax *stax)
{{< /proto >}}

---

{{< proto evl_trylock_stax >}}
int evl_trylock_stax(struct evl_stax *stax)
{{< /proto >}}

---

{{< proto evl_unlock_stax >}}
void evl_unlock_stax(struct evl_stax *stax)
{{< /proto >}}

---

{{<lastmodified>}}
