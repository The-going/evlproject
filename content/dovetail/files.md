---
title: "File tracking"
weight: 120
---

A companion core usually wants its [device drivers]({{< relref
"core/kernel-api/_index.md" >}}) to export a file interface to
applications. It may even [generalize]({{< relref
"core/_index.md#everything-is-a-file" >}}) this to all of the
resources it provides, which can then be implemented by device drivers
and referred to by common file descriptors from applications. For
this, we need a way for the core to maintain its own per-process index
of files which may support out-of-band I/O, since we will not be
allowed to reuse the in-band
[VFS](https://www.kernel.org/doc/html/latest/filesystems/vfs.html)
services from the out-of-band context for this purpose
(e.g. `current->files` may not be safely accessed from the out-of-band
stage by the companion core). Dovetail provides a simple mechanism for
tracking file descriptor creation and deletion happening system-wide
from the companion core, based on calling hooks directly from the
[VFS](https://www.kernel.org/doc/html/latest/filesystems/vfs.html),
which the companion core can connect to by overriding their empty
_weak_ definitions:

- [install_inband_fd()]({{< relref "#install_inband_fd" >}}) is
  invoked whenever a new file descriptor is installed into the file
  descriptor table of the current process.

- [replace_inband_fd()]({{< relref "#replace_inband_fd" >}}) is
  invoked whenever the file referred to by an existing file descriptor
  has changed.

- [uninstall_inband_fd()]({{< relref "#uninstall_inband_fd" >}}) is
  called whenever a file descriptor is closed.

In addition, the file description structure maintained by the
[VFS](https://www.kernel.org/doc/html/latest/filesystems/vfs.html)
(`struct file`) contains an opaque pointer named `oob_data`, which
drivers may use freely for keeping any information related to
out-of-band handling. The _oob\_data_ member comes in handy as a way
for the companion core to figure out whether a given file (struct)
supports out-of-band requests. If non-NULL, it may assume the
out-of-band capable driver set it for such a file when
[opening](http://man7.org/linux/man-pages/man2/open.2.html) some
device. For instance, [install_inband_fd()]({{< relref
"#install_inband_fd" >}}) can check this information to determine
whether the file which is being indexed by the in-band kernel does
relate to a file which supports out-of-band operations too, eventually
storing the [ file descriptor, file description structure ] tuple into
its own table if so.

> As an example, the [EVL core]({{< relref
"core/under-the-hood/_index.md" >}}) uses this tracking capabilities
for implementing its own interface to out-of-band file operations such
as [oob_ioctl()]({{< relref "core/user-api/io/_index.md#oob_ioctl"
>}}), [oob_read()]({{< relref "core/user-api/io/_index.md#oob_read"
>}}) and [oob_write()]({{< relref
"core/user-api/io/_index.md#oob_write" >}}).

---

{{< proto install_inband_fd >}}
void __weak install_inband_fd(unsigned int fd, struct file *file,
			      struct files_struct *files)
{{< /proto >}}

{{% argument fd %}}
The file descriptor which is being installed in the file descriptor
table of the current task so as to refer to _file_.
{{% /argument %}}

{{% argument file %}}
The file which is indexed on _fd_.
{{% /argument %}}

{{% argument files %}}
The file table of the current task.
{{% /argument %}}

[install_inband_fd()]({{< relref "#install_inband_fd" >}}) is invoked
whenever a new file descriptor is installed into the file descriptor
table of the current process, typically when indexing a newly
[opened](http://man7.org/linux/man-pages/man2/open.2.html) file. This
implies that [install_inband_fd()]({{< relref "#install_inband_fd"
>}}) happens once the file structure it refers to is fully built,
representing a connection to some character-based device for some
driver. In its simplest form,
[duplicating](http://man7.org/linux/man-pages/man2/dup.2.html) a
descriptor would also cause [install_inband_fd()]({{< relref
"#install_inband_fd" >}}) to be invoked for the new descriptor.

NOTE: `files->file_lock` is locked on entry to this call, the
in-band stage accepts interrupts.

---

{{< proto replace_inband_fd >}}
void __weak replace_inband_fd(unsigned int fd, struct file *file,
			      struct files_struct *files)
{{< /proto >}}

{{% argument fd %}}
The file descriptor which is being reassigned to a new _file_
in the file descriptor table of the current task.
{{% /argument %}}

{{% argument file %}}
The file which is indexed on _fd_.
{{% /argument %}}

{{% argument files %}}
The file table of the current task.
{{% /argument %}}

[replace_inband_fd()]({{< relref "#replace_inband_fd" >}}) is invoked
whenever the file referred to by an existing file descriptor has
changed, typically as a result of calling
[dup2(2)](http://man7.org/linux/man-pages/man2/dup.2.html), asking to
replace an active descriptor.

NOTE: `files->file_lock` is locked on entry to this call, the
in-band stage accepts interrupts.

---

{{< proto uninstall_inband_fd >}}
void __weak uninstall_inband_fd(unsigned int fd, struct file *file,
			      struct files_struct *files)
{{< /proto >}}

{{% argument fd %}}
The file descriptor which is being removed from the file descriptor
table of the current task.
{{% /argument %}}

{{% argument file %}}
The file which was indexed on _fd_.
{{% /argument %}}

{{% argument files %}}
The file table of the current task.
{{% /argument %}}

[uninstall_inband_fd()]({{< relref "#uninstall_inband_fd" >}}) is
called whenever a file descriptor is closed, either explicitly [by the
current process](http://man7.org/linux/man-pages/man2/close.2.html) or
implicitly by the in-band kernel, as a result of calling [dup(2) or
any of its
derivatives](http://man7.org/linux/man-pages/man2/dup.2.html), or
releasing the resources of an exiting process.

{{% notice warning %}}
Unlike with other notifications, the process cleanup operations may be called
from a process context which is different from the
process owning the resources being released. Therefore, the implementation of
[uninstall_inband_fd()]({{< relref "#uninstall_inband_fd" >}}) should never assume
that `current->mm` or `current->files` are relevant in the context of that
particular call.
{{% /notice %}}

NOTE: `files->file_lock` is locked on entry to this call, the
in-band stage accepts interrupts.

---

{{<lastmodified>}}
