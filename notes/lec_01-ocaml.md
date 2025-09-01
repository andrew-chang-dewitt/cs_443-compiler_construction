---
title: "Compilers notes: ocaml crash course"
description: "Intruductory lecture notes."
keywords:
  - "ocaml"
  - "lecture notes"
  - "computer science"
  - "cs 443"
  - "illinois tech"
meta:
  byline: Andrew Chang-DeWitt
  published: "2025-05-16T00:00-09:00"
---

ocaml is a statically typed functional language w/ strong similarities to Haskell (basically just forget about monads). basic syntax includes expressions, declarations, etc. see below for some examples.

these examples are all lines that can be executed in `utop`, an ocaml repl.

some basic expressions/values:

```ocaml
(* basic arithmetic *)
1 + 2;; (* => : int = 3 *)

(* strings *)
"Hello";; (* => : string = "Hello" *)
(* string concatenation w/ `^` operator *)
"Hello " ^ "World";; (* => : string = "Hello World" *)

(* unit value *)
();; (* => - : unit = () *)
```

exceptions can be thrown (or "raised"), custom exceptions can be created, & exception handling is done w/ `try ... with ...`:

```ocaml
(* expressions can throw *)
1 / 0;; (* => Exception: Division_by_zero. *)

(* throw exceptions w/ `raise` keyword *)
raise Division_by_zero;; (* => Exception: Division_by_zero. *)

(* declare your own exceptions *)
exception Foo;; (* => exception Foo *)
raise Foo;; (* => Exception: Foo. *)

(* exception handling *)
try 1 / 0
with Division_by_zero -> -1
   | Foo -> -2;; (* => - : int = -1 *)
```

compound types exist, such as pairs/tuples:

```ocaml
(* pairs *)
(1, 2);; (* => - : int * int = (1, 2) *)
(* parens are optional *)
1, 2;; (* => - : int * int = (1, 2) *)

(* not limited to two values *)
1, 2, 3;; (* => - : int * int * int = (1, 2, 3) *)
true, false;; (* => - : bool * bool = (true, false) *)

(* pairs have some builtin functions *)
fst (1, 2);; (* => - : int = 1 *)
snd (1, 2);; (* => - : int = 2 *)
(* but they don't work when airty != 2 *)
fst (1, 2, 3);; (* => Error: ... *)

```

bindings can be done as expression or as declarations:

```ocaml
(* binding expressions *)
let x = 5
 in x + 1;; (* => - : int = 6 *)
let x = (1, 2)
 in fst x;; (* => - : int = 1 *)
(* can pattern match in bindings *)
let (x, y) = (1, 2)
 in x + y;; (* => - : int = 3 *)

(* binding declarations *)
x = 5;; (* => val x: int = 5 *)
x + 1;; (* => - : int = 6 *)
```

functions can be bound to identifiers:

```ocaml
(* types can be specified *)
let double (x: int): int = x * 2;; (* => val double : int -> int = <fun> *)
(* but they can often be inferred *)
let double x = x * 2;; (* => val double : int -> int = <fun> *)
```

comparisons are all overloaded (note `=` is for equality, `==` is almost never what you want (used for referential equality)):

```ocaml
1 = 2;; (* => - : bool = false *)
```

parametric polymorphism:

```ocaml
let eq (x: 'a) (y: 'a): bool = x = y;; (* => val eq : 'a -> 'a -> bool = <fun> *)
```

recursion requires a `rec` keyword in your declaration (otherwise it will throw an unbound identifier exception at compile time):

```ocaml
let rec fact (n: int): int = if n <= 1 then 1
                             else n * (fact (n - 1));;
(* => val fact : int -> int = <fun> *)
```

some useful type constructs:

```ocaml
(* optional values *)
let div a b = try Some (a / b)
              with Division_by_zero -> None;; (* => val div : int -> int -> int option *)
div 2 2;; (* - : int option = Some(1) *)
div 1 0;; (* - : int option = None *)
let foo opt = match opt with
                  | Some x => x * 2
                  | None -> -1;; (* => val foo : int option -> int *)
foo Some(1);; (* - : int = 2 *)
foo None;; (* - : int = -1 *)
```

lists:

```
[];; (* => - 'a list = [] *)
[1];; (* => - int list = [1] *)
```

lists have some syntactic quirks&mdash;`::` for cons, `;` for separating elements:

```ocaml
1::2::[];; (* => - int list = [1; 2] *)
[1; 2];; (* => - int list = [1; 2] *)
[1, 2];; (* => - int * int list = [(1, 2)] *)
```

we can match on lists:

```ocaml
let hd l = match l with
               | [] -> None
               | h::_ -> Some h;; (* => 'a list -> 'a option = <fn> *)
```

and recur:

```ocaml
let rec map (f: 'a -> 'b) (l: 'a list) : 'b list =
  match l with
      | []   -> []
      | h::t -> (f h)::(map f t);; (* => ('a -> 'b) -> 'a list -> 'b list = <fn> *)
```

defining types, basic example is records:

```ocaml
(* using pairs *)
type person = string * int
let me : person = ("andrew", 35);;
(* using records *)
type person = { name: string; age: int; }
let me : person = { name = "andrew"; age = 35 }
me.name;; (* => - : string = "andrew" *)
```

algebraic data types:

```ocaml
(* some redefinitions of option & list as examples *)
type 'a my_opt = None | Some of 'a
Some (1);; (* => - : int my_opt = Some (1) *)
type 'a my_list = Nil | Cons of 'a * 'a my_list
Cons (1, Nil);; (* => - : int my_list = Cons(1, Nil) *)

(* let's try an expression type *)
type expr = Int of Int | Plus of exp * exp
Int 5;; (* => - : exp = Int 5 *)
Plus (Int 1, Int 2);; (* => - : exp = Plus (Int 1, Int 2) *)
```
