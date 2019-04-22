---
title: "Installing the out-of-band stage"
menuTitle: "OOB stage installation"
weight: 52
---

Before you can direct the incoming interrupt flow to out-of-band
handlers, you need to install the [out-of-band interrupt stage]({{<
relref "dovetail/pipeline/_index.md#two-stage-pipeline"
>}}). Conversely, you need to remove the out-of-band stage from the
interrupt pipeline when you are done with receiving out-of-band
events.

---

{{< proto enable_oob_stage >}}
int enable_oob_stage(const char *name)
{{< /proto >}}

{{% argument name %}}
A symbolic name describing the high priority interrupt stage which is
being installed. This information is merely used in kernel messages,
so it should be short but descriptive enough. For instance, the EVL
core installs the "EVL" stage.
{{% /argument %}}

This call enables the out-of-band stage context in the interrupt
pipeline, which in turn allows an autonomous core to [install
out-of-band handlers]({{< relref
"dovetail/pipeline/irq_handling.md" >}}) for interrupts.  It
returns zero on success, or a negated error code if something went
wrong:

-EBUSY		The out-of-band stage is already enabled.

---

{{< proto disable_oob_stage >}}
void disable_oob_stage(void)
{{< /proto >}}

This call disables the out-of-band stage context in the interrupt
pipeline. From that point, the interrupt flow is exclusively directed
to the in-band stage.

{{% notice warning %}}
This call does **NOT** perform any serialization with ongoing
interrupt handling on remote CPUs whatsoever. The autonomous core must
synchronize with remote CPUs before calling `disable_oob_stage()` to
prevent them from running out-of-band handlers while the out-of-band
stage is being dismantled. This is particularly important if these
handlers belong to a dynamically loaded module which might be unloaded
right after `disable_oob_stage()` returns. In that case, you certainly
don't want the _.text_ section containing interrupt handlers to vanish
while they are still running.
{{% /notice %}}

---
