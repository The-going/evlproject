---
title: "Condition variable"
weight: 15
---

EVL's condition variable behave like the [POSIX
one](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_cond_wait.html)
for the most part. It can be used for synchronizing EVL threads on the
state of some shared information in a raceless way. Proper
serialization is enforced by pairing a condition variable with a
[mutex]({{% relref "core/user-api/mutex/_index.md" %}}).

#### int evl_new_condvar(struct evl_condvar *cv, int clockfd, const char *fmt, ...)

---

#### int evl_open_condvar(struct evl_condvar *cv, const char *fmt, ...)

---

#### int evl_wait(struct evl_condvar *cv, struct evl_mutex *mutex)

---

#### int evl_timedwait(struct evl_condvar *cv, struct evl_mutex *mutex, const struct timespec *timeout)

---

#### int evl_signal(struct evl_condvar *cv)

---

#### int evl_signal_thread(struct evl_condvar *cv, int thrfd)

---

#### int evl_broadcast(struct evl_condvar *cv)

---

#### int evl_close_condvar(struct evl_condvar *cv)

---

#### EVL_CONDVAR_INITIALIZER(__name, __clock)
