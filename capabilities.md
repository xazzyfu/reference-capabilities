A System of Reference Capabilities
==================================


|              | Access Rights        | Aliasing Rights          |
|-------------:|----------------------|--------------------------|
|         all  |  a (read and write)  |  * (globally aliasable)  |
|  restricted  |  r (read only)       |  + (locally aliasable)   |
|        none  |  n (none)            |  - (not aliasable)       |


|     |   *   |   +   |   -   |
|:---:|:-----:|:-----:|:-----:|
|  a  |   .   |  mut  |   .   |
|  r  |  imm  |  box  |   .   |
|  n  |  syn  |   .   |  iso  |


|  Unused combination  |       Reason      |
|:--------------------:|:-----------------:|
|          a *         |  Not thread-safe  |
|          a -         |      Unneeded     |
|          r -         |      Unneeded     |
|          n +         |      Unneeded     |


|  capability  |  property      |  sendable  |  alias capabilities  |   field capabilities   |
|:------------:|----------------|:----------:|:--------------------:|:----------------------:|
|      mut     |  thread-local  |     no     |      mut or box      |        unchanged       |
|      imm     |  immutable     |     yes    |      imm or box      |   imm unless sendable  |
|      box     |  mut or imm    |     no     |         box          |   box unless sendable  |
|      syn     |  synchronised  |     yes    |         syn          |            .           |
|      iso     |  isolated      |     yes    |          .           |            .           |


```
iso a

with relaxed(a) as x:
  # x is mut
  # a is not accessible
  # only sendable enclosing capabilities are accessible

# a is now accessible again
# x is no longer accessible
```

```
iso a

with relaxed(a) as box x:
  # x is box
  # same rules otherwise
```

```
# o is an imm reference to an object with an iso field a
imm o

with relaxed(o.a) as x:
  # x is box
  # because x inherits the access restrictions of o
```

```
# o is a box reference to an object with an iso field a
box o

with relaxed(o.a) as x:
  # x is box
  # because x inherits the access restrictions of o
```

```
syn a

with locked(a) as x:
  # a is locked
  # x is mut
  # a is still accessible
  # only sendable enclosing capabilities are accessible
  # a is unlocked at the end of the block

# x is no longer accessible
```

```
syn a

with wlocked(a) as x:
  # sames rules exactly
```

```
syn a

with wlocked(a) as box x:
  # x is box
  # sames rules otherwise
```

```
syn a

with rlocked(a) as x:
  # a is reader-locked
  # x is box
  # same rules otherwise
  # a is unlocked at the end of the block
```

```
asy a

with schedule(a) as x:
  # the contents of this block are executed asynchronously
  # x is mut
  # a is still accessible
  # only non iso sendable enclosing capabilities are accessible

# x is no longer accessible
```

```
asy a

with schedule(a) as box x:
  # the contents of this block are executed asynchronously
  # x is box
  # same rules otherwise
```

```
asy a
iso b

with schedule(a) as x, consume(b) as y:
  # x is mut
  # y is iso
  # same rules otherwise
```

```
asy a
iso b

with schedule(a) as x, consume(b) as mut y:
  # x is mut
  # y is mut
  # the same would work with box y, imm y, syn y, asy y
  # because a consumed reference can become any capability
  # same rules otherwise
```
