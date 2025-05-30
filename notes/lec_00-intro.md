# intro

what is a compiler?

given a source code as text, a compiler generates machine code which can be
linked w/ other machine code objects to create an executable:

```
 foo.c ---[compiler]--> foo.s ~~~[assembler]~~~> foo.o ~~~[linker]~~~> foo
|                            |                              |
|                            |                    lib1.o ~~~+
/------- this class ---------/                    lib2.o ~~~+
                                                  lib1.o ~~~+
```

remember other ways of taking source code & getting something that can be
executed from it:

1. source code -> compiler -> binary/assembly (this class, e.g.
   C/Rust/Haskell/...)
2. source code -> interpreter (cs 440, e.g. Racket/Python/JavaScript/...)
3. source code -> compiler -> bytecode -> VM (e.g. Java)

## compilation phases

compilation is done in phases:

```
                                           /-(analysis)       /-(optimization)       /-(optimization)
                                           |  /               |  /                   |  /
                                           v /                v /                    v /
 source --[lexer]--> tokens --[parser]--> ast --[lowering]--> ir --[code gen]--> target
|                                            |                                         |
|                                            |                                         |
/--------------- front end ------------------/--------------- back end ----------------/
```

front end is specific to one language & back end is specific to one
machine/architecture&mdash;this creates an opportunity for compilers to swap
out front & back ends so long as they share the same intermediate
representation (IR). an important example of this is `llvm` (low level virtual
machine).

## what's next?

in this course, students build a small ML compiler, lexing & parsing a source
language into a higher-order, typed AST (language) w/ structured data, nested
expressions, & unlimited variables; then converting closures & lifting it into
a first-order language (mini C); then generating from that an IR w/ flat
expressions; allocating registers from that into an LLVM compatible language w/
only 32 hardware registers (variables); before finally selecting instructions
to output RISC-V assembly code.

by project 3, an LLVM representation will be built; however, before getting
there, students will spend time with a smaller language: IITran (a Fortran
derivative designed to be easy to compile) as a way of understanding lexing &
parsing & directly targeing LLVM. afterwards, projects 3 & 4 will backtrack to
IR generation & closure conv./lifting before learning about optimizations &
register allocation in projects 5 & 6.
