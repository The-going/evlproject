---
menuTitle: "Function index"
title: "libevl function index"
weight: 1
---

### Thread services

<div>
<style>
#usvctable {
      width: 30%;
}
#usvctable td {
      text-align: center;
}
</style>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/index.html#evl_attach_thread" target="_blank">evl_attach_thread()</a></td>
    <td>N/A</td> 
    <td>&#9989;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/index.html#evl_attach_self" target="_blank">evl_attach_self()</a></td>
    <td>N/A</td> 
    <td>&#9989;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/index.html#evl_detach_thread" target="_blank">evl_detach_thread()</a></td>
    <td>&#9661;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/index.html#evl_detach_self" target="_blank">evl_detach_self()</a></td>
    <td>&#9661;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/index.html#evl_get_self" target="_blank">evl_get_self()</a></td>
    <td>&#9866;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/index.html#evl_switch_oob" target="_blank">evl_switch_oob()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/index.html#evl_switch_inband" target="_blank">evl_switch_inband()</a></td>
    <td>&#9661;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/index.html#evl_is_inband" target="_blank">evl_is_inband()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/thread/index.html#evl_get_state" target="_blank">evl_get_state()</a></td>
    <td>&#9866;</td> 
    <td>&#10005;</td> 
  </tr>
</table>
</div>

### Scheduler services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/scheduling/index.html#evl_get_schedattr" target="_blank">evl_get_schedattr()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/scheduling/index.html#evl_set_schedattr" target="_blank">evl_set_schedattr()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  <tr>
    <td><a href="/core/user-api/scheduling/index.html#evl_control_sched" target="_blank">evl_control_sched()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/scheduling/index.html#evl_get_cpustate" target="_blank">evl_get_cpustate()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

### Clock services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/clock/index.html#evl_new_clock" target="_blank">evl_new_clock()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/clock/index.html#evl_read_clock" target="_blank">evl_read_clock()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/clock/index.html#evl_set_clock" target="_blank">evl_set_clock()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/clock/index.html#evl_get_clock_resolution" target="_blank">evl_get_clock_resolution()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/clock/index.html#evl_sleep_until" target="_blank">evl_sleep_until()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/clock/index.html#evl_udelay" target="_blank">evl_udelay()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
</table>
</div>

### Timer services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/timer/index.html#evl_new_timer" target="_blank">evl_new_timer()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/timer/index.html#evl_set_timer" target="_blank">evl_set_timer()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/timer/index.html#evl_get_timer" target="_blank">evl_get_timer()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

### Mutex services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_create_mutex" target="_blank">evl_create_mutex()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_new_mutex" target="_blank">evl_new_mutex()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_open_mutex" target="_blank">evl_open_mutex()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_lock_mutex" target="_blank">evl_lock_mutex()</a></td>
    <td>&#9651;<sup>4</sup></td>
    <td>&#10005;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_trylock_mutex" target="_blank">evl_trylock_mutex()</a></td>
    <td>&#9651;<sup>4</sup></td>
    <td>&#10005;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_timedlock_mutex" target="_blank">evl_timedlock_mutex()</a></td>
    <td>&#9651;<sup>4</sup></td>
    <td>&#10005;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_unlock_mutex" target="_blank">evl_unlock_mutex()</a></td>
    <td>&#9866;<sup>3</sup></td> 
    <td>&#10005;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_get_mutex_ceiling" target="_blank">evl_get_mutex_ceiling()</a></td>
    <td>&#9866;</td>
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_set_mutex_ceiling" target="_blank">evl_set_mutex_ceiling()</a></td>
    <td>&#9866;</td>
    <td>&#9989;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/mutex/index.html#evl_close_mutex" target="_blank">evl_close_mutex()</a></td>
    <td>&#9661;</td>
    <td>&#9989;</td>
  </tr>
</table>
</div>

