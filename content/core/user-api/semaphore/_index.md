---
title: "Counting semaphore"
weight: 20
---

#### int evl_new_sem(struct evl_sem *sem, int clockfd, int initval, const char *fmt, ...)

---

#### int evl_open_sem(struct evl_sem *sem, const char *fmt, ...)

---

#### int evl_close_sem(struct evl_sem *sem)

---

#### int evl_get(struct evl_sem *sem)

---

#### int evl_timedget(struct evl_sem *sem, const struct timespec *timeout)

---

#### int evl_put(struct evl_sem *sem)

---

#### int evl_tryget(struct evl_sem *sem)

---

#### int evl_getval(struct evl_sem *sem)

---

#### EVL_SEM_INITIALIZER(__name, __clockfd, __initval)
