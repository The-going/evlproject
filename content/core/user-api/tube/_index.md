---
menuTitle: "Tube"
weight: 55
---

### A lightweight FIFO messaging mechanism

The tube is a lighweight and flexible FIFO data structure which you
can use to send any type of data between threads. Since the algorithm
manipulating a tube is lockless, threads can use it regardless of
their respective [execution stage]({{< relref
"dovetail/altsched/_index.md#altsched-theory" >}}), therefore a tube
could also be used as a very simple [inter-stage messaging system]({{<
relref "#inter-stage-tube" >}}).  In addition, the tube supports the
multi-reader and multi-writer paradigms. The intent is to provide a
basic mechanism which can either be used "as is" for fully
non-blocking send/receive operations, or as a building block for
implementing featureful message queues including sleeping wait for
input and output congestion control.

At its core, an EVL _tube_ is basically a one-way linked list
conveying so-called _canisters_ in a FIFO manner. Each canister is a
container for a data item (or payload) a thread wants to send to
another thread.

![Alt text](/images/tube.png "Lightweight data tube")

#### Creating a tube {#tube-creation}

The following steps are required to create a tube:

1. You first need to declare the C structure type of the canister
which is going to convey data through that tube. This is done by using
the `DECLARE_EVL_TUBE_CANISTER(canister_tag, data_type)` macro in your
code, which should be given the name of the new canister type (used as
the C structure tag), and the C type of the data to be conveyed in
that canister. For instance, the following snippet would declare the
`j1587_canister` type, conveying J1587 protocol data (some automotive
diagnostic protocol, could be anything else of use of course):

  ```
  #include <evl/tube.h>

  /*
     The following macro call defines the C struct type
     describing a canister as:
    
     struct j1587_canister {
          struct j1587_data payload;
          struct j1587_canister *next;
     };
   */
  struct j1587_data {
  	 uint16_t pid;
  	 uint16_t ecu;
	 unsigned char data[17];
  };
  DECLARE_EVL_TUBE_CANISTER(j1587_canister, struct j1587_data);
  ```

	This macro expands to a complete C `struct j1587_canister`
  definition, laying out the information needed to convey one item of
  type `struct j1587_data`.

2. Then you need to declare the tube data structure itself, mentioning
which kind of canister is going to flow through it. Note that a tube
is fit for one specific type of canister, you cannot use multiple
types of canister with a single tube. Such declaration is obtained by
expanding the `DECLARE_EVL_TUBE(tube_tag, canister_tag)` macro, which
receives the name of the new tube type (used as a C structure tag) and
the canister tag passed earlier to `DECLARE_EVL_TUBE_CANISTER()`.

  ```
  #include <evl/tube.h>

  /*
     The following macro call defines the C struct type
     describing a tube conveying canisters which in turn contain
     a payload of type j1587_data:
    
     struct j1587_tube {
          ...
     };
   */
  DECLARE_EVL_TUBE(j1587_tube, j1587_canister);
  ```

3. Finally, you need to initialize the tube by a call to
[evl_init_tube()]({{< relref "#evl_init_tube" >}}), passing it a
memory area which should be large enough to store as many canisters as
required. This should be seen as the maximum number of canisters which
can be queued into the tube at once. In other words, this is the
maximum number of messages such queue can hold without running out of
memory. The initialization code can refer to the declarations of the
canister and tube types obtained by DECLARE_EVL_TUBE_CANISTER() and
DECLARE_EVL_TUBE():

  ```
  #include <stdio.h>
  #include <evl/tube.h>

  /* The tube can convey up to 16 floating-point values at once. */
  static DECLARE_EVL_TUBE_CANISTER(j1587_canister, struct j1587_data) canisters[16];
  static DECLARE_EVL_TUBE(j1587_tube, j1587_canister) tube;

  evl_init_tube(&tube, canisters, 16);
  printf("j1587 tube can convey %ld messages concurrently\n",
  		      tube.max_items); /* should display as '16' */
  ```

Once the tube is initialized, threads can communicate over it by
calling [evl_send_tube()]({{< relref "#evl_send_tube" >}}) and
[evl_receive_tube()]({{< relref "#evl_receive_tube" >}}).

{{% notice note %}}
You can use any valid C type to hold the payload into a
canister. However, keep in mind that conveying a payload entails two
copies of the corresponding data: first to install it into the
canister when [evl_send_tube()]({{< relref "#evl_send_tube" >}}) is
called, next when [evl_receive_tube()]({{< relref "#evl_receive_tube"
>}}) extracts it from the canister at the other end to private memory,
so you definitely want to keep the payload size reasonable. If you need to
convey large bulks of data as single messages flowing through the
tube, then you should consider declaring a canister type which only stores
a pointer to the final data, so that only that pointer needs to be copied, e.g.:
{{% /notice %}}

  ```
  #include <evl/tube.h>

  /*
     The following macro call defines the C struct type
     describing a canister as:
    
     struct massive_bitmap_canister {
          void *payload;
          struct massive_bitmap_canister *next;
     };
   */
  DECLARE_EVL_TUBE_CANISTER(massive_bitmap_canister, void *);
  ```

