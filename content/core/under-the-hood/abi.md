---
menuTitle: "ABI revisions"
title: "The EVL ABI"
weight: 10
---

Since we have service calls between the applications and the EVL core,
we depend on a certain format of request codes, arguments and return
values when exchanging information between both ends, like the
contents of the argument structure passed to the various
[ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) and
[oob_ioctl()]({{< relref "core/user-api/io/_index.md#oob_ioctl" >}})
requests. This convention forms the EVL ABI. As hard as we might try
to keep the ABI stable over time, there are times when this cannot be
achieved, at the very least because we may want to make new features
added to the EVL core visible to applications. Generally speaking, EVL
being a new kid in the old dual kernel town, it is too early - for the
time being - to write the current convention in stone. This fact of
life calls for a way to unambiguously:

- determine which (range of subsequent) revision(s) of the ABI is
  _implemented_ by the running EVL core, from the application side.

- state which revision of the ABI an application _requires_.

In some instance, the EVL core may also be able to provide a
backward-compatible interface to applications which spans multiple
subsequent ABI revisions. Because **ABI stability is a strong goal**,
backward compatibility may become the norm as the EVL core feature set
stabilizes.

To this end, the following apply (since ABI revision #19):

- an ABI is versioned. Each version is represented by a non-null
  integer starting at 1, incremented by one after each revision.

- the EVL core defines two symbols, namely
  [EVL_ABI_BASE](https://git.evlproject.org/linux-evl.git/tree/include/uapi/evl/control.h?h=evl/next)
  and
  [EVL_ABI_LEVEL](https://git.evlproject.org/linux-evl.git/tree/include/uapi/evl/control.h?h=evl/next),
  which represent the range of ABI revisions the core supports
  (inclusively), from oldest (EVL_ABI_BASE) to current (EVL_ABI_LEVEL)
  respectively. Differing values means that the EVL core honors
  multiple subsequent revisions by way of backward compatible support
  of some sort.

{{% notice tip %}}
By convention, changes to EVL_ABI_BASE and/or
EVL_ABI_LEVEL belong to the same GIT commit which triggered the ABI
revision. This way, it is easy to map a revision to a particular
change unambiguously. For the same reason, non-related changes to the
ABI appear in separate commits, each one forming a particular ABI
revision.
{{% /notice %}}

- [libevl]({{< relref "core/user-api/_index.md" >}}) states the ABI
  revision it follows by defining the
  [EVL_ABI_PREREQ](https://git.evlproject.org/libevl.git/tree/include/evl/evl.h?h=next)
  symbol. When [libevl initializes]({{< relref
  "core/user-api/init/_index.md" >}}), this prerequisite is checked
  against the range of ABI revisions the running EVL core
  advertises. If a mismatch is detected, the initialization fails and
  -ENOEXEC is returned to the caller; otherwise the execution proceeds
  normally. An easy way to figure out which ABI a [libevl]({{< relref
  "core/user-api/_index.md" >}}) installation requires is to ask the
  [evl command]({{< relref "core/commands.md" >}}) about it as
  follows:

  ```
  ~ # evl -V
  evl.0 -- #1c6115c (2020-03-06 16:24:00 +0100) [requires ABI 19]
  ```

#### Locating ABI boundaries and prereqs

When developing (with) the EVL core, it may sometimes be useful to
find which code changes in the GIT commit history correspond to ABI
revisions and conversely. It may also be useful to determine the
history of ABI prerequisites in [libevl]({{< relref
"core/user-api/_index.md" >}}), so that you can map any particular
point the commit history with the ABI revision it is compatible
with. For this, you can use a simple script called
[git-evlabi](https://git.evlproject.org/linux-evl.git/tree/scripts/git-evlabi?h=evl/next),
which is available in the
[linux-evl](https://git.evlproject.org/linux-evl.git) repository. You
just need to make sure this script is executable and reachable from
the $PATH variable.

git-evlabi looks for EVL ABI definitions or prereqs in GIT trees
cloned from either
[linux-evl](https://git.evlproject.org/linux-evl.git) or
[libevl](https://git.evlproject.org/libevl.git) respectively. It does
so depending on the value and type of the GIT object mentioned (or
not) in the command:

```
$ git evlabi -h
usage: git-evlabi [-r <rev>][--git-dir <dir>][-s][-h] [<object>]
```

- if \<object\> is a regular commit (SHA-1) hash, it extracts the ABI
  information for that particular commit.

- if \<object\> matches a GIT refname (typically a branch name), it
  scans the commit history backward from that point, displaying the
  ABI information for each revision encountered during the traversal.

- if \<object\> is omitted, it defaults to HEAD, which falls back to
  the refname case above.

The output is of the form:

  \<start\> \<range\> \<revision\>  [\<shortlog\>], where:

  - \<start\> is the predecessor of the earliest commit
    implementing/requesting the ABI revision
    
  - \<range\> is the span of the ABI revision in the history (usable
    with _git log_ for instance).
    
  - \<revision\> is the ABI revision number
    
  - \<shortlog\> is the subject line describing \<start\>

> e.g. asking for all EVL ABI revisions in the kernel tree
```
{rpm@cobalt} cd ~/git/linux-evl && git evlabi evl/next
4bc9b71134f0 96c899ef3a6c..bc7ca4ad0b5c  19  
02ae0b8783cb 7f082bc5bef1..96c899ef3a6c  18  
c2a51a7e68a7 56e217799766..7f082bc5bef1  17  
5266e26ca729 857c199beead..56e217799766  16  
54559813955b 91f7d9d3bc73..857c199beead  15  
fb0de980dac9 7795aa5979a1..91f7d9d3bc73  14  
437fee2b84ee 1b627b1977a1..7795aa5979a1  13  
60c937d72a72 03f35888596c..1b627b1977a1  12  
db431047b9bd 68b176e50081..03f35888596c  11  
8f5b07b678ea 92f04516c8dd..68b176e50081  10  
b0eb03db5c77 8f9bb48957ba..92f04516c8dd   9  
b5f675b3e2c0 5e640e734385..8f9bb48957ba   8  
7892bfe4886e 9ddba326d41b..5e640e734385   7  
9ddba326d41b 61e9a6b78aa5..9ddba326d41b   6  
badb3bf8cee8 ba708547b903..61e9a6b78aa5   5  
ba708547b903 7d9aa309a81d..ba708547b903   4  
06b78dd09bfa 3dd263191153..7d9aa309a81d   3  
6170f4f1d9fd fd56dfbfdbbf..3dd263191153   2  
800484643595 f8ef2539553e..fd56dfbfdbbf   1  
f8ef2539553e fec5d8902f49..f8ef2539553e   0  
```

With the information above, it is easy to figure out the span of a
given ABI revision in the kernel code (here ABI 17 starting at commit
#c2a51a7e68a7), as follows:

> e.g. looking for span of ABI 17 in the kernel tree
```
{rpm@cobalt} git log --oneline 56e217799766..7f082bc5bef1
7f082bc5bef1 evl/work: export interface to modules
54b365d416c7 evl/stax: export interface to modules
c268cadb0396 evl/stax: force the caller to in-band mode on WOSX
77d1de6add6e evl/clock: y2038: remove use of struct timex
05fc1d0e5e96 evl/thread: fix rescheduling leak
c2a51a7e68a7 evl/thread: replace SIGEVL_ACTION_HOME with a RETUSER event
```

The same logic applies to the
[libevl](https://git.evlproject.org/libevl.git) tree, this time for
determining the ABI prerequisites of commits in the API.

> e.g. walking the history of ABI prerequisites
```
{rpm@cobalt} git evlabi next
dde33b5 1c6115c..dde33b5  19  
```

Which means that looking at the current state of branch _next_ in the
[libevl](https://git.evlproject.org/libevl.git) tree, ABI revision 19
in the EVL core starts being a prerequisite for the code starting at
commit #dde33b5.

Conversely, you could ask for matching a particular ABI revision to a
commit, using the -r switch.

> e.g. matching a particular ABI revision (asking for the subject line too)
```
{rpm@cobalt} git evlabi -r 16 -s
5266e26ca729 857c199beead..56e217799766  16  evl: introduce synchronous breakpoint support
```

{{% notice info %}}
git-evlabi cannot detect ABI revisions earlier than 19 in the
[libevl](https://git.evlproject.org/libevl.git) tree, since the
marker it looks for was introduced by that particular revision.
{{% /notice %}}

---

{{<lastmodified>}}
