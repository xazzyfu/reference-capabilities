A System of Reference Capabilities
==================================


Formalising the Capabilities
----------------------------

References afford two fundamental operations:
- *dereferencing* to access the referenced object
- *aliasing* to create a new reference to the referenced object

A reference capability is therefore defined by these two axes:
- the *dereferencing rights* of the reference
- the *aliasing rights* of the reference

We introduce three kinds of *dereferencing rights*:
- read and write
- read only
- no dereferencing allowed

We introduce three kinds of *aliasing rights*:
- globally aliasable across multiple threads
- locally aliasable only within the current thread
- not aliasable at all


|              | Dereferencing Rights | Aliasing Rights          |
|-------------:|----------------------|--------------------------|
|         all  |  a (read and write)  |  * (globally aliasable)  |
|  restricted  |  r (read only)       |  + (locally aliasable)   |
|        none  |  n (none)            |  - (not aliasable)       |


We name five reference capabilities (using the above notations):


|     |   *   |   +   |   -   |
|:---:|:-----:|:-----:|:-----:|
|  a  |   .   | `mut` |   .   |
|  r  | `imm` | `box` |   .   |
|  n  | `syn` |   .   | `iso` |


So for example the `mut` capability allows the referenced object to be read and
modified, but the reference cannot be passed (i.e. *aliased*) to another thread.


Some combinations are discarded as either un-thread-safe or simply superfluous.


|  Unused combination  |       Reason      |
|:--------------------:|:-----------------:|
|          a *         |  Not thread-safe  |
|          a -         |      Unneeded     |
|          r -         |      Unneeded     |
|          n +         |      Unneeded     |


Considerations for Thread Safety
--------------------------------

An object is *thread-safe* if at every point in time either:
- no more than one thread can access it
- no thread can modify it

Each capability represents an invariant *thread-safe* property of the
referenced object:


|  capability  |  property                   |
|:------------:|:---------------------------:|
|     `mut`    |  thread-local               |
|     `imm`    |  immutable                  |
|     `box`    |  thread-local or immutable  |
|     `syn`    |  synchronised               |
|     `iso`    |  isolated                   |


The main idea is that *data* can either be:
- *thread-local*, in which case it can be modified and freely aliased
  *within the thread*, but never shared with another thread
- *immutable*, in which case it can be shared across threads
- *synchronised*, meaning *protected by a synchronisation mechanism*
  (such as a mutex), in which case it can be shared across threads
- *isolated*, meaning only reachable through a single reference,
  in which case that reference can be passed from thread to thread


Everything that follows is only about making sure that all references to the
same *data* (i.e. *objects*) always agree on the *state* of that *data*: for
example an object should never be simultaneously referenced as *immutable*
(`imm`) and *thread-local* (`mut`).


References may only be aliased to references with a compatible capability:


|  capability  |  alias capabilities  |
|:------------:|:--------------------:|
|     `mut`    |    `mut` or `box`    |
|     `imm`    |    `imm` or `box`    |
|     `box`    |        `box`         |
|     `syn`    |        `syn`         |
|     `iso`    |          .           |


The `box` capability is particular in that it means the *data* is either
*thread-local* or *immutable*, but we don't know which. Therefore a `box`
reference cannot modify the object - only read it, in case it's *immutable*,
but it also cannot share it with another thread in case it's *thread-local*.


A capability is said to be *sendable* if it is incompatible with `mut`. In that
case such a reference can be safely transferred to another thread.


|  capability  |  sendable  |
|:------------:|:----------:|
|     `mut`    |     no     |
|     `imm`    |     yes    |
|     `box`    |     no     |
|     `syn`    |     yes    |
|     `iso`    |     yes    |


A fundamental consideration for thread-safety is that a reference exposes not
only the referenced object, but also the objects it itself references through
its fields, and the objects *they* reference through their fields, and so on.

Therefore, reference capabilities which might have aliases in other threads
(such as `imm` or `box`) must necessarily *alter* the capability of their `mut`
fields: otherwise these `mut` references could be reached by several threads
(since an `imm` reference can be shared accross threads).

The simplest solution is to assign to these fields the capability of the
original `imm` or `box` reference. This ensures that `mut` fields of an `imm`
object are in fact *immutable* themselves.


```python
class MutableField:
  mut a : Value

# o is an imm reference to an object that would normally have a mut field
imm o : MutableField

# o.a is seen as imm instead of mut

imm a = o.a
# a is imm too

# o can be aliased to box
box b = o

# b.a is seen as box instead of mut, which is compatible with a and o.a

# Since b is box, writing to b.a is forbidden, so this cannot happen:
#  mut c = Value()
#  b.a = c
```


An interesting consequence is that `box` fields of an `imm` object will always
refer to effectively *immutable* data (since they cannot have `mut` aliases).
So we also *alter* the `box` fields of `imm` objects to reflect this.


