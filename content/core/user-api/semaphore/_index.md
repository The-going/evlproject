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

#### int evl_get_sem(struct evl_sem *sem)

---

#### int evl_timedget_sem(struct evl_sem *sem, const struct timespec *timeout)

---

#### int evl_put_sem(struct evl_sem *sem)

---

#### int evl_tryget_sem(struct evl_sem *sem)

---

#### int evl_peek_sem(struct evl_sem *sem)

---

#### EVL_SEM_INITIALIZER(__name, __clockfd, __initval)
