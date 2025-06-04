# parsing lr(k) grammars

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
operate on via transition table made of two parts. first, the table tells us
what ACTION to take on the stack; then, the table tells us what next state to
go to:

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