|  capability  |   field capabilities    |
|:------------:|:-----------------------:|
|     `mut`    |        unchanged        |
|     `imm`    |  `imm` unless sendable  |
|     `box`    |  `box` unless sendable  |
|     `syn`    |            .            |
|     `iso`    |            .            |


So far, we don't need to worry about `syn` and `iso` capabilities since they
disallow *dereferencing*. However, if that were entirely true they would be
quite useless...


Relaxing `iso` References
-------------------------

The `iso` capability represents an invariant that can be expressed thus:

All objects that are reachable through a given `iso` reference are either:
- *isolated* (*only* reachable through that one `iso` reference)
- *immutable* (not having any `mut` references)
- *synchronised* (reachable through a `syn` reference)

The idea is that objects which are already *immutable* or *synchronised* do not
need to be *isolated* on top of that because they are already thread-safe.

The `iso` capability disallows casual *dereferencing* because it's an easy way
to make sure the invariant of an `iso` reference will not be broken by aliasing
its fields. Instead, we introduce a new special operation on `iso` references we
call *relaxing* which allows access to their fields under controlled conditions.

A solution we find to be the simplest to understand is to introduce the idea of
an *isolated scope*: a kind of scope through which only *sendable* references
can pass, but which is completely impermeable to `mut` and `box` references.

In a nutshell, only *sendable* references from enclosing scopes can be accessed
inside an *isolated scope*, and *sendable* references created inside can live
past the end of the *isolated scope*.

The `iso` invariant can then be temporarily *relaxed* inside an *isolated scope*
with the guarantee that it will be restored at the end of the scope, since
nothing that shouldn't can *leak* through the scope.


```python
class VariousFields:
  mut f : Value
  imm o : Value

iso a : VariousFields

imm b : Value
mut c : Value

with relaxed(a) as mut x:
  mut f = x.f
  imm o = x.o
  # a is not accessible because its iso invariant is temporarily lifted
  x.o = b # OK: b is accessible because it's imm
  x.f = c # ERROR! c is not accessible because it's mut

# a is now accessible again and still respects the iso invariant
# x is no longer accessible, otherwise it would break the iso invariant of a
# f is no longer accessible, otherwise it would break the iso invariant of a
# o is still accessible because it's imm
```


An `iso` reference can be *relaxed* either as `mut` or as `box`:


```python
iso a

with relaxed(a) as box x:
  # x is box
  # same rules otherwise
```


Only one final difficulty remains: when an object that may be reachable from
multiple threads has an `iso` field, several threads could simultaneously
*relax* this `iso` reference. Such an `iso` reference must thus disallow
modifying the reachable objects.

We find the simplest solution is to merely detect this situation at the site
where an `iso` is relaxed and impose the `box` capability for the relaxed
reference:

```python
class IsoField:
    iso a

imm o # references an IsoField

# Error!
with relaxed(o.a) as mut x:
    # x cannot be mut because o is imm
```

```python
class IsoField:
    iso a

imm o # references an IsoField

with relaxed(o.a) as box x:
  # OK
  # x inherits the access restrictions of o
```

```python
class IsoField:
    iso a

box o # references an IsoField

with relaxed(o.a) as box x:
  # OK
  # x inherits the access restrictions of o
```


Consuming References
--------------------

Although they cannot be aliased, `iso` references can be transferred by
invalidating the original reference. This operation maintains the `iso`
invariant intact.

In this way, an `iso` reference can be transferred to another thread. This
is thread-safe because the original thread must necessarily give up its own
reference, and the `iso` invariant guarantees it doesn't retain references to
objects that may be modified through the `iso` reference.


```python
iso a

iso b = consume a
# a is now null
# b still respects the iso invariant
```


A consumed reference can be turned into any capability: no matter what that
capability is, the `iso` invariant of the original reference guarantees there
can be no reference that would contradict the invariant of the new capability.


```python
iso a0, a1, a2

# a consumed reference can become any capability
mut b0 = consume a0
box b1 = consume a1
imm b2 = consume a2
```


Conversely, any capability can be consumed, but if the `iso` invariant cannot
be statically inferred, a runtime verification using reference counts will be
performed instead and an exception will be raised if the reference does not in
fact respect the `iso` invariant.


```python
mut a

# non iso references can also be consumed
# but a runtime check using reference counts is performed to ensure it
# respects the iso invariant. If the check fails an exception is raised
iso b = consume a
```

```python
iso a

# to ensure at compilation time that the consume operand is already
# iso and that no runtime check will need to be performed,
# the operand can be cast to iso, which will fail to compile otherwise
iso b = consume iso a
```


Synchronising `syn` References
------------------------------

The `syn` capability represents a similar invariant to the `iso` capability:

