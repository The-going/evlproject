---
title: "Switch to in-band context"
menuTitle: "evl_switch_inband"
weight: 45
---

### int evl_switch_inband(void)

Applications are unlikely to ever use this call explicitly: it
switches the calling thread to the in-band [execution stage]({{%relref
"dovetail/pipeline/_index.md#two-stage-pipeline" %}}), for running
under the main kernel supervision. Any EVL thread which issues a
system call to the main kernel will be switched to the in-band context
automatically, so in most cases there should be no point in dealing
with this manually in applications.

`evl_switch_inband()` is defined for the rare circumstances where some
high-level API based on the EVL core library might have to enforce a
particular execution stage, based on a deep knowledge of how EVL works
internally. Entering a syscall-free section of code for which the
in-band mode needs to be guaranteed on entry would be the only valid
reason to call `evl_switch_inband()`.

{{% notice warning %}}
Forcing the current execution stage between in-band and out-of-band
stages is a heavyweight operation: this entails two thread context
switches both ways, as the switching thread is offloaded to the
opposite scheduler. You really don't want to force this explicitly
unless you definitely have to and fully understand the implications of
it runtime-wise. Bottom line is that **switching the execution stage
to in-band from within a time-critical code is a clear indication that
something is wrong** in such code.
{{% /notice %}}

### Return value

This call returns zero on success, or a negated error code if
something went wrong:

-EPERM		The current thread is not attached to the EVL core.

Anything else is likely an EVL bug. Congratulations.
