---
menuTitle: "Core calls"
title: "Core syscalls"
weight: 10
---

An EVL application interacts with so-called EVL [elements]({{< relref
"core/_index.md#evl-core-elements" >}}). Each element usually exports
a mix of [in-band](http://man7.org/linux/man-pages/man2/ioctl.2.html)
and [out-of-band]({{< relref "core/user-api/io/_index.md" >}}) file
operations implemented by the EVL core (in a few cases, only either
side is implemented). Therefore, an EVL application uses regular file
descriptors to interact with elements, either directly or indirectly
by calling routines from the standard
[glibc](https://www.gnu.org/software/libc/) or [libevl]({{< relref
"core/user-api/_index.md" >}}) whenever there is a real-time
requirement. To sum up, the system calls involved in the interface
between an application and the EVL core are exclusively
[open(2)](http://man7.org/linux/man-pages/man2/open.2.html),
[close(2)](http://man7.org/linux/man-pages/man2/close.2.html),
[read(2)](http://man7.org/linux/man-pages/man2/read.2.html),
[write(2)](http://man7.org/linux/man-pages/man2/write.2.html),
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html),
[mmap(2)](http://man7.org/linux/man-pages/man2/mmap.2.html) for the
in-band side, and [oob_ioctl()]({{< relref
"core/user-api/io/_index.md#oob_ioctl" >}}), [oob_read()]({{< relref
"core/user-api/io/_index.md#oob_read" >}}), [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}) for the out-of-band side.

### The EVL ABI

Since we have system calls between the applications and the EVL core,
we depend on a certain format of request codes, arguments and return
values when exchanging information between both ends. This convention
forms the EVL ABI. As hard as we might try to keep the ABI stable over
time, there are times when this cannot be achieved, at the very least
because we may want to make new features added to the EVL core visible
to applications. Generally speaking, EVL being a new kid in the old
dual kernel town, it is too early - for the time being - to write the
current convention in stone. This fact of life calls for a way to
unambiguously:

- determine which revision of the ABI is _implemented_ by the running
  EVL core, from the application side.

- state which revision of the ABI an application _requires_.

In some instance, the EVL core may also be able to provide a
backward-compatible interface to applications which spans multiple
subsequent ABI revisions. Once the dust has settled on the EVL core
development - which should happen in a foreseeable future - ABI
stability should become a strong goal, making backward compatibility
between ABI revisions a must.

---

{{<lastmodified>}}
