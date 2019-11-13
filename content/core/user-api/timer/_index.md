---
menuTitle: "Timer(fd)"
title: "Timers"
weight: 30
---

#### int evl_new_timer(int clockfd)

---

#### int evl_set_timer(int efd, struct itimerspec *value, struct itimerspec *ovalue)

---

#### int evl_get_timer(int efd, struct itimerspec *value)

---

### Events pollable from a timer file descriptor

The [evl_poll()]({{< relref "core/user-api/poll/_index.md" >}})
interface can monitor the following events occurring on a timer
file descriptor:

- _POLLIN_ and _POLLRDNORM_ are set whenever a timeout event has fired
  for the timer, which is yet to be collected by the parent
  process. Timer events should be waited for and collected by a call
  to [oob_read()]({{< relref "core/user-api/io/_index.md#oob_read"
  >}}).
