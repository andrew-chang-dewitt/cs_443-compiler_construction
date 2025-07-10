# optimizations

a summary of content from lectures 15, 16, & 17 on loop optimization, static
single assignment, & function body optimizations.

## loop optimizations

loops are important to optimize because most programs spend the vast majority
of their time in a loop. some common loop optimizations include:

- loop invariant removal
- induction variable elimination
- loop unrolling
- loop fusion
- loop fission
- loop peeling
- loop interchange
- loop tiling
- loop parallelization
- software pipelining

### what is a loop?

in terms of a control flow graph (cfg), since that's what we're considering
when we're talking about compiler optimizations, a loop is defined as follows:

```
∵   S ⊆ cfg

    S is strongly connected
    ∃ node h ∈ S
        s.t. ∄ any edge from cfg \ S to S \ {h}

∴   S is a loop
```

however, this definition is quite hard to use in a compiler setting. an
equivalent definition is one made by use of _dominators_ & _back edges_.

- **dominator**: some node, `d`, is said to
  _dominate_ another node, `n`, if every path
  to `n` must go through `d`.
- **back edge**: any edge from `n` to it's
  dominator `d`

putting these two together, we arrive at an equivalent definition of a loop:

```
∵   S ⊆ cfg,
    ∃ d, n ∈ S

    dominator[n] ≡ d
    ∃ back edge n->d

∴   S is a loop
```

why does this matter? it's much easier to reason about _dominators_ than
_strongly connected_ subsets of graphs.

> [!IMPORTANT]
> dominators can be easily found (meaning loops can be easily found) using the
> following forward must **dataflow analysis** (initializing `out[n], in[n] :=
> cfg`):
> 
> <pre><code>
> ∵   out[n] = set of nodes that dominate n:
>         in[n]  := ∩<sub>n'∈pred[n]</sub> out[n']  <span class="comment">// intersection of out sets for all predecessors</span>
>         out[n] := in[n] ∪ {n}        <span class="comment">// a node always dominates itself</span>
> </pre></code>

dominators can be represented as a _dominator tree_, a model that is perhaps
more readily built/understood by a program.  this tree is built by drawing
edges from each node to its _immediate dominator_, that is, the dominator
(other than the node itself) that is dominated by other dominators (other than
the node itself):

```
this cfg:           becomes this dominator tree:

     (1)<--\          (1)
      |    |           |
      v    |           v
 /-->(2)   |          (2)
 |  /   \  |         /   \
 |  |   |  |         |   |
 |  v   v  |         v   v
 \-(3) (4)-/        (3) (4)-----\
      /   \            / \ \     \
      |   |            |  \ \-\   \
      v   v            v   \   \   \
     (5) (6)          (5)  |   |   |
    /   \/             |   v   v   v
    |   |              |  (8) (0) (6)
    v   v              v
/->(7) (8)            (7)
|  /     \             |
|  |     |             v
|  v     v            (9)
\-(9)-->(0)
```

as you might notice, the dominator tree above verifies our dataflow analysis equations. e.g. each node is dominated by any and all ancestors, but not by any children, aunts/uncles, or cousins, which is equivalent to our `in[n]` & `out[n]` equations above.

### loop optimization: invariant removal

moves a calculation that never changes from inside a loop to outside the loop
so that it is only executed once.

first, we need to be able to identify if some given node, `n`, is a loop invariant:

<pre><code>
∵   n := node of instruction of form `%x = opc op<sub>1</sub>, op<sub>2</sub>, ..., op<sub>N</sub>`:
        opc := the operation
        operands := {op<sub>1</sub>, op<sub>2</sub>, ..., op<sub>N</sub>}

    ∀ op<sub>i</sub> ∈ operands:
        op<sub>i</sub> is constant
      ∨ ∀ def_reaches[op<sub>i</sub>, n] is from outside the loop
      ∨ (|def_reaches[op<sub>i</sub>, n]| ≡ 1 ∧ def_reaches[op<sub>i</sub>, n] is loop invariant)

∴   n is loop invariant
</pre></code>

then, we need to _hoist_ `n` from inside the loop to the _pre-header_ node. for example:

```
before hoisting:    after hoisting:

   (1)                 (1)
    |                   |
    v                   v
   (h)<-\              (n)
    |   |               |
    v   |               v
   (3)  |              (h)<-\
    |   |               |   |
    v   |               v   |
   (n)  |              (3)  |
    |   |               |   |
    v   |               v   |
   (4)  |              (4)  |
    |   |               |   |
    v   |               v   |
   (5)--/              (5)--/
    |                   |
    v                   v
   (6)                 (6)
```

> [!WARNING]
> care must be taken while hoisting `n` to avoid changing the
> reaching definition of any value defined by `n` for any node between the new
> location of `n` & the old location.

to know if some node, `n`, is safe to hoist:

<pre><code>
∵   n := `%x = opc op<sub>1</sub>, op<sub>2</sub>, ..., op<sub>N</sub>`,      <span class="comment">// some node in loop w/ instruction defining %x</span>
    S := some loop in cfg,
    n ∈ S

    ∀ l ∈ L : dominate[n, l]                <span class="comment">// if n dominates every loop exit where %x is live</span>
      ∧ ∃! m ∈ S : defines[%x, m]           <span class="comment">// and %x is only defined once in the loop</span>
      ∧ ¬ live[%x, p]                       <span class="comment">// and %x is not live at the pre-header</span>
    where
        X := ∃ x->y : ∀ x ∈ S, y ∈ cfg / S, <span class="comment">// set of all loop exits </span>
        L := { l : ∀ x ∈ X, live[%x, x] },  <span class="comment">// where %x is live</span>
        p := pred[header[S]]                <span class="comment">// pre-header node</span>

