---
title: "Pipeline Stop"
date: 2018-07-01T15:13:13+02:00
weight: 4
draft: false
---

In some circumstances, it may be required to stop all activities in
the whole machine, forcing all CPUs to come to a stall before running
a user-defined handler. A non-pipelined kernel would call the
`stop_machine()` service for this purpose.

However, interrupt pipelining would still allow out-of-band activity
to take place on the head stage as `stop_machine()` would only affect
the root stage onto which the in-band kernel code runs. For the
purpose of enforcing a complete stall encompassing all execution
stages, the `stop_machine_pipelined()` service has been introduced.

A typical example of such use can be found in the modifications
brought to the `ftrace` support code on ARM, which make sure that
execution is stopped on all CPUs regardless of the interrupt stage,
before the kernel .text section can be altered safely:

> Stopping all CPUs before poking into the .text section

```
--- a/arch/arm/kernel/ftrace.c
+++ b/arch/arm/kernel/ftrace.c
@@ -44,7 +44,7 @@ static int __ftrace_modify_code(void *data)
 
 void arch_ftrace_update_code(int command)
 {
-	stop_machine(__ftrace_modify_code, &command, NULL);
+	stop_machine_pipelined(__ftrace_modify_code, &command, NULL);
 }
 
 #ifdef CONFIG_OLD_MCOUNT
```

{{% notice note %}}
`stop_machine_pipelined()` may be called from in-band code only.
{{% /notice %}}