### Event services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/event/index.html#evl_create_event" target="_blank">evl_create_event()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/event/index.html#evl_new_event" target="_blank">evl_new_event()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/event/index.html#evl_open_event" target="_blank">evl_open_event()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/event/index.html#evl_wait_event" target="_blank">evl_wait_event()</a></td>
    <td>&#9866;<sup>4</sup></td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/event/index.html#evl_timedwait_event" target="_blank">evl_timedwait_event()</a></td>
    <td>&#9866;<sup>4</sup></td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/event/index.html#evl_signal_event" target="_blank">evl_signal_event()</a></td>
    <td>&#9866;<sup>4</sup></td>
    <td>&#10005;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/event/index.html#evl_signal_thread" target="_blank">evl_signal_thread()</a></td>
    <td>&#9866;<sup>4</sup></td>
    <td>&#10005;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/event/index.html#evl_broadcast_event" target="_blank">evl_broadcast_event()</a></td>
    <td>&#9866;<sup>4</sup></td>
    <td>&#10005;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/event/index.html#evl_close_event" target="_blank">evl_close_event()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

### Flags services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#evl_create_flags" target="_blank">evl_create_flags()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#evl_new_flags" target="_blank">evl_new_flags()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#evl_open_flags" target="_blank">evl_open_flags()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#evl_wait_flags" target="_blank">evl_wait_flags()</a></td>
    <td>&#9651;<sup>4</sup></td>
    <td>&#10005;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#evl_timedwait_flags" target="_blank">evl_timedwait_flags()</a></td>
    <td>&#9651;<sup>4</sup></td>
    <td>&#10005;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#evl_trywait_flags" target="_blank">evl_trywait_flags()</a></td>
    <td>&#9866;<sup>4</sup></td>
    <td>&#9989;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#evl_peek_flags" target="_blank">evl_peek_flags()</a></td>
    <td>&#9866;</td>
    <td>&#9989;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#evl_post_flags" target="_blank">evl_post_flags()</a></td>
    <td>&#9866;<sup>4</sup></td>
    <td>&#9989;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/flags/index.html#evl_close_flags" target="_blank">evl_close_flags()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

### Semaphore services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/sem/index.html#evl_create_sem" target="_blank">evl_create_sem()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/sem/index.html#evl_new_sem" target="_blank">evl_new_sem()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/sem/index.html#evl_open_sem" target="_blank">evl_open_sem()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/sem/index.html#evl_get_sem" target="_blank">evl_get_sem()</a></td>
    <td>&#9651;<sup>4</sup></td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/sem/index.html#evl_timedget_sem" target="_blank">evl_timedget_sem()</a></td>
    <td>&#9651;<sup>4</sup></td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/sem/index.html#evl_tryget_sem" target="_blank">evl_tryget_sem()</a></td>
    <td>&#9866;<sup>4</sup></td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/sem/index.html#evl_peek_sem" target="_blank">evl_peek_sem()</a></td>
    <td>&#9866;</td>
    <td>&#9989;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/sem/index.html#evl_put_sem" target="_blank">evl_put_sem()</a></td>
    <td>&#9866;<sup>4</sup></td>
    <td>&#9989;</td>
  </tr>
  <tr>
    <td><a href="/core/user-api/sem/index.html#evl_close_sem" target="_blank">evl_close_sem()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

### Polling services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/poll/index.html#evl_new_poll" target="_blank">evl_new_poll()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/poll/index.html#evl_add_pollfd" target="_blank">evl_add_pollfd()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/poll/index.html#evl_mod_pollfd" target="_blank">evl_mod_pollfd()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/poll/index.html#evl_del_pollfd" target="_blank">evl_del_pollfd()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/poll/index.html#evl_poll_sem" target="_blank">evl_poll_sem()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/poll/index.html#evl_timedpoll_sem" target="_blank">evl_timedpoll_sem()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
</table>
</div>

### Memory heap services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/heap/index.html#evl_init_heap" target="_blank">evl_init_heap()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/heap/index.html#evl_extend_heap" target="_blank">evl_extend_heap()</a></td>
    <td>&#9866;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/heap/index.html#evl_alloc_block" target="_blank">evl_alloc_block()</a></td>
    <td>&#9651;<sup>3</sup></td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/heap/index.html#evl_free_block" target="_blank">evl_free_block()</a></td>
    <td>&#9651;<sup>3</sup></td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/heap/index.html#evl_check_block" target="_blank">evl_check_block()</a></td>
    <td>&#9651;<sup>3</sup></td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/heap/index.html#evl_destroy_heap" target="_blank">evl_destroy_heap()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/heap/index.html#evl_heap_raw_size" target="_blank">evl_heap_raw_size()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/heap/index.html#evl_heap_size" target="_blank">evl_heap_size()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/heap/index.html#evl_heap_used" target="_blank">evl_heap_used()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

