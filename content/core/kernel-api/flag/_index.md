---
title: "Kernel flag"
weight: 25
---

![Alt text](/images/wip.png "To be continued")

#### void evl_init_flag(struct evl_flag *wf)

---

#### void evl_destroy_flag(struct evl_flag *wf)

---

#### int evl_wait_flag_timeout(struct evl_flag *wf, ktime_t timeout, enum evl_tmode timeout_mode)

---

#### int evl_wait_flag(struct evl_flag *wf)

---

#### struct evl_thread *evl_wait_flag_head(struct evl_flag *wf)

---

#### void evl_raise_flag_nosched(struct evl_flag *wf)

---

#### void evl_raise_flag(struct evl_flag *wf)

---

#### void evl_pulse_flag_nosched(struct evl_flag *wf)

---

#### void evl_pulse_flag(struct evl_flag *wf)

---

#### void evl_flush_flag_nosched(struct evl_flag *wf, int reason)

---

#### void evl_flush_flag(struct evl_flag *wf, int reason)

---

#### DEFINE_EVL_FLAG(__name)

---

{{<lastmodified>}}
