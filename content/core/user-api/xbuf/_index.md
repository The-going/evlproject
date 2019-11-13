---
menuTitle: "Cross-buffer"
weight: 35
---

#### int evl_new_xbuf(size_t i_bufsz, size_t o_bufsz,  const char *fmt, ...)

---

### Events pollable from a cross-buffer descriptor

The [evl_poll()]({{< relref "core/user-api/poll/_index.md" >}})
interface can monitor the following events occurring on a cross-buffer
file descriptor:

- _POLLIN_ and _POLLRDNORM_ are set whenever data coming from the
  inband side is available for reading by a call to [oob_read()]({{<
  relref "core/user-api/io/_index.md#oob_read" >}}).

- _POLLOUT_ and _POLLWRNORM_ are set whenever there is still room in
  the output ring buffer for sending more data to the inband side
  using [oob_write()]({{< relref
  "core/user-api/io/_index.md#oob_write" >}}).

Conversely, you can also use the inband
[poll(2)](http://man7.org/linux/man-pages/man2/poll.2.html) on a
cross-buffer file descriptor, monitoring the following events:

- _POLLIN_ and _POLLRDNORM_ are set whenever data coming from the
  out-of-band side is available for reading, by a call to
  [read(2)](http://man7.org/linux/man-pages/man2/read.2.html).
  
- _POLLOUT_ and _POLLWRNORM_ are set whenever there is still room in
  the output ring buffer for sending more data to the out-of-band
  side, using
  [write(2)](http://man7.org/linux/man-pages/man2/write.2.html).
