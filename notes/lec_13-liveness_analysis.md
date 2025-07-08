# liveness analysis

a variable is _live_ when its value is needed, e.g.:

```
int f(int x) {
                 <---- x is live
  int a = x + 2;
                 <---- a & x are live
  int b = a * a;
                 <---- b & x are live
  int c = b + x;
                 <---- c is live
  return c;
}
```

observing this example, we see that each variable lives from the point after it
is initialized to the last point where it is used.

> [NOTE!] _liveness_ !== scope

why does this matter?

1. because if two variables share the same scope, but are not alive at the same
   time, then this means we can optimize by having these variables share
   registers
2. some other analyses are possible when thinking in terms of liveness, such as
   defining constraints of non-lexical lifetimes in rustc's borrow checker
   implementation

## a simple overview

to begin, we model the program being compiled as a _control flow graph_ (CFG),
composed of individual instructions as nodes & control flow "directions" as
edges:

```
/->[ Move ]<-\
|     |      |
|     v      |
|  [ Binop ] |
|     |      |
|     v      |
|  [ If ]----/
|     |
|     v
|  [ Unop ]
|     |
|     v
\--[ Jump ]
```

liveness is then associated with edges in our graph. e.g., from `a = b + 1`, we
can derive a CFG as:

```
     |
     |  live: b
     v
[ Mov a, b ]
     |
     |  live: a
     v
[ Add a, 1 ]
     |
     |  live: a (maybe)
     v
```

we can naively compile this to:

```
Move rax, rax
Add rax, 1
```

however, as line 1 should make painfully clear, it would be missing an
important optimization. as seen by the liveness annotation for `a` & `b` above,
neither variable is alive at the same time as the other, thus we can optimize
by simply skipping the `Move rax, rax` instruction entirely.

## `use[s]` & `def[s]`

liveness is based on _uses_ & _definitions_, modeled as sets of variable names related to another variable.

- `use[s]`: set of variables used by s
- `def[s]`: set of variables defined by s
  (most instructions write only 1 variable, a convention enforced by llvm to be
  exactly one for every instruction)

e.g.:

```
∵ a = b + c

  use[a] = {b,c}
  def[a] = {a}

∵ a = a + 1

  use[a] = {a}
  def[a] = {a}
```

## formal def of liveness

a variable `v` is _live_ on edge `e` if:

there is:

- a node `n` in the CFG such that `use[n]` contains `v` &&
- a directed path from `e` to `n` such that for every statement `s'` on the
  path, `def[s']` does not contain `v`

## a naive algorithm

```
for every variable v:
  for every n in use[v]:
    walk CFG backward from n until:
      a def[v] is found
      OR a previously visited node has been reached
      AND mark v as live on each edge traversed
```

this works, but is inefficient (time complexity of O(num edges * |use[v]| * num
of variables) ~= O(n<sup>3</sup>)). how can we make it better?
