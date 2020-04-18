---
menuTitle: "Misc. services"
title: Miscellanea
weight: 70
---

{{< proto evl_get_version >}}
struct evl_version evl_get_version(void)
{{< /proto >}}

This function returns the available version information about the
current [libevl API]({{< relref "core/user-api/_index.md" >}}) and
[EVL core]({{< relref "core/_index.md" >}}).

A structure of type `struct evl_version` containing such information
is returned (this call cannot fail). The definition of this type is as
follows:

```
struct evl_version {
	int api_level;	/* libevl.so: __EVL__ */
	int abi_level;	/* core: EVL_ABI_PREREQ, -1 for ESHI */
	const char *version_string;
};
```

- `api_level` matches the value carried by the `__EVL__`
  macro-definition when the [libevl]({{< relref
  "core/user-api/_index.md" >}}) code was compiled. A list of released
  API versions is available [in this document]({{< relref
  "core/user-api/api-revs.md" >}}).

- the `abi_level` is dynamically returned from the [EVL core]({{<
  relref "core/_index.md" >}}) running on the current machine. Details
  about ABI management in EVL can be found [in this document]({{<
  relref "core/under-the-hood/abi.md" >}}).

- a version string which collates all the revision information
  available for pretty-printing.

---

{{<lastmodified>}}
