---
title: "Kernel thread"
weight: 5
---

![Alt text](/images/wip.png "To be continued")

#### evl_run_kthread(__kthread, __fn, __priority, __fmt, __args...)

---

#### evl_run_kthread_on_cpu(__kthread, __cpu, __fn, __priority, __fmt, __args...)

---

#### void evl_cancel_kthread(struct evl_kthread *kthread)

---

#### int evl_kthread_should_stop(void)

---

#### evl_set_kthread_priority(struct evl_kthread *thread, int priority)

---

#### int evl_sleep_until(ktime_t timeout)

---

#### int evl_sleep(ktime_t delay)

---

#### int evl_set_thread_period(struct evl_clock *clock, ktime_t idate, ktime_t period)

---

#### int evl_wait_thread_period(unsigned long *overruns_r)

---

{{<lastmodified>}}
