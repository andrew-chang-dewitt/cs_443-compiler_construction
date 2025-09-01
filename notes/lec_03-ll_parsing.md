---
title: "Compilers notes: ll parsing"
description: "Intruductory lecture notes."
keywords:
  - "parsing"
  - "fsm"
  - "context free grammars"
  - "cfg"
  - "pl theory"
  - "lecture notes"
  - "computer science"
  - "cs 443"
  - "illinois tech"
meta:
  byline: Andrew Chang-DeWitt
  published: "2025-06-02T00:00-06:00"
---

dfas/regex are super powerful for lexing, but not enough for parsing. easy
example: it's impossible for a DFA or regex to make sure there is exactly one
close paren for each open paren.

why? dfas can't count non-finite numbers (deterministic **finite** automaton).

so if dfas can't do it, what can?

## context free grammars (CFG)

we'll use a common grammar for specifying other grammars: backus-naur form
(BNF). it's a set of _nonterminals_ defined in terms of _productions_ composed
of one or more other nonterminals & _terminals_:

```BNF
e ::= x | num | e bop e | uop e
```

in this example, `e` is a _nonterminal_ that can be expanded to one of four
_productions_. productions `x` & `num` are _terminals_ while `binop` & `uop`
are productions composed of nonterminals.

using a slightly altered definition of `e` as `e ::= n | e + e | (e)`, let's
show how to expand it using productions of `e` until we can match our input of
`1 + (2 + 3)`:

```
e -> e + e
     1 + e
     1 + (e)
     1 + (e + e)
     1 + (2 + e)
     1 + (2 + 3)
```

you might observe that we always expanded the leftmost item first.
alternatively, we can do rightmost expansion first:

```
e -> e + e
     e + (e)
     e + (e + e)
     e + (e + 3)
     e + (2 + 3)
     1 + (2 + 3)
```

while these two derivations of `e` are the same in this example, they can
possibly differ in some grammars/for some inputs. we call each of these the
_leftmost derivation_ and the _rightmost derivation_.

## parse trees

a way of showing the decisions made while expanding an expression to match a
derivation.

for the derivation example above, we can draw the following parse tree:

```
                      <e>
                  ___/ | \___
                 /     |     \
               <e>    "+"    <e>
                |        ___/ | \___
                |       /     |     \
               "1"    "("    <e>    ")"
                         ___/ | \___
                        /     |     \
                      <e>    "+"    <e>
                       |             |
                       |             |
                      "2"           "3"
```

## ambiguity

a grammar can be _ambiguous_ if the order in which you expand nonterminals can
effect the parsing outcome. to illustrate this, consider the following grammar
& sentence:

```
e ::= num | e + e | e - e

1 + 2 + 3 + 4 -> ((1 + 2) + 3) + 4, (1 + 2) + (3 + 4), ...
```

as seen in the expansion, this grammar (as given) is ambiguous. a common (easy)
solution is to force parens as a means of specifying _associativity_:

```
e ::= num | e + num | e - num | e + (e) | e - (e)
```

another source: _precidence_:

```
e ::= e + e | e - e | e * e | e / e

1 + 2 * 3 / 4 -> ?
```

to make this unambiguous, we _factor_ out productions:

```
f ::= num | (e)
t :: f | t * f | t / f
e ::= t | e + t | e - t
```

## the "dangling else" parsing problem

given the grammar `s ::= if e s | if e s else s`, how to parse `if e1 if e2 s1 else s2`?

solution--factor s into three nonterminals: `closedstmt`, `openstmt`, & `stmt`.

```
closedstmt ::= if e closedstmt else closedstmt
               | ... (non-if stmts not ending w/ an openstmt)
openstmtn  ::= if e closedstmt else openstmt
               | if e stmt
               | ... (non-if stmts ending w/ an openstmt)
stmt       ::= openstmt | closedstmt
```

let's try that out:

```
           stmt
            |
         openstmt
          /    \
       "if e1" stmt
                 |
             closedstmt
       ______/ | \___ \___
      /        |     \    \
"if e2" closedstmt "else" closedstmt
            |                 |
           "s1"              "s2"
```

## recursive descent parsers

a _recursive descent_ (or _predictive_) parser is a v simple algorithm that's
appropriate for very simple grammars. an example:

given a grammar of just `if ... then ... else ...` statements & variables:

```
e ::= if e then e else e
    | x
```

a small predictive parser is:

```
let rec parse_e (l: token list) =
  match l with
  | IF::l' ->
      (match parse_e l' with
       | THEN::l'' ->
         (match parse_e l'' with
          | ELSE::l''' -> parse_e l'''
          | [] -> error ())
       | [] -> error ())
  | (VAR x)::l' -> l'
  | [] -> error ()
```

the only reason this can be such a simple example is because the grammar is so
simple as well. we can see this with a few simple changes.

e.g. adding sequencing of expressions:

```
e ::= if e then e else e
    | x
    | e ; e
```

a small predictive parser is:

```
let rec parse_e (l: token list) =
  match l with
  | IF::l' ->
      (match parse_e l' with
       | THEN::l'' ->
         (match parse_e l'' with
          | ELSE::l''' -> parse_e l'''
          | [] -> error ())
       | [] -> error ())
  | (VAR x)::l' -> l'
  | _ -> (match parse_e l with ...) // ERROR: infinite loop on non-matching tokens!
```

or adding dangling `if`s:

```
e ::= if e then e else e
    | x
    | e ; e
```

a small predictive parser is:

```
let rec parse_e (l: token list) =
  match l with
  | IF::l' ->
      (match parse_e l' with
       | THEN::l'' ->
         (match parse_e l'' with
          | ELSE::l''' -> parse_e l'''
          | [] -> error ())
       | [] -> error ())
  | (VAR x)::l' -> l'
  | IF::l' -> ...                // ERROR: conflict w/ previous `IF` branch!
```

## LL(k) languages

grammars that Just Work<sup>TM</sup> w/ recursive descent algorithms are
classified as _LL(1)_, meaning **l**eft-to-right reading, **l**eftmost
derivation, w/ **k** token lookahead. the previous example was **ll(1)**.

while ll(k) languages are mathematically interesting, they aren't practical for
real programming usage. instead, we'll use **lr(k)** languages most of the
time, covered in the next lecture.
