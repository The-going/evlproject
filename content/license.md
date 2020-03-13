---
title: "Licensing terms"
---

SPDX license identifiers are used throughout the code to state the
licensing terms of each file clearly. This boils down to:

- [GPL-2.0](https://spdx.org/licenses/GPL-2.0.html) for the EVL core
  in kernel space.

- [GPL-2.0] (https://spdx.org/licenses/GPL-2.0.html) WITH
  [Linux-syscall-note](https://spdx.org/licenses/Linux-syscall-note.html)
  for the UAPI bits exported to user-space, so that [libevl]({{<
  relref "core/user-api/_index.md" >}}) knows at build time about the
  [ABI details]({{< relref "core/under-the-hood/abi.md" >}}) of the
  system call interface implemented by the EVL core.

- [MIT](https://spdx.org/licenses/MIT.html) for all code from
  [libevl](https://git.evlproject.org/libevl.git), which implements
  the EVL system call wrappers, a few utilities and test programs.
