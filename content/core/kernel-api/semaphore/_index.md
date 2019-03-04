---
title: "Kernel semaphores"
weight: 20
---

#### void evl_init_ksem(struct evl_ksem *ksem, unsigned int value)

---

#### void evl_destroy_ksem(struct evl_ksem *ksem)

---

#### int evl_down_timeout(struct evl_ksem *ksem, ktime_t timeout)

---

#### int evl_down(struct evl_ksem *ksem)

---

#### int evl_trydown(struct evl_ksem *ksem)

---

#### void evl_up(struct evl_ksem *ksem)
