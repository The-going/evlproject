---
title: "Stage escalation"
weight: 60
---

Sometimes you may need to escalate the current execution stage from
in-band to out-of-band, only for running a particular routine. This
can be done using `run_oob_call()`. For instance, the EVL core is
using this service to escalate calls to its rescheduling procedure to
the out-of-band stage, as described in the discussion about [switching
task contexts]({{% relref
"dovetail/altsched.md#context-switching" %}}) with Dovetail's
support for alternate scheduling.

---

{{< proto run_oob_call >}}
int run_oob_call(int (*fn)(void *arg), void *arg)
{{< /proto >}}


{{% argument fn %}}
The address of the routine to execute on the out-of-band stage.
{{% /argument %}}

{{% argument arg %}}
The routine argument.
{{% /argument %}}

run_oob_call() first switches the current execution stage to
out-of-band - if need be - then calls the routine with hard interrupts
disabled (i.e. disabled in the CPU). Upon return, the integer value
returned by _fn()_ is passed back to the caller.

Because the routine may switch the execution stage back to in-band for
the calling context, run_oob_call() restores the original stage only
if it did not [change in the meantime]({{% relref
"dovetail/altsched.md#inband-switch" %}}). In addition, the
interrupt log of the current CPU is synchronized before returning to
the caller. The following matrix describes the logic for determining
which epilogue should be performed before leaving run_oob_call(),
depending on the active stage on entry to the latter and on return
from _fn()_:

|   On entry to run_oob_call() | At exit from _fn()_   |  Epilogue
| :------ |:------ | ------
|  out-of-band | out-of-band | sync current stage if not stalled
|  in-band | out-of-band | switch to in-band + sync both stages
|  out-of-band | in-band | sync both stages
|  in-band | in-band | sync both stages

{{% notice note %}}
run_oob_call() is a lightweight operation that switches the CPU to the
out-of-band interrupt stage for the duration of the call, whatever the
underlying context may be. This is different from switching a task
context to the out-of-band mode _indefinitely_ by offloading it to the
autonomous core for scheduling. The latter operation would involve [a
more complex procedure]({{% relref
"dovetail/altsched.md#oob-switch" %}}).
{{% /notice %}}

---

{{<lastmodified>}}
