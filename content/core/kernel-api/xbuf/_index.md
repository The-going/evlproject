---
title: "Using cross-buffers"
weight: 40
---

#### struct evl_xbuf *evl_get_xbuf(int efd, struct evl_file **efilpp)

---

#### void evl_put_xbuf(struct evl_file *efilp)

---

#### ssize_t evl_read_xbuf(struct evl_xbuf *xbuf, void *buf, size_t count, int f_flags)

---

#### ssize_t evl_write_xbuf(struct evl_xbuf *xbuf, const void *buf, size_t count, int f_flags)
