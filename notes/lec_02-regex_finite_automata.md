# lexing, finite state machines, & regex

remember the compiler structure:

```
                                           /-(analysis)       /-(optimization)       /-(optimization)
                                           |  /               |  /                   |  /
                                           v /                v /                    v /
 source --[lexer]--> tokens --[parser]--> ast --[lowering]--> ir --[code gen]--> target
|                                            |                                         |
|                                            |                                         |
/--------------- front end ------------------/--------------- back end ----------------/
```

the first step in the front end of a compiler, _lexing_ (or _lexical
analysis_), is the process of taking the text of a source file & converting it
to a stream of _tokens_ (e.g. individual words/symbols/etc. of meaning).
typically this process is done using _regex_.

## regex

core expressions:

- `ε`: empty string
- `abc`: exactly the string (_literal_)
- <code>R<sub>1</sub>R<sub>2</sub></code>: <code>R<sub>1</sub></code> followed by <code>R<sub>2</sub></code> (_concatenation_)
- <code>R<sub>1</sub> | R<sub>2</sub></code>: <code>R<sub>1</sub></code> or <code>R<sub>2</sub></code> (_alternation_)
- <code>R<sup>\*</sup></code>: zero or more `R` (_Kleene star_)

some helpful expressions (but each can be built from the above):

- <code>R<sup>+</sup></code> one or more `R`
- `R?` Optional `R`
- `[a-z]` a, b, c, d, ..., z; also works for sub ranges & other ordered
  alphabets like digits

## regular grammar & lexical analysis

tokens can be specified using a _regular grammar_ (based on regex):

<pre><code>digit ::= [0-9]
alpha ::= [a-z]
ident ::= alpha (alpha | digit)<sup>\*</sup>
  num ::= digit<sup>+</sup>
</code></pre>

the lexer than uses these grammars to convert matching strings to tokens:

```
ident -> IDENT s
num -> NUM s
"while" -> WHILE
"+" -> PLUS ...
```

### some examples

- `while (i<5)` -> WHILE; OPAREN; IDENT "i"; LT; NUM 5; CPAREN
- `while i<5)` -> WHILE; IDENT "i"; LT; NUM 5; CPAREN
- `whole (i<5)` -> IDENT "whole"; OPAREN; IDENT "i"; LT; NUM 5; CPAREN

> [!NOTE] lexer doesn't catch syntax errors, it simply transforms any substring
> into a token whenever possible regardless of semantic or syntactic meaning.

## finite state machines (FSMs)

a **machine** that can be in one of a _finite number of states_. changes state
by reading input. each state can be limited on what inputs it responds to &
which state it can move to in response.

### deterministic finite automaton (DFA)

a state machine that formally defines exactly one transition for every state
**and** every possible input. in more mathematical terms: over a given
_alphabet_ of possible inputs (`Σ`), there exists a set of possible states
(`Q`), a transition function mapping every combination of states & inputs to a
new state (`δ: Q x Σ -> Q`), a starting state (<code>q<sub>0</sub></code>), and
a set of _accepting states_ (`F`). it is important to note that `δ` must be
_total_ and _deterministic_ (e.g. there exists one and only one transition for
every combination of state & symbol), hence the domain of `δ` being defined as
the _cross product_ of `Q` & `Σ`).

the notion of _accepting states_ is the other detail that sets DFA apart from
some other FSM. by specifying `F` as a subset of `Q`, a DFA can also determine
if the given inputs are "accepted" or "rejected". in more concrete terms, it is
a way of saying if the inputs are invalid or erroneous.

#### some examples

consider the following simple example of a DFA that parses only the characters
`a` or `b` can be defined as only accepting the exact string of `"a"` by the
following definition of `δ`:

<pre><code>Σ  := {a,b}
Q  := {0,1,2,e}
q<sub>0</sub> := 0
F  := {2}

                  --> ( 1 ) -[a,b]
                 /             |    __
               [b]             |   |  |
               /               v   v  |
δ :=  --> ( 0 )                ( e ) [a,b]
              \                ^   |__|
              [a]              |
                \              |
                 --> (( 2 )) -[a,b]
