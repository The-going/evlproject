---
title: "Polling file descriptors"
weight: 45
---

#### int evl_new_poll(void)

---

#### int evl_add_pollfd(int efd, int newfd, unsigned int events)

---

#### int evl_del_pollfd(int efd, int delfd)

---

#### int evl_mod_pollfd(int efd, int modfd, unsigned int events)

---

#### int evl_timedpoll(int efd, struct evl_poll_event *pollset, int nrset, struct timespec *timeout)

---

#### int evl_poll(int efd, struct evl_poll_event *pollset, int nrset)
