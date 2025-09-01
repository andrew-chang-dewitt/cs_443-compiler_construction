---
title: "Compilers notes: dataflow analysis"
description: "Intruductory lecture notes."
keywords:
  - "dataflow analysis"
  - "reaching definitions"
  - "optimizations"
  - "compiler construction"
  - "pl theory"
  - "lecture notes"
  - "computer science"
  - "cs 443"
  - "illinois tech"
meta:
  byline: Andrew Chang-DeWitt
  published: "2025-07-08T00:00-06:00"
---

a general framework for analysing control flow graphs by

> producing a list of _facts_ that are available going into & out of each node

this means that each node does the following (in the eyes of dataflow
analysis):

- generates some set of facts
- kills some set of facts

the steps:

1. define facts, gen & kill sets
2. define constraints
3. convert constraints to equations (for use in algorithm)
4. initialize facts for each node (consistent w/ if sets are increasing or
   decreasing as the cfg is traversed)

some examples of dataflow analyses are:

- liveness
- reaching defs
- optimizations

let's review liveness in terms of dataflow analysis

## ex: liveness

1. define facts & gen/kill sets:

   ```
   facts   := live variables
   gen[n]  := use[n] // set of variables referenced (live) at node
   kill[n] := def[n] // set of variables defined at node
   ```

2. constraints:

   ```
   in[n]  ⊇ gen[n]
   out[n] ⊇ in[n'] if n' succ[n]
   in[n]  ⊇ out[n] / kill[n]
   ```

3. equations:

   <pre><code>
   out[n] := ∪<sub>n'∈succ[n]</sub> in[n']
   in[n]  := gen[n] ∪ (out[n] / kill[n])
   </pre></code>

4. initialize sets:

   ```
   out[n] := ∅
   in[n]  := ∅
   ```

## ex: reaching defs

which definition of `variable` might reach the current node?

```
 ( 1: b = a + 2 ) // out[1] = {1}
         |        // in[2]  = {1}
         v
 ( 2: c = b * b ) // out[2] = {1,2}
         |        // in[3]  = {1,2}
         v
 ( 3: b = c + 1 ) // out[3] = {2,3}
         |        // in[4]  = {2,3}, note 2 still reaches node 4,
         |        //                 even though c is no longer live
         v
 ( 4: ret b * a )
```

generalizing this:

1. define facts & gen/kill sets:

   ```
   facts   := nodes who's defs might reach current node
   gen[n]  := current node if it defines a variable
   kill[n] := all other nodes that define that variable if
              current node defines a variable
   ```

2. constraints:

   ```
   out[n] ⊇ gen[n]
   in[n]  ⊇ out[n'] if n' ∈ pred[n]
   out[n] ⊇ in[n] / kill[n]
   ```

3. equations:

   <pre><code>
   in[n]  := ∪<sub>n'∈pred[n]</sub> out[n']
   out[n] := gen[n] ∪ (in[n] / kill[n])
   </pre></code>

4. initialize sets:

   ```
   out[n] := ∅
   in[n]  := ∅
   ```

## types of dataflow analyses

|      | backward                                                                        | forward                                            |
| ---- | ------------------------------------------------------------------------------- | -------------------------------------------------- |
| may  | liveness:<br />what variables _may_ be<br />needed from _n_?                    | reaching defs:<br />what defs _may_ reach _n_?     |
| must | very busy exprs:<br />what exprs _will_ be defined<br />on every path from _n_? | available exprs:<br />what exprs _must_ reach _n_? |

## generic dataflow analysis, in code

so how to translate the 4-step system to code? as an example here, we
disect the generalized pattern given in [project 5](../prj_5/), in the file
`src/dataflow.ml`.

first, how is this module used? looking at `src/opt.ml`, the dataflow module is initialized w/ a

```ocaml
module ExpDataflow = Dataflow.Make
                       (struct type t = var end)
                       (struct type t = inst
                               let compare a b =
                                 (* This may not do the right thing, but it'll
                                  * do something, which is good enough to just
                                  * treat the set like a list *)
                                 if a < b then -1 else
                                   if a = b then 0 else 1
                        end)
```