### Proxy services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/proxy/index.html#evl_create_proxy" target="_blank">evl_create_proxy()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/proxy/index.html#evl_new_proxy" target="_blank">evl_new_proxy()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/proxy/index.html#evl_send_proxy" target="_blank">evl_send_proxy()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/proxy/index.html#evl_vprint_proxy" target="_blank">evl_vprint_proxy()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/proxy/index.html#evl_print_proxy" target="_blank">evl_print_proxy()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/proxy/index.html#evl_printf" target="_blank">evl_printf()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

#### Cross-buffer services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/xbuf/index.html#evl_create_xbuf" target="_blank">evl_create_xbuf()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/xbuf/index.html#evl_new_xbuf" target="_blank">evl_new_xbuf()</a></td>
    <td>&#9661;</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

### I/O services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/io/index.html#oob_read" target="_blank">oob_read()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/io/index.html#oob_write" target="_blank">oob_write()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/io/index.html#oob_ioctl" target="_blank">oob_ioctl()</a></td>
    <td>&#9651;</td> 
    <td>&#10005;</td> 
  </tr>
</table>
</div>

### Tube services

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/tube/index.html#evl_init_tube" target="_blank">evl_init_tube()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/tube/index.html#evl_send_tube" target="_blank">evl_send_tube()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/tube/index.html#evl_receive_tube" target="_blank">evl_receive_tube()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/tube/index.html#evl_get_tube_size" target="_blank">evl_get_tube_size()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/tube/index.html#evl_init_tube_rel" target="_blank">evl_init_tube_rel()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/tube/index.html#evl_send_tube_rel" target="_blank">evl_send_tube_rel()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/tube/index.html#evl_receive_tube_rel" target="_blank">evl_receive_tube_rel()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/tube/index.html#evl_get_tube_size_rel" target="_blank">evl_get_tube_size_rel()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

### Misc routines

<div>
<table id="usvctable">
  <col width="40%">
  <col width="12%">
  <col width="12%">
  <tr>
    <th>Function name</th>
    <th>EVL Switch<sup>1</sup></th> 
    <th>non-EVL<sup>2</sup></th> 
  </tr>
  <tr>
    <td><a href="/core/user-api/init/index.html#evl_init" target="_blank">evl_init()</a></td>
    <td>N/A</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/misc/index.html#evl_get_version" target="_blank">evl_get_version()</a></td>
    <td>&#9866;</td> 
    <td>&#9989;</td> 
  </tr>
  <tr>
    <td><a href="/core/user-api/misc/index.html#evl_sigdebug_handler" target="_blank">evl_sigdebug_handler()</a></td>
    <td>N/A</td> 
    <td>&#9989;</td> 
  </tr>
</table>
</div>

<sup>1</sup> Defines the stage switching behavior for out-of-band
callers:

- &#9651; the core may switch the current stage, promoting the caller
  to out-of-band mode if running in-band at the time of the call.

- &#9661; the core may switch the current stage, demoting the caller
  to in-band mode if running out-of-band at the time of the call.

- &#9866; the call does not entail any stage switch.

<sup>2</sup> Whether this call is also available to non-EVL threads,
i.e. threads not [attached]({{< relref
"core/user-api/thread/_index.md#evl_attach_self" >}}) to the EVL core.

<sup>3</sup> Except if the caller undergoes the [SCHED_WEAK]({{<
relref "core/user-api/scheduling/_index.md#SCHED_WEAK" >}}) policy, in
which case it is switched back to in-band mode if it has released the
last [EVL mutex]({{< relref
"core/user-api/mutex/_index.md#evl_mutex_lock" >}}) it holds by the end
of the call.

<sup>4</sup> As an exception, if this synchronization object was
statically initialized (`EVL_*_INITIALIZER()`), this routine may
switch the caller to the in-band stage in order to finalize the
construction before carrying out the requested operation. This is
required only once in the object's lifetime.

---

{{<lastmodified>}}
