---
menuTitle: "Memory heap"
weight: 53
---

### Dynamic memory allocation from out-of-band context

EVL applications usually have to manage the RAM resource dynamically,
so that the number of live objects they may need to maintain, and for
how long they would have to do so, does not have to be defined or even
known in advance. For this purpose, `libevl` implements a memory heap
manager usable from the [out-of-band context]({{< relref
"dovetail/altsched.md" >}}), from which objects can be allocated or
released dynamically in a time-bounded fashion. The EVL heap manager
organizes the portion of RAM the application has provided, [allocating
chunks]({{< relref "#evl_alloc_heap" >}}) from this area and
[releasing them]({{< relref "#evl_alloc_heap" >}}) on request. There
is no arbitrary limit on the number of heaps an application can
create, which is only limited to the available system resources. There
is a limit on the largest heap one can create though, which is 2Gb.

#### How an EVL heap works

The memory allocation scheme is a variation of the algorithm described
in a USENIX 1988 paper called "Design of a General Purpose Memory
Allocator for the 4.3BSD Unix Kernel" by Marshall K. McKusick and
Michael J. Karels. You can find it at [various
locations](http://docs.FreeBSD.org/44doc/papers/kernmalloc.pdf) on the
Internet.

An EVL heap organizes the memory it has been given as a set of
fixed-size pages where allocated blocks live, with each page worth 512
bytes of storage. Pages can be either part of the free pool, or busy
storing user data. A busy page either contains one or multiple blocks
(aka _chunks_), or it may be part of a larger block which spans
multiple contiguous pages in memory. Pages containing chunks (as
opposed to pages representing a portion of a larger block) are grouped
by common chunk size, which is always a power of 2. Every allocation
request is rounded to the next power of 2, with a minimum of 16 bytes,
e.g. calling [evl_alloc_heap()]({{< relref "#evl_alloc_heap" >}}) for
60 bytes will reserve a 64-byte chunk internally.

To this end, the heap manager maintains:

- a free page pool. At initialization time, most of the memory area
  passed to [evl_init_heap()]({{< relref "#evl_init_heap" >}}) is
  forming this pool. This pool is maintained in an [AVL
  tree](https://en.wikipedia.org/wiki/AVL_tree) to ensure time-bounded
  operations on inserting and removing pages.

- an array of list of pages used as a fast block cache, where each
  entry links pages occupied by chunks of the same size. Since the
  base page is 512 bytes long, the chunk size may range from 2^4 to
  2^8 bytes, which gives five entries. Within a single page, up to 32
  chunks of 16 bytes each are available, 16 chunks for 32 bytes chunks
  and so on, up to 2 chunks of 256 bytes.

The allocation strategy is as follows:

- if the application requests a chunk which is not larger than half
  the size of a page (i.e. 2^8 or 256 bytes), then the fast block
  cache is first searched for a free chunk. For instance, a request
  for allocating 24 bytes would cause a lookup into the fast cache for
  a free 32-byte long chunk. If no free chunk is available from the
  cache for that requested size, a new page is pulled from the free
  pool, added to the cache for the corresponding size, and the
  allocation proceeds from there, returning one chunk from the newly
  allocated page to the caller.

- if the requested chunk is larger than half the size of a page
  (i.e. >= 2^9 bytes), a set of contiguous pages which covers the
  entire allocation request is directly pulled from the free pool. In
  this case, the fast block cache is not used.

![Alt text](/images/heap.png "EVL heap management")

#### Some runtime characteristics

- the implementation is thread-safe, using an EVL [mutex]({{< relref
  "core/user-api/mutex/_index.md" >}}) internally to serialize callers
  while updating the heap state.

- O(1) access guarantee on allocation and release of free chunks into
  the fast block cache, ranging from 16 to 256 bytes. This is the
  typical allocation pattern an EVL heap is good at handling very
  quickly.

- O(log n) access guarantee on allocation and release of pages from/to
  the free pool.

- the EVL heap yields limited internal fragmentation for small chunks
  which can fit into the fast block cache. Since there is no meta-data
  associated to busy blocks pulled from the cache, the memory overhead
  is basically zero in this case. The larger the chunk, the larger the
  internal fragmentation since all requested sizes are aligned on the
  next power of 2, up to 2^9. So asking for 257 bytes would actually
  consume an entire 512-byte page internally, directly pulled from the
  free pool.

- for precise information about the runtime performance of the EVL
  heap manager on your platform, you may want to have a look at the
  output of the `heap_torture` test with verbosity turned on, as
  follows:

```
# /usr/evl/tests/heap-torture -v
```

#### Other options for a real-time capable memory allocator

- Another option for a deterministic memory allocator would be
[TLSF](http://www.gii.upv.es/tlsf/), which would easily work on top of
EVL's out-of-band context with some limited adaptation. It is slightly
faster than the EVL heap on average, but internal fragmentation for
the typical use cases it was confronted to looks much higher.

- The fragmentation issue of the original TLSF implementation seems to
have led to a [later
implementation](https://github.com/mattconte/tlsf) addressing the
problem.

These are only examples which come to mind. There should be no
shortage of memory allocators you could adapt for running EVL
applications. You would only need to make sure to use EVL services
exclusively in that code, like [mutexes]({{< relref
"core/user-api/mutex/_index.md" >}}) if you need a thread-safe
implementation.

> Initializing a memory heap

This is how one could create a memory heap out of a static array of
characters defined in a program:

```
	#include <evl/heap.h>

	static char heap_storage[EVL_HEAP_RAW_SIZE(1024 * 1024)]; /* 1Mb heap */
	static struct evl_heap runtime_heap;

	int init_runtime_heap(void)
	{
		int ret;
		
		ret = evl_init_heap(&runtime_heap, heap_storage, sizeof heap_storage);
		...

		return ret;
	}
```

Likewise, but this time using
[malloc(3)](http://man7.org/linux/man-pages/man3/malloc.3.html) to get
the raw storage for the new heap:

```
	#include <stdlib.h>
	#include <evl/heap.h>

	static struct evl_heap runtime_heap;

	int init_runtime_heap(void)
	{
		const size_t raw_size = EVL_HEAP_RAW_SIZE(1024 * 1024); /* 1Mb heap */
		void *heap_storage;
		int ret;

		heap_storage = malloc(raw_size);
		...
		ret = evl_init_heap(&runtime_heap, heap_storage, raw_size);
		...

		return ret;
	}
```

### Memory heap services {#memory-heap-services}

{{< proto evl_init_heap >}}
int evl_init_heap(struct evl_heap *heap, void *mem, size_t size)
{{< /proto >}}

This service initializes a heap, based on a memory area provided by
the caller. This area should be large enough to contain both the user
payload, and the meta-data the heap manager is going to need for
maintaining such payload. The length of such area is called the _raw
size_, as opposed to the lesser amount of bytes actually available for
storing the user payload which is called the _user size_.

{{% argument heap %}}
An in-memory heap descriptor is constructed by `evl_init_heap()`,
which contains ancillary information other calls will need. _heap_
is a pointer to such descriptor of type `struct evl_heap`.
{{% /argument %}}

{{% argument mem %}}
The start address of the _raw_ memory area which is given to the heap
manager for serving dynamic allocation requests.
{{% /argument %}}

{{% argument size %}}
The size (in bytes) of the raw memory area starting at _mem_. This size
includes the space required to store the meta-data needed for
maintaining the new heap. You should use the
`EVL_HEAP_RAW_SIZE(user_size)` macro to determine the total amount
of memory which should be available from _mem_ for creating a heap
offering up to _user\_size_ bytes of payload data. For instance, you
would use `EVL_HEAP_RAW_SIZE(8192)` as the value of the _size_
argument for creating a 8Kb heap.
{{% /argument %}}

Zero is returned on success. Otherwise, a negated error code is
returned:

-	-EINVAL is returned if _size_ is invalid, cannot be used to derive
        a proper user size. A proper user size should be aligned on
	a page boundary (512 bytes), cover at least one page of storage,
	without exceeding 2Gb.

-	Since `evl_init_heap()` creates a mutex for protecting access to
	the heap meta-data, any return code returned by
	[evl_new_mutex()]({{< relref "core/user-api/mutex/_index.md#evl_new_mutex" >}}).

---

{{< proto evl_extend_heap >}}
int evl_extend_heap(struct evl_heap *heap, void *mem, size_t size)
{{< /proto >}}

Add more storage space to an existing heap. The extent space should be
large enough to contain both the additional user payload, and the
meta-data the heap manager is going to need for maintaining such
payload.

{{% argument heap %}}
The in-memory mutex descriptor previously constructed by
[evl_init_heap()]({{< relref "#evl_init_heap" >}}).
{{% /argument %}}

{{% argument mem %}}
The start address of the _raw_ memory area which is added to the heap.
{{% /argument %}}

{{% argument size %}}
Like with [evl_init_heap()]({{< relref "#evl_init_heap" >}}), the size
(in bytes) of the raw memory area starting at _mem_ which includes the
space required to store the meta-data needed for maintaining the
additional set of pages. You should use the
`EVL_HEAP_RAW_SIZE(user_size)` macro to determine the total amount of
memory which should be available from _mem_ for creating an extent
offering up to _user\_size_ bytes of additional payload data. For
instance, you would use `EVL_HEAP_RAW_SIZE(8192)` as the value of the
_size_ argument for creating a 8Kb extent.
{{% /argument %}}

Zero is returned on success. Otherwise, a negated error code is
returned:

-	-EINVAL is returned if _size_ is invalid, cannot be used to derive
        a proper user size. A proper user size should be aligned on
	a page boundary (512 bytes), cover at least one page of storage,
	without exceeding 2Gb.

---

{{< proto evl_destroy_heap >}}
void evl_destroy_heap(struct evl_heap *heap)
{{< /proto >}}

This call dismantles a memory heap. The user storage is left unspoiled
by the deletion process, and no specific action is taken regarding the
raw storage received from either [evl_init_heap()]({{< relref
"#evl_init_heap" >}}) or [evl_extend_heap()]({{< relref
"#evl_extend_heap" >}}) when the heap was active. Such storage should
be further released by the caller if need be.

{{% argument heap %}}
The in-memory mutex descriptor previously constructed by
[evl_init_heap()]({{< relref "#evl_init_heap" >}}) to be dismantled.
{{% /argument %}}

---

{{< proto evl_alloc_block >}}
void *evl_alloc_block(struct evl_heap *heap, size_t size)
{{< /proto >}}

Allocate a chunk of memory from a given heap.

{{% argument heap %}}
The in-memory mutex descriptor previously constructed by
[evl_init_heap()]({{< relref "#evl_init_heap" >}}).
{{% /argument %}}

{{% argument size %}}
The amount of bytes to allocate.
{{% /argument %}}

A valid memory pointer is returned on success, otherwise NULL. The
following is true for any chunk of memory returned by
`evl_alloc_block()`:

- if _size_ is smaller than 512 bytes, it is rounded up to the next
  power of 2, with a minimum of 16 bytes.

- if _size_ is larger than 512 bytes, it is rounded up to the next
  512-byte boundary.

- the address of the new chunk is aligned on a 16-byte boundary.

---

{{< proto evl_free_block >}}
int evl_free_block(struct evl_heap *heap, void *block)
{{< /proto >}}

Release a previously allocated chunk of memory, returning it to the
originating _heap_.

{{% argument heap %}}
The in-memory mutex descriptor previously constructed by
[evl_init_heap()]({{< relref "#evl_init_heap" >}}).
{{% /argument %}}

{{% argument block %}}
The address of a chunk to release which was originally obtained from
[evl_alloc_block()]({{< relref "#evl_alloc_block" >}}).
{{% /argument %}}

---

{{< proto evl_check_block >}}
int evl_check_block(struct evl_heap *heap, void *block)
{{< /proto >}}

Check if _block_ is an active memory chunk living in _heap_. The
thoroughness of this test depends on whether `libevl` was compiled
with the optimizer disabled (i.e. -O0 passed to the compiler, causing
the `__OPTIMIZE__` macro flag NOT to be defined at build time). If the
optimizer was enabled, this routine may not be able to detect whether
_block_ is part of a valid data page. Therefore, you would probably
rely on this service only with debug builds.

{{% argument heap %}}
The in-memory mutex descriptor previously constructed by
[evl_init_heap()]({{< relref "#evl_init_heap" >}}).
{{% /argument %}}

{{% argument block %}}
The address of a block to check for sanity.
{{% /argument %}}

Zero is returned if _block_ which was originally obtained from
[evl_alloc_block()]({{< relref "#evl_alloc_block" >}}) and has not yat
been released at the time of the call. Otherwise, a negated error code is returned:

-    -EINVAL if _block_ is not an active memory chunk previously returned
     by [evl_alloc_block()]({{< relref "#evl_alloc_block" >}}).

---

{{< proto evl_heap_raw_size >}}
size_t evl_heap_raw_size(struct evl_heap *heap)
{{< /proto >}}

Return the [raw size]({{< relref "#memory-heap-services" >}}) of the
memory storage associated with _heap_. This size includes the storage
which may have been further added to the heap by calling
[evl_extend_heap()]({{< relref "#evl_extend_heap" >}}).

{{% argument heap %}}
The in-memory mutex descriptor previously constructed by
[evl_init_heap()]({{< relref "#evl_init_heap" >}}).
{{% /argument %}}

Return the raw size (in bytes) of the memory storage associated with
_heap_.

---

{{< proto evl_heap_size >}}
size_t evl_heap_size(struct evl_heap *heap)
{{< /proto >}}

Return the user (or payload) size of the memory storage associated
with _heap_. This size includes the payload storage which may have
been further added to the heap by calling [evl_extend_heap()]({{<
relref "#evl_extend_heap" >}}). This is the actual amount of bytes
available for storing user data, which is lesser than the [raw heap
size]({{< relref "#evl_heap_raw_size" >}}) since it does not account
for the meta-data.

{{% argument heap %}}
The in-memory mutex descriptor previously constructed by
[evl_init_heap()]({{< relref "#evl_init_heap" >}}).
{{% /argument %}}

Return the user size (in bytes) of the memory storage associated with
_heap_.

---

{{< proto evl_heap_used >}}
size_t evl_heap_used(struct evl_heap *heap)
{{< /proto >}}

Return the amount of space already consumed from the user (or payload)
area.

{{% argument heap %}}
The in-memory mutex descriptor previously constructed by
[evl_init_heap()]({{< relref "#evl_init_heap" >}}).
{{% /argument %}}

Return the amount of space (in bytes) consumed within _heap_. This
value is lower or equal to the user (or payload) size.

---

{{<lastmodified>}}
