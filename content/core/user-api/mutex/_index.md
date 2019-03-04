---
title: "Mutex"
weight: 10
---

#### int evl_new_mutex(struct evl_mutex *mutex, int clockfd, const char *fmt, ...)

---

#### int evl_new_mutex_ceiling(struct evl_mutex *mutex, int clockfd, unsigned int ceiling, const char *fmt, ...)

---

#### int evl_open_mutex(struct evl_mutex *mutex, const char *fmt, ...)

---

#### int evl_lock(struct evl_mutex *mutex)

---

#### int evl_timedlock(struct evl_mutex *mutex, const struct timespec *timeout)

---

#### int evl_trylock(struct evl_mutex *mutex)

---

#### int evl_unlock(struct evl_mutex *mutex)

---

#### int evl_set_mutex_ceiling(struct evl_mutex *mutex, unsigned int ceiling)

---

#### int evl_get_mutex_ceiling(struct evl_mutex *mutex)

---

#### int evl_close_mutex(struct evl_mutex *mutex)

---

#### EVL_MUTEX_INITIALIZER(__name, __clock)

---

#### EVL_MUTEX_CEILING_INITIALIZER(__name, __clock, __ceiling)