∴   n is safe to hoist
</code></pre>

#### an example

given the following loop:

```llvm
l0:
  %i = bitcast i32 0 to i32
  br %l1
l1:
  %i = add i32 %i 1
  %t = add i32 %a %b // %a & %b are defined outside %l1 -> %t is invariant
  %el = getelementptr i32, i32* %arr, i32 %i
  store i32 %t, i32* %el
  %lt = icmp lt i32 %i %N
  br i1 %lt, label %l1, label %l2
l2:
  ret %t
```

invariant removal can identify that `%t` is invariant & recomputed every loop
iteration, thus optimizing the loop by moving the definition of `%t` to `%l0`:

```llvm
l0:
  %i = bitcast i32 0 to i32
  %t = add i32 %a %b // move %t here to avoid recomputation
  br %l1
l1:
  %i = add i32 %i 1
  %el = getelementptr i32, i32* %arr, i32 %i
  store i32 %t, i32* %el
  %lt = icmp lt i32 %i %N
  br i1 %lt, label %l1, label %l2
l2:
  ret %t
```

### loop optimization: induction variable

similar to loop invariant removal, the loop induction variable optimization
avoids needlessly recomputating value inside the loop—this time based on loop
induction variables.

this means that if some instruction inside the loop only relies on the loop induction variable, it's definition can be hoisted to before the loop & based on the induction variable, then the recalculation of it can be simplified to rely on itself instead of the induction variable. often, the loop induction variable can be removed or replaced by this value too. 

<pre><code>
∵   n := node of instruction of form `%x = opc op<sub>1</sub>, op<sub>2</sub>, ..., op<sub>N</sub>`:
        opc := the operation
        operands := {op<sub>1</sub>, op<sub>2</sub>, ..., op<sub>N</sub>}

    ∀ op<sub>i</sub> ∈ operands:
        loop_induction_var[op<sub>i</sub>, n]
      ∨   op<sub>i</sub> is constant
        ∧ ∀ def_reaches[op<sub>i</sub>, n] is from outside the loop
        ∧ (|def_reaches[op<sub>i</sub>, n]| ≡ 1 ∨ def_reaches[op<sub>i</sub>, n] is loop invariant)

∴   n is loop invariant
</pre></code>

#### an example

given the following loop:

```llvm
l0:
  %i = bitcast i32 0 to i32 // %i is a loop induction variable
l1:
  %t1 = mul i32 %i 4 // %t1 computed from %i
  %t2 = add i32 %a %t1
  %s = add i32 %s %t2
  %i = add i32 %i 1
  %lt = icmp lt %i 100
  br i1 %lt, label %l1, label %l2
l2:
  ... // use %s here...
```

notice `%t1` is computed from `%i`, the loop induction variable. this can be
optimized by moving the initialization of `%t1` out of the loop, then
incrementing it by 4 instead of multiplying `%i` by 4 since addition is cheaper
than multiplication.

```llvm
l0:
  %i = bitcast i32 0 to i32
  %t1 = bitcast i32 0 to i32 // move %t1 def here
l1:
  %t1 = add i32 %t1 4 // %t1 now computed from %t1 via addition
  %t2 = add i32 %a %t1 // note %t2 computed from %t1 & %a (a loop invariant)
  %s = add i32 %s %t2
  %i = add i32 %i 1
  %lt = icmp lt %i 100
  br i1 %lt, label %l1, label %l2
l2:
  ... // use %s here...
```

this first optimization allows for another:

```llvm
l0:
  %i = bitcast i32 0 to i32
  %t1 = bitcast i32 0 to i32
  %t2 = bitcast i32 %a to i32 // initialize %t2 from %a before loop
l1:
  %t1 = add i32 %t1 4
  %t2 = add i32 %t2 4 // %t2 now computed from %t1 via addition
  %s = add i32 %s %t2
  %i = add i32 %i 1
  %lt = icmp lt %i 100
  br i1 %lt, label %l1, label %l2
l2:
  ... // use %s here...
```

since `%t2` depended on `%t1`, `%t2` was now being computed from a loop
induction variable. by moving the initialization of `%t2` out of the loop, we
can optimize the loop further.

notice `%t1` is now only used to change `%t1`, this means we can elminate it:

```llvm
l0:
  %i = bitcast i32 0 to i32
  %t2 = bitcast i32 %a to i32
l1:
  %t2 = add i32 %t2 4
  %s = add i32 %s %t2
  %i = add i32 %i 1
  %lt = icmp lt %i 100
  br i1 %lt, label %l1, label %l2
l2:
  ... // use %s here...
```

this loop could still be simplified even further by pre-computing the end value
from `%a` (perhaps using `%a + 400`), then comparing to that in `%lt`, thus
allowing `%i` to be elminated:

```llvm
l0:
  %t2 = bitcast i32 %a to i32
  %e = add i32 %a 400
l1:
  %t2 = add i32 %t2 4
  %s = add i32 %s %t2
  %lt = icmp lt %t2 %e
  br i1 %lt, label %l1, label %l2
l2:
  ... // use %s here...
```