All objects that are reachable through a given `syn` reference are either:
- *synchronised* (*only* reachable through `syn` references to the same object)
- *immutable* (not having any `mut` references)

The idea is that objects which are already *immutable* do not need to be
*synchronised* since they are already thread-safe. Also, other objects which
are *synchronised* too (but with their own *synchronisation mechanism*) do not
need to be *synchronised* again with another *synchronisation mechanism*.

The difference with `iso` is that there may be several `syn` references to the
same object, and they may be shared across several threads.

The *synchronisation mechanism* ensures thread-safety by enforcing that only
one `syn` reference to a given object may be dereferenced at a time.

We consider two suitable synchronisation mechanisms:
- variations on the traditional *mutex* lock
- variations on the asynchronous *actor model*

The *mutex* works by blocking until the resource is free and granting only one
access at a time. Variations include the *readers-writers* lock, which allows
read only accesses to occur simultaneously, and *recursive locks*, which allow a
thread to reacquire the same lock without deadlocking.

The *actor model* instead allows only asynchronous accesses to the resource,
which are enqueued as *tasks* into a queue and executed one at a time by the
*actor* in its own thread. Variations include the possibility to wait until a
*task* is done and retrieve some result, using a kind of *future* mechanism.

This leads us to consider two distinct flavors of `syn`, so we introduce a new
capability we call `asy` to distinguish them:
- `syn` references will use a lock
- `asy` references will use a queue


Relaxing `syn` References
-------------------------

To access the fields of a `syn` reference, we use the same *isolated scope*
solution as for `iso`, which prevents references to the *protected data*
leaking past the point where the synchronisation mechanism is released:


```python
syn a

with locked(a) as mut x:
  # a is locked here
  # x is mut
  # a is still accessible
  # only sendable enclosing capabilities are accessible
  # a is unlocked at the end of the block

# x is no longer accessible
```


A `syn` reference can be *relaxed* (i.e. *locked*) either as `mut` or as `box`:


```python
syn a

with locked(a) as box x:
  # x is box
  # sames rules otherwise
```


We can introduce a distinction between *writelocks* and *readlocks*,
assuming a *readers-writers* lock is used instead of a simple mutex:


```python
syn a

with wlocked(a) as mut x:
  # a is writer-locked
  # x is mut
  # sames rules exactly
```

```python
syn a

with rlocked(a) as box x:
  # a is reader-locked
  # x is box
  # same rules otherwise
  # a is unlocked at the end of the block
```


Or better yet, we can infer whether to *writelock* or *readlock* based only on
the capability of the relaxed reference:


```python
syn a

with locked(a) as mut x:
  # a is writer-locked because x is mut
```

```python
syn a

with locked(a) as box x:
  # a is reader-locked because x is box
```


Relaxing `asy` references
-------------------------

For asynchronous execution one further subtlety applies: all references from
enclosing scopes must actually be *captured* (meaning aliased) in order to be
accessible in the *isolated scope*, since the execution takes place later in a
different context. This means `iso` references from enclosing scopes may not
be accessed inside an asynchronous scope (since that would imply aliasing).

Additionally, since the code in the scope is executed asynchronously, nothing
can live past the end of the scope.


```python
asy a

with schedule(a) as mut x:
  # the contents of this block are executed asynchronously
  # x is mut
  # a is still accessible
  # only non iso sendable enclosing capabilities are accessible

# x is no longer accessible
```


An `asy` reference can be *relaxed* (i.e. *scheduled*) as either `mut` or `box`:


```python
asy a

with schedule(a) as box x:
  # the contents of this block are executed asynchronously
  # x is box
  # same rules otherwise
```


If an `iso` reference is needed inside the asynchronous scope, it must be
consumed in advance:


```python
asy a
iso b

with schedule(a) as mut x, consume(b) as iso y:
  # x is mut
  # y is iso
  # same rules otherwise
```

```python
asy a
iso b

with schedule(a) as mut x, consume(b) as mut y:
  # x is mut
  # y is mut
  # the same would work with box y, imm y, syn y, asy y
  # because a consumed reference can become any capability
  # same rules otherwise
```


A *future* mechanism may be introduced to allow waiting until task is executed
and retrieving computed results:


```python
asy a

# f represents a future computation on a
f = Future(a)

# a computation can be scheduled on a through f
with schedule(f) as mut x:
  # same rules as with schedule(a)

# f is able to wait until the scheduled computation completes
wait(f)
```

```python
asy a

# Future can take a consumed reference as second argument
f = Future(a, consume List[Value]())

# f then behaves exactly like a mut reference to the same object
f.append(Value(1))

with schedule(f) as mut x:
  # f is accessible within the asynchronous computation
  f.append(Value(2))

# but once a task is scheduled through f
# accessing the referenced object implictly causes f to wait

# f waits first until the computation is complete
value = f.pop()
```
