# Chapter 9: Syntax-directed translation and intermediate code

<!-- contents -->
- [9.1 Syntax-directed definitions: synthesised and inherited attributes, annotated parse trees](#91-syntax-directed-definitions-synthesised-and-inherited-attributes-annotated-parse-trees)
- [9.2 Three-address code and its kinds of instruction](#92-three-address-code-and-its-kinds-of-instruction)
- [9.3 Quadruples, triples and indirect triples, in C](#93-quadruples-triples-and-indirect-triples-in-c)
- [9.4 Generating three-address code for expressions with Yacc](#94-generating-three-address-code-for-expressions-with-yacc)
- [9.5 Boolean expressions, if and while: labels and backpatching](#95-boolean-expressions-if-and-while-labels-and-backpatching)
- [9.6 Type checking and type conversion](#96-type-checking-and-type-conversion)
- [9.7 Practice: from source to three-address code for a new program](#97-practice-from-source-to-three-address-code-for-a-new-program)
<!-- /contents -->

## 9.1 Syntax-directed definitions: synthesised and inherited attributes, annotated parse trees

### Actions with a theory behind them

Chapter 8 ended with a parser that did more than say yes or no. `calc` computed `10 + 2.5 * 60` to 160 as it parsed; `post` wrote `pos init rate 60 * + =`; `tiny` printed every Tiny statement in postfix. In each, a few lines of C sat beside the productions, and the order in which the parser reduced put the results together in the right order. It worked, and section 8.6 explained *why* it worked for postfix. But the written paper does not ask for C. It asks: "Write a syntax-directed definition for a desk calculator and draw the annotated parse tree for `3 * 5 + 4`", or "Give an SDD that enters the type of each identifier in a declaration into the symbol table; show the annotated tree for `float a, b, c`."

Those questions use a notation that says the same thing as a Yacc action, without saying *when* the code runs: a value attached to each node of the parse tree, and a rule, attached to each production, that says how to compute it. This section gives you that notation, shows that your Yacc programs were already examples of it, and adds the one kind of value Yacc cannot compute in the obvious way: one that flows *down* the tree instead of up.

### Notes written on the tree

Chapter 1 called the symbol table the compiler's notebook. Here is a smaller notebook: every node of the parse tree gets a sticky note with a few fields on it, such as "value", "type", "code". The translation is a set of instructions for filling in the notes: "the value on an `E` made by `E + T` is the value on its `E` child plus the value on its `T` child". Fill in every note, and the note at the root holds the answer.

The picture is exact about *what* is written; it is silent about *order*. A note can only be filled in once the notes it reads are filled in, and the instructions do not say which to fill first. Working that out, and making it fit the order in which a real parser builds the tree, is the whole subject of the second half of this section.

### Attributes and semantic rules

<!-- teaches: sdd = syntax-directed definition, attribute, semantic rule -->
A *syntax-directed definition* (SDD) is a grammar with two things added:

- **attributes**: each grammar symbol may have named values, written `E.val`, `T.type`, `L.inh`. Every node of the parse tree labelled with that symbol has its own copy of each attribute. A terminal's attributes come from the scanner: `digit.lexval` is the digit's value, which in Yacc was `yylval`.
- **semantic rules**: each production has rules that compute attributes of its symbols from other attributes of the same production. When one symbol appears twice in a production, the copies are numbered, `E1` on the right for the `E` on the left: `E -> E1 + T` with the rule `E.val = E1.val + T.val`.

Here is the SDD of a desk calculator in the Dragon book's form, the paper's favourite. `n` is the end marker (a newline); `L` is a whole line.

| Production | Semantic rule |
|---|---|
| L -> E n | L.val = E.val |
| E -> E1 + T | E.val = E1.val + T.val |
| E -> T | E.val = T.val |
| T -> T1 * F | T.val = T1.val × F.val |
| T -> F | T.val = F.val |
| F -> ( E ) | F.val = E.val |
| F -> digit | F.val = digit.lexval |

Compare it line by line with `calc.y` of section 8.3. `E.val` is `$$`, `E1.val` is `$1`, `T.val` is `$3`; the rule `E.val = T.val` is the default action `$$ = $1`; `digit.lexval` is the value the scanner put in `yylval`. The SDD is the Yacc program with the C removed and the names made readable.

A parse tree with the values of its attributes written in, is called an *annotated* (or *decorated*) parse tree. To annotate one, draw the parse tree as in section 5.1, then evaluate each node's rule, children before parents. Here is `3 * 5 + 4 n` without the `L` and `n` at the top, which add nothing but the final copy. Each `F` box shows the `digit` leaf below it, to fit the page.

<!-- fig: id=calc-tree
title: Annotated parse tree of 3 * 5 + 4, desk-calculator SDD
kind: tree
dir: down
box e19: E.val = 19
box e15: E.val = 15
box plus: `+`
box t4: T.val = 4
box f4: F.val = 4\ndigit.lexval = 4
box t15: T.val = 15
box t3: T.val = 3
box star: `*`
box f5: F.val = 5\ndigit.lexval = 5
box f3: F.val = 3\ndigit.lexval = 3
e19 -> e15
e19 -> plus
e19 -> t4
t4 -> f4
e15 -> t15
t15 -> t3
t15 -> star
t15 -> f5
t3 -> f3
hot: e19
-->
![Tree, top to bottom. E.val = 19 leads to E.val = 15. E.val = 19 leads to +. E.val = 19 leads to T.val = 4. E.val = 15 leads to T.val = 15. T.val = 4 leads to F.val = 4 digit.lexval = 4. T.val = 15 leads to T.val = 3. T.val = 15 leads to *. T.val = 15 leads to F.val = 5 digit.lexval = 5. T.val = 3 leads to F.val = 3 digit.lexval = 3. E.val = 19 is highlighted.](fig/09-sdt-intermediate-code/calc-tree.svg)

Read the tree from the leaves. The three `F`s take their digits' values, 3, 5 and 4. The `T` above the 3 copies it (T -> F). The `T` with the `*` multiplies its `T1` and its `F`: 3 × 5 = 15. The `E` above it copies 15 (E -> T). On the right, `T` copies 4. At the root, E -> E1 + T gives 15 + 4 = 19. Every value came *up* from the children; no node needed anything from its parent or its siblings.

### Synthesised attributes

<!-- teaches: synthesised-attribute = synthesised attribute, annotated parse tree, S-attributed definition -->
An attribute is *synthesised* when its value at a node is computed only from the attributes of that node's children (and of the node itself). Every attribute of the calculator is synthesised: `E.val` is made from `E1.val` and `T.val`, both children. A tree whose attributes are all synthesised can be annotated in one pass, bottom-up: any order that visits the children before the parent will do, and a postorder walk is such an order.

An SDD with only synthesised attributes is called *S-attributed*. It suits an LR parser perfectly, because an LR parser *is* a bottom-up, postorder builder of the tree (section 8.6): when it reduces by a production, every child has just been finished, and its values are sitting on the value stack. So each semantic rule becomes the action of its production, with `$$` on the left and `$1`, `$2` ... on the right, and Yacc evaluates the whole SDD as it parses, without ever building the tree. That is exactly what `calc` did.

### Inherited attributes

<!-- teaches: inherited-attribute = inherited attribute, L-attributed definition -->
Not every value flows up. In the C declaration `float a, b, c;`, the type `float` is written once, at the start, and every name in the list must receive it. In the parse tree, the type sits in a subtree to the *left* of the list, and has to travel across to the list and then *down* to each name. Here is the classic SDD; `addtype` enters a name with its type in the symbol table.

| Production | Semantic rules |
|---|---|
| D -> T L | L.inh = T.type |
| T -> int | T.type = integer |
| T -> float | T.type = float |
| L -> L1 , id | L1.inh = L.inh; addtype(id.entry, L.inh) |
| L -> id | addtype(id.entry, L.inh) |

`T.type` is synthesised, from the keyword below it. `L.inh` is different: the rule for D -> T L sets the `L`'s attribute from its *sibling* `T`, and the rule for L -> L1 , id sets `L1.inh` from its *parent* `L`. An attribute is *inherited* when its value at a node comes from the node's parent or siblings. In the annotated tree for `float a, b, c`, the type goes across once and then down the spine of `L`s:

<!-- fig: id=decl-tree
title: Annotated tree of float a, b, c
kind: tree
dir: down
box d: D
box t: T.type\nfloat
box l: L.inh\nfloat
box l1: L.inh\nfloat
box idc: `id` c
box l2: L.inh\nfloat
box idb: `id` b
box ida: `id` a
d -> t
d -> l
l -> l1
l -> idc
l1 -> l2
l1 -> idb
l2 -> ida
hot: l l1 l2
-->
![Tree, top to bottom. D leads to T.type float. D leads to L.inh float. L.inh float leads to L.inh float. L.inh float leads to id c. L.inh float leads to L.inh float. L.inh float leads to id b. L.inh float leads to id a. L.inh float, L.inh float and L.inh float are highlighted.](fig/09-sdt-intermediate-code/decl-tree.svg)

The order of evaluation is forced by where the values come from. First `T.type`, from its leaf. Then the top `L.inh`, from `T`. Then each lower `L.inh`, from the `L` above it. And at each `L`, once its `inh` is known, `addtype` runs for the `id` it holds: c at the top `L`, then b, then a at the bottom. A value goes down only after it has come from the left. An SDD where every inherited attribute of a symbol in a production depends only on the attributes of the parent and of the symbols to its *left* is called *L-attributed*; this one is, and so is every S-attributed SDD. Such an SDD can be evaluated in one walk of the tree, left to right, depth first, which is the order of a top-down parser and, with one trick, of a bottom-up one too.
<!-- risk: definition of L-attributed (inherited attributes depend only on the parent's inherited attributes and on symbols to the left; Dragon 2nd ed. 5.2.4 also allows the symbol's own attributes without cycles); check the wording against Dragon 5.2.4 -->

### Inherited attributes in Yacc: looking under the plate

A bottom-up parser has no node for `L` until the `L` is complete, so where does `L.inh` live while the names are being reduced? The answer is on the stack. When the parser reduces the first name of `float a, b, c` to `L`, the stack holds `T L`'s left part: the `T`, with its `type`, is the plate *just below* the names being reduced. Yacc lets an action look at that plate: `$0` is the value of the symbol directly below the right side being reduced. In `L -> id`, `$1` is the `id` and `$0` is whatever is beneath it, which, in this grammar, is always the `T`.

That is the trick that lets an LR parser handle this L-attributed SDD with no tree and no global variable. `decl.y` does it. The scanner returns the type word and the names as strings:

<!-- code: id=ch9-setup -->
```console
$ mkdir cdlab/ch9
$ cd cdlab/ch9
```

<!-- code: id=decl-write -->
```console
$ cat > decl.l <<'END'
%{
/* decl.l: the scanner for decl.y */
#include <string.h>
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
int|float|char              { yylval.s = strdup(yytext); return TYPE; }
[A-Za-z_][A-Za-z0-9_]*      { yylval.s = strdup(yytext); return ID; }
[ \t\n]+                    { /* blanks */ }
.                           { return yytext[0]; }
%%
END
$ cat > decl.y <<'END'
%{
/* decl.y: declarations such as float a, b, c; each name gets the type
   that the list inherits from its declaration */
#include <stdio.h>
#include <stdlib.h>
int yylex(void);
void yyerror(const char *s);
%}
%union {
    char *s;
}
%token <s> TYPE ID
%%
decls : /* nothing */
      | decls decl
      ;
decl  : TYPE list ';'        { free($1); }
      ;
list  : ID                   { printf("%-6s %s\n", $<s>0, $1); free($1); }
      | list ',' ID          { printf("%-6s %s\n", $<s>0, $3); free($3); }
      ;
%%
void yyerror(const char *s)
{
    fprintf(stderr, "%s\n", s);
}

int main(void)
{
    return yyparse();
}
END
```

The grammar is the SDD's: `decl` is D -> T L, with the type word as `T`; `list` is L, left-recursive. The two `list` actions are `addtype`, here printing the type and the name instead of writing a symbol table. `$<s>0` is `L.inh`: the value of the symbol under the right side. The `<s>` says which member of the union to use, because Bison cannot know the type of a symbol that is not in this production. In `list : list ',' ID`, the symbol under the right side is again the `TYPE`, because the inner `list` was reduced from the plates above it; `$0` is the same plate at every level. `decl`'s action frees the type word once the whole list has used it.

<!-- code: id=decl-run -->
```console
$ yacc -d decl.y
$ flex decl.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o decl
$ printf 'float rate, init, pos;\nint i, n;\n' | ./decl
float  rate
float  init
float  pos
int    i
int    n
```

Each name came out with its declaration's type: the inherited attribute reached every leaf. Notice the order: `rate` first. The figure's top-down order handed the type to `c` first, but `decl` gives it to `a` first, because a left-recursive list is reduced from its leftmost name up (`rate` is the innermost `list`). Both are correct evaluations of the same SDD; the rules say what each `L.inh` is, not when to print.

`$0` is safe only when you know what is under the right side every time the production is reduced. Here a `list` only ever appears right after a `TYPE`. If the grammar also allowed `list` somewhere else, say after `extern`, `$0` would there be something else entirely, and Bison could not warn you.

### When it goes wrong

**Calling an attribute inherited because it is on the right side.** `T.val` in E -> E1 + T is synthesised: it was computed at the `T` node, from the `T`'s children, and is only *read* here. Synthesised or inherited is decided by where an attribute's rule sits: in a production where the symbol is on the left (synthesised) or on the right (inherited).

**Annotating in the wrong order.** For an inherited attribute, filling in the notes bottom-up gives a leaf nothing to read: `addtype(a, L.inh)` before `L.inh` is known. Draw the dependency arrows first (from the attributes a rule reads to the one it sets) and follow them.

**An SDD with a cycle.** Rules such as `A.s = B.i` and `B.i = A.s + 1` in one production make each attribute wait for the other; no order exists. The paper sometimes asks you to spot this.

**`$0` in a production that can appear in two places.** The value under the right side differs, and the program reads garbage or crashes. Use `$0` only where one symbol always precedes the production.

**`$0` without `<member>` under `%union`.** Bison stops with `error: $0 of 'list' has no declared type`: it cannot tell the member, since the symbol below is not part of the production. Write `$<s>0`.
<!-- checked: how=run date=2026-11-21 ref="scratch decl.y with $0: Bison 3.8.2 error: $0 of 'list' has no declared type" -->

### In one line each

1. An SDD is a grammar with attributes on its symbols and a semantic rule for each attribute on each production.
2. An annotated parse tree shows every attribute's value at every node.
3. Synthesised: computed from the children; inherited: from the parent or siblings.
4. S-attributed SDDs are evaluated by any bottom-up walk, and by Yacc's actions as it parses.
5. L-attributed SDDs take inherited values only from the parent and from symbols to the left; in Yacc, `$<member>0` reads the symbol just below the right side.

**Check yourself**

<!-- q: idea=synthesised-attribute type=apply wrong="the * node's value is computed before its children are known" -->
1. Draw the annotated parse tree for `(3 + 4) * 5 n` with the desk-calculator SDD, and list the order in which a bottom-up parser computes the `val` attributes.

<details>
<summary>Answer</summary>

L.val = 35 at the root, from E.val = 35, from T.val = 35, from T -> T1 * F with T1.val = 7 and F.val = 5. T1 comes from F -> ( E ), whose E.val = 7 is E -> E1 + T with E1.val = 3 and T.val = 4. In bottom-up order: F = 3, T = 3, E1 = 3, F = 4, T = 4, E = 7 (the `+`), F = 7 (the parentheses), T1 = 7, F = 5, T = 35 (the `*`), E = 35, L = 35. The parentheses' node gets 7 *before* the 5 is even read: the parser reduces F -> ( E ) when it sees `)`.

</details>

<!-- q: idea=inherited-attribute type=decide wrong="every attribute on the right side is inherited" -->
2. For each attribute, say whether it is synthesised or inherited: (a) `T.type` in D -> T L; (b) `L.inh` in D -> T L; (c) `L1.inh` in L -> L1 , id; (d) `E1.val` in E -> E1 + T. Is the declaration SDD S-attributed, L-attributed, both or neither?

<details>
<summary>Answer</summary>

(a) Synthesised: its rule is in T -> float, where `T` is on the left; D -> T L only reads it. (b) Inherited: set in D -> T L, where `L` is on the right, from its sibling. (c) Inherited: set from the parent `L`. (d) Synthesised: set in its own production, read here. The SDD is L-attributed (every inherited attribute comes from the parent or from the left) and not S-attributed (it has inherited attributes).

</details>

<!-- q: idea=inherited-attribute type=predict wrong="$0 is the type only in the first list production" -->
3. Add the line `char ok;` to the input of `decl` and predict the output. Then predict what happens if the line is `char ;` instead.

<details>
<summary>Answer</summary>

<!-- code: id=decl-predict -->
```console
$ printf 'char ok;\n' | ./decl
char   ok
$ printf 'char ;\n' | ./decl; echo "exit $?"
syntax error
exit 1
```

`char   ok` for the first: one name, reduced by `list : ID`, with the `TYPE` under it. The second is a syntax error, since a `list` needs at least one `ID`; `decl` prints `syntax error` and exits with status 1.

</details>

## 9.2 Three-address code and its kinds of instruction

### Between the tree and the machine

Chapter 1 traced `pos = init + rate * 60;` through every phase, and phase 4 turned the tree into four short lines:

```text
t1 = inttofloat(60)
t2 = id3 * t1
t3 = id2 + t2
id1 = t3
```

That was the first sight of *three-address code*, the intermediate language of this book and of the written paper. Section 8.6's postfix already put the operations in the right order; what it lacked was names for the results. A stack machine keeps them on its stack; a real processor keeps them in registers and memory, and needs to be told where. Three-address code gives every intermediate result a name, so that each instruction says exactly what it reads and where it writes.

Why have an intermediate language at all, instead of writing machine code straight from the parse tree? Because of a multiplication. A company with three source languages (C, C++, Fortran) and four processors (x86-64, ARM, RISC-V, MIPS) would need twelve compilers if each went straight from source to machine. With one intermediate language in the middle, it needs three front ends that each write the intermediate code and four back ends that each read it: seven pieces, and the optimiser of chapter 10 is written once, for the intermediate code, and serves all of them. GCC's GIMPLE, which you saw in section 1.4, is exactly such a language, and it is very close to three-address code.

### One operation per line

<!-- teaches: three-address-code = three-address code, temporary, address -->
Three-address code (TAC) is a sequence of instructions, each with **at most one operator** on its right side. An instruction has at most three *addresses*: two operands and a result, hence the name. An address is one of three things:

- a **name** from the source program, `pos`, `rate` (in a real compiler, a pointer to its symbol-table entry; chapter 1 wrote `id3`);
- a **constant**, `60`, `2.5`;
- a **temporary**, a name the compiler invents, `t1`, `t2`, for a value that the source program never named, such as the result of `rate * 60`.

So `pos = init + rate * 60` cannot be one instruction: it has two operators. It becomes two, the first writing its result into a temporary that the second reads:

```text
t1 = rate * 60
t2 = init + t1
pos = t2
```

That is the code chapter 1 wrote, without the type conversion (Tiny has one numeric type; section 9.6 brings conversions back). The order of the instructions is the order of section 8.6's postfix, `init rate 60 * +`, with a temporary named for each operator. It is the same postorder walk of the tree. A temporary is an unlimited supply of fresh names: the intermediate code pretends the machine has as many registers as it likes, and the code generator of section 10.6 later decides which temporaries live in real registers and which in memory.

### The kinds of instruction

The Dragon book lists these kinds, and exam answers are expected to use them as written:

| Kind | Form | Example | Means |
|---|---|---|---|
| assignment, binary | `x = y op z` | `t1 = rate * 60` | arithmetic or logic on two operands |
| assignment, unary | `x = op y` | `t2 = minus c` | negation, logical not, conversion (`inttofloat`) |
| copy | `x = y` | `pos = t2` | move a value |
| unconditional jump | `goto L` | `goto L1` | the next instruction is the one labelled `L` |
| conditional jump | `if x goto L`, `ifFalse x goto L` | `ifFalse t1 goto L2` | jump if `x` is true (or false) |
| conditional jump on a comparison | `if x relop y goto L` | `if i < n goto L1` | `relop` is `<`, `<=`, `==`, `!=`, `>`, `>=` |
| procedure call | `param x`, `call p, n`, `y = call p, n`, `return y` | `param x` / `t3 = call f, 1` | pass n parameters, then call |
| indexed copy | `x = y[i]`, `x[i] = y` | `t2 = a[t1]` | `i` is an offset in bytes, not an index |
| address and pointer | `x = &y`, `x = *y`, `*x = y` | `t5 = &rate` | address of, load through, store through |

A label is a name for an instruction, written `L1:` before it. Some books number every instruction instead (`100: t1 = ...`) and jump to numbers; section 9.5 does that too, because backpatching needs numbers.

Two kinds hide a trap each. A **conditional jump on a comparison** takes a comparison of two addresses directly, `if i < n goto L1`, so a comparison used only to decide a jump needs no temporary; a comparison whose value is *stored*, as in `f = a < b`, needs one: `t1 = a < b` and `f = t1`. And in an **indexed copy**, the index is an *offset in bytes* from the start of the array. For an array of `int`s of 4 bytes each, `a[i]` is `t1 = i * 4` followed by `t2 = a[t1]`. Writing `t2 = a[i]` is the most common mistake in this kind of question: it is right only for an array of one-byte elements.

### A loop by hand

Here is a loop in C-like Tiny, with an array: the sum of the first `n` elements of `a`.

```c
s = 0;
i = 0;
while (i < n) {
    s = s + a[i];
    i = i + 1;
}
```

Translate it one statement at a time, keeping the rule of one operator per line, and lay out the loop with two labels: one where the test starts, so that the end of the body can jump back to it; and one after the loop, where the test jumps when it fails.

```text
    s = 0
    i = 0
L1: ifFalse i < n goto L2
    t1 = i * 4
    t2 = a[t1]
    t3 = s + t2
    s = t3
    t4 = i + 1
    i = t4
    goto L1
L2:
```

Walk it for a moment. The test comes first, at `L1`, and jumps out to `L2` when it is false; the body follows it; and the body ends with a jump back to the test, never to the body's first line, because the test must run again before every pass. `ifFalse i < n goto L2` uses the comparison form directly, so the test needs no temporary. `a[i]` became an offset computation and an indexed copy. And `s = s + a[i]`, one line of C, became four lines of TAC.

### Running three-address code

Three-address code is not only a notation for the paper. Every line of it is also a line of C, if you allow yourself labels and `goto`, which C has. That gives you a way to test a translation done by hand: write it as C, run it, and compare its answer with the original. Save this as `tac.c`:

<!-- code: id=tac -->
```c
/* tac.c: the same loop twice, as C and as three-address code.
   The second version uses only TAC instructions: one operator per line,
   labels and gotos, and array access as base[offset] in bytes. */
#include <stdio.h>

int a[5] = { 3, 1, 4, 1, 5 };
int n = 5;

int sum_c(void)
{
    int s = 0, i = 0;

    while (i < n) {
        s = s + a[i];
        i = i + 1;
    }
    return s;
}

int sum_tac(void)
{
    int s, i, t1, t2, t3, t4;
    char *base = (char *)a;     /* a's address, counted in bytes */

    s = 0;
    i = 0;
L1: if (i >= n) goto L2;        /* ifFalse i < n goto L2 */
    t1 = i * 4;                 /* the offset of a[i]: 4 bytes per int */
    t2 = *(int *)(base + t1);   /* t2 = a[t1] */
    t3 = s + t2;
    s = t3;
    t4 = i + 1;
    i = t4;
    goto L1;
L2: return s;
}

int main(void)
{
    printf("as C:   %d\n", sum_c());
    printf("as TAC: %d\n", sum_tac());
    printf("sizeof(int) = %zu\n", sizeof(int));
    return 0;
}
```
```text
as C:   14
as TAC: 14
sizeof(int) = 4
```

`sum_tac` is the hand translation, line for line. Two lines needed C's help to say what TAC says. `ifFalse i < n goto L2` became `if (i >= n) goto L2`, the same test turned round. And `t2 = a[t1]`, with `t1` in bytes, became `*(int *)(base + t1)`: take `a`'s address as a pointer to bytes, move `t1` bytes along, and read an `int` there. That is what the indexed copy means, and the program proves the offset is right: both versions add up to 14, 3 + 1 + 4 + 1 + 5. The last line shows the width of an `int` on this machine, 4, which is where the 4 in `t1 = i * 4` came from.

Change `t1 = i * 4;` to `t1 = i;` and run it again (question 2 does): the TAC version reads bytes from the middle of integers and gives a wrong sum, which is exactly what `t2 = a[i]` means in TAC.

### When it goes wrong

**Two operators on one line.** `t1 = init + rate * 60` is not three-address code. Each operator gets its own instruction and its own temporary.

**The index used as an offset.** `t2 = a[i]` for an array of 4-byte `int`s reads the wrong bytes. Multiply by the element's width first; for a two-dimensional array `a[i][j]` of `int` with 10 columns, the offset is `(i * 10 + j) * 4`.

**The loop's jump back goes to the body.** `goto` the line after the test means the test runs once, before the first pass, and never again: an endless loop for every loop that should stop. The jump back goes to the label of the test.

**A temporary used before it is written, or written twice.** In code made from an expression, each temporary is written once and read later. Two instructions writing `t1` is the sign of a translation that reused a name; it may still compute the right thing, but chapter 10's optimisations assume one writer per temporary in an expression's code.

**A comparison stored without a temporary.** `f = a < b` is a binary assignment, `t1 = a < b` then `f = t1`, or directly `f = a < b`, which is one operator, so it is fine. The `if x relop y goto L` form is only for jumps.

### In one line each

1. Three-address code has at most one operator per instruction and at most three addresses: names, constants and temporaries.
2. An expression becomes one instruction per operator, in postorder, each writing a fresh temporary.
3. The kinds: binary, unary, copy, goto, if/ifFalse goto, if relop goto, param/call/return, indexed copy, address and pointer.
4. A loop is a labelled test that jumps out when false, the body, and a jump back to the test.
5. An array index in TAC is an offset in bytes: multiply by the element's width.

**Check yourself**

<!-- q: idea=three-address-code type=apply wrong="a[i] is t = a[i] in TAC" -->
1. Write three-address code for `x = a[i] * b[j] - c` where `a` and `b` are arrays of 8-byte `double`s.

<details>
<summary>Answer</summary>

```text
t1 = i * 8
t2 = a[t1]
t3 = j * 8
t4 = b[t3]
t5 = t2 * t4
t6 = t5 - c
x = t6
```

Each array read needs its offset first, in bytes: 8 for a `double`. The multiplication comes before the subtraction because `*` binds tighter; within the multiplication, the left operand is computed first, by the postorder rule. `x = t5 - c` in place of the last two lines is also correct and one line shorter; it is the copy that chapter 1's optimiser removed.

</details>

<!-- q: idea=three-address-code type=predict wrong="a wrong offset only reads a wrong element" -->
2. In `tac.c`, change `t1 = i * 4;` to `t1 = i;` and predict whether `sum_tac` still gives 14, and why. Then run it.

<details>
<summary>Answer</summary>

No. With `t1 = i`, the loop reads an `int` starting 0, 1, 2, 3 and 4 bytes from the start of `a`: only the first read is a whole element. The others take bytes from two neighbouring elements; on this little-endian machine the reads at offsets 1, 2 and 3 are huge numbers made of a high byte and the next element's low bytes, and the sum is garbage.

<!-- code: id=tac-bad -->
```console
$ sed 's/t1 = i \* 4;/t1 = i;/' tac.c > tacbad.c
$ gcc -Wall -Wextra tacbad.c -o tacbad
$ ./tacbad
as C:   14
as TAC: 16843012
sizeof(int) = 4
```

The C version is unchanged, which is the point of keeping it beside the TAC: when the two disagree, the hand translation is wrong.

</details>

<!-- q: idea=three-address-code type=apply wrong="do-while tests before the body" -->
3. Write three-address code for this loop, which runs its body at least once (the `do ... while` of C):

```c
do {
    i = i + 1;
} while (a[i] < v);
```

`a` holds 4-byte `int`s.

<details>
<summary>Answer</summary>

```text
L1: t1 = i + 1
    i = t1
    t2 = i * 4
    t3 = a[t2]
    if t3 < v goto L1
```

The body comes first, the test last, and the test jumps *back* when it is true: one label, one conditional jump, no `goto`. The Dragon book uses this very loop to show three-address code, with the instructions numbered from 100 instead of labelled, so expect it in the paper in that form too.
<!-- risk: the Dragon book's do-while example a[i] < v appears in 2nd ed. section 6.2 with numbered instructions; check the reference -->

</details>

## 9.3 Quadruples, triples and indirect triples, in C

### Three-address code in memory

Three-address code as you have written it is text. A compiler does not keep text: the optimiser of chapter 10 has to look at an instruction's operator, find which instructions read a temporary, move instructions and delete them, and doing that on strings would be slow and clumsy. So the instructions are kept in a table of records, one record per instruction, with a field for each part. The paper asks for exactly these tables: "Write the quadruples, triples and indirect triples for `a = b * -c + b * -c`." It is one of the most predictable questions in the whole paper, and the three answers differ in one idea only: how an instruction refers to a result computed by another.

### Quadruples

<!-- teaches: quadruple = quadruple -->
A *quadruple* is a record with four fields: `op`, `arg1`, `arg2` and `result`. `x = y + z` is the quadruple (`+`, `y`, `z`, `x`). A unary operator leaves `arg2` empty; a copy `x = y` has the operator `=` and `y` in `arg1`. Jumps put the target label in `result`.

Take the paper's expression. `-c` is a unary minus, written `minus` in a table to tell it from subtraction. Its three-address code, in postorder, is:

```text
t1 = minus c
t2 = b * t1
t3 = minus c
t4 = b * t3
t5 = t2 + t4
a = t5
```

and its quadruples, numbered from 0, are:

| | op | arg1 | arg2 | result |
|---|---|---|---|---|
| (0) | minus | c | | t1 |
| (1) | * | b | t1 | t2 |
| (2) | minus | c | | t3 |
| (3) | * | b | t3 | t4 |
| (4) | + | t2 | t4 | t5 |
| (5) | = | t5 | | a |

The table is the code, column by column. Temporaries are named, so an instruction refers to an earlier result by its name, `t1`, and the instructions can be moved about freely: wherever `(1)` ends up, it still reads `t1`. The cost is that every temporary is a name in the symbol table, needing a place to live.

### Triples

<!-- teaches: triple = triple -->
A *triple* drops the `result` field. A temporary's value is referred to by the *number of the triple that computes it*, written in parentheses: instead of `t1`, the argument is `(0)`, "the result of triple 0".

| | op | arg1 | arg2 |
|---|---|---|---|
| (0) | minus | c | |
| (1) | * | b | (0) |
| (2) | minus | c | |
| (3) | * | b | (2) |
| (4) | + | (1) | (3) |
| (5) | = | a | (4) |

Read triple (1): multiply `b` by the result of triple (0). Triple (5) is the assignment, written with `=` as the operator, the target `a` as `arg1` and the value `(4)` as `arg2`. A triple that stores into a real name always looks like that, since there is no `result` field to hold the name.

Triples save the space of the `result` field and of the temporaries' names. The cost shows as soon as the optimiser moves an instruction. If triple (2) is moved, every triple that says `(2)` must be found and changed; the numbers are positions, and positions change. In quadruples, nothing would need changing.

### Indirect triples

<!-- teaches: indirect-triple = indirect triple -->
*Indirect triples* keep the triples and add a second list: the *order* in which the instructions run, as a list of pointers to triples. The Dragon book numbers that list from 35, to make clear it is a different list from the triples:

| instruction | triple |
|---|---|
| 35 | (0) |
| 36 | (1) |
| 37 | (2) |
| 38 | (3) |
| 39 | (4) |
| 40 | (5) |
<!-- risk: the Dragon book (2nd ed. Fig. 6.12) numbers the instruction list of indirect triples from 35; check -->

At first this looks like a waste: a second table that only counts. Its point is that the optimiser reorders instructions by reordering *this* list, and leaves the triples, and the `(n)` references inside them, untouched. Triple (1) always refers to triple (0), wherever either one appears in the running order.

| Representation | Fields | Refers to a result by | Moving an instruction |
|---|---|---|---|
| quadruples | op, arg1, arg2, result | the temporary's name | free; nothing else changes |
| triples | op, arg1, arg2 | the triple's number | every `(n)` that refers to it must change |
| indirect triples | triples + an instruction list | the triple's number | change the list only |

### tables.c: all three from the code

Habit 7 says to check hand work with a program. `tables.c` reads three-address code, one instruction per line, in the three assignment shapes, `x = y op z`, `x = op y` (with `-` for unary minus) and `x = y`, and prints the three tables. It treats a name made of `t` and digits as a temporary. Save it:

<!-- code: id=tables mode=compile -->
```c
/* tables.c: three-address code to quadruples, triples and indirect triples.
   Reads one instruction per line, in one of three shapes:
       x = y op z        x = op y        x = y
   A name made of t and digits (t1, t12) is a temporary. */
#include <stdio.h>
#include <string.h>
#include <ctype.h>

#define MAX 100

struct quad {                    /* one instruction as a quadruple */
    char op[8], arg1[16], arg2[16], result[16];
} q[MAX];
int n;                           /* instructions read */
int at[MAX];                     /* at[i]: the triple that computes q[i] */

int istemp(const char *s)
{
    return s[0] == 't' && isdigit((unsigned char)s[1]);
}

void row(int i, const char *a, const char *b, const char *c)
{
    char line[80];
    int len = sprintf(line, "(%d)  %-6s %-6s %s", i, a, b, c);

    while (len > 0 && line[len - 1] == ' ')   /* no blanks at the end */
        line[--len] = '\0';
    puts(line);
}

void targ(char *out, const char *a)  /* in a triple, a temporary is (i) */
{
    strcpy(out, a);
    if (istemp(a))
        for (int i = 0; i < n; i++)
            if (strcmp(q[i].result, a) == 0)
                sprintf(out, "(%d)", at[i]);
}

int main(void)
{
    char line[100], w[4][16], a1[16], a2[16], res[40];
    int k, t = 0;                /* t: the next triple's number */

    while (n < MAX && fgets(line, sizeof line, stdin)) {
        k = sscanf(line, "%15s = %15s %15s %15s", w[0], w[1], w[2], w[3]);
        if (k < 2)
            continue;            /* a blank or unreadable line */
        strcpy(q[n].result, w[0]);
        if (k == 4) {            /* x = y op z */
            strcpy(q[n].op, w[2]);
            strcpy(q[n].arg1, w[1]);
            strcpy(q[n].arg2, w[3]);
        } else if (k == 3) {     /* x = op y; unary - is written minus */
            strcpy(q[n].op, strcmp(w[1], "-") == 0 ? "minus" : w[1]);
            strcpy(q[n].arg1, w[2]);
        } else {                 /* x = y, a copy */
            strcpy(q[n].op, "=");
            strcpy(q[n].arg1, w[1]);
        }
        n++;
    }

    printf("Quadruples\n     op     arg1   arg2   result\n");
    for (int i = 0; i < n; i++) {
        sprintf(res, "%-6s %s", q[i].arg2, q[i].result);
        row(i, q[i].op, q[i].arg1, res);
    }

    printf("\nTriples\n     op     arg1   arg2\n");
    for (int i = 0; i < n; i++) {
        targ(a1, q[i].arg1);
        targ(a2, q[i].arg2);
        if (istemp(q[i].result)) {           /* its number is its name */
            at[i] = t;
            row(t++, q[i].op, a1, a2);
        } else if (strcmp(q[i].op, "=") == 0) {
            at[i] = t;                       /* x = y: one triple */
            row(t++, "=", q[i].result, a1);
        } else {                             /* x = y op z: compute, then store */
            row(t++, q[i].op, a1, a2);
            sprintf(a2, "(%d)", t - 1);
            at[i] = t;
            row(t++, "=", q[i].result, a2);
        }
    }

    printf("\nIndirect triples\ninstruction  triple\n");
    for (int i = 0; i < t; i++)
        printf("%-11d  (%d)\n", 35 + i, i);
    return 0;
}
```

**Reading.** The loop reads a line with `fgets` (habit 8) and splits it with `sscanf` into up to four words around the `=`. The number of words matched tells the shape: four for `x = y op z`, three for `x = op y`, two for a copy. Each instruction becomes one `struct quad`; the quadruple table is then a plain printout of the array.

**Triples.** The second loop turns quadruples into triples, keeping in `at[i]` the number of the triple that computes the result of quadruple `i`. An argument that is a temporary is looked up by `targ`: the quadruple that wrote it, and from `at`, that quadruple's triple, printed as `(n)`. There are three cases for the result. A temporary result needs no field: the triple's number names it. A copy into a real name is one triple `= x y`. And `x = y op z` with a real name `x`, such as `pos = init + t1`, needs *two* triples, one to compute and one to store, because a triple has no result field; that is why triple numbers can run ahead of quadruple numbers.

**Indirect triples.** The instruction list, numbered from 35, points at each triple in order, as the table above did.

`row` prints a numbered row and trims the blanks that an empty field leaves at the end of the line.

<!-- code: id=tables-run -->
```console
$ gcc -Wall -Wextra tables.c -o tables
$ printf 't1 = - c\nt2 = b * t1\nt3 = - c\nt4 = b * t3\nt5 = t2 + t4\na = t5\n' | ./tables
Quadruples
     op     arg1   arg2   result
(0)  minus  c             t1
(1)  *      b      t1     t2
(2)  minus  c             t3
(3)  *      b      t3     t4
(4)  +      t2     t4     t5
(5)  =      t5            a

Triples
     op     arg1   arg2
(0)  minus  c
(1)  *      b      (0)
(2)  minus  c
(3)  *      b      (2)
(4)  +      (1)    (3)
(5)  =      a      (4)

Indirect triples
instruction  triple
35           (0)
36           (1)
37           (2)
38           (3)
39           (4)
40           (5)
```

The three tables are the ones worked by hand above. Now the running statement, in its short form, where the last instruction stores straight into `pos`:

<!-- code: id=tables-pos -->
```console
$ printf 't1 = rate * 60\npos = init + t1\n' | ./tables
Quadruples
     op     arg1   arg2   result
(0)  *      rate   60     t1
(1)  +      init   t1     pos

Triples
     op     arg1   arg2
(0)  *      rate   60
(1)  +      init   (0)
(2)  =      pos    (1)

Indirect triples
instruction  triple
35           (0)
36           (1)
37           (2)
```

Two quadruples, but three triples: `pos = init + t1` has no temporary result, so its triple computes `init + (0)` and a second triple, `= pos (1)`, stores it. The paper accepts either form for the quadruples, `pos = init + t1` or a temporary and a copy; for the triples, the store must be a triple of its own.

### Why the optimiser cares

Section 10.2 will notice that `t3 = minus c` computes what `t1` already holds, and that `t4 = b * t3` is then `b * t1`, which `t2` already holds. Removing them changes the three representations in different ways:

- **Quadruples**: delete (2) and (3), and change `t4` to `t2` in (4). The other rows stay as they are.
- **Triples**: delete (2) and (3), and every later triple moves up two places. Old (4) becomes (2), and must now say `+ (1) (1)`; old (5) becomes (3) and must say `= a (2)`. Every reference after the deletion has to be renumbered.
- **Indirect triples**: drop 37 and 38 from the instruction list and change triple (4)'s `(3)` to `(1)`. The other triples keep their numbers.

That is the whole argument for quadruples in optimising compilers, and for indirect triples where space matters: both survive editing.

### When it goes wrong

**A result field in a triple.** Writing `(1) * b (0) t2` is a quadruple with a number in front. Triples have three fields; the number in parentheses *is* the result.

**Numbering triples by quadruple numbers.** When an assignment to a real name needs two triples, every triple after it is one further along. Count the triples as you write them, or check with `tables`.

**Unary minus written as subtraction.** `-` with one operand is a different operator from `-` with two; tables use `minus` (or `uminus`), and an `arg2` left empty.

**`tables` prints nothing for a line.** It reads only the three assignment shapes, with blanks around every word: `t1=-c` is one word to `sscanf`. Write `t1 = - c`.

### In one line each

1. A quadruple is (op, arg1, arg2, result); temporaries are named, and instructions can be moved freely.
2. A triple is (op, arg1, arg2); a result is referred to as `(n)`, the number of the triple that computes it.
3. An assignment to a real name is a triple of its own: `= x (n)`.
4. Indirect triples add an instruction list of pointers to triples; reordering changes only the list.
5. `tables` turns three-address code into all three, to check hand work.

**Check yourself**

<!-- q: idea=quadruple type=apply wrong="a = b op c needs no temporary in quadruples" -->
1. Write the three-address code, quadruples and triples for `x = (a + b) * (a + b - c)`. Check with `tables`.

<details>
<summary>Answer</summary>

```text
t1 = a + b
t2 = a + b
t3 = t2 - c
t4 = t1 * t3
x = t4
```

The quadruples are those five lines as table rows; the triples are (0) `+ a b`, (1) `+ a b`, (2) `- (1) c`, (3) `* (0) (2)`, (4) `= x (3)`. The code computes `a + b` twice; a translator works on the tree, and the tree has two `+` nodes. Removing the second is section 10.3's job.

<!-- code: id=tables-check -->
```console
$ printf 't1 = a + b\nt2 = a + b\nt3 = t2 - c\nt4 = t1 * t3\nx = t4\n' | ./tables | sed -n '/^Triples/,/^$/p'
Triples
     op     arg1   arg2
(0)  +      a      b
(1)  +      a      b
(2)  -      (1)    c
(3)  *      (0)    (2)
(4)  =      x      (3)
```

</details>

<!-- q: idea=triple type=decide wrong="triples are always better because they are smaller" -->
2. A compiler's optimiser moves instructions often; its code generator only reads the instructions once, in order. Which representation would you choose for each, and why?

<details>
<summary>Answer</summary>

For the optimiser, quadruples (or indirect triples): moving or deleting an instruction leaves every other instruction's references valid. Plain triples would need every `(n)` after a change to be renumbered. For a code generator that only reads the final code once, plain triples are fine and save space; nothing moves any more. Real compilers mostly keep quadruple-like instructions throughout, because memory is cheap and editing is constant.

</details>

<!-- q: idea=indirect-triple type=apply wrong="reordering indirect triples renumbers the triples" -->
3. In the indirect triples of `a = b * -c + b * -c`, the optimiser wants triple (2) to run first, before (0). What changes, and what stays the same?

<details>
<summary>Answer</summary>

Only the instruction list changes: it becomes 35 → (2), 36 → (0), 37 → (1), 38 → (3), 39 → (4), 40 → (5). Every triple, and every `(n)` inside a triple, stays exactly as it was: (3) still multiplies `b` by `(2)`, wherever (2) runs. With plain triples, moving (2) to the front would renumber it (0) and shift (0) and (1) to (1) and (2), and every reference to them would change.

</details>

## 9.4 Generating three-address code for expressions with Yacc

### From postfix to instructions

Section 8.7's `tiny` printed `pos = init rate 60 * +`. Section 9.2 showed that the same order, with a temporary named for each operator, *is* three-address code: `t1 = rate * 60`, `t2 = init + t1`, `pos = t2`. This section makes the parser write that code. It is the fifth file of the mini compiler, and the lab question "generate three-address code for an arithmetic expression using Lex and Yacc" is answered by its first half.

The change from postfix is small but it matters. Postfix only had to print each operator after its operands; three-address code has to *say* which operands, by name. When the parser reduces `e * e`, the action must know where the two operands' values are, the name `rate` and the constant `60`, so that it can write `t1 = rate * 60`, and the `+` above it must then know that the multiplication's value is in `t1`. Every expression needs to carry the name of the place where its value will be. That is an attribute, and section 9.1 has the language for it.

### The SDD: an address for every expression

<!-- teaches: tac-generation = translation to three-address code, E.addr, newtemp, gen -->
The Dragon book's SDD for this job gives every `E` a synthesised attribute `E.addr`, the address (name, constant or temporary) that holds its value, and uses two helpers: `new Temp()` (here `newtemp()`), which makes a fresh temporary, and `gen(...)`, which emits one instruction.

| Production | Semantic rules |
|---|---|
| S -> id = E ; | gen(id.lexeme '=' E.addr) |
| E -> E1 + E2 | E.addr = newtemp(); gen(E.addr '=' E1.addr '+' E2.addr) |
| E -> E1 * E2 | E.addr = newtemp(); gen(E.addr '=' E1.addr '*' E2.addr) |
| E -> ( E1 ) | E.addr = E1.addr |
| E -> id | E.addr = id.lexeme |
| E -> num | E.addr = num.lexeme |

A name or a number needs no instruction: it is its own address. An operator makes a new temporary, emits one instruction that fills it from the two operands' addresses, and passes the temporary up as its own address. Parentheses pass the address through unchanged. The rules for `-` and `/` are those of `+` with the operator changed.
<!-- risk: the Dragon book's translation scheme for expressions (2nd ed. Fig. 6.19/6.20) uses E.addr, new Temp() and gen; check the attribution of the notation -->

There is a second way the paper writes it, with a second synthesised attribute, `E.code`, the instructions that compute `E`, built by joining strings: `E.code = E1.code || E2.code || gen(E.addr '=' E1.addr '+' E2.addr)`, where `||` joins code. The two say the same thing. The `code` form builds the code of each subtree and joins it at the parent; the form above *emits* each instruction as soon as its node is finished. Because an LR parser finishes the nodes in postorder, emitting at once puts every instruction exactly where the joined `code` would have put it, with no strings to build. That is why Yacc programs use the second form, and the paper accepts either.

Annotate the tree of `pos = init + rate * 60;` with it, bottom-up, as section 9.1 did with `val`:

| Node, in the order the parser finishes it | addr | Instruction emitted |
|---|---|---|
| E -> id (`init`) | `init` | |
| E -> id (`rate`) | `rate` | |
| E -> num (`60`) | `60` | |
| E -> E1 * E2 | `t1` | `t1 = rate * 60` |
| E -> E1 + E2 | `t2` | `t2 = init + t1` |
| S -> id = E ; | | `pos = t2` |

`init` is reduced first but needs no code; its address waits on the value stack until the `+` uses it, two reductions later.

### expr-tac.y

In Yacc, `E.addr` is `$$`, a string, so the value type is the `char *` of chapter 8's `%union`. The scanner is chapter 8's `tiny.l` unchanged: it already sends every name and number as a copy in `yylval.s`, which is exactly an `id.lexeme` or a `num.lexeme`. Flex can read it from its own folder; `lex.yy.c` is written in the current folder, where it finds this folder's `y.tab.h`. Save the parser as `expr-tac.y`:

<!-- code: id=expr-tac-write -->
```console
$ cat > expr-tac.y <<'END'
%{
/* expr-tac.y: Tiny assignments to three-address code; the scanner is
   chapter 8's tiny.l */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
int yylex(void);
void yyerror(const char *s);
extern int yylineno;
extern int errors;
int temps;                        /* temporaries used so far */

char *newtemp(void)               /* t1, t2, ...: a fresh name each time */
{
    char name[16];

    sprintf(name, "t%d", ++temps);
    return strdup(name);
}

char *gen(char *a, const char *op, char *b)   /* t = a op b; return t */
{
    char *t = newtemp();

    printf("%s = %s %s %s\n", t, a, op, b);
    free(a);
    free(b);
    return t;
}
%}
END
```

That is the prologue: the helpers, in C. The declarations and the rules follow, added with `>>`:

<!-- code: id=expr-tac-write2 -->
```console
$ cat >> expr-tac.y <<'END'
%union {
    char *s;                      /* an address: a name, a number or a temporary */
}
%token <s> ID NUM
%token IF WHILE ASSIGN EQ NE LT LE GT GE
%token PLUS MINUS TIMES DIVIDE LPAREN RPAREN SEMI
%type  <s> expr
%left PLUS MINUS
%left TIMES DIVIDE
%%
program : /* nothing */
        | program stmt
        ;
stmt    : ID ASSIGN expr SEMI      { printf("%s = %s\n", $1, $3); free($1); free($3); }
        ;
expr    : expr PLUS expr           { $$ = gen($1, "+", $3); }
        | expr MINUS expr          { $$ = gen($1, "-", $3); }
        | expr TIMES expr          { $$ = gen($1, "*", $3); }
        | expr DIVIDE expr         { $$ = gen($1, "/", $3); }
        | LPAREN expr RPAREN       { $$ = $2; }
        | ID
        | NUM
        ;
%%
void yyerror(const char *s)
{
    fflush(stdout);
    fprintf(stderr, "line %d: %s\n", yylineno, s);
    exit(1);
}

int main(void)
{
    yyparse();
    if (errors > 0) {
        fprintf(stderr, "%d lexical error(s)\n", errors);
        return 1;
    }
    return 0;
}
END
```

Hold it against the SDD, rule by rule.

**`newtemp`** keeps a counter and returns `t1`, `t2` ... as fresh strings, so that each can be freed like any other value. **`gen`** is the rule shared by the four operators: make a temporary, emit `t = a op b`, free the two operands' strings (they have been printed and are needed no more) and return the temporary as the new address.

**The productions.** Each operator's action is `$$ = gen($1, "op", $3)`: E.addr = newtemp() and gen(...) in one call. `( expr )` passes `$2` up. `ID` and `NUM` have no action, so the default `$$ = $1` passes the lexeme up as the address. `%type <s> expr` gives `expr` its type, as section 8.6 said a nonterminal whose value is used needs. The statement emits the final copy, `pos = t2`, from the name and the expression's address.

Everything else is `tiny.y`'s frame: every token of `tiny.l` declared, the error function that ends the half-written line, the count of lexical errors.

### Build and run

<!-- code: id=expr-tac-run -->
```console
$ yacc -d expr-tac.y
$ flex ../ch8/tiny.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o expr-tac
$ printf 'pos = init + rate * 60;\na = b * (c - d) / (c - d);\nx = 5;\n' | ./expr-tac
t1 = rate * 60
t2 = init + t1
pos = t2
t3 = c - d
t4 = b * t3
t5 = c - d
t6 = t4 / t5
a = t6
x = 5
```

The running statement gives the three instructions of section 9.2, `t1` and `t2` in postorder. The second statement shows what a translator working one node at a time does with a repeated subexpression: `c - d` is computed twice, into `t3` and `t5`, because the tree has two such nodes. And `/` comes last although it is written last, while `*` comes before it: they are on one level and `%left`, so `b * (c - d)` is finished first. A lone number needs no temporary: `x = 5` is a single copy. The temporaries are numbered through the whole program, not per statement, so no two statements ever share a name.

The empty input and the inputs that must be rejected:

<!-- code: id=expr-tac-errors exit=1 -->
```console
$ ./expr-tac < /dev/null
$ printf 'p = a * ;\n' | ./expr-tac; echo "exit $?"
line 1: syntax error
exit 1
$ printf 'q = 2 * (a + b;\n' | ./expr-tac
t1 = a + b
line 1: syntax error
```

The empty program has no code. In the last line the inner `a + b` was complete, and its instruction emitted, before the parser found the missing `)`: code is written as the parser goes, so a syntax error leaves the code that came before it. A real compiler throws that code away once it knows there was an error; for a lab program the message is enough.

### Fewer temporaries

Look at `t2` and the copy `pos = t2`: the addition could have written straight into `pos`. A translator can do that by giving the expression its target, an inherited attribute, so that the topmost operator writes `pos = init + t1` directly. Most textbook answers do not, because chapter 10's *copy propagation* removes such copies anyway, and the simple scheme here is what the paper expects. Two temporaries for two operators, and one copy, is the usual answer.

### When it goes wrong

**Every instruction reads `t1`.** `newtemp` returned the same buffer each time, a `static char name[16]` returned without `strdup`: every address points at one string, which ends up holding the last name made. Each temporary needs its own memory.

**Names come out as later tokens.** The `ID` alternative was written `{ $$ = yytext; }`, so the address points into Flex's buffer, which the lookahead overwrites: section 8.6's trap again, now inside the instructions. The scanner's copy in `yylval.s` is the address; leave `ID` and `NUM` with no action.

**Instructions in the wrong order.** An action that prints before its subtrees are done, in a mid-rule action for instance, writes the operator's instruction before its operands'. Emit in the action at the end of the production, when both operands are complete.

**`free` of an address used twice.** `( expr )` passes `$2` up instead of copying it, so the string is owned by the parent now; freeing it in the parentheses' action as well would free it twice. Each address is freed once, by the action that consumes it.

### In one line each

1. Every expression carries `E.addr`, a name, a constant or a temporary, as a synthesised attribute.
2. An operator's rule makes a new temporary, emits `t = a op b` and passes `t` up; a name or number is its own address.
3. Emitting at each reduction gives the same code as the `E.code` concatenation, because reductions come in postorder.
4. In Yacc, `E.addr` is `$$` of type `char *`, from `%union` and `%type <s> expr`.
5. A repeated subexpression is computed again; removing the repeat is chapter 10's work.

**Check yourself**

<!-- q: idea=tac-generation type=predict wrong="the instructions follow the source from left to right" -->
1. Predict the code `expr-tac` writes for `y = a - b - c * d;` and for `z = (a);`. Then check.

<details>
<summary>Answer</summary>

<!-- code: id=expr-tac-predict -->
```console
$ printf 'y = a - b - c * d;\nz = (a);\n' | ./expr-tac
t1 = a - b
t2 = c * d
t3 = t1 - t2
y = t3
z = a
```

`t1 = a - b` first, since `-` groups to the left and `a - b` is finished before the second `-` is shifted. Then `c * d`, `t2`, because the second `-` cannot be reduced until its right operand, `c * d`, is complete. Then `t3 = t1 - t2` and `y = t3`. `z = (a)` makes no temporary: the parentheses pass the address `a` through, and the statement is the copy `z = a`. The numbering goes on from the first statement, so this run's temporaries start at `t1` again only because it is a new run.

</details>

<!-- q: idea=tac-generation type=apply wrong="a temporary is needed for every node of the tree" -->
2. Write the annotated tree (only the `addr` of each `E`) and the code for `x = (a + b) * (a + b)` using the SDD above. How many temporaries, and why not fewer?

<details>
<summary>Answer</summary>

The left `( a + b )` is an E with addr `t1` (the parentheses copy the addr of the inner `+`, which emitted `t1 = a + b`); the right one emits `t2 = a + b` and has addr `t2`; the `*` emits `t3 = t1 * t2`; then `x = t3`. Three temporaries, one per operator node; names and parentheses make none. The SDD has no memory of what it emitted before, so it cannot see that `t2` equals `t1`. The DAG of section 10.3 does.

</details>

<!-- q: idea=tac-generation type=apply wrong="unary minus needs no temporary" -->
3. Add unary minus to `expr-tac.y`, so that `a = -b * c;` gives `t1 = minus b`, `t2 = t1 * c`, `a = t2`. Write the production, its action and the precedence line.

<details>
<summary>Answer</summary>

Add `%right UMINUS` as the last precedence line, and the production

```text
        | MINUS expr %prec UMINUS  { char *t = newtemp();
                                     printf("%s = minus %s\n", t, $2);
                                     free($2);
                                     $$ = t; }
```

The unary operator needs a temporary like any other operator: its value is new. `%prec UMINUS` makes it bind tighter than `*`, so `-b * c` is `(minus b) * c`, as section 8.3 arranged for the calculator.

</details>

## 9.5 Boolean expressions, if and while: labels and backpatching

### The loop in prog.tiny

Since chapter 2, `prog.tiny` has ended with a loop that no parser in this book has accepted:

```text
while (pos >= 100)
    pos = pos - 7.25 / (rate != 0);
```

`tiny-rd` stopped at it in section 6.1 and `tiny` in section 8.7, both with "syntax error" at the `while`. Tiny gets its `if` and `while` now, and with them the last two kinds of three-address instruction an expression never needed: labels and jumps. The first half of this section builds `tiny-tac.y`, which translates whole Tiny programs, using labels made as it parses. The second half answers a harder question that the paper asks every year: how does a one-pass translator jump to an instruction it has not written yet? The answer is called *backpatching*.

### The shape of the code

<!-- teaches: control-flow-tac = code layout for if and while, label, jumping code -->
Section 9.2 laid out a `while` loop by hand. The three statements have one layout each, and every translator, by hand or by program, follows these:

| Statement | Three-address code |
|---|---|
| `if (B) S1` | code for B; `ifFalse B goto Lnext`; code for S1; `Lnext:` |
| `if (B) S1 else S2` | code for B; `ifFalse B goto Lelse`; code for S1; `goto Lend`; `Lelse:` code for S2; `Lend:` |
| `while (B) S` | `Lbegin:` code for B; `ifFalse B goto Lend`; code for S; `goto Lbegin`; `Lend:` |

In the `while`, the label `Lbegin` comes *before* the code for `B`: the test's own instructions, such as `t1 = a + b` for `while (a + b < c)`, must run again before every pass, so the jump back must land on them, not on the `ifFalse`. In the `if ... else`, the `goto Lend` at the end of S1 is what keeps the program from falling into S2.

Each layout needs one or two labels that do not exist yet when the statement starts. In a translator that emits code as it parses, the label names can be made up front: when the parser reaches `while`, it takes a fresh label number and prints `L1:` at once; when it has read the condition, it takes another for `Lend` and prints the `ifFalse` that jumps to it, although `L2:` itself will only be printed after the body. A name can be used before the line it names is written. Making labels up front only needs a counter, like the temporaries'.

### Labels in Yacc: values of mid-rule actions

The labels must be printed at points *inside* a production: `L1:` before the condition, `ifFalse` between the condition and the body. Section 8.6 printed in the middle of a production with a mid-rule action. Here the action must also *remember* its label number for the end of the production, and a mid-rule action can have a value, `$<n>$`, which later actions read by its position, `$<n>2`. In

```text
stmt : WHILE                { $<n>$ = ++labels; printf("L%d:\n", $<n>$); }
       LPAREN expr RPAREN   { $<n>$ = ++labels; printf("ifFalse %s goto L%d\n", $4, $<n>$); }
       stmt                 { printf("goto L%d\nL%d:\n", $<n>2, $<n>6); }
```

the symbols are numbered `WHILE` 1, first action 2, `LPAREN` 3, `expr` 4, `RPAREN` 5, second action 6, `stmt` 7. The first action takes label 1 (`Lbegin`) and prints it; the second takes label 2 (`Lend`) and prints the jump to it; the last prints the jump back to `$<n>2` and then label `$<n>6`. `<n>` names the `int` member of the union, because Bison cannot know the type of a mid-rule action's value.

For `if`, the `else` is optional. Section 8.5 settled the dangling `else` with `%prec LOWER_THAN_ELSE`, and `tiny-tac.y` does the same. It also uses `$<n>0` from section 9.1 so that the two endings of an `if` can find the label of the `ifFalse`, which is the plate just below them.

Tiny also needs comparisons. A comparison may decide a jump (`while (pos >= 100)`), or be a value in arithmetic (`7.25 / (rate != 0)`, where `rate != 0` is 1 or 0). The translator keeps a comparison as *text*, `"pos >= 100"`, until it knows which: a jump uses the text directly, in `ifFalse pos >= 100 goto L2`, and an operator that needs it as a value first stores it in a temporary, `t3 = rate != 0`. The text has blanks in it and an address never does, which is how the program tells the two apart.

### tiny.l, with else and braces

Tiny's statements need three more tokens: `else`, and `{` and `}` to make a block of several statements the body of a loop. `sed` copies chapter 8's `tiny.l` with three rules added, after the `while` rule and after the `;` rule:

<!-- code: id=tiny-l9 exit=1 -->
```console
$ sed -e 's/^"while" .*/&\n"else"        { return ELSE; }/' -e 's/^";" .*/&\n"{"           { return LBRACE; }\n"}"           { return RBRACE; }/' ../ch8/tiny.l > tiny.l
$ diff ../ch8/tiny.l tiny.l
24a25
> "else"        { return ELSE; }
43a45,46
> "{"           { return LBRACE; }
> "}"           { return RBRACE; }
```

`diff` shows the three new lines and that nothing else changed; it exits with status 1 because the files differ, which here is the point.

### tiny-tac.y

The parser comes in three pieces. First the helpers, in C:

<!-- code: id=tiny-tac-1 -->
```console
$ cat > tiny-tac.y <<'END'
%{
/* tiny-tac.y: Tiny to three-address code; the scanner is tiny.l */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
int yylex(void);
void yyerror(const char *s);
extern int yylineno;
extern int errors;

int temps, labels;              /* t1, t2 ... and L1, L2 ... used so far */

char *newtemp(void)             /* a fresh temporary's name */
{
    char buf[16];

    sprintf(buf, "t%d", ++temps);
    return strdup(buf);
}

char *value(char *a)            /* a comparison's text gets a temporary */
{
    char *t;

    if (strchr(a, ' ') == NULL) /* a name, a number or a temporary */
        return a;
    t = newtemp();
    printf("%s = %s\n", t, a);
    free(a);
    return t;
}

char *binop(char *a, const char *op, char *b)  /* emit t = a op b; return t */
{
    char *t;

    a = value(a);
    b = value(b);
    t = newtemp();
    printf("%s = %s %s %s\n", t, a, op, b);
    free(a);
    free(b);
    return t;
}

char *relop(char *a, const char *op, char *b)  /* the text "a op b" */
{
    char *c;

    a = value(a);
    b = value(b);
    c = malloc(strlen(a) + strlen(op) + strlen(b) + 3);
    sprintf(c, "%s %s %s", a, op, b);
    free(a);
    free(b);
    return c;
}
%}
END
```

`newtemp` and `binop` are `expr-tac.y`'s `newtemp` and `gen`, with one addition: `binop` passes both operands through `value`, which leaves an address alone and turns a comparison's text into a temporary. `relop` builds the text of a comparison, after making sure its own operands are addresses (so `a + b < c` becomes `t1 = a + b` and the text `t1 < c`).

Then the declarations and the statements:

<!-- code: id=tiny-tac-2 -->
```console
$ cat >> tiny-tac.y <<'END'
%union {
    char *s;                    /* an address, or a comparison's text */
    int n;                      /* a label's number */
}
%token <s> ID NUM
%token IF ELSE WHILE ASSIGN EQ NE LT LE GT GE
%token PLUS MINUS TIMES DIVIDE LPAREN RPAREN SEMI LBRACE RBRACE
%type  <s> expr
%nonassoc LOWER_THAN_ELSE
%nonassoc ELSE
%nonassoc EQ NE LT LE GT GE
%left PLUS MINUS
%left TIMES DIVIDE
%%
program : /* nothing */
        | program stmt
        ;
stmt    : ID ASSIGN expr SEMI      { $3 = value($3);
                                     printf("%s = %s\n", $1, $3);
                                     free($1);
                                     free($3); }
        | LBRACE stmts RBRACE
        | IF LPAREN expr RPAREN    { $<n>$ = ++labels;
                                     printf("ifFalse %s goto L%d\n", $3, $<n>$);
                                     free($3); }
          else_part
        | WHILE                    { $<n>$ = ++labels; printf("L%d:\n", $<n>$); }
          LPAREN expr RPAREN       { $<n>$ = ++labels;
                                     printf("ifFalse %s goto L%d\n", $4, $<n>$);
                                     free($4); }
          stmt                     { printf("goto L%d\nL%d:\n", $<n>2, $<n>6); }
        ;
else_part : stmt %prec LOWER_THAN_ELSE
                                   { printf("L%d:\n", $<n>0); }
        | stmt ELSE                { $<n>$ = ++labels;
                                     printf("goto L%d\nL%d:\n", $<n>$, $<n>0); }
          stmt                     { printf("L%d:\n", $<n>3); }
        ;
stmts   : /* nothing */
        | stmts stmt
        ;
END
```

**The union** has a second member, `n`, for label numbers. **The precedence lines** put the comparisons below `+` and `-`, so that `a + b < c` compares a sum, and make them `%nonassoc`, so that `a < b < c` is a syntax error rather than a puzzle.

**Assignment** passes the expression through `value`, so `f = a < b;` stores the comparison in a temporary first.

**`if`.** After the condition, a mid-rule action takes a label and prints the `ifFalse`. Then `else_part`, which is either a statement alone, or a statement, `else` and another. With no `else`, the end of the statement is the place the `ifFalse` jumps to, so it prints that label, `$<n>0`, the value of the mid-rule action just below `else_part` on the stack. With an `else`, it takes a second label for the end, prints `goto` to it and then the `ifFalse`'s label, which is where the `else` branch starts; after the second statement it prints the end label, `$<n>3`.

**`while`** is the layout above, as the three actions you read in the last part.

**A block**, `{ stmts }`, needs no action: its statements print their own code in order.

Last, the expressions and the user code:

<!-- code: id=tiny-tac-3 -->
```console
$ cat >> tiny-tac.y <<'END'
expr    : expr EQ expr             { $$ = relop($1, "==", $3); }
        | expr NE expr             { $$ = relop($1, "!=", $3); }
        | expr LT expr             { $$ = relop($1, "<", $3); }
        | expr LE expr             { $$ = relop($1, "<=", $3); }
        | expr GT expr             { $$ = relop($1, ">", $3); }
        | expr GE expr             { $$ = relop($1, ">=", $3); }
        | expr PLUS expr           { $$ = binop($1, "+", $3); }
        | expr MINUS expr          { $$ = binop($1, "-", $3); }
        | expr TIMES expr          { $$ = binop($1, "*", $3); }
        | expr DIVIDE expr         { $$ = binop($1, "/", $3); }
        | LPAREN expr RPAREN       { $$ = $2; }
        | ID
        | NUM
        ;
%%
void yyerror(const char *s)
{
    fflush(stdout);
    fprintf(stderr, "line %d: %s\n", yylineno, s);
    exit(1);
}

int main(void)
{
    yyparse();
    if (errors > 0) {
        fprintf(stderr, "%d lexical error(s)\n", errors);
        return 1;
    }
    return 0;
}
END
$ yacc -d tiny-tac.y
$ flex tiny.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o tiny-tac
```

`yacc` builds it without a conflict: the one shift-reduce conflict of the `if` is settled by the two `%nonassoc` lines, as in `dangle2.y`.

### prog.tiny, translated at last

<!-- code: id=tiny-tac-prog -->
```console
$ ./tiny-tac < ../ch2/prog.tiny
init = 10
rate = 2.5
t1 = rate * 60
t2 = init + t1
pos = t2
L1:
ifFalse pos >= 100 goto L2
t3 = rate != 0
t4 = 7.25 / t3
t5 = pos - t4
pos = t5
goto L1
L2:
```

The first three statements are `expr-tac`'s. Then the loop in the `while` layout: `L1:` before the test, the test as a jump on the comparison's text, the body, and the jump back to `L1`. Inside the body, `rate != 0` is a value, so it got `t3` before the division used it. The program that every parser in the book had refused is translated whole.

An `if` with an `else`, nested, with a block:

<!-- code: id=tiny-tac-if -->
```console
$ printf 'if (a < b) { x = 1; } else { if (c) x = 2; y = 3; }\nf = a < b;\n' | ./tiny-tac
ifFalse a < b goto L1
x = 1
goto L2
L1:
ifFalse c goto L3
x = 2
L3:
y = 3
L2:
t1 = a < b
f = t1
```

The outer `if` has an `else`: `ifFalse` to `L1`, the `then` part, `goto L2` over the `else` part, `L1:` and the `else` part, `L2:`. Inside the `else` block, the inner `if (c)` has none: its `ifFalse` jumps to `L3`, printed right after `x = 2`, and `y = 3` follows. A condition that is a plain name, `c`, is tested as it is. The last line stores a comparison, which takes a temporary.

### Boolean operators and the jump to nowhere

<!-- teaches: backpatching = backpatching, truelist, falselist, short-circuit code -->
Tiny has no `&&` or `||`, but C has, and so does the paper: "Using backpatching, generate code for `a < b || c < d && e < f`." These operators are evaluated by *jumping*: in `B1 || B2`, if `B1` is true, the whole is true and `B2` is never computed. So the code for a condition is not a value but a set of jumps: some go to wherever "true" leads, the others to wherever "false" leads. This is *short-circuit*, or *jumping*, code.

The trouble is the targets. When the translator emits `if a < b goto _` for the first comparison, where should it jump? To the code for "true", which is the `then` part of an `if`, or the body of a `while`, or, inside `||`, the end of the whole condition; none of it has been written yet, and the instruction number it will get is unknown. Making labels up front, as `tiny-tac` does, works for statements, but inside a boolean expression there are many jumps and it is not known which label each one needs until the operators above it are reduced.

*Backpatching* solves it by leaving holes. Each jump whose target is unknown is emitted with an empty target, written `_`, and its instruction number is put on a list. Every boolean expression B carries two such lists as synthesised attributes:

- `B.truelist`: the jumps, still unfilled, to be taken when B is true;
- `B.falselist`: those to be taken when B is false.

When the translator later learns where one of these lists must go, it *backpatches*: it fills in the target of every instruction on the list. Three helpers do the work: `makelist(i)` makes a list holding instruction i; `merge(p1, p2)` joins two lists; `backpatch(p, i)` fills the target i into every instruction on p. Instructions are numbered (from 100 here, as in the Dragon book), since a hole is filled with an instruction number. The rules, with `nextinstr` the number of the next instruction to be emitted:

| Production | Semantic rules |
|---|---|
| B -> E1 rel E2 | B.truelist = makelist(nextinstr); B.falselist = makelist(nextinstr + 1); emit `if E1 rel E2 goto _`; emit `goto _` |
| B -> B1 \|\| M B2 | backpatch(B1.falselist, M.instr); B.truelist = merge(B1.truelist, B2.truelist); B.falselist = B2.falselist |
| B -> B1 && M B2 | backpatch(B1.truelist, M.instr); B.truelist = B2.truelist; B.falselist = merge(B1.falselist, B2.falselist) |
| B -> ! B1 | B.truelist = B1.falselist; B.falselist = B1.truelist |
| B -> ( B1 ) | B.truelist = B1.truelist; B.falselist = B1.falselist |
| B -> true | B.truelist = makelist(nextinstr); emit `goto _` |
| B -> false | B.falselist = makelist(nextinstr); emit `goto _` |
| M -> ε | M.instr = nextinstr |
<!-- risk: the backpatching translation scheme as in Dragon 2nd ed. Fig. 6.43; check each rule against the book -->

`M` is an empty nonterminal whose only job is to remember where `B2`'s code starts, `nextinstr` at the moment `M` is reduced. In `B1 || M B2`, if `B1` is false the program must try `B2`, so `B1`'s false jumps are filled with `M.instr`; if `B1` is true, the whole is true, so `B1`'s true jumps join `B2`'s on the result's truelist, still unfilled. `&&` is the mirror: `B1` true means "go on to `B2`". `!` swaps the lists and emits nothing.

### bool.y: backpatching in Yacc

`bool.y` reads one boolean expression whose comparisons are between names or numbers, emits its jumping code with backpatching, and prints the instructions, the holes still open, and the two lists left at the top. Its scanner:

<!-- code: id=bool-l -->
```console
$ cat > bool.l <<'END'
%{
/* bool.l: the scanner for bool.y */
#include <string.h>
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
"||"                        { return OR; }
"&&"                        { return AND; }
"<="|">="|"=="|"!="|"<"|">" { yylval.s = strdup(yytext); return RELOP; }
"!"                         { return NOT; }
true                        { return TRUE; }
false                       { return FALSE; }
[A-Za-z_][A-Za-z0-9_]*      { yylval.s = strdup(yytext); return ID; }
[0-9]+                      { yylval.s = strdup(yytext); return ID; }
[ \t\n]+                    { /* blanks */ }
.                           { return yytext[0]; }
%%
END
```

The helpers, in the prologue:

<!-- code: id=bool-1 -->
```console
$ cat > bool.y <<'END'
%{
/* bool.y: jumping code for a boolean expression, by backpatching.
   The instructions are numbered from 100; _ is a jump not yet filled in. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
int yylex(void);
void yyerror(const char *s);

#define FIRST 100                  /* the number of the first instruction */
struct instr {
    char text[40];                 /* "if a < b goto" or "goto" */
    int target;                    /* where it jumps; -1 while unknown */
} code[200];
int nextinstr = FIRST;             /* the number the next instruction gets */

struct list {                      /* instructions with a hole in them */
    int n, at[50];
};

void emit(const char *text)
{
    strcpy(code[nextinstr - FIRST].text, text);
    code[nextinstr - FIRST].target = -1;
    nextinstr++;
}

struct list *makelist(int i)       /* a list holding instruction i only */
{
    struct list *l = malloc(sizeof *l);
    l->n = 1;
    l->at[0] = i;
    return l;
}

struct list *emptylist(void)       /* a list with no holes */
{
    struct list *l = malloc(sizeof *l);
    l->n = 0;
    return l;
}

struct list *merge(struct list *a, struct list *b)
{
    for (int i = 0; i < b->n; i++)
        a->at[a->n++] = b->at[i];
    free(b);
    return a;
}

void backpatch(struct list *l, int target)   /* fill each hole in l */
{
    for (int i = 0; i < l->n; i++)
        code[l->at[i] - FIRST].target = target;
    free(l);
}
END
```

and the grammar, which is the table above:

<!-- code: id=bool-2 -->
```console
$ cat >> bool.y <<'END'
void show(const char *name, struct list *l)
{
    printf("%s:", name);
    for (int i = 0; i < l->n; i++)
        printf(" %d", l->at[i]);
    printf("\n");
}
%}
%union {
    char *s;                       /* a name, a number or an operator */
    int i;                         /* M: an instruction number */
    struct {
        struct list *t, *f;        /* B.truelist and B.falselist */
    } b;
}
%token <s> ID RELOP
%token OR AND NOT TRUE FALSE
%type  <b> b
%type  <i> m
%left OR
%left AND
%right NOT
%%
END
```

<!-- code: id=bool-2b -->
```console
$ cat >> bool.y <<'END'
line : b                 { for (int i = 0; i < nextinstr - FIRST; i++) {
                               printf("%d: %s ", FIRST + i, code[i].text);
                               if (code[i].target < 0)
                                   printf("_\n");
                               else
                                   printf("%d\n", code[i].target);
                           }
                           show("truelist", $1.t);
                           show("falselist", $1.f);
                         }
     ;
b    : b OR m b          { backpatch($1.f, $3);
                           $$.t = merge($1.t, $4.t);
                           $$.f = $4.f; }
     | b AND m b         { backpatch($1.t, $3);
                           $$.t = $4.t;
                           $$.f = merge($1.f, $4.f); }
     | NOT b             { $$.t = $2.f; $$.f = $2.t; }
     | '(' b ')'         { $$ = $2; }
     | ID RELOP ID       { char text[40];
                           $$.t = makelist(nextinstr);
                           $$.f = makelist(nextinstr + 1);
                           snprintf(text, sizeof text, "if %s %s %s goto", $1, $2, $3);
                           emit(text);
                           emit("goto");
                           free($1); free($2); free($3); }
     | TRUE              { $$.t = makelist(nextinstr);
                           $$.f = emptylist();
                           emit("goto"); }
     | FALSE             { $$.t = emptylist();
                           $$.f = makelist(nextinstr);
                           emit("goto"); }
     ;
m    : /* nothing */     { $$ = nextinstr; }
     ;
END
```

<!-- code: id=bool-2c -->
```console
$ cat >> bool.y <<'END'
%%
void yyerror(const char *s)
{
    fprintf(stderr, "%s\n", s);
}

int main(void)
{
    return yyparse();
}
END
```

<!-- code: id=bool-build -->
```console
$ yacc -d bool.y
$ flex bool.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o bool
```

The instructions live in an array, `code`, each with its text and its target, -1 while it is a hole. A list is a small array of instruction numbers. The value of a `b` is a pair of lists, a C `struct` inside the union, read in the actions as `$1.t` and `$1.f`. `m` is the marker `M`, with an `int` value. Everything else is the table, one production per row.

### The paper's question, worked

<!-- code: id=bool-run -->
```console
$ echo 'a < b || c < d && e < f' | ./bool
100: if a < b goto _
101: goto 102
102: if c < d goto 104
103: goto _
104: if e < f goto _
105: goto _
truelist: 100 104
falselist: 103 105
```

Follow it reduction by reduction. `&&` binds tighter than `||`, so the tree is `a < b || (c < d && e < f)`.

1. `a < b` is reduced: instructions 100 (`if a < b goto _`) and 101 (`goto _`); truelist {100}, falselist {101}.
2. `||` is shifted, then `M` is reduced with nothing read: M.instr = 102.
3. `c < d`: instructions 102 and 103; truelist {102}, falselist {103}.
4. `&&` is shifted, then the second `M`: M.instr = 104.
5. `e < f`: instructions 104 and 105; truelist {104}, falselist {105}.
6. Reduce `B1 && M B2` (`c < d && e < f`): backpatch B1's truelist {102} with 104: if `c < d`, go on to test `e < f`. The result's truelist is {104}, its falselist the merge {103, 105}.
7. Reduce `B1 || M B2` (the whole): backpatch B1's falselist {101} with 102: if `a < b` is false, try the right side. Truelist: merge {100} and {104}; falselist {103, 105}.

Three jumps were filled while parsing, and four holes are left: 100 and 104 go wherever "true" leads, 103 and 105 wherever "false" leads. Those are filled by the statement that uses the expression: an `if` backpatches the truelist with the start of its `then` part and the falselist with what follows. That is the Dragon book's own answer to this question, instruction for instruction.
<!-- risk: claim that this output matches the Dragon book's worked example (2nd ed. Fig. 6.47) for x < 100 || x > 200 && x != y; check -->

### When it goes wrong

**The jump back of a `while` goes to the `ifFalse`.** `L1:` printed after the condition's own instructions skips them on every pass after the first: the loop tests a stale value. The label goes before the condition's code.

**A missing `goto Lend` in `if ... else`.** The `then` part falls through into the `else` part. Every `then` part followed by an `else` ends in a jump over it.

**Mid-rule values read with the wrong number.** A mid-rule action counts as a symbol: in the `while` production the second action is `$6`, not `$5`. A wrong number reads the `RPAREN`'s value, which is garbage, and prints a label such as `L0` or a large number.

**Backpatching the wrong list.** In `B1 || M B2`, it is B1's *false*list that gets M.instr; in `&&` it is the *true*list. Ask: if B1 is false, does `||` need to look at B2? Yes, so B1's false jumps go to B2.

**Forgetting `M`.** Without the marker, the translator does not know where B2's code starts: by the time `B1 || B2` is reduced, `nextinstr` is past B2.

### In one line each

1. `if` is `ifFalse B goto Lnext`, S, `Lnext:`; `if ... else` adds `goto Lend` and `Lelse:`; `while` puts `Lbegin:` before the test and `goto Lbegin` after the body.
2. A translator that knows the layout makes its labels up front and keeps their numbers in mid-rule action values, `$<n>$`.
3. `&&`, `||` and `!` compile to jumping code: the value of a condition is where the program goes.
4. Backpatching emits jumps with empty targets, keeps them on `truelist` and `falselist`, and fills them when the target is known.
5. The marker `M -> ε` records where the second operand's code starts.

**Check yourself**

<!-- q: idea=control-flow-tac type=apply wrong="the jump back goes to the ifFalse line" -->
1. Write by hand the code `tiny-tac` gives for `while (i < n) { s = s + i * i; i = i + 1; } if (s > 100) s = 100;`, then check.

<details>
<summary>Answer</summary>

<!-- code: id=tiny-tac-check -->
```console
$ printf 'while (i < n) { s = s + i * i; i = i + 1; }\nif (s > 100) s = 100;\n' | ./tiny-tac
L1:
ifFalse i < n goto L2
t1 = i * i
t2 = s + t1
s = t2
t3 = i + 1
i = t3
goto L1
L2:
ifFalse s > 100 goto L3
s = 100
L3:
```

`L1:`, `ifFalse i < n goto L2`, `t1 = i * i`, `t2 = s + t1`, `s = t2`, `t3 = i + 1`, `i = t3`, `goto L1`, `L2:`, then `ifFalse s > 100 goto L3`, `s = 100`, `L3:`. The label numbers continue from the loop to the `if`, as the temporaries do.

</details>

<!-- q: idea=backpatching type=apply wrong="every jump is filled in when the expression is parsed" -->
2. Using backpatching, write the jumping code for `x < 100 || x > 200 && x != y`, numbered from 100, and give the truelist and falselist of the whole expression. Check with `bool`.

<details>
<summary>Answer</summary>

It has the same shape as the worked example, `B1 || (B2 && B3)`, so the same numbers come out with different comparisons:

```text
100: if x < 100 goto _
101: goto 102
102: if x > 200 goto 104
103: goto _
104: if x != y goto _
105: goto _
```

truelist {100, 104}, falselist {103, 105}.

<!-- code: id=bool-check -->
```console
$ echo 'x < 100 || x > 200 && x != y' | ./bool
100: if x < 100 goto _
101: goto 102
102: if x > 200 goto 104
103: goto _
104: if x != y goto _
105: goto _
truelist: 100 104
falselist: 103 105
```

</details>

<!-- q: idea=backpatching type=predict wrong="! emits a new instruction" -->
3. Predict `bool`'s output for `!(a < b) && c == d`: how many instructions, which holes are filled, and the two lists. Then run it.

<details>
<summary>Answer</summary>

Four instructions. `a < b` emits 100 and 101 with truelist {100}, falselist {101}; `!` swaps them, emitting nothing: truelist {101}, falselist {100}. M = 102. `c == d` emits 102 and 103. The `&&` backpatches the left side's truelist, {101}, with 102: if `a < b` is false, test `c == d`. Result: truelist {102}, falselist {100, 103}. So `100: if a < b goto _` is now a jump to *false*: exactly what `!` means.

<!-- code: id=bool-not -->
```console
$ echo '!(a < b) && c == d' | ./bool
100: if a < b goto _
101: goto 102
102: if c == d goto _
103: goto _
truelist: 102
falselist: 100 103
```

</details>

## 9.6 Type checking and type conversion

### Where inttofloat went

Chapter 1's intermediate code started with `t1 = inttofloat(60)`, and `expr-tac` writes `t1 = rate * 60` with no conversion. Tiny has one kind of number, so it never needed one; C has `int` and `float` and `double`, and the machine has different instructions for each: an integer add and a floating-point add are different operations on different registers, and adding an `int`'s bits to a `float`'s bits gives nonsense. So before the translator can write `t2 = rate * t1`, it must know the *type* of each operand, check that the operator applies to them, and insert a conversion where they differ. That is *type checking*, the last job of the semantic analysis phase of section 1.2.

### Types as one more synthesised attribute

<!-- teaches: type-checking = type checking, type expression, coercion, widening -->
Give every `E` a second synthesised attribute, `E.type`, beside `E.addr`. A name's type comes from the symbol table, filled by declarations such as section 9.1's `addtype`; a constant's from its form (`60` is `integer`, `2.5` is `float`). An operator's rule checks the two types and decides the result's:

| Production | Semantic rules |
|---|---|
| E -> E1 + E2 | if E1.type = integer and E2.type = integer: E.type = integer; gen(E.addr '=' E1.addr '+' E2.addr) |
| | else if one is integer and the other float: convert the integer one, `t = inttofloat(x)`; E.type = float; gen the float `+` |
| | else if both are float: E.type = float; gen the float `+` |
| | else: E.type = type_error; report "operands of + have incompatible types" |

The conversion is a *coercion*: the compiler inserts it without being asked. Converting an `int` to a `float` is a *widening*: every `int` value has a `float` value close to it, so nothing is lost by the conversion that is not lost anyway in floating point. The reverse, `float` to `int`, drops the fraction, and C does it silently on assignment while many languages refuse.
<!-- risk: claim that int to float is a widening conversion with no loss "beyond floating point's own"; large ints above 2^24 lose precision in a 32-bit float; check wording against Dragon 6.5.2 -->

Run the rules on `pos = init + rate * 60;` with `pos`, `init` and `rate` declared `float`:

| Node | type | Instruction |
|---|---|---|
| `rate` | float | |
| `60` | integer | |
| `rate * 60` | float, after converting the 60 | `t1 = inttofloat(60)`, `t2 = rate * t1` |
| `init + ...` | float | `t3 = init + t2` |
| `pos = ...` | float = float: no conversion | `pos = t3` |

That is chapter 1's code exactly, four instructions with `inttofloat` first: the conversion is emitted where the types meet, at the `*`, before the multiplication that needs it. Had `pos` been an `int`, the assignment would need `t4 = floattoint(t3)` (or, in a strict language, an error).

### When it goes wrong

**The conversion in the wrong place.** Writing `t3 = init + t2` and then converting the *result* is wrong: by then a float multiply has been done on an integer's bits. The conversion goes to the operand, before the operator that needs it.

**A type error that still emits code.** When the rule finds `type_error`, it must report it and not pretend: the compiler stops producing object code, though it may go on checking, to report more errors.

**Check yourself**

<!-- q: idea=type-checking type=apply wrong="the conversion goes after the operator" -->
1. With `int i, j;` and `float x;`, write the three-address code, with conversions, for `x = i / j + x;`. Then say what C prints for `x` when `i` is 7, `j` is 2 and `x` is 0, and why it is not 3.5.

<details>
<summary>Answer</summary>

```text
t1 = i / j
t2 = inttofloat(t1)
t3 = t2 + x
x = t3
```

`i / j` has two integer operands, so it is an *integer* division with an integer result, 3; only the `+` meets a float, so the conversion comes after the division, and `x` becomes 3.0. The types decide at each node, from that node's operands alone: no rule looks ahead to the float on the far side of the `+`. To get 3.5, a conversion must be asked for before the division, `(float)i / j`.

</details>

## 9.7 Practice: from source to three-address code for a new program

You can now write a syntax-directed definition and annotate a parse tree with synthesised and inherited attributes (9.1), write three-address code by hand (9.2), turn it into quadruples, triples and indirect triples (9.3), make Yacc generate it for expressions (9.4) and for `if` and `while`, with backpatching for boolean operators (9.5), and insert type conversions (9.6). The written paper's intermediate-code question usually takes a short program or expression through several of these at once; the lab asks for the generator. These tasks climb from hand work checked by the chapter's programs to a new program that *runs* three-address code, so that you can test any translation, yours or a compiler's, by its result. Work in `~/cdlab/ch9`; do each hand answer on paper first, then check it with `tables`, `bool` and `tiny-tac`.

### Guided

<!-- q: idea=synthesised-attribute type=apply wrong="the leftmost bit is added last" -->
**1. An SDD for binary numbers.** The grammar `N -> L`, `L -> L1 B | B`, `B -> 0 | 1` describes binary numerals such as `1011`. Write an SDD with a synthesised attribute `val` that gives the numeral's value in decimal, and draw the annotated parse tree for `1011`.

<details>
<summary>Answer</summary>

| Production | Semantic rule |
|---|---|
| N -> L | N.val = L.val |
| L -> L1 B | L.val = 2 × L1.val + B.val |
| L -> B | L.val = B.val |
| B -> 0 | B.val = 0 |
| B -> 1 | B.val = 1 |

The list is left-recursive, so the innermost `L` holds the leftmost bit, and each step up shifts the value left by one bit (× 2) and adds the next bit. For `1011`, from the bottom: L = 1 (the first `1`); L = 2 × 1 + 0 = 2; L = 2 × 2 + 1 = 5; L = 2 × 5 + 1 = 11 at the top; N.val = 11. The tempting mistake is to add each bit times its power of two, which needs to know how many bits come *after* it, an attribute that this grammar does not supply bottom-up. Doubling at each step needs no such count.

</details>

<!-- q: idea=triple type=apply wrong="unary minus takes two arguments" -->
**2. Three tables for a new expression.** Write the three-address code, the quadruples and the triples for `x = -(a + b) * (c - d) + (a + b)`. Check with `tables`.

<details>
<summary>Answer</summary>

In postorder, with a temporary per operator:

```text
t1 = a + b
t2 = - t1
t3 = c - d
t4 = t2 * t3
t5 = a + b
t6 = t4 + t5
x = t6
```

<!-- code: id=practice-tables -->
```console
$ printf 't1 = a + b\nt2 = - t1\nt3 = c - d\nt4 = t2 * t3\nt5 = a + b\nt6 = t4 + t5\nx = t6\n' | ./tables
Quadruples
     op     arg1   arg2   result
(0)  +      a      b      t1
(1)  minus  t1            t2
(2)  -      c      d      t3
(3)  *      t2     t3     t4
(4)  +      a      b      t5
(5)  +      t4     t5     t6
(6)  =      t6            x

Triples
     op     arg1   arg2
(0)  +      a      b
(1)  minus  (0)
(2)  -      c      d
(3)  *      (1)    (2)
(4)  +      a      b
(5)  +      (3)    (4)
(6)  =      x      (5)

Indirect triples
instruction  triple
35           (0)
36           (1)
37           (2)
38           (3)
39           (4)
40           (5)
41           (6)
```

The unary minus is (1) `minus (0)`, with one argument; the subtraction `c - d` is an ordinary binary `-`. `a + b` appears twice, as (0) and (4); the translator has no reason to notice, and section 10.3's DAG will join them.

</details>

### On your own

<!-- q: idea=backpatching type=apply wrong="the parentheses change which list is backpatched" -->
**3. Backpatching a new condition.** Using backpatching, write the numbered jumping code for `(a < b && c > d) || !(e == f)`, starting at 100, with the truelist and falselist of the whole. Say which jumps were filled during the parse and why. Check with `bool`.

<details>
<summary>Answer</summary>

<!-- code: id=practice-bool -->
```console
$ echo '(a < b && c > d) || !(e == f)' | ./bool
100: if a < b goto 102
101: goto 104
102: if c > d goto _
103: goto 104
104: if e == f goto _
105: goto _
truelist: 102 105
falselist: 104
```

`a < b` emits 100 and 101; M = 102; `c > d` emits 102 and 103. The `&&` backpatches `a < b`'s truelist {100} with 102. The parentheses pass the lists through. M = 104; `e == f` emits 104 and 105, with truelist {104} and falselist {105}; the `!` swaps them: truelist {105}, falselist {104}. The `||` backpatches the left side's falselist, {101, 103}, with 104: if the `&&` part is false, try the right side. Result: truelist {102, 105}, falselist {104}. Three instructions were filled (100, 101, 103); 102, 104 and 105 wait for the statement that uses the condition.

</details>

<!-- q: idea=control-flow-tac type=apply unseen="gcd-tiny" wrong="the else branch needs no jump over it" -->
**4. A whole program by hand.** Write the three-address code, with labels, for this Tiny program, which computes the greatest common divisor of 48 and 18 by repeated subtraction. Then save it as `gcd.tiny` and check your answer with `tiny-tac`.

```c
a = 48;
b = 18;
while (a != b) {
    if (a > b) a = a - b;
    else b = b - a;
}
```

<details>
<summary>Answer</summary>

<!-- code: id=gcd -->
```console
$ cat > gcd.tiny <<'END'
a = 48;
b = 18;
while (a != b) {
    if (a > b) a = a - b;
    else b = b - a;
}
END
$ ./tiny-tac < gcd.tiny
a = 48
b = 18
L1:
ifFalse a != b goto L2
ifFalse a > b goto L3
t1 = a - b
a = t1
goto L4
L3:
t2 = b - a
b = t2
L4:
goto L1
L2:
```

The `while` takes `L1` (its start) and `L2` (its end); the `if` inside takes `L3` (the start of the `else` part) and `L4` (the end of the `if`). Notice the end of the loop body: `L4:` and then `goto L1`, two lines in a row. The `if` statement ends at `L4`, and the `while` adds its jump back after it. Section 10.5's peephole optimiser will notice that a jump to `L4` is then a jump to a `goto L1`.

</details>

### A lab program from a blank file

<!-- q: idea=three-address-code type=build unseen="tac-interpreter" wrong="ifFalse jumps when the condition is true" -->
**5. tacrun.c: run three-address code.** Write `tacrun.c`, an interpreter for the three-address code that `tiny-tac` writes. It reads the code on standard input, one instruction per line: a label line `L1:`; a copy `x = y`; a binary assignment `x = y op z`, where `op` is one of `+ - * /` or the comparisons `< <= > >= == !=` (which give 1 or 0); a unary minus `x = minus y`; `goto L`; and `if a goto L` or `ifFalse a goto L`, where `a` is an address or a comparison `y relop z`. Words are separated by blanks. An address is a number (it starts with a digit, `.` or `-`, as in `-3`, which an optimiser may write) or a name; every name starts at 0, and values are `double`s. It runs the code from the first line, and when it falls off the end, prints `name = value` (with `%g`) for each name that is not a temporary (`t` followed by a digit), in the order the names were first met. On a jump to a label that does not exist, it prints `tacrun: no label L9` to standard error and exits with status 1; after 100,000 instructions it stops with `tacrun: stopped after 100000 steps` and status 1, so that an endless loop ends.

**Done when** `./tacrun` with the code of `gcd.tiny` (task 4) as its input prints exactly the model's output below. You can test that without anyone marking it: in the understudy folder (the one that holds `book/`), in the terminal, run `python3 scripts/us_practice.py check compiler-design 9.7 --in ~/cdlab/ch9`. It builds your `tacrun.c` and compares what it prints for this input with the model. Then test it, as habit 3 asks, on an empty input, on `goto L9`, and on `x = 1`, `L1:`, `goto L1`, an endless loop.

<details>
<summary>Answer</summary>

<!-- code: id=tacrun check="gcc -Wall -Wextra tacrun.c -o tacrun && ./tacrun" stdin="a = 48\nb = 18\nL1:\nifFalse a != b goto L2\nifFalse a > b goto L3\nt1 = a - b\na = t1\ngoto L4\nL3:\nt2 = b - a\nb = t2\nL4:\ngoto L1\nL2:\n" -->
```c
/* tacrun.c: run three-address code. Reads the code on standard input,
   one instruction per line:
       L:   x = y   x = y op z   x = minus y   goto L
       if a goto L   ifFalse a goto L   (a may be y relop z)
   Every name starts at 0. At the end it prints each name that is not a
   temporary (t and digits), in the order it was first met. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAXLINES 500
#define MAXVARS 100
#define MAXSTEPS 100000

char code[MAXLINES][80];            /* the program, one line each */
int ncode;
char name[MAXVARS][16];             /* the names met so far ... */
double val[MAXVARS];                /* ... and their values */
int nvars;

int istemp(const char *s)
{
    return s[0] == 't' && isdigit((unsigned char)s[1]);
}

int var(const char *s)              /* the index of name s, made if new */
{
    for (int i = 0; i < nvars; i++)
        if (strcmp(name[i], s) == 0)
            return i;
    if (nvars == MAXVARS) {
        fprintf(stderr, "tacrun: too many names\n");
        exit(1);
    }
    strcpy(name[nvars], s);
    val[nvars] = 0;
    return nvars++;
}

double get(const char *s)           /* a number's value, or a name's */
{
    if (isdigit((unsigned char)s[0]) || s[0] == '.' || s[0] == '-')
        return strtod(s, NULL);
    return val[var(s)];
}

double apply(double a, const char *op, double b)
{
    if (strcmp(op, "+") == 0) return a + b;
    if (strcmp(op, "-") == 0) return a - b;
    if (strcmp(op, "*") == 0) return a * b;
    if (strcmp(op, "/") == 0) return a / b;
    if (strcmp(op, "<") == 0) return a < b;
    if (strcmp(op, "<=") == 0) return a <= b;
    if (strcmp(op, ">") == 0) return a > b;
    if (strcmp(op, ">=") == 0) return a >= b;
    if (strcmp(op, "==") == 0) return a == b;
    if (strcmp(op, "!=") == 0) return a != b;
    fprintf(stderr, "tacrun: unknown operator %s\n", op);
    exit(1);
}

int label(const char *l)            /* the line that holds "l:" */
{
    char want[20];

    snprintf(want, sizeof want, "%s:", l);
    for (int i = 0; i < ncode; i++)
        if (strcmp(code[i], want) == 0)
            return i;
    fprintf(stderr, "tacrun: no label %s\n", l);
    exit(1);
}

int main(void)
{
    char w[6][16];
    int pc = 0, steps = 0, k;       /* pc: the line being run */

    while (ncode < MAXLINES && fgets(code[ncode], sizeof code[ncode], stdin)) {
        code[ncode][strcspn(code[ncode], "\n")] = '\0';
        ncode++;
    }
    while (pc < ncode) {
        if (++steps > MAXSTEPS) {
            fprintf(stderr, "tacrun: stopped after %d steps\n", MAXSTEPS);
            return 1;
        }
        k = sscanf(code[pc], "%15s %15s %15s %15s %15s %15s",
                   w[0], w[1], w[2], w[3], w[4], w[5]);
        if (k <= 0 || w[0][strlen(w[0]) - 1] == ':') {    /* blank or label */
            pc++;
        } else if (strcmp(w[0], "goto") == 0) {
            pc = label(w[1]);
        } else if (strcmp(w[0], "if") == 0 || strcmp(w[0], "ifFalse") == 0) {
            double c = k == 6 ? apply(get(w[1]), w[2], get(w[3])) : get(w[1]);
            int jump = strcmp(w[0], "if") == 0 ? c != 0 : c == 0;
            pc = jump ? label(w[k - 1]) : pc + 1;
        } else if (k == 3) {                               /* x = y */
            val[var(w[0])] = get(w[2]);
            pc++;
        } else if (k == 4 && strcmp(w[2], "minus") == 0) { /* x = minus y */
            val[var(w[0])] = -get(w[3]);
            pc++;
        } else if (k == 5) {                               /* x = y op z */
            val[var(w[0])] = apply(get(w[2]), w[3], get(w[4]));
            pc++;
        } else {
            fprintf(stderr, "tacrun: cannot read: %s\n", code[pc]);
            return 1;
        }
    }
    for (int i = 0; i < nvars; i++)
        if (!istemp(name[i]))
            printf("%s = %g\n", name[i], val[i]);
    return 0;
}
```
```text
a = 6
b = 6
```

**How it runs.** The whole program is read into `code` first, because a jump may go forwards to a line not yet read. `pc` is the program counter, the line being run. Each line is split into up to six words, and the first word and the number of words tell the kind: a word ending in `:` is a label, which does nothing; `goto` sets `pc` to the label's line; `if` and `ifFalse` compute their condition (a comparison if there are six words, `if a relop b goto L`, otherwise one address) and jump or fall through; and the three assignment shapes are told apart by their word count, three, four or five. `label` searches for the line `L1:` every time, which is slow for a long program and plenty for a lab one. Names live in two parallel arrays, and `var` adds a name the first time it is met, which also fixes the order of the final printout.

The usual mistake is in `ifFalse`: it jumps when the condition is *false*, 0. Getting it backwards gives a loop that runs zero times or never ends; the step limit catches the second.

Now the whole chain: Tiny source, through `tiny-tac`, into `tacrun`. `prog.tiny` should leave `pos` at 160 after the assignment and then subtract 7.25 until it is below 100:

<!-- code: id=tacrun-chain -->
```console
$ gcc -Wall -Wextra tacrun.c -o tacrun
$ ./tiny-tac < gcd.tiny | ./tacrun
a = 6
b = 6
$ ./tiny-tac < ../ch2/prog.tiny | ./tacrun
init = 10
rate = 2.5
pos = 94.75
```

160 − 9 × 7.25 = 94.75, the first value below 100: the translation of the loop is right. And the tests of habit 3:

<!-- code: id=tacrun-tests exit=1 -->
```console
$ ./tacrun < /dev/null
$ printf 'goto L9\n' | ./tacrun; echo "exit $?"
tacrun: no label L9
exit 1
$ printf 'x = 1\nL1:\ngoto L1\n' | ./tacrun
tacrun: stopped after 100000 steps
```

`tacrun` is a test bench for chapter 10. An optimisation is correct only if the optimised code leaves every variable with the value the original code left; `tacrun` prints those values, so any optimised code can be checked by running both versions and comparing.

</details>