#### Using tubes for inter-process messaging {#inter-process-tube}

A special form of tube can be used for transferring data between
processes, using the *__rel()_ interface variant, which stands for
_relative addressing_. As this implies, all internal references within
the tube data structure use base-offset addressing instead of absolute
memory pointers, so that such data structure can be mapped onto a
piece of memory shared between processes via [mmap(2)](
http://man7.org/linux/man-pages/man2/mmap.2.html). `DECLARE_EVL_TUBE_REL()`
should be used to define the C type of the new inter-process tube. In
addition, [evl_init_tube_rel()]({{< relref "#evl_init_tube_rel" >}})
should be used for initializing the such tube,
[evl_send_tube_rel()]({{< relref "#evl_send_tube" >}}) for pushing
data through it, [evl_receive_tube_rel()]({{< relref
"#evl_receive_tube_rel" >}}) for pulling available messages from it
and so on.

Of course, you still need to refrain from conveying absolute pointers
referring to a particular process address space into the message
payload.

#### Using tubes for out-of-band &#8660; in-band messaging {#inter-stage-tube}

Since sending and receiving to/from a tube is performed locklessly and
does not involve any system call, this data structure can be used for
implementing a basic message queue between threads which may belong to
different stages, which is an alternative to using a
[cross-buffer]({{< relref "core/user-api/xbuf/_index.md" >}}).

#### Blocking input and/or output congestion control {#synchronizing-tube-ops}

A tube is inherently non-blocking, it neither imposes any policy nor
provides support for sleeping in absence of input or for dealing with
output congestion. It only provides a very simple lockless mechanism
for transferring arbitrary data between peers. This means that
[evl_send_tube()]({{< relref "#evl_send_tube" >}}) may return a
failure status (i.e. boolean _false_) if no free canister is available
for conveying data at the time of the call. Conversely,
[evl_receive_tube()]({{< relref "#evl_receive_tube" >}}) may also
return a failure status in case no data is immediately available at
the receiving end of the tube. If you need the sender(s) to handle
output congestion by sleeping until canisters are free for sending to
the other side, or the receiver(s) to sleep until some data is
available for input, the trick is to combine a tube with the proper
synchronization mechanisms. For instance:

- if both the sender(s) and receiver(s) run on the [out-of-band
stage]({{< relref "dovetail/altsched/_index.md#altsched-theory" >}}),
then a pair of [EVL semaphores]({{< relref
"core/user-api/semaphore/_index.md" >}}) would suffice: the sender
would [wait on a semaphore]({{< relref
"core/user-api/semaphore/_index.md#evl_get_sem" >}}) counting the
number of free canisters in the tube before attempting to [push a new
data item]({{< relref "#evl_send_tube" >}}), and the receiver would
wait on the other semaphore counting the number of canisters available
for [reading from the tube]({{< relref "#evl_receive_tube"
>}}). Conversely, the receiver would [signal the semaphore]({{< relref
"core/user-api/semaphore/_index.md#evl_put_sem" >}}) counting the free
canisters after it has successfully pulled a data item, and the sender
would signal the input semaphore after it has successfully pushed a
new data item.

- if the sender or the receiver run on the [out-of-band stage]({{<
relref "dovetail/altsched/_index.md#altsched-theory" >}}) but its peer
may only run in-band, you could use one [EVL semaphore]({{< relref
"core/user-api/semaphore/_index.md" >}}), and a [proxy]({{< relref
"core/user-api/proxy/_index.md" >}}) associated with a regular
[eventfd(2)](http://man7.org/linux/man-pages/man2/eventfd.2.html).
How to synchronize two threads which belong to distinct execution
stages using such combo is explained in [this document]({{< relref
"core/user-api/proxy/_index.md#proxy-synchronization-example" >}}).

The following figure summarizes these options for synchronizing tube
operations:

![Alt text](/images/tubesync.png "Synchronizing tube operations")

### Tube services {#tube-services}

{{< proto evl_init_tube >}}
void evl_init_tube(struct {tube_tag} *tube, struct {canister_tag} freevec[], int count)
{{< /proto >}}

Initialize a tube data structure for process-local use, which can only
happen once the canister and tube types [have been defined]({{< relref
"#tube-creation" >}}).

{{% argument tube %}}
The address of the tube structure to initialize. The C type of the new
tube should have been declared earlier by the [DECLARE_EVL_TUBE()]({{<
relref "#tube-creation" >}}) macro.
{{% /argument %}}

{{% argument freevec %}}
The start address of an array of canisters which should be used to
convey the payload through the tube. The C type of the basic element
should have been declared earlier by the
[DECLARE_EVL_TUBE_CANISTER()]({{< relref "#tube-creation" >}})
macro.
{{% /argument %}}

{{% argument count %}}
The number of elements in _freevec_.
{{% /argument %}}

---

{{< proto evl_init_tube_rel >}}
void evl_init_tube_rel({tube_tag}, {canister_tag}, void *mem, size_t memsize)
{{< /proto >}}

Initialize a relative-addressing tube data structure for inter-process
use, which can only happen once the canister and tube types [have been
defined]({{< relref "#tube-creation" >}}). Unlike with the
[process-local variant]({{< relref "#evl_init_tube" >}}), the
relative-addressing variant requires both the tube meta-data and the
canisters to be part of the same (shareable) memory segment so that
all peers have easily access to both of them.

{{% argument tube_tag %}}
The C tag of the tube type to initialize. The C type of the new
tube should have been declared earlier by the [DECLARE_EVL_TUBE_REL()]({{<
relref "#tube-creation" >}}) macro.
{{% /argument %}}

{{% argument mem %}}
The start address of the shared memory area where both the tube
meta-data and canisters will live.
{{% /argument %}}

{{% argument memsize %}}
The length in bytes of _mem_. Because a portion of _mem_ is reserved
for storing meta-data, the number of canisters available for conveying
actual payload through the tube is lesser than `memsize /
sizeof(struct <canister_tag>)`. You can use
[evl_get_tube_size_rel()]({{< relref "#evl_get_tube_size_rel" >}}) to
calculate the exact amount of memory you would need for storing a
process-shared tube given the canister type and a maximum number of
messages flowing concurrently through the tube. The returned value
should be used to allocate the shared memory segment, then passed to
[evl_init_tube_rel]({{< relref "#evl_init_tube_rel" >}}).
{{% /argument %}}

This macro returns the tube address which should be used in other
relative-addressing calls. You can retrieve the number of canisters
available with this tube by fetching the `max_items` members from the
tube C type.

---

{{< proto "evl_send_tube" >}}
bool evl_send_tube[_rel](struct {tube_tag} *tube, {payload})
{{< /proto >}}

Send a data item through a tube in a FIFO manner. The __rel_ variant
should be used for sending to a relative-addressing tube. This payload
is first copied to a free canister, which is then queued for
consumption by the receiving end.

{{% argument tube %}}
The address of the tube to push the data to.
{{% /argument %}}

{{% argument payload %}}
The data to push to the tube. This argument is passed by reference to
the [evl_send_tube()]({{< relref "#evl_send_tube" >}}) macro.
{{% /argument %}}

This macro returns a boolean _true_ value on success, or _false_ in
case no free canister was available for sending more data at the time
of the call. If you need a mechanism to have the sender block on
output contention, you may want to have a [look at this section]({{<
relref "#synchronizing-tube-ops" >}}).

---

{{< proto "evl_receive_tube" >}}
bool evl_receive_tube[_rel](struct {tube_tag} *tube, {payload})
{{< /proto >}}

Receive the next available data item from a tube. The __rel_ variant
should be used for receiving from a relative-addressing tube. The
incoming payload is eventually copied to the _payload_ argument before
the conveying canister is made available anew for further sending.

{{% argument tube %}}
The address of the tube to pull data from.
{{% /argument %}}

{{% argument payload %}}
A variable to be assigned the payload data received from the
tube. This argument is passed by reference to the
[evl_receive_tube()]({{< relref "#evl_receive_tube" >}}) macro.
{{% /argument %}}

This macro returns a boolean _true_ value on success, or _false_ in
case no free canister was available for sending more data at the time
of the call. If you need a mechanism to have the receiver block on
lack of input, you may want to have a [look at this section]({{<
relref "#synchronizing-tube-ops" >}}).

---

{{< proto "evl_get_tube_size" >}}
bool evl_get_tube_size[_rel]({tube_tag}, int count)
{{< /proto >}}

Calculate the exact amount of memory required for storing a tube data
structure given the canister type and a maximum number of messages
flowing concurrently through that tube. This size represents the total
amount of memory which is needed for storing a complete tube data
structure.

{{% argument tube_tag %}}
The C tag of the tube type to get the size of.
{{% /argument %}}

{{% argument count %}}
The number of messages which can flow concurrently through the tube.
{{% /argument %}}

Return the size in bytes representing the amount of memory required
for storing both the meta-data and the canisters for the given tube
type.

---

{{<lastmodified>}}
