---
title: "Compilers notes: parsing lr(k) grammars"
description: "Intruductory lecture notes."
keywords:
  - "parsing"
  - "fsm"
  - "shift-reduce parsers"
  - "cfg"
  - "pl theory"
  - "lecture notes"
  - "computer science"
  - "cs 443"
  - "illinois tech"
meta:
  byline: Andrew Chang-DeWitt
  published: "2025-06-04T00:00-06:00"
---

in comparison to ll(k) grammars, l**r**(k) grammars work by producing the
**r**ightmost derivation:

- **l**eft-to-right reading
- **r**ightmost derivation
- **k** symbol lookahead

## shift-reduce parsers

an algorithm known to parse lr(k) grammars, _shift reduce_ parsers are a state
machine operating on a stack via two actions:

1. _shift_ a token onto the stack
2. _reduce_ the top of the stack to a nonterminal by a production

### an example

given this grammar:

```
S ::= e
e ::= n | e + n | e + (e)
```

parse `1 + (2 + 3)$`:

```
stack:          input:          comment:
[        ]      1 + (2 + 3)$
         1        + (2 + 3)$    shift 1 to stack
         e        + (2 + 3)$    reduce n -> e
       e +          (2 + 3)$    shift
     e + (           2 + 3)$    shift
    e + (2             + 3)$    shift
    e + (e             + 3)$    reduce n -> e
  e + (e +               3)$    shift
e + (e + 3                )$    shift
    e + (e                )$    reduce e + n -> e
   e + (e)                 $    shift
         e                 $    reduce e + (e) -> e
         S                 $    reduce e -> S
```

okay, that's good and all, but how do we know when to shift vs when to reduce?

### return of DFA

shift-reduce parsers use DFA to make decisions. modeling terminals &
nonterminals on the stack as edges, we treat the stack as a set of states to
operate on via transition table made of one of two things: an ACTION or a GOTO.
an ACTION modifies the stack based on the current state & next terminal; while
a GOTO tells us what state to go to next based on the current state and the
next nonterminal:

1. ACTION(state, terminal)
   - sn: shift state n onto stack
   - rn: reduce using rule n
   - a: accept
   - error (leave the table blank?)
2. GOTO(state, nonterminal)
   - next state

#### an example

this example is building a parsing table for a very simple LR(0) grammar. We
begin by defining our grammar as a list of productions:

```
1 S' -> S$
2 S  -> (L)
3 S  -> x
4 L  -> S
5 L  -> L, S
```

then expand that into a list of _items_, e.g. a list of productions that
indicate where we are (using `.`) in that production:

```
S' -> .S$
S' -> S.$
S  -> .(L)
S  -> (.L)
S  -> (L.)
S  -> (L).
S  -> .x
S  -> x.
L  -> .S
L  -> S.
L  -> .L,S
L  -> L.,S
L  -> L,.S
L  -> L,S.
```

from here, we can build a DFA showing how to get from some set of items (a
state) to another set of items based on what token is next:

```
                              2__________
                             |           |
        1___________  /-[S]->| S' -> S.$ |
       |            |/       |___________|
       | S' -> .S$  |
       | S  -> .(L) |         3________
       | S  -> .x   |\       |         |
       |____________| \-[x]->| S -> x. |<-----\
           |                 |_________|      |
          [(]  /---------\      ^             |
           |   |         |      |             |
        4__v___v___      |      /             |
       |           |     |     /              |
       | S -> .x   |----[(]   /               |
/----->| S -> (.L) |         /                |
|      | L -> .L,S |----[x]-/ 5________       |
|      | L -> .S   |         |         |      |
|      | S -> .(L) |----[S]->| L -> S. |      |
|      |___________|         |_________|      |
|            |                                |
|           [L]                               |
|            |                                |
|       6____v_____           7__________     |
|      |           |         |           |    |
|      | S -> (L.) |----[)]->| S -> (L). |    |
|      | L -> L.,S |         |___________|    |
|      |___________|                          |
|            |                                |
|           [,]                               |
|            |                                |
|       8____v_____   /-[x]-------------------/
|      |           | /
\-[(]--| L -> L,.S |/         9__________
       | S -> .(L) |         |           |
       | S -> .x   |----[S]->| L -> L,S. |
       |___________|         |___________|
```

while this graph visualizes the DFA nicely, in practice it's hard to use.
instead, we typically build transition tables mapping a state to a transition.
to build a table from the above graph, we'll label the rows by the given state
numbers, then create a column for each possible terminal or nonterminal,
finally filling in the cells for every state that has transitions for that
terminal/nonterminal.

| state | (   | )   | x   | ,   | $   | S   | L   |
| ----- | --- | --- | --- | --- | --- | --- | --- |
| 1     | s4  |     | s3  |     |     | 2   |     |
| 2     |     |     |     |     | a   |     |     |
| 3     | r3  | r3  | r3  | r3  | r3  |     |     |
| 4     | s4  |     | s3  |     |     | 5   | 6   |
| 5     | r4  | r4  | r4  | r4  | r4  |     |     |
| 6     |     | s7  |     | s8  |     |     |     |
| 7     | r2  | r2  | r2  | r2  | r2  |     |     |
| 8     | s4  |     | s3  |     |     | 9   |     |
| 9     | r5  | r5  | r5  | r5  | r5  |     |     |

this transition table can then be used to quickly look up what to do for a
given token while in the current state. we can see it in action by parsing the
text `(x,(y))$`:

```
1 S' -> S$
2 S  -> (L)
3 S  -> x
4 L  -> S
5 L  -> L,S

stack:  token: input:    comments:
1              (x,(y))$
14      (       x,(y))$  shift state 4 onto stack
143     x        ,(y))$  shift state 3 onto stack
14      S        ,(y))$  reduce using S -> x
145     S        ,(y))$  goto 5
14      L        ,(y))$  reduce using L -> S
146     L        ,(y))$  goto 6
1468    ,         (y))$  shift state 8 onto stack
14684   (          y))$  shift state 4 onto stack
146843  y           ))$  shift state 3 onto stack
14684   S           ))$  reduce using S -> x
146845  S           ))$  goto 5
14684   L           ))$  reduce using L -> S
146846  L           ))$  goto 6
1468467 )            )$  shift state 7 onto stack
1468    S            )$  reduce using S -> (L)
14689   S            )$  goto 9
14      L            )$  reduce using L -> L,S
146     L            )$  goto 6
1467    )             $  shift 7 onto stack
1       S             $  reduce using S -> (L)
12      S             $  accept
```
