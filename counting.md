Reference Counting for Reference Capabilities
=============================================

About Reference Counting
------------------------

In a nutshell, the idea of reference counting is to associate each object to
a counter keeping track of the number of references to that object over time
(e.g. using a dedicated integer field in the object's memory):
- when a new alias is created, the reference count is incremented
- when an alias is dropped, the reference count is decremented
- when the reference count drops down to zero, the object is freed

This is usually reduced to two fundamental reference counting operations:
- `INCREF`: increment the reference count
- `DECREF`: decrement the count and free the object if it reaches zero

In multi-threaded programs where objects can be shared between multiple threads,
care must be taken to ensure `INCREF` and `DECREF` are thread-safe. A simple
solution is to use atomic increment and decrement instructions. However these
are more expensive than the bare non-atomic increment and decrement.

To combine the best of both worlds, we will leverage the system of reference
capabilities so as to use atomic instructions only when it's necessary.


Two Categories of References
----------------------------

Depending on their capability, we distinguish two categories of references:
- *owning* references: `iso`, `syn`, `asy`
- *open* references: `mut`, `box`, `imm`

References like `iso`, `syn` and `asy` can be said to have *ownership* of all
*mutable* objects they can reach without *relaxing* another *owning* reference:
such *mutable* objects are only reachable from the *owning* `iso`, `syn` or
`asy` references.

By opposition, references like `mut`, `box` and `imm` are said to be *open*:
they allow *dereferencing* and external aliases to the objects they can reach.

The same object may simultaneously have *owning* and *open* references: `iso`,
`syn` and `asy` can be temporarily *relaxed* as `mut` or `box`, and they can
also *own* `mut` or `box` references to themselves via their fields:


```python
class LinkedList:
    mut LinkedList next

# o is is an owning reference
iso o = consume LinkedList()

# o can be relaxed as an open reference
# o can own an open reference to itself
with relaxed o as mut x:
    x.next = x
```

This distinction is useful because *owning* references are *isolated* as soon as
there is only one *owning* reference to this object, even if there are multiple
*open* references as well: the *open* references are necessarily only reachable
through the *owning* reference.


A Dual Reference Count
----------------------

We introduce a dual reference count with two counters instead of only one:
- an *open* reference count for *open* references
- an *owning* reference count for *owning* references

This makes it easy to check at runtime if an *owning* reference is *isolated*:
`syn` and `asy` references can be *consumed* when their *owning* count is one.

To facilitate dealing with two reference counts, we set the convention that as
long as the *owning* count is nonzero, the *open* count must be increased by
one (i.e. be one greater than the actual number of *open* references).

That way when the *open* count drops to zero, we know we can free the memory
without needing to inspect the *owning* count.

When the *owning* count drops to zero we can also free the memory immediately
regardless of the *open* count because all the existing *open* reference are
necessarily *owned* be the *owning* reference. In fact we can free the memory
of every object *owned* by the *owning* reference immediately.

Under the hood, this dual reference count can actually be implemented by using
objects with a standard single reference count, and allocating a second object
that holds a reference to the first when it is consumed to `syn` or `asy`. For
`iso` references we actually don't need an *owning* count because it would
always be one.


Distinguishing Between `box` and `box`
--------------------------------------

The `box` capability is particular in that it refers to *data* that is either
*mutable* or *immutable*, but we generally don't know which at compile-time.

However this information is crucial at runtime: whether an object is *mutable*
or *immutable* determines whether it must be *owned* by an *owning* reference
or whether external aliases are allowed.

When *consuming* a `mut` or `imm` reference, we will need to check whether
the *mutable* objects that are *directly reachable* from it (i.e. without
*relaxing* other *owning* references) are in fact *only* reachable from it
(i.e. *owned* by it).

Of course, for an `imm` reference all *directly reachable* objects are seen as
*immutable* themselves, so we're talking here about the *directly reachable*
objects that might become *mutable* once the `imm` reference is *consumed* into
something else.

Conversely, we want to allow all reachable *immutable* objects to have external
aliases without compromising our ability to check if a reference is *isolated*.
Again, we're speaking here of the objects that would remain *immutable* even
after the reference is *consumed*.

Therefore we need a way to distinguish between `box` references that may have
`mut` aliases, even if those `mut` aliases are currently *altered* to `imm`
because they are fields of an `imm` reference, and `box` references that refer
to intrinsically *immutable* objects, i.e. that do not have any `mut` aliases.

In a nutshell, when *consuming* a `mut` or `imm` reference, we need to know if
*directly reachable* `box` references will refer to *mutable* or *immutable*
objects after the reference is *consumed*.


```python
class VariousFields:
  mut f : Value
  box g : Value
  imm o : Value
  box p : Value

mut a : VariousFields
a.g = a.f
a.p = a.o

imm b = consume a

# p is an external alias of a box and an imm field of b
imm p = b.p

mut a = consume b  # OK
# p is an alias of a.p and a.o and all their capabilities are compatible

imm b = consume a

# p is an external alias of a box and a mut field of b
imm g = b.g

mut a = consume b  # ERROR!
# g would be an alias of a.g and a.f, but g is imm and a.f would be mut
```


Tagging `box` References
------------------------

Each `box` reference will store a tag alongside the actual pointer to the
referenced object denoting whether it was aliased from a `mut` or an `imm`.

In what follows, we will use this notation:
 - `box(0)`: a `box` reference tagged as aliased from `mut`
 - `box(1)`: a `box` reference tagged as aliased from `imm`

On some architectures we may take advantage of alignment requirements leaving
unused bits in the address representation to fold this information into the
actual pointer itself.

When an `imm` object has a `mut` field, the capability of the field is altered
to `imm`, so when aliasing the field to `box` is should be tagged as `box(1)`:
what matters is the way the field is currently seen.

In the same way, `box` fields of `imm` objects are altered to `imm` and should
be tagged as `box(1)` regardless of the tag of the field.

When aliasing from `box` to `box`, the new `box` alias will get the same tag as
the first reference.

When a `box` object has a `mut` field, the capability of the field is altered
to `box`: in that case, should the field be aliased to `box(0)` or `box(1)` ?

The anwser is the tag should be the same as that of the root `box` reference:
`box(0)` can have `mut` aliases and `box(1)` can have `imm` aliases, so they
must have the same behavior as their aliases.

Finally, when a `box` reference has a `box` field, should the field be aliased
to `box(0)` or `box(1)` ? Again, if the root reference is `box(0)` it should
behave like a `mut` reference, so the alias will get the same tag as the field.
But if the root reference is `box(1)`, it should behave like an `imm` and the
alias will be `box(1)` regardless of the tag of the field.


| reference | box alias |
|-----------|-----------|
| `mut`     | `box(0)`  |
| `box(0)`  | `box(0)`  |
| `box(1)`  | `box(1)`  |
| `imm`     | `box(1)`  |


|  reference -> field  | box alias |
|:--------------------:|:---------:|
| `box(0)` -> `box(0)` | `box(0)`  |
| `box(0)` -> `box(1)` | `box(1)`  |
| `box(1)` -> `box(0)` | `box(1)`  |
| `box(1)` -> `box(1)` | `box(1)`  |


In a nutshell, when aliasing from `box` to `box` the new alias will get the
same tag as the first reference, except if it is a field of a `box(1)` object:
in that case it must also be `box(1)`.

This all ensures that:
- if a reference is `box(1)`, it has no `mut` alias.
- if a reference can be aliased to `box(0)`, it has no `imm` alias.

Note that a `box(1)` reference can have aliases that are `mut` fields of an
`imm` object and thus *altered* to `imm`, but they cannot become `mut` again
while the `box(1)` reference exists as the `imm` object cannot be *consumed*,
since it has a `mut` field with external aliases (the `box(1)` alias).


Atomic Reference Counting
-------------------------

Depending on the capability of a reference, incrementing or decrementing its
reference count will require atomic operations in order to be thread-safe:


| capability | reference count | atomic |
|:----------:|:---------------:|:------:|
|    `mut`   |      *open*     |   no   |
|    `imm`   |      *open*     |   yes  |
|    `iso`   |        .        |    .   |
|    `syn`   |     *owning*    |   yes  |
|    `asy`   |     *owning*    |   yes  |
|  `box(0)`  |      *open*     |   no   |
|  `box(1)`  |      *open*     |   yes  |


When *relaxing*  an `iso`, `syn` or `asy` reference as `box` we can tag it as
`box(0)`, since it will necessarily only be reachable from a single thread at
a time.

The only exception is when we use a *readlock*: this grants multiple threads
read-only access, so the *relaxed* `box` reference must be tagged as `box(1)`
so that it uses atomic reference counting.


|       capability       | box alias |
|:----------------------:|:----------|
| `iso` / `syn` / `asy`  | `box (0)` |
| `syn` (readlocked)     | `box (1)` |



Relaxing `syn`, `asy` and `iso` References
------------------------------------------

When *relaxing* a reference into a `mut` or `box`, the *open* reference count
will be incremented for the new `mut` or `box` alias. When this alias goes out
of scope at the end of the isolated scope, or if it is manually deleted first,
the *open* reference count will be decremented. Since the *open* reference
count is increased by 1 when the *owning* count is nonzero, this decrement will
not drop the reference count to zero.

This way, a *relaxed* `mut` or `box` reference can be treated like any other
`mut` or `box` reference, with no need to worry about the object being freed
because the *open* reference count dropped to zero.


```python
syn a
# normal count: 1

with locked(a) as mut x:
  # normal count: 2

# normal count: 1
```

With `syn` and `asy` references, the *owning* reference count should also be
artifically incremented at the beginning of the scope and decremented at the
end (with `DECREF`). This ensures the object will not be freed before the end
of the scope even if the `syn` or `asy` reference is deleted inside the scope.

That is desirable because the *synchronisation mechanism* still needs to be
released at the end of the scope.


```python
syn a
# owning count: 1

with locked(a) as mut x:
  # owning count: 2
  # deleting a does not free the object immediately
  del a
  # owning count: 1

  # the owning count drops to zero at the end of the scope
  # and the object is freed
```


When *relaxing* an `iso` reference, the `iso` reference is not accessible from
inside the *isolated scope* anyway, so such precautions are not required.


Consuming `mut` References
--------------------------

To *consume* a `mut` reference, we first need to determine if it is *isolated*.
Sometimes this can be done with static analyis at compile-time, but in general
we may need to do a runtime check (and raise an exception if the check fails).

The runtime *isolation* check boils down to this:
1. Collect all `mut` and `box(0)` objects *directly reachable* from the root.
2. Sum the number of `mut` and `box(0)` references in each collected object.
3. Sum the *normal* reference counts of each collected object.

Note that the root object must be included in the collected objects.

Step 1 is achieved by recursively following `mut` and `box (0)` fields. The
visited object can be collected in a linked-list. In order to avoid memory
allocations during this step, each object can have a dedicated pointer field
used to link to the previous object in the list. This field can then double
as a flag for already collected objects: if the pointer is null, the object
hasn't been collected yet.

The root reference is *isolated* if and only if the sum computed in step 2 is
exactly one less than the sum computed in step 3. The difference accounts for
the root reference itself. If the difference is greater than one, it means
there are external references and the object is not *isolated*.

This runtime check can seem fairly expensive: it's computational complexity is
proportional to the size of the graph of *thread-local* object reachable from
the root reference. But it's still a fraction of the cost of creating all these
objects in the first place, and we can expect a *thread-local* object graph
will usually only need be *consumed* once into a *synchronised* or *immutable*
state. In addition, in many cases the `iso` capability can be used instead.


Consuming `imm` References
--------------------------

To *consume* an `imm` reference, the steps are very similar to the `mut` case:
1. Collect all `mut` and `box(0)` objects *directly reachable* from the root.
2. Sum the number of `mut` and `box(0)` references in each collected object.
3. Sum the *normal* reference counts of each collected object.

In step 1, the `mut` and `box (0)` objects are currently *immutable* since the
the root reference is `imm`, and `mut` and `box` fields are *altered* to `imm`.
But once the `imm` reference is *consumed* they will no longer be *immutable*.
The root reference must also be collected for the same reason.

Since an `imm` reference is *sendable*, any of the collected objects might be
accessible from multiple threads. But they are also all currently *immutable*,
so the set of reachable `mut` and `box (0)` objects cannot change while they
are being collected. However, another thread could concurrently be trying to
*consume* the same root object, or one of the objects that we need to collect
in step 1. This of course means that the root object is not *isolated*, and
both threads should fail to *consume*.

In order to avoid allocating memory during the *consume* operation, we'll use
a dedicated pointer field on the object to create a linked list of collected
objects. But there is a complication compared to *consuming* `mut` references:
another thread could be doing the same thing concurrently. So we need to mark
each collected object with an identifier unique to the current thread, e.g. an
address in that threads's stack. This requires one more field on the object.

Updating the linked list pointer and the thread identifier field should be done
atomically, and if an object is already marked by another thread it should not
be collected, and the *consume* operation should fail.

At this stage, we now have a linked list of all reachable `mut` and `box (0)`
objects as well as the root object. Step 2 presents no complications, but in
step 3 we need to atomically read the *open* reference count of each object
and sum them, but even this is not enough: another thread could concurrently
be creating and dropping aliases to the collected objects in such a way that
the sum we obtain is less than the actual total at a single point in time.

To detect this scenario, we've come up with the following scheme:
- the *open* reference count has reserved bit
- when updating the count of an `imm` or `box (1)`, that bit must be set
- before summing the collected objects, we reset each bit to zero
- after tallying the sum, we check that each bit is still zero

If the reserved bit of any of the collected objects is set, it means another
thread modified the reference counts while we were tallying them. This means
that the object is not *isolated*, so the *consume* operation must fail.


Consuming `box` References
--------------------------

- For `box(0)` references the protocol is exactly the same as for `mut`.
- For `box(1)` references the protocol is exactly the same as for `imm`.


Consuming `syn` and `asy` References
------------------------------------

For `syn` and `asy` references, *consuming* is easy:
- if the *owning* reference count is 1, go ahead, the object is *isolated*
- if the *owning* reference count greater than 1, the object is not *isolated*

This is because *synchronised* objects guarantee that there can be no external
`mut` or `box` reference to any directly reachable object, so if there is only
one *owning* reference we know the whole object graph is *isolated*.

There can be internal `mut` or `box` references to the root object, but they
use the *open* count.

So all we need to do is atomically read the *owning* reference count.

