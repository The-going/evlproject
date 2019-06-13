---
title: "Event"
weight: 15
---

An EVL event object has the same semantics than a POSIX [condition
variable](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_cond_wait.html)
for the most part. It can be used for synchronizing EVL threads on the
state of some shared information in a raceless way. Proper
serialization is enforced by pairing an event with a
[mutex]({{% relref "core/user-api/mutex/_index.md" %}}).

#### int evl_new_event(struct evl_event *evt, int clockfd, const char *fmt, ...)

---

#### int evl_open_event(struct evl_event *evt, const char *fmt, ...)

---

#### int evl_wait(struct evl_event *evt, struct evl_mutex *mutex)

---

#### int evl_timedwait(struct evl_event *evt, struct evl_mutex *mutex, const struct timespec *timeout)

---

#### int evl_signal(struct evl_event *evt)

---

#### int evl_signal_thread(struct evl_event *evt, int thrfd)

---

#### int evl_broadcast(struct evl_event *evt)

---

#### int evl_close_event(struct evl_event *evt)

---

#### EVL_EVENT_INITIALIZER(__name, __clock)