</code></pre>

> [!NOTE] the double parens (often a circle in drawn versions of this diagram)
> denotes the _accepting state_. additionally, every state has at least one
> arrow leading out of it and between all the outbound arrows, all letters in
> `Σ` are accounted for.

a perhaps more interesting example, a DFA that matches any string containint at
least one `a` and zero or more of any other character in `Σ`:

<pre><code>Σ  := {a,b}
Q  := {0,1}
q<sub>0</sub> := 0
F  := {1}
             ____             ____
            |    /           |    /
            |  [b]           |  [a,b]
            v  /             v  /
δ :=  --> ( 0 ) --[a]--> (( 1 ))
</code></pre>

and a DFA for any string beginning and ending w/ `b`:

<pre><code>Σ  := {a,b}
Q  := {0,1,2,e}
q<sub>0</sub> := 0
F  := {2}

             ____            ____
            |    /          |    /
            |  [a,b]        |  [a]
            v  /            v  /
          ( e )           ( 1 )
            ^             |   ^
            |            [b]  |
           [a]            |  [a]
            |             v   |
δ :=  --> ( 0 ) --[b]--> (( 2 ))
                            ^  \
                            |  [b]
                            |____\
</code></pre>

finally, a DFA for matching exactly two `a`s:

<pre><code>Σ  := {a,b}
Q  := {0,1,2,3}
q<sub>0</sub> := 0
F  := {2}
             ____           ____             ____           ____
            |    /         |    /           |    /         |    /
            |  [b]         |  [b]           |  [b]         |  [a,b]
            v  /           v  /             v  /           v  /
δ :=  --> ( 0 ) --[a]--> ( 1 ) --[a]--> (( 2 )) --[a]--> ( 3 )
</code></pre>

#### on DFAs & regex:

all valid regex can be converted to DFAs; however, it is not done directly.
instead, each part of a regex is first converted to a _nondeterministic finite
automaton_ (NFA), then connected with transitions to form a DFA where each
"state" is itself an NFA.

### nondeterministic finite automaton (NFA)

as the name implies, an NFA is like a DFA, but allows for nondeterministic
transitions by way of _silent transitions_ of `{ε}` instead of requiring
exactly one transition for every state & symbol. in mathematical terms, the
domain & range of an NFAs transition function is `δ: Q x Σ x {ε} -> Q`.

#### some examples

rewriting our "beginning & anding w/ `b`" example from a DFA to an NFA looks like:

<pre><code>Σ  := {a,b}
Q  := {0,1,2}
q<sub>0</sub> := 0
F  := {2}

                             ____
                            |    /
                            |  [a]
                            v  /
 { note no transition }   ( 1 )
 { for 'a' here       }   |   ^
            .            [b]  |
            .             |  [a]
            .             v   |
δ :=  --> ( 0 ) --[b]--> (( 2 ))
                            ^  \
                            |  [b]
                            |____\
</code></pre>

from this example, we can see that zero or more transitions allows us an NFA to
implicitly reject w/out needing to specify all possible paths. for regex, this
means we can build parts of a complex expression w/out needing each one to be
total & deterministic over the entirety of `Q` & `Σ`, allowing for easily
connecting separate NFA. to see this in action, we can build the regex
`^b.*|.*b$` (where `^` & `$` match the beginning and ending of a string) as two
separate NFA first:

<pre><code>Σ  := {a,b}
Q  := {0,1,2,3}
q<sub>0a</sub> := 0
q<sub>0b</sub> := 2
F  := {1,3}

                             ____
                            |    /
                            |  [a,b]
                            v  /
δ<sub>a</sub> := --> ( 0 ) --[b]--> (( 1 ))

             /----[a]------\
            /               \
            v               |
δ<sub>a</sub> := --> ( 2 ) --[b]--> (( 3 ))
            ^  \            ^  \
            |  [a]          |  [b]
            |____\          |____\
</code></pre>

then we can simply "glue" those together with empty transitions from a unified
starting state to make one NFA:

<pre><code>Σ  := {a,b}
Q  := {0',0,1,2,3}
q<sub>0</sub> := 0'
F  := {1,3}

                                    ____
                                   |    /
                                   |  [a,b]
                                   v  /
          [ε]--> ( 0 ) --[b]--> (( 1 ))
           /
δ := ( 0' )          /----[a]------\
           \        /               \
            \       v               |
           [ε]--> ( 2 ) --[b]--> (( 3 ))
                    ^  \            ^  \
                    |  [a]          |  [b]
                    |____\          |____\
</code></pre>

#### connecting NFAs to regex, part 1

armed w/ this idea of glueing unrelated NFAs (or DFAs) together into a single
NFA, we can generalized the core regex expressions [from above](#regex) into
new NFAs:

##### `ε`:

<pre><code>q<sub>0</sub> := --> (( 0 ))
where
  Σ := ∅
  Q,F := {0}
</pre></code>

##### `abc`:

<pre><code>q<sub>0</sub> := --> ( 0 ) --[a]--> (( 1 ))
where
  Σ := {a}
  Q := {0,1}
  F := {1}
</pre></code>

##### <code>R<sub>1</sub>R<sub>2</sub></code>:

<pre><code>q<sub>0</sub> := --> ( R<sub>1</sub> NFA ) --[ε]--> (( R<sub>2</sub> NFA ))
where
  Σ := ∅
  Q,F := {R<sub>1</sub> NFA, R<sub>2</sub> NFA}
</code></pre>

##### <code>R<sub>1</sub> | R<sub>2</sub></code>:

<pre><code>          [ε]--> (( R<sub>1</sub> NFA ))
           /
δ := ( 0' )
           \
            \
           [ε]--> (( R<sub>2</sub> NFA ))
where
  Σ := ∅
  Q := {R<sub>1</sub> NFA, R<sub>2</sub> NFA}
  F := {R<sub>2</sub> NFA}
</code></pre>

##### <code>R<sup>\*</sup></code>:

<pre><code>               /---------------[ε]--------------|
              /                                 v
q<sub>0</sub> := --> ( 0 ) --[ε]--> ( R<sub>1</sub> NFA ) --[ε]--> (( 1 ))
                             ^                 /
                             |-----[ε]--------/
where
  Σ := ∅
  Q := {0, R NFA, 1}
  F := {1}
</code></pre>

from here, we have all the tools we need to to represent as complex of a regex
as we want as an NFA, but the end goal is to convert it to a DFA (becuase DFAs
are known to be more powerful).

#### converting NFAs to DFAs

luckily, any NFA can be represented as a DFA by the following process:

starting from a set of all (i.e. the only) starting states, follow all possible
paths for each transition from those states in the NFA and union each state
reached into a new set of states. see a trivial example below using the OR NFA
from above:

```
Σ := {a,b}
Q := {0',0,1,2,3}
q := 0'
F := {1,3}
                                    ____
                                   |    /
                                   |  [a,b]
                                   v  /
          [ε]--> ( 0 ) --[b]--> (( 1 ))
           /
δ := ( 0' )          /----[a]------\
           \        /               \
            \       v               |
           [ε]--> ( 2 ) --[b]--> (( 3 ))
                    ^  \            ^  \
                    |  [a]          |  [b]
                    |____\          |____\
```

we can redraw this as a DFA like so:

```
Σ' := {a,b}
Q' := {{0',0,2},{2},{1,3},{3},{1,2}}
q' := {0',0,2}
F' := {{1,3},{3},{1,2}}
                                     ____                  ____
           {starts & ends w/ b}.    |    /                |    /
                                .   |  [b] /----[a]--|    |  [a]
                                .   v  /  /          v    v  /
                      /-[b]--> (( {1,3} ))          (( {1,2} ))
                     /                  ^           /        .
                    /                   |---[b]----/         .
δ := -->( {0',0,2} )                                          .{starts w/ b, but ends w/ a}
                    \               |---[a]----\
                     \              v           \
                      \-[a]--> ( {2} )          (( {3} )) . . .{starts w/ a, but ends w/ b}
                              . ^  \  \          ^  ^  \
                             .  |  [a] \----[b]--|  |  [b]
        {starts & ends w/ a}.   |____\              |____\
```
