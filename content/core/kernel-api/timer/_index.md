---
title: "Timers"
weight: 35
---

![Alt text](/images/wip.png "To be continued")

#### evl_init_timer(__timer, __clock, __handler, __rq, __flags)

---

#### evl_init_core_timer(__timer, __handler)

---

#### evl_init_timer_on_cpu(__timer, __cpu, __handler)

---

#### void evl_destroy_timer(struct evl_timer *timer)

---

#### void evl_start_timer(struct evl_timer *timer, ktime_t value, ktime_t interval)

---

#### void evl_stop_timer(struct evl_timer *timer)

---

#### ktime_t evl_get_timer_date(struct evl_timer *timer)

---

#### ktime_t xntimer_get_interval(struct evl_timer *timer)

---

#### ktime_t evl_get_timer_delta(struct evl_timer *timer)

---

#### ktime_t evl_get_stopped_timer_delta(struct evl_timer *timer)

---

{{<lastmodified>}}
