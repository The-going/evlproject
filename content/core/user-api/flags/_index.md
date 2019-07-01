---
menuTitle: "Flags"
weight: 17
---

### Synchronizing on a group of flags {#synchronizing-on-flags}

An event flag group is an efficient mechanism for synchronizing
multiple threads, based on a 32bit value, where each individual bit
represents the state of a particular event defined by the
application. Therefore, each group defines 32 individual events, from
bit #0 to bit #31. Threads can wait for bits to be posted by other
threads. Although EVL's event flag group API is somewhat reminiscent
of the [eventfd
interface](http://man7.org/linux/man-pages/man2/eventfd.2.html), it is
much simpler:

- on the send side, posting the _bits_ event mask to an event _group_
  is equivalent to performing atomically a bitwise OR operation such
  as _group |= bits_.
 
- on the receive side, waiting for events means blocking until _group_
  becomes non-zero, at which point this value is returned to the
  thread heading the wait queue and the group is reset to zero in the
  same move. In other words, all bits set are immediately consumed by
  the first waiter and cleared in the group atomically.

![Alt text](/images/flags.png "Event flag group")

{{% notice tip %}}
Unlike with the
[eventfd](http://man7.org/linux/man-pages/man2/eventfd.2.html), there
is no semaphore semantics associated to an event flag group, you may
want to consider the EVL [semaphore]({{< relref
"core/user-api/semaphore/_index.md" >}}) feature instead if this is
what you are looking for.
{{% /notice %}}

### Event flag group services
