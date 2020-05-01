---
menuTitle: "Misc. services"
title: Miscellanea
weight: 70
---

A set of ancillary services which are not directly related to [EVL
elements]({{< relref "core/user-api/_index.md#evl-core-elements" >}}).

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

{{< proto evl_sigdebug_handler >}}
void evl_sigdebug_handler(int sig, siginfo_t *si, void *ctxt)
{{< /proto >}}

This routine is a basic signal handler for the [SIGDEBUG]({{< relref
"core/user-api/thread/_index.md#hm-sigdebug" >}}) signal which simply
prints out the HM diagnostics received to _stdout_ then returns.

`libevl` does _not_ install this handler by default, this is up to
your application to do so if need be.

_sig_, _si_ and _ctxt_ correspond to the parameters received by the
_sa\_sigaction_ handler which your application should install using
[sigaction()](http://man7.org/linux/man-pages/man2/sigaction.2.html)
(`SA_SIGINFO` must be set in the action flags).

---

{{<lastmodified>}}
