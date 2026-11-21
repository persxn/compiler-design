# Chapter 1: Follow one statement through a compiler

<!-- contents -->
- [1.1 What happens between prog.c and a.out: watching gcc stop after each stage](#11-what-happens-between-progc-and-aout-watching-gcc-stop-after-each-stage)
- [1.2 The phases of a compiler, traced on pos = init + rate * 60](#12-the-phases-of-a-compiler-traced-on-pos--init--rate--60)
- [1.3 Tokens, lexemes and patterns](#13-tokens-lexemes-and-patterns)
- [1.4 The symbol table: the notebook every phase shares](#14-the-symbol-table-the-notebook-every-phase-shares)
- [1.5 Passes, interpreters, cross compilers and compiler-construction tools](#15-passes-interpreters-cross-compilers-and-compiler-construction-tools)
- [1.6 How a lab exam program is written, built, run and marked](#16-how-a-lab-exam-program-is-written-built-run-and-marked)
- [1.7 Practice: trace a new statement through every phase by hand](#17-practice-trace-a-new-statement-through-every-phase-by-hand)
<!-- /contents -->

## 1.1 What happens between prog.c and a.out: watching gcc stop after each stage

### One command, four jobs

You have typed `gcc prog.c` hundreds of times. A moment later there is an `a.out`, it runs, and you never had to think about what happened in between. Then the compiler-design lab begins and the first viva question is "what does gcc do with your file?", and the first long question in the paper asks you to explain, with an example, the steps a compiler goes through inside (section 1.2 answers it). The single command that has always just worked suddenly needs explaining.

The honest answer is that `gcc` is not one job. It is four jobs done one after another, and three of them are done by separate programs that `gcc` starts for you and then hides. This section makes `gcc` stop after each job, so you can open the file each one leaves behind and see with your own eyes what it did. Every later chapter of this book lives inside one of those four jobs, so it is worth seeing them once.

### An assembly line of stations

Picture a factory assembly line. Raw material goes in at one end. Each station does one kind of work on what it is handed, then passes its product to the next station, and a finished product comes out at the far end. No station needs to know how the others work; it only needs to know what it receives and what it must hand on.

<!-- teaches: compile-pipeline = compile pipeline -->
`gcc` is such a line with four stations:

1. the **preprocessor** receives your C file and hands on C text with the headers pasted in and the macros replaced;
2. the **compiler proper** receives that C text and hands on assembly language for your processor;
3. the **assembler** receives the assembly and hands on an object file of machine code;
4. the **linker** receives one or more object files, joins them with the C library, and hands on a program you can run.

This chain of four stations is the *compile pipeline*. The picture is true for these four stations in a very literal way: they really are separate steps, and between each pair there really is a file you can look at. It stops being true in two places. The linker does not only take what the line hands it; it also pulls in finished parts from outside, such as the code of `printf`, which you never wrote. And when you open up the second station, the compiler proper, in section 1.2, you will find a smaller line inside it whose stations do not take turns so neatly.

### Setting up the work folder

All the work in this book goes in a folder called `cdlab` in your home folder, with one folder inside it per chapter. When a command prints your home folder, this book shows it as `/home/you`; your own user name stands there. Open a terminal in your home folder and type this. `mkdir -p cdlab/ch1` makes the folder and, because of `-p`, the `cdlab` folder above it, with no complaint if they already exist; `cd cdlab/ch1` moves you into it. The `cat > pos.c <<'END'` line writes everything up to the line `END` into a new file called `pos.c`, so you do not need an editor yet.

<!-- code: id=ch1-setup -->
```console
$ mkdir -p cdlab/ch1
$ cd cdlab/ch1
$ cat > pos.c <<'END'
#include <stdio.h>

#define RATE 2.5

double pos, init = 10.0, rate = RATE;

int main(void)
{
    pos = init + rate * 60;
    printf("pos = %g\n", pos);
    return 0;
}
END
$ gcc -Wall -Wextra pos.c -o pos
$ ./pos
pos = 160
```

The middle line of `main` is the statement this whole book follows: `pos = init + rate * 60;`. Every chapter comes back to it. The rest of the file is only there to give it something to work on: three variables of type `double`, declared outside `main` (you will see in section 1.4 why they are outside), a macro `RATE`, and a `printf` so the program shows its answer. Ten plus two and a half times sixty is 160, and that is what it printed.

Two habits are already in that build line, and the book keeps them from here on. `-Wall -Wextra` turns on the warnings that point at real bugs; treat each one as a bug to fix before the examiner sees it. `-o pos` names the output; never give the output the name of a source file, as you will see at the end of this section.

### Station 1: the preprocessor

<!-- teaches: preprocessor = preprocessor -->
The *preprocessor* is the station that handles every line starting with `#`. `gcc -E` runs the first station and stops. Its product is still C, so you can read it:

<!-- code: id=stage-preprocess -->
```console
$ gcc -E pos.c -o pos.i
$ wc -l pos.c pos.i
   12 pos.c
  825 pos.i
  837 total
$ grep -n ' printf (' pos.i
476:extern int printf (const char *__restrict __format, ...);
$ tail -n 9 pos.i
# 5 "pos.c"
double pos, init = 10.0, rate = 2.5;

int main(void)
{
    pos = init + rate * 60;
    printf("pos = %g\n", pos);
    return 0;
}
```

`wc -l` counts the lines of each file, and `grep -n` prints the lines that contain a piece of text, each with its line number. Your twelve-line file went in and a file of several hundred lines came out. The `grep` shows where they came from: the line `#include <stdio.h>` was replaced by the whole text of the header file `stdio.h` (and the headers that it includes in turn), and far down inside that text is the declaration of `printf`, the line that tells the compiler what `printf` takes and returns. That is all `#include` ever does: it pastes one file into another, as text.

The `tail` shows the end of the file, which is your own code. Look at the declaration line: `RATE` has gone, and `2.5` stands in its place. The `#define` line itself has gone too. The preprocessor replaced the name with its text everywhere it appeared and then threw the definition away. The lines that begin with `#` and a number, such as `# 5 "pos.c"`, are line markers: notes saying "what follows came from line 5 of `pos.c`". They are how the next station can still report an error in the line of *your* file, not line 700 of `pos.i`.

The point to take from this station is what it does *not* do. It does not know what a variable is, or whether `rate * 60` makes sense. It works on text: lines, names to replace, files to paste. If you had written `#define RATE hello`, it would have put `hello` there just as happily.

### Station 2: the compiler proper

`gcc -S` runs the first two stations and stops, leaving the compiler's product in a file ending `.s`:

<!-- code: id=stage-compile -->
```console
$ gcc -S pos.c
$ grep -n -E 'main:|mulsd|addsd|call' pos.s
31:main:
42:     mulsd   %xmm0, %xmm1
44:     addsd   %xmm1, %xmm0
51:     call    printf@PLT
```
<!-- checked: how=reworded date=2026-11-21 ref="run as stage-compile on x86-64 GCC 13.3; ARM case now named in the text" h=bed77797 -->

This is assembly language: one line per machine instruction, still written as text. The `grep` picks out the lines that matter. `main:` is a label, the place where `main` starts. `mulsd` multiplies two `double` values and `addsd` adds them: that is your `rate * 60` and your `init + ...`, now as instructions for an x86-64 processor. The `call` line jumps to `printf`. (On an ARM machine, such as a Raspberry Pi, the grep finds only `main:`: there the same work is done by `fmul`, `fadd` and `bl printf`.)

This is the station the rest of the book is about. It is the only one that understands C: it found the names, worked out that `*` must happen before `+`, checked that the types fit, and chose instructions. Section 1.2 opens it up.

### Station 3: the assembler

<!-- teaches: assembler = assembler, object file -->
The *assembler* turns assembly text into machine code, in an *object file*. `gcc -c` runs three stations and stops before the linker. Its product, `pos.o`, is machine code, which is not text any more, so instead of opening it you ask the tool `nm` to list the names inside it:

<!-- code: id=stage-assemble -->
```console
$ gcc -c pos.c
$ nm pos.o
0000000000000000 D init
0000000000000000 T main
0000000000000000 B pos
                 U printf
0000000000000008 D rate
```

The assembler turned each line of assembly into the bytes the processor really executes, and it kept a list of names: the functions and variables this file defines, and the ones it uses but does not define. Each line of `nm` is one name. The letter tells you what kind: `T` is code (the "text" section), `D` is a variable with a starting value (`init` and `rate`), `B` is a variable that starts at zero (`pos`, which you gave no value). The line that matters most is the one with `U` and no address: `printf` is *used* in this file but *undefined* here. The assembler has left a hole with a name on it.

That is why `pos.o` cannot run yet. Its `call` to `printf` points at nothing.

### Station 4: the linker

<!-- teaches: linker = linker -->
The *linker* joins object files and libraries into one program. Run `gcc` on the object file, with no stopping flag, and only the last station has work left:

<!-- code: id=stage-link -->
```console
$ gcc pos.o -o pos
$ ./pos
pos = 160
$ ls
pos  pos.c  pos.i  pos.o  pos.s
```

The linker took `pos.o`, found the code of `printf` in the C library, joined the two, filled every hole with a real address, and wrote a program that runs. `ls` shows the product of every station side by side: `pos.c` (yours), `pos.i` (preprocessor), `pos.s` (compiler), `pos.o` (assembler) and `pos` (linker).

| You type | It stops after | It leaves | What is in it |
|---|---|---|---|
| `gcc -E pos.c -o pos.i` | the preprocessor | `pos.i` | C text, headers pasted, macros replaced |
| `gcc -S pos.c` | the compiler proper | `pos.s` | assembly text for this processor |
| `gcc -c pos.c` | the assembler | `pos.o` | machine code and a list of names, some undefined |
| `gcc pos.c -o pos` | the linker | `pos` | a program you can run |

### gcc is a manager, not a worker

When you type plain `gcc pos.c -o pos`, all four stations run and the in-between files are made in a temporary folder and deleted. `gcc` itself does none of the work; it is a *driver* that starts the right programs in the right order. The option `-###` makes it print the commands it would run, without running them. Each command line it prints starts with a space and then the program's name, so this keeps just that part:

<!-- code: id=driver -->
```console
$ gcc -### pos.c -o pos 2>&1 | grep -o '^ [^ ]*'
 /usr/libexec/gcc/x86_64-linux-gnu/13/cc1
 as
 /usr/libexec/gcc/x86_64-linux-gnu/13/collect2
```
<!-- checked: how=reworded date=2026-11-21 ref="run as driver on GCC 13.3; folder variation now said in the text" h=ad1412ea -->

Three programs. (The folder in front of `cc1` and `collect2` differs between distributions and GCC versions; the three names do not.) `cc1` is the compiler proper (the preprocessor is built into it in modern GCC, which is why there is no separate preprocessor program here). `as` is the assembler. `collect2` is a small helper that runs the linker, `ld`. Each reads a file that the one before wrote. The assembly line is real.

### Which station complained?

When a build fails, the shape of the message tells you which station stopped the line, and that tells you where to look. Here is the same program broken three ways. First, a misspelt header name:

<!-- code: id=break-header mode=error -->
```c
#include <stdoi.h>

double pos, init = 10.0, rate = 2.5;

int main(void)
{
    pos = init + rate * 60;
    return 0;
}
```
```text
break-header.c:1:10: fatal error: stdoi.h: No such file or directory
    1 | #include <stdoi.h>
      |          ^~~~~~~~~
compilation terminated.
```

`fatal error` and "No such file or directory": the preprocessor could not find a file to paste. Nothing after it ran. Second, a missing semicolon:

<!-- code: id=break-semicolon mode=error -->
```c
#include <stdio.h>

double pos, init = 10.0, rate = 2.5;

int main(void)
{
    pos = init + rate * 60
    printf("pos = %g\n", pos);
    return 0;
}
```
```text
break-semicolon.c: In function 'main':
break-semicolon.c:7:27: error: expected ';' before 'printf'
    7 |     pos = init + rate * 60
      |                           ^
      |                           ;
    8 |     printf("pos = %g\n", pos);
      |     ~~~~~~
```

A file name, a line number, a column and `error:`: that is the compiler proper. It understands C, so it can say exactly where the C stopped making sense. Notice that it reports the semicolon as missing *before* `printf`, on the next line: the compiler only knew something was wrong when it met a word that could not follow `60`. Third, a function that is declared but whose code never reaches `gcc`. Imagine `scale` is written in another file, `scale.c`, and you forgot to put that file on the command line:

<!-- code: id=break-link mode=error varies -->
```c
#include <stdio.h>

double scale(double x);

double pos, init = 10.0, rate = 2.5;

int main(void)
{
    pos = init + rate * 60;
    printf("pos = %g\n", scale(pos));
    return 0;
}
```
```text
/usr/bin/ld: /tmp/cc1ma4Mf.o: in function `main':
break-link.c:(.text+0x3d): undefined reference to `scale'
collect2: error: ld returned 1 exit status
```

The compiler proper is satisfied: the declaration on line 3 tells it what `scale` takes and returns, which is all it needs to compile the call. Then the linker fails with `undefined reference` and `ld returned 1 exit status`. There is no useful line number in the linker's complaint, because the linker never sees your lines; it sees object files with holes, and one hole has a name that nothing given to it fills. A misspelt call such as `prnitf(...)` with no declaration ends the same way on GCC 13: the compiler only warns that it has never heard of `prnitf`, and the linker then fails. From GCC 14 the compiler itself stops with an error at that line, which is kinder. Either way, the warning was the real clue, which is one more reason to treat warnings as bugs.

| The message looks like | The station | Look at |
|---|---|---|
| `fatal error: x.h: No such file or directory` | preprocessor | the `#include` lines |
| `pos.c:7:27: error: ...` | compiler proper | that line, and the one before it |
| `undefined reference to ...`, `ld returned 1 exit status` | linker | the spelling of the name; which files you gave `gcc` |

One mistake no message warns you about. `gcc pos.c -o pos.c` is a valid command: it compiles your file and then writes the program *over your source file*. Your C is gone and `pos.c` now holds machine code. Always give `-o` a name without `.c`.

### In one line each

1. `gcc` is a driver that runs four stations: preprocessor, compiler proper, assembler, linker.
2. `-E`, `-S` and `-c` stop after the first, second and third, leaving `.i`, `.s` and `.o` files you can inspect.
3. The preprocessor works on text only; the compiler proper is the one that understands C; the assembler makes machine code with named holes; the linker fills the holes from other object files and the C library.
4. An error's shape names its station: `fatal error` and a file name, `file:line:col: error`, or `undefined reference`.

**Check yourself**

<!-- q: idea=preprocessor type=decide wrong="the macro is replaced by the compiler" -->
1. You suspect a macro is being replaced with the wrong text. Which one of `pos.i`, `pos.s` or `pos.o` do you open, with which command do you make it, and why not the other two?

<details>
<summary>Answer</summary>

`pos.i`, made with `gcc -E pos.c -o pos.i`. Macro replacement is the preprocessor's job, and `pos.i` is exactly the preprocessor's product: your code, as C, after replacement. `pos.s` is assembly, where the macro has long since turned into instructions and constants, and `pos.o` is machine code you cannot read at all. If you picked `pos.s`, you were thinking of the compiler as the one that handles `#define`; it never sees the `#define`, only its result.

</details>

<!-- q: idea=linker type=fix wrong="an undefined reference is a syntax error in main.c" -->
2. Your lab program is split into two files: `main.c` calls `next_token()`, which is written in `scanner.c`. You build with `gcc -Wall -Wextra main.c -o main` and get `undefined reference to 'next_token'`. Which station failed, and what is the fix?

<details>
<summary>Answer</summary>

The linker. `main.c` compiled into an object file with a hole named `next_token`, and the only other things the linker was given (the C library) do not fill it. The code is in `scanner.c`, which you never gave to `gcc`. Fix: `gcc -Wall -Wextra main.c scanner.c -o main`. Staring at `main.c` for a typo is the usual time-waster here; an `undefined reference` is almost never about the line that makes the call.

</details>

<!-- q: idea=compile-pipeline type=predict wrong="removing the include stops the build" -->
3. Predict: you delete the line `#include <stdio.h>` from `pos.c` and build with `gcc -Wall -Wextra`. Which station complains, does the build stop, and does the program run?

<details>
<summary>Answer</summary>

Try it without editing `pos.c`: `grep -v` copies every line that does *not* contain `#include` into a new file.

<!-- code: id=no-include -->
```console
$ grep -v '#include' pos.c > noinc.c
$ gcc -Wall -Wextra noinc.c -o noinc
noinc.c: In function ‘main’:
noinc.c:9:5: warning: implicit declaration of function ‘printf’ [-Wimplicit-function-declaration]
    9 |     printf("pos = %g\n", pos);
      |     ^~~~~~
noinc.c:1:1: note: include ‘<stdio.h>’ or provide a declaration of ‘printf’
  +++ |+#include <stdio.h>
    1 |
noinc.c:9:5: warning: incompatible implicit declaration of built-in function ‘printf’ [-Wbuiltin-declaration-mismatch]
    9 |     printf("pos = %g\n", pos);
      |     ^~~~~~
noinc.c:9:5: note: include ‘<stdio.h>’ or provide a declaration of ‘printf’
$ ./noinc
pos = 160
```

The compiler proper complains, but only with a warning: without the pasted header it has no declaration of `printf`, so it falls back on a built-in idea of what `printf` is. The build does not stop, and the linker still finds `printf` in the C library, so the program runs. If you predicted a preprocessor error: the preprocessor has nothing to complain about, since there is no `#include` line left to fail. From GCC 14 an implicit declaration is an error by default, so there the compiler proper stops the build and no program is made; do not rely on the warning, put the `#include` back.
<!-- checked: how=practices date=2026-11-21 ref="PRACTICES.md Facts that change: GCC 14 implicit declarations are errors" h=e6516c9b -->

</details>

## 1.2 The phases of a compiler, traced on pos = init + rate * 60

### Inside the second station

In section 1.1 the compiler proper, `cc1`, took the line `pos = init + rate * 60;` and a moment later there were `mulsd` and `addsd` instructions in `pos.s`. Between those two, something read a row of characters and worked out that `pos`, `init` and `rate` are variables, that the `*` must be done before the `+`, that 60 is a whole number being multiplied by a `double`, and which instructions would do all that. That something is what compiler design is about, and it is also the paper's favourite long question: "explain the phases of a compiler, showing the output of each phase for the statement position = initial + rate * 60". This section answers it, once, in full, on this book's version of that statement.

<!-- teaches: compiler-phase = phase, phases of a compiler -->
### A smaller assembly line

Open up the compiler station and you find a smaller assembly line inside it. Each of its stations is called a *phase*: a step that takes the program in one form and hands it on in another form, a little closer to machine code. The textbook compiler has six phases, plus one shared table, the symbol table: a record of every name in the program (§1.4).

<!-- fig: id=phases
title: The phases of a compiler
kind: flow
dir: down
round src: characters\n`pos = init + rate * 60;`
box lex: Lexical analyzer
box syn: Syntax analyzer
box sem: Semantic analyzer
box icg: Intermediate code generator
box opt: Code optimizer
box cg: Code generator
round tgt: target code
store st: Symbol table
src -> lex
lex -> syn: tokens
syn -> sem: tree
sem -> icg: checked tree
icg -> opt: three-address code
opt -> cg
cg -> tgt
st -- lex
st -- sem
st -- cg
hot: lex syn sem icg
shade: opt cg
-->
![Flow chart, top to bottom. characters pos = init + rate * 60; leads to Lexical analyzer. Symbol table is joined to Lexical analyzer. Symbol table is joined to Semantic analyzer. Symbol table is joined to Code generator. Lexical analyzer leads to Syntax analyzer (tokens). Syntax analyzer leads to Semantic analyzer (tree). Semantic analyzer leads to Intermediate code generator (checked tree). Intermediate code generator leads to Code optimizer (three-address code). Code optimizer leads to Code generator. Code generator leads to target code. Lexical analyzer, Syntax analyzer, Semantic analyzer and Intermediate code generator are highlighted. Code optimizer and Code generator are greyed out.](fig/01-one-statement/phases.svg)

*Figure 1.2a: The six phases of a compiler with the symbol table beside them. The four front-end phases are highlighted; the two back-end phases are grey.*

The picture of stations handing products down a line is good for remembering the order and what passes between the phases, and it is exactly how the paper wants the answer drawn. It breaks in two places, and both matter in the lab. First, the phases do not take turns over the whole program. The syntax analyzer is in charge; whenever it needs the next piece of the input it calls the lexical analyzer, which reads just far enough to hand back one piece. So the first two stations run interleaved, a piece at a time, not one after the other. You will write exactly that arrangement in chapter 6, where your parser calls your scanner. Second, the symbol table is not a station at all. It is a notebook lying beside the line that every phase writes in or reads from (section 1.4 is about it). That is why it is drawn off to the side, joined to the line rather than in it.

### The trace, one phase at a time

Here is the statement going through all six phases. At each one, look at what the phase receives, what it hands on, and what it wrote in the notebook. This is also the answer the paper expects; learn the shape of each line.

**Phase 1, lexical analysis.** The lexical analyzer (also called the *scanner*) receives a row of characters: `p`, `o`, `s`, a space, `=`, a space, `i`, ... It groups them into the meaningful pieces of the language and drops the spaces. Each piece comes out as a *token*, written in angle brackets:

```text
<id,1> <=> <id,2> <+> <id,3> <*> <60> <;>
```

`pos`, `init` and `rate` are all names, so all three come out with the same token name, `id`. What tells them apart is the number after the comma: the line in the symbol table where the scanner wrote that name down. When the scanner met `pos` it had never seen it, so it wrote `pos` on line 1 of the notebook; `init` went on line 2 and `rate` on line 3. The operators and the semicolon need nothing more than their own name. The number 60 is shown simply as `<60>`, the way the Dragon book draws it. (That is Aho, Lam, Sethi and Ullman's *Compilers: Principles, Techniques and Tools*, named after the dragon on its cover; this book uses its notation because the papers do.) Section 1.3 says exactly what a token is and how the scanner decides where one ends.

**Phase 2, syntax analysis.** The syntax analyzer (the *parser*) receives the tokens and works out how they group, according to the grammar of the language. Its product is a *syntax tree*: each operator is a node whose children are its operands.

```text
          =
        /   \
   <id,1>     +
            /   \
       <id,2>     *
                /   \
           <id,3>    60
```

This tree is where the rule "multiply before you add" lives. The `*` sits lower than the `+`, so the multiplication is one of the `+` node's operands: it has to be worked out before the addition can happen. Nothing in the token row said so; the parser knew it from the grammar. The classic expression grammar that builds exactly this shape is `E -> E + T | T`, `T -> T * F | F`, `F -> ( E ) | id`, and chapter 5 is about how it does so.

**Phase 3, semantic analysis.** The semantic analyzer checks that the tree makes sense as a program: every name is declared, and every operator is given operands of types it can work on. It reads the types from the notebook. In the textbook version of this example, `pos`, `init` and `rate` are floating-point, and 60 is an integer. A machine cannot multiply a floating-point value by an integer directly, so the semantic analyzer puts a conversion into the tree:

```text
          =
        /   \
   <id,1>     +
            /   \
       <id,2>     *
                /   \
           <id,3>   inttofloat
                        |
                        60
```

The tree is the same shape, with one new node, `inttofloat`, above the 60. This is the step students most often leave out of their answer, and it is the only thing that changes in this phase, so the examiner looks for it.

**Phase 4, intermediate code generation.** The tree is good for checking but awkward to turn into instructions. So the next phase walks the tree from the bottom up and writes it out as a list of simple instructions, each doing at most one operation and storing the result in a new temporary name (`t1`, `t2` ...):

```text
t1 = inttofloat(60)
t2 = id3 * t1
t3 = id2 + t2
id1 = t3
```

This form is called *three-address code*, because each instruction names at most three places: two operands and a result. Read it against the tree: the deepest node (the conversion) comes first, then the `*` that uses it, then the `+` that uses that, then the assignment. Chapter 9 generates it with a program.

**Phase 5, code optimization.** The optimizer receives three-address code and hands back code that does the same thing in less time or space. Two improvements are possible here. Converting the constant 60 to floating-point does not need to wait until the program runs: the compiler can do it once, now, and write `60.0`. That removes the first instruction. And `t3` exists only to be copied into `id1`, so the addition can store into `id1` directly:

```text
t1 = id3 * 60.0
id1 = id2 + t1
```

Four instructions became two, and the program still computes exactly the same value. Chapter 10 teaches these improvements by name.

**Phase 6, code generation.** The code generator maps the three-address code onto the real machine: it chooses registers, loads values from memory, operates on them and stores the result. In the Dragon book's simple machine, where `R1` and `R2` are registers, `F` means floating-point and `#` marks a constant:

```text
LDF  R2, id3
MULF R2, R2, #60.0
LDF  R1, id2
ADDF R1, R1, R2
STF  id1, R1
```

Load `rate` into R2, multiply it by 60.0, load `init` into R1, add R2 into R1, store R1 into `pos`. Here the notebook is read once more: the code generator needs to know where in memory each `id` lives.

### Watching gcc do the same thing

The trace above is the textbook's. Your own compiler can show you that real compilers do much the same. GCC can write out its intermediate forms as it goes, as files, with `-fdump-tree-...` options. You add one option per form you want, and each writes a file named after your source. Two are useful here: `-fdump-tree-original` writes the checked tree, written back out as C-like text, and `-fdump-tree-gimple` writes GCC's three-address code, which GCC calls GIMPLE.

<!-- code: id=dump-phases -->
```console
$ gcc -c -fdump-tree-original -fdump-tree-gimple pos.c
$ ls pos.c.*
pos.c.005t.original  pos.c.006t.gimple
$ grep 'pos =' pos.c.*.original
  pos = rate * 6.0e+1 + init;
  printf ((const char * restrict) "pos = %g\n", pos);
$ cat pos.c.*.gimple
int main ()
{
  int D.3181;

  {
    rate.0_1 = rate;
    _2 = rate.0_1 * 6.0e+1;
    init.1_3 = init;
    _4 = _2 + init.1_3;
    pos = _4;
    pos.2_5 = pos;
    printf ("pos = %g\n", pos.2_5);
    D.3181 = 0;
    return D.3181;
  }
  D.3181 = 0;
  return D.3181;
}

```
<!-- checked: how=reworded date=2026-11-21 ref="run as dump-phases on GCC 13.3; version variation now said in the text" h=1bfb46b1 -->

The number in each file name (`005t`, `006t`) and the exact text of the dumps change between GCC versions, which is why the commands say `pos.c.*`. Look at the `original` line first. The 60 has already become `6.0e+1`: a floating-point constant, 60.0. GCC's semantic analysis saw an integer meeting a `double`, and since the integer was a constant, it converted it on the spot instead of inserting an `inttofloat` that runs later. You can find the same constant in the assembly from section 1.1, stored as the two 32-bit halves of a `double`, low half first (this is GCC on x86-64; other processors label and lay it out differently). `grep -A 2` prints each matching line and the 2 lines after it:

<!-- code: id=const-in-asm -->
```console
$ grep -A 2 '^\.LC0:' pos.s
.LC0:
        .long   0
        .long   1078853632
```
<!-- checked: how=reworded date=2026-11-21 ref="run as const-in-asm on x86-64; 1078853632 = 0x404E0000 checked by hand; other targets now named in the text" h=55d5bb12 -->

The number 1078853632 is 404E0000 in hexadecimal, the high half of 60.0 in the 64-bit `double` format, and the low half is 0. No `int` 60 is left anywhere in the program; the conversion was done once, at compile time, by the compiler.

Now the `gimple` dump. Find the lines between the inner `{` and the `printf` call: each does at most one operation, into a temporary with a made-up name such as `_2`, exactly the rule of three-address code. GCC copies `rate` into a temporary, multiplies it by `6.0e+1`, copies `init`, adds, and stores into `pos`. Set it beside the optimized code from phase 5, `t1 = id3 * 60.0` and `id1 = id2 + t1`: the same multiply, then the same add, then the store. The differences are small. GCC names its temporaries differently, it makes extra copies of the globals, and it wrote the addition as `_2 + init.1_3` with the operands swapped, which is allowed because swapping the operands of `+` does not change the result. The `D.3181` lines are GCC's own bookkeeping for `return 0`.

### Front end and back end

<!-- teaches: front-end = front end, analysis -->
The six phases fall into two groups. The first four (lexical, syntax and semantic analysis, and intermediate code generation) are the *front end*, also called the *analysis* part: they take the source program apart, check it, and produce the intermediate code. Everything the front end does depends on the source language (C's rules for names, C's grammar, C's types) and nothing depends on the machine. Three-address code says `t1 = id3 * 60.0`, not `mulsd`.

<!-- teaches: back-end = back end, synthesis -->
The last two (optimization and code generation) are the *back end*, also called the *synthesis* part: they build the target program out of the intermediate code. The code generator depends entirely on the target machine and not at all on the source language; it would do the same work if the three-address code had come from a Pascal program. The optimizer sits on the boundary, and some books call a machine-independent optimizer a "middle end"; in the paper, put it with the back end unless your lecturer drew it otherwise.

Why build compilers in two halves? Suppose you want compilers for 3 languages on 4 machines. Written whole, that is 3 × 4 = 12 compilers. Split at the intermediate code, it is 3 front ends and 4 back ends, 7 pieces, and every new language costs one front end, not one compiler per machine. GCC is built this way: C, C++ and Fortran have their own front ends that all produce GIMPLE, and they share the back ends for every processor GCC supports. The cost is that the intermediate code has to be general enough for every language and every machine, so neither half can take shortcuts that rely on the other.

### When the trace goes wrong in the paper

These are the mistakes that cost marks on this question, each with the fix.

**The symbol table drawn as a seventh station.** It goes beside the line, joined to the phases, not between two of them. No product flows through it.

**No `inttofloat`.** The semantic analyzer's output is drawn identical to the parser's. If the types are mixed, the conversion node must appear, and the intermediate code must start with it. If the question says all the variables are integers, there is no conversion, and you should say so in one line so the examiner sees you checked.

**More than one operator in a line of three-address code.** `t1 = id3 * 60 + id2` is not three-address code. One operator, one result, per line.

**The tree built in the wrong order.** If `+` sits below `*`, the tree computes `(init + rate) * 60`. Check by reading the tree back: the operator nearest the root is done last.

**The optimizer changing what the program computes.** Folding `inttofloat(60)` into `60.0` and removing the copy into `t3` are safe. Reordering to `id1 = id2 * 60.0 + id3` is not; check that your two optimized lines compute the same value as the four you started from.

### In one line each

1. Lexical analysis turns characters into tokens and enters each new name in the symbol table.
2. Syntax analysis groups the tokens into a syntax tree, where operator precedence is decided.
3. Semantic analysis checks names and types and inserts conversions such as `inttofloat`.
4. Intermediate code generation flattens the tree into three-address code.
5. Optimization shortens the three-address code without changing its meaning.
6. Code generation maps it onto registers and instructions.
7. Phases 1 to 4 are the front end (source-language dependent); 5 and 6 the back end (machine dependent).

**Check yourself**

<!-- q: idea=compiler-phase type=predict wrong="the conversion is always applied to the constant" -->
1. Suppose `rate` is an `int` while `pos` and `init` are floating-point (the statement is unchanged). Predict the semantic analyzer's tree and the three-address code.

<details>
<summary>Answer</summary>

Now `rate * 60` is an integer times an integer, so the multiplication is an integer one and needs no conversion. Its result meets `init`, a float, at the `+`, so the conversion goes above the `*` node:

```text
t1 = id3 * 60
t2 = inttofloat(t1)
t3 = id2 + t2
id1 = t3
```

If you put `inttofloat(60)` first again, you applied the textbook answer from memory instead of reading the types. The conversion goes wherever an integer value meets a float operand, and here that is the result of the multiplication.

</details>

<!-- q: idea=compiler-phase type=fix wrong="any short code is three-address code" -->
2. A classmate's intermediate code for the running statement is below. Find the two faults.

```text
t1 = id3 * 60
t2 = id2 + t1
id1 = t2
```

<details>
<summary>Answer</summary>

First, the `inttofloat` is missing: 60 is an integer and `id3` is floating-point, so the code must convert 60 (or use 60.0 and say the conversion was done at compile time, which belongs to the optimizer, not to intermediate code generation). Second, and following from it, `t1 = id3 * 60` multiplies values of two different types, which no single machine instruction does. The three-address form itself is fine: one operator per line. The tempting wrong answer is "it is just the optimized version"; it is not, because the optimized version has `60.0`, a floating-point constant.

</details>

<!-- q: idea=front-end type=decide wrong="a new language needs a new code generator" -->
3. A team has a working C compiler for x86-64 built as a front end and a back end. They want a compiler for a new language on the same machine. Which phases must they write, and which can they reuse?

<details>
<summary>Answer</summary>

They write a new front end: lexical, syntax and semantic analysis, and intermediate code generation for the new language, producing the same intermediate code the C front end produces. They reuse the optimizer and the code generator unchanged, because those depend only on the intermediate code and the machine. Saying "a new code generator" is the usual slip: the machine has not changed, so neither does the part that knows the machine.

</details>

## 1.3 Tokens, lexemes and patterns

### What exactly did the scanner hand on?

In section 1.2 the first phase turned `pos = init + rate * 60;` into `<id,1> <=> <id,2> <+> <id,3> <*> <60> <;>`. Some questions were left open. Why do three different names all become `id`, while `=` and `+` each keep their own name? Where did the spaces go? What decides that `rate` is one piece and not four? The paper asks the same thing in a standard form, "distinguish between token, lexeme and pattern, with examples", and the lab asks you to write the program that does it, from chapter 2 on. This section gives the three words their exact meaning, on the running statement.

### Words and parts of speech

Think about how you read an English sentence: "The clerk stamped the form." Before you can understand it, you have already cut the row of letters into words, and you know without effort that "clerk" and "form" are nouns and "stamped" is a verb. The grammar of English talks about nouns and verbs, not about "clerk" and "form" in particular: "a sentence can be a noun phrase followed by a verb" works for every noun there is. And you have rules in your head for what counts as a word at all, so "stamped" is one word, not "stam" and "ped".

That is the whole idea. The particular words are *lexemes*, their parts of speech are *tokens*, and the rules for what forms a word of each kind are *patterns*. The picture stops being true in one useful way. In English the same word can be a noun or a verb depending on the sentence ("a stamp", "to stamp"). In a programming language the scanner decides a lexeme's token without looking at the rest of the statement: `rate` is an identifier wherever it appears. That is what lets a scanner be a small, fast program that knows nothing of grammar.

### The three words, exactly

<!-- teaches: lexeme = lexeme -->
A *lexeme* is a sequence of characters in the source program that forms one meaningful unit. In the running statement the lexemes are `pos`, `=`, `init`, `+`, `rate`, `*`, `60` and `;`. A lexeme is concrete: it is those characters, at that place in the file.

<!-- teaches: token = token, token name, attribute value -->
A *token* is what the scanner hands to the parser for one lexeme: a pair, `<token-name, attribute-value>`. The token name is the kind of unit: `id` for any identifier, `num` for any number, and one name per operator or punctuation mark. The attribute value carries whatever the later phases need to know about this particular lexeme: for an identifier, the line of the symbol table where its name is written (§1.4); for a number, its value. Many lexemes share one token name; that is the point of having token names.

<!-- teaches: pattern = pattern -->
A *pattern* is the rule that describes which lexemes belong to a token name. The pattern for `id` is "a letter, followed by any number of letters and digits" (in C, the underscore counts as a letter). The pattern for `num` is "one or more digits" (for whole numbers). The pattern for the token `+` is just the one character `+`. Chapter 3 writes patterns precisely, as regular expressions; for now plain English is enough.

Here is the running statement in one table. Make a table like this whenever the paper asks you to "identify the tokens".

| Lexeme | Token name | Attribute value | Pattern (in words) |
|---|---|---|---|
| `pos` | `id` | symbol table line 1 | letter followed by letters or digits |
| `=` | `=` | none | the character `=` |
| `init` | `id` | symbol table line 2 | letter followed by letters or digits |
| `+` | `+` | none | the character `+` |
| `rate` | `id` | symbol table line 3 | letter followed by letters or digits |
| `*` | `*` | none | the character `*` |
| `60` | `num` | the value 60 | one or more digits |
| `;` | `;` | none | the character `;` |

Eight lexemes, eight tokens. In full, the 60 is the token `<num,60>`; the trace in section 1.2 wrote just `<60>`, which is how the Dragon book draws it to save space. Use whichever form your lecturer used, and be consistent.

Notice what is missing from the table: the four spaces. Spaces, tabs and newlines separate lexemes but are not lexemes; the scanner reads past them and produces no token. Comments are treated the same way. In C, the preprocessor has already removed the comments before the compiler proper ever sees the code (`pos.c` has none, but a comment you add to it is gone from `pos.i`); in a language without a preprocessor, the scanner skips them itself (you write that in section 2.4).

### Why bother with token names?

Look at the statement from the parser's side. The parser only ever looks at token names, so this is what it sees:

```text
id = id + id * num ;
```

That is the reason tokens exist. The grammar from section 1.2, `E -> E + T | T`, `T -> T * F | F`, `F -> ( E ) | id`, has one rule for `id`, and that rule covers `pos`, `init`, `rate` and every other name anyone will ever write. If the parser had to work on characters, its grammar would need rules for how names are spelled, how numbers are spelled, and where spaces may go, all tangled with the rules for expressions. Splitting the work keeps both halves simple: the scanner knows spelling and nothing about structure, the parser knows structure and nothing about spelling. The attribute values ride along so that nothing is lost: when the semantic analyzer later needs to know *which* `id` it is looking at, the symbol table line is right there in the token.

There is one wrinkle you will meet in the lab. In C, `int` and `while` are spelled exactly like identifiers, so they match the `id` pattern too, yet each must be its own token, a *keyword*. Two patterns match the same lexeme, and the scanner needs a rule to choose. Section 2.2 handles it with a keyword table.

### Which phase rejects it?

A wrong program is rejected by the first phase whose rules it breaks, and once you know what each phase checks you can predict which one it is. GCC's messages show this directly. Here are three versions of the running statement, each broken for a different phase. In the first, a letter is stuck to the number:

<!-- code: id=lex-error mode=error -->
```c
double pos, init = 10.0, rate = 2.5;

int main(void)
{
    pos = init + rate * 60x;
    return 0;
}
```
```text
lex-error.c: In function 'main':
lex-error.c:5:25: error: invalid suffix "x" on integer constant
    5 |     pos = init + rate * 60x;
      |                         ^~~
```

`60x` matches no pattern at all. It starts like a number, but no number pattern lets an `x` follow the digits, and it cannot be an identifier because identifiers cannot start with a digit. So the problem is found while cutting the characters into lexemes, before any grammar is involved. This is a *lexical error*. (GCC calls the `x` a "suffix" because C numbers may legally end in letters such as `L` or `u`, and `x` is not one of them.) In the second, every lexeme is fine but the order is not:

<!-- code: id=syntax-error mode=error -->
```c
double pos, init = 10.0, rate = 2.5;

int main(void)
{
    pos = init + rate * ;
    return 0;
}
```
```text
syntax-error.c: In function 'main':
syntax-error.c:5:25: error: expected expression before ';' token
    5 |     pos = init + rate * ;
      |                         ^
```

The scanner is perfectly happy: `pos`, `=`, `init`, `+`, `rate`, `*`, `;` are all good lexemes. The parser is not, because the grammar says a `*` must have an operand on each side, and here the `;` comes where the right-hand operand should be. Read the message: GCC talks in tokens. It says what it expected (an expression), and it names the token where it got stuck (`';' token`). That is a *syntax error*. In the third, everything is spelled and ordered correctly, but one name is wrong:

<!-- code: id=semantic-error mode=error -->
```c
double pos, init = 10.0, rate = 2.5;

int main(void)
{
    pos = init + rat * 60;
    return 0;
}
```
```text
semantic-error.c: In function 'main':
semantic-error.c:5:18: error: 'rat' undeclared (first use in this function); did you mean 'rate'?
    5 |     pos = init + rat * 60;
      |                  ^~~
      |                  rate
semantic-error.c:5:18: note: each undeclared identifier is reported only once for each function it appears in
```

`rat` is a perfectly good identifier, and `id = id + id * num ;` is perfectly good grammar. Only when the semantic analyzer looks the name up in the symbol table and finds no declaration for it does the program fail. That is a *semantic error*: the program is well formed, but it does not make sense.

### When you count tokens in the paper

The paper loves "how many tokens are there in this statement?". Three slips account for nearly all lost marks.

**Counting the spaces, or the comment.** They produce no tokens. `a  =  b ;` has four tokens however many spaces it has.

**Splitting a string literal.** `"pos = %g\n"` is one lexeme and one token, a string literal, even though it has spaces and an `=` inside it. The scanner reads from the opening quote to the closing quote as one unit.

**Splitting a name at a digit or an underscore.** `rate2` and `init_val` are one identifier each; the pattern lets digits and underscores follow the first letter.

### In one line each

1. A lexeme is the actual characters of one unit in the source.
2. A token is the pair `<token-name, attribute-value>` the scanner hands to the parser for it.
3. A pattern is the rule saying which lexemes belong to a token name.
4. White space and comments produce no tokens.
5. The parser sees only token names, which is what keeps its grammar small.

**Check yourself**

<!-- q: idea=token type=apply wrong="the string literal is several tokens" -->
1. How many tokens are in this C statement? List them: `printf("i = %d, &i = %x", i, &i);`

<details>
<summary>Answer</summary>

Ten: `printf`, `(`, `"i = %d, &i = %x"`, `,`, `i`, `,`, `&`, `i`, `)`, `;`. The whole string, quotes included, is one string-literal token; the `=`, `&` and `%` inside it are characters of that lexeme, not operators. If you got more than ten, you split the string. `printf` itself is an identifier, not a keyword: it is the name of a library function, which is exactly why `prnitf` in section 1.1 got only a warning from the compiler and failed at the linker.

</details>

<!-- q: idea=pattern type=explain wrong="lexeme and token are the same thing" -->
2. A friend says "lexeme and token are two words for the same thing". Using `rate` and `init` from the running statement, explain the difference in two or three sentences, and say where the pattern comes in.

<details>
<summary>Answer</summary>

`rate` and `init` are two different lexemes: different characters at different places in the source. They produce the same token name, `id`, because both match the identifier pattern, a letter followed by letters or digits; their tokens differ only in the attribute value (symbol table lines 3 and 2). So many lexemes can map to one token name, and the pattern is the rule that decides which ones do. The friend's view is tempting for operators like `+`, where the lexeme and the token name are spelled the same, but even there the token is the scanner's classification, not the characters.

</details>

<!-- q: idea=token type=predict wrong="pos=init+rate*60; has fewer tokens" -->
3. Predict the token stream for `pos=init+rate*60;`, written with no spaces at all. Is it the same as for the spaced version? Then say which phase rejects `pos = init + rate * 60 60;` and why.

<details>
<summary>Answer</summary>

Exactly the same eight tokens, `<id,1> <=> <id,2> <+> <id,3> <*> <num,60> <;>`: spaces separate lexemes but are never part of them, and `=`, `+` and `*` cannot be part of an identifier, so the scanner knows where each name ends without a space. The second statement scans fine, into `... <num,60> <num,60> <;>`, since `60` is a good lexeme twice. The parser rejects it: no grammar rule allows one number straight after another. A syntax error, not a lexical one; the tempting answer "lexical, because something is wrong with the numbers" forgets that each number on its own matches the pattern.

</details>

## 1.4 The symbol table: the notebook every phase shares

<!-- teaches: symbol-table-idea = symbol table -->
The scanner in section 1.3 handed on `<id,1>`, and the code generator in section 1.2 needed to know where `id1` lives in memory. The 1 points into the *symbol table*: one record per name in the program, which the phases fill in as they learn things about that name.

Think of it as the compiler's notebook, lying open beside the assembly line. Each phase writes what it has just found out on the name's line and reads what earlier phases wrote. Here is the notebook for the running statement, filled in as the phases go by:

| Line | Name | Written by the scanner | Added by the semantic analyzer | Added by the code generator |
|---|---|---|---|---|
| 1 | `pos` | the name | kind: variable, type: `double` | its memory location |
| 2 | `init` | the name | kind: variable, type: `double` | its memory location |
| 3 | `rate` | the name | kind: variable, type: `double` | its memory location |

The scanner wrote only the names, because names are all a scanner knows. The semantic analyzer found the declaration `double pos, init = 10.0, rate = RATE;` and wrote each name's type, then read those types back to decide that `rate * 60` needs a conversion. The code generator gave each variable a place in memory and read it back to write `LDF R2, id3`.

Part of this notebook survives into the object file. Ask `nm` for the sizes as well (`-S`):

<!-- code: id=nm-sizes -->
```console
$ nm -S pos.o
0000000000000000 0000000000000008 D init
0000000000000000 0000000000000057 T main
0000000000000000 0000000000000008 B pos
                 U printf
0000000000000008 0000000000000008 D rate
```

The second column is each name's size in bytes, in hexadecimal: 8 for each `double`, because the compiler wrote down the type of each name and the type fixed its size. The first column is where the name sits within its section of the object file (`init` at 0 and `rate` at 8, right after `init`'s 8 bytes): the memory place the back end gave it, still relative to the start of that section until the linker fixes the final address. The assembler's list of names from section 1.1 is the last page of the compiler's notebook, kept so that the linker can find these names later.

The picture breaks in one important way. A real program has many scopes: a variable declared inside `main` is a different variable from a global of the same name, and it disappears when `main`'s block ends. So a real symbol table has one page per scope, opened when a block starts and closed when it ends, and a name is looked up on the innermost page first. That is also why this chapter's `pos.c` declares its variables outside `main`: a variable local to `main` lives in `main`'s part of the stack, has no name in the object file, and would not have shown up in `nm` at all.
<!-- checked: how=run date=2026-11-21 ref="nm on a local double in main, GCC 13.3, with and without -g: only main and printf listed" h=7614ff0d -->

**The usual mistake.** Drawing the symbol table as one of the phases, or saying "the lexical analyzer creates the symbol table" and stopping there. The scanner usually makes the entry, but every phase reads or writes it; the notebook belongs to the whole line.

**Check yourself**

<!-- q: idea=symbol-table-idea type=predict wrong="the scanner makes a new entry for every occurrence" -->
1. The statement is `pos = pos + rate * 60;`. How many symbol-table entries does the scanner make for it, and what is the token stream?

<details>
<summary>Answer</summary>

Two entries: `pos` on line 1 and `rate` on line 2. The second time the scanner meets `pos` it looks the name up, finds it already on line 1, and reuses that line, so the stream is `<id,1> <=> <id,1> <+> <id,2> <*> <60> <;>`. If you said three entries, you pictured one entry per occurrence; the notebook has one line per name, which is how all uses of `pos` end up meaning the same variable.

</details>

<!-- q: idea=symbol-table-idea type=decide wrong="the parser fills in the types" -->
2. In the table above, which phase would notice that `rat` in section 1.3's third broken program was never declared, and what would it find in the notebook?

<details>
<summary>Answer</summary>

The semantic analyzer. The scanner happily makes an entry for `rat` (it is a good identifier), but no declaration ever gives that entry a type. When the semantic analyzer goes to read the type of `rat` it finds nothing, and reports `'rat' undeclared`. The parser never looks at the notebook's types; to the parser `rat` is just `id`.

</details>

## 1.5 Passes, interpreters, cross compilers and compiler-construction tools

Four terms from the lecture turn up in short questions and in the viva: "is gcc a one-pass compiler?", "how is an interpreter different?", "what is a cross compiler?", "name some compiler-construction tools". Each is one idea, and your own machine shows all four. `--version` makes a program print its version and stop; `flex` and `bison` are the tools in the last paragraph, and `head -n 1` keeps the first line of `bison`'s answer. (On Windows, `python3` may be called `python` or `py`.)

<!-- code: id=four-terms -->
```console
$ gcc -dumpmachine
x86_64-linux-gnu
$ python3 -c 'init = 10.0; rate = 2.5; pos = init + rate * 60; print(pos)'
160.0
$ flex --version
flex 2.6.4
$ bison --version | head -n 1
bison (GNU Bison) 3.8.2
```
<!-- checked: how=reworded date=2026-11-21 ref="the next line says what to do when flex or bison is missing; Windows python3 name already noted" h=bd0fa100 -->

If `flex` or `bison` answers `command not found`, install both now; on Ubuntu that is `sudo apt install flex bison`.
<!-- checked: how=run date=2026-11-21 ref="apt-cache policy flex bison on Mint 22.3 (Ubuntu 24.04 base): flex 2.6.4, bison 3.8.2" h=be9a413e -->

<!-- teaches: compiler-pass = pass, one-pass, multi-pass -->
**A pass** is one reading of the whole program, in some form, from start to end, writing it out in another form. Phases are the logical steps of section 1.2; passes are how they are grouped when the compiler actually runs. Since the parser calls the scanner for one token at a time, the first four phases can share a single pass. A *one-pass* compiler does everything in one reading; it needs little memory, but it must handle a `goto` to a label it has not reached yet (section 9.5 shows the trick). A *multi-pass* compiler writes an intermediate form and reads it again: slower and hungrier, but with room for better optimization. GCC is plainly multi-pass: section 1.1 showed `cc1`, `as` and `collect2` each reading the whole of a file the previous one wrote, and inside `cc1` the program is walked many more times.

<!-- teaches: interpreter = interpreter -->
**An interpreter** runs the source program directly, statement by statement, instead of producing a target program to run later. The `python3` line ran the running statement and printed the answer straight away; no `a.out` appeared. The cost: the analysis is redone each time you run, and many errors, such as a misspelt name, are found only when that line runs. (Python in fact compiles each file to an intermediate *bytecode* first and then interprets that, a mix of both.)

<!-- teaches: cross-compiler = cross compiler -->
**A cross compiler** runs on one kind of machine and produces code for another. `gcc -dumpmachine` names the machine your `gcc` produces code for, here 64-bit x86 Linux, which is also the machine it runs on, so it is a native compiler. A `gcc` that runs on this laptop but writes code for an ARM board would print an ARM name instead. Cross compilers exist because the target is often too small or too slow to run a compiler, like a washing machine's microcontroller.

**Compiler-construction tools** write parts of a compiler for you from a description. *Scanner generators* take patterns and produce a scanner: Lex, and its free version Flex, which you use from chapter 4. *Parser generators* take a grammar and produce a parser: Yacc, and its free version GNU Bison, from chapter 8. The Dragon book's list goes on with syntax-directed translation engines, code-generator generators, data-flow analysis engines and compiler-construction toolkits; in the paper, name all six and describe the first two.

**The usual mistake.** Calling a compiler that is *written in* another language a cross compiler. A C compiler written in Python that produces x86 code for an x86 machine is not a cross compiler; what decides it is where the produced code runs, compared with where the compiler runs.

**Check yourself**

<!-- q: idea=cross-compiler type=decide wrong="a cross compiler is written in another language" -->
1. You build `pos` on your x86-64 laptop. Your friend copies the compiled `pos` program to her ARM-based Raspberry Pi and it will not run. Was your `gcc` a cross compiler? What would you need?

<details>
<summary>Answer</summary>

No: your `gcc` ran on x86-64 and produced x86-64 code (it says so with `-dumpmachine`), so it is a native compiler and its output runs only on x86-64. You need either a cross compiler on your laptop that produces ARM code, or to build `pos` with the native `gcc` on the Pi itself. The language the compiler is written in has nothing to do with it.

</details>

<!-- q: idea=compiler-pass type=explain wrong="phase and pass are the same" -->
2. The examiner asks: "Your compiler has six phases. Does that make it a six-pass compiler?" Answer in two sentences.

<details>
<summary>Answer</summary>

No. Phases are the logical steps; a pass is one reading of the whole program, and several phases can run in one pass, as the scanner and parser do when the parser asks for one token at a time. A compiler with six phases might have one pass or several, depending on how the phases are grouped.

</details>

## 1.6 How a lab exam program is written, built, run and marked

<!-- teaches: lab-exam-cycle = lab exam cycle -->
In the lab exam you get a question slip, a blank file, a time limit, and an examiner who will watch it run and then ask about it. The same cycle works for every question in this book, and it is worth practising on something small before the programs get big. Here is a typical warm-up question: *write a C program that reads one line and counts the operators `+ - * / =` in it.*

**1. Write the smallest version that could work.** One file, named after the job: `ops.c`. Here is a first attempt, with a bug in it that the compiler can see:

<!-- code: id=ops-warn mode=warns stdin="pos = init + rate * 60;\n" -->
```c
#include <stdio.h>

int main(void)
{
    int c, count = 0;

    while ((c = getchar()) != EOF && c != '\n') {
        if (c == '+' || c == '-' || c == '*' || c == '/')
            count++;
        if (c = '=')
            count++;
    }
    printf("operators: %d\n", count);
    return 0;
}
```
```text
ops-warn.c: In function 'main':
ops-warn.c:10:13: warning: suggest parentheses around assignment used as truth value [-Wparentheses]
   10 |         if (c = '=')
      |             ^

operators: 25
```

**2. Build with warnings on, and treat each one as a bug.** The warning points at the exact line: `c = '='` is an assignment, not a comparison. It stores `'='` in `c`, and since that value is not zero the test is always true, so `count` goes up once for every character in the line, on top of the real `+` and `*`: the running statement has 23 characters, and 23 plus 2 is the 25 it printed. The program ran and printed a confident wrong number, and without `-Wall` you might have handed it in. Fix it with `c == '='`.

**3. Run it on a normal input and an empty one,** and from chapter 2 on, one input it must reject. `printf` feeds a line to the program as if you had typed it:

<!-- code: id=ops-cycle -->
```console
$ cat > ops.c <<'END'
#include <stdio.h>

int main(void)
{
    int c, count = 0;

    while ((c = getchar()) != EOF && c != '\n') {
        if (c == '+' || c == '-' || c == '*' || c == '/')
            count++;
        if (c == '=')
            count++;
    }
    printf("operators: %d\n", count);
    return 0;
}
END
$ gcc -Wall -Wextra ops.c -o ops
$ printf 'pos = init + rate * 60;\n' | ./ops
operators: 3
$ printf '' | ./ops
operators: 0
```

No warnings, three operators in the running statement, and zero for empty input instead of a crash or a hang. The empty case is the one examiners try first, because it is the one students forget.

**4. Be ready to explain every line.** In many labs the viva carries a large share of the marks (the split between write-up, program, output and viva differs; ask for your own lab's scheme): why `c` is an `int` and not a `char` (`getchar` returns `EOF`, a value no `char` can hold), why the loop also stops at `'\n'`, why `=` is tested in an `if` of its own (it could join the others; ask yourself which reads better to an examiner). If you cannot say why a line is there, take it out or learn it before the exam.
<!-- checked: how=reworded date=2026-11-21 ref="marking split now said to vary by lab" h=3d0e840f -->

**The usual mistake.** Editing `ops.c` and running `./ops` again without rebuilding. The symptom: your change "does nothing", and the old output appears. `./ops` is the program `gcc` made last time; only `gcc` turns the new source into a new program. Rebuild after every edit, and you will never debug a program you are not running.

**Check yourself**

<!-- q: idea=lab-exam-cycle type=decide wrong="it compiles, so it is finished" -->
1. Your program builds with one warning, `unused variable 'i'`, and gives the right answer on the sample input. The examiner is walking over. What do you do in the next thirty seconds, and why?

<details>
<summary>Answer</summary>

Delete the unused variable (or use it, if you meant to), rebuild with `gcc -Wall -Wextra`, and run the sample input and an empty input once more. An unused variable is harmless on its own, but it often means a line you meant to write is missing, and a clean build is the first thing an examiner can check. "It works, so leave it" is the tempting answer; it leaves the examiner's first question pointing at your weakest line.

</details>

<!-- q: idea=lab-exam-cycle type=fix wrong="the edit was wrong" -->
2. You change the `printf` in `ops.c` to print `count = %d`, type `printf 'a+b\n' | ./ops`, and still see `operators: 1`. What happened?

<details>
<summary>Answer</summary>

You ran the old program: the edit changed `ops.c`, but `ops` was built before the edit. Run `gcc -Wall -Wextra ops.c -o ops` and try again. Hunting for a fault in the new line is the usual waste of time here; nothing is wrong with it, it simply has not been compiled.

</details>

## 1.7 Practice: trace a new statement through every phase by hand

You can now make `gcc` stop after each station and say what each file holds, name the six phases of a compiler and show what each hands on for `pos = init + rate * 60;`, split a statement into lexemes and tokens, and say what the symbol table holds and who writes it. This practice proves you can do that on statements you have not seen, by hand, the way the paper asks, and then check your hand work against what `gcc` really does. Work each task on paper before you open its answer.

### Guided

<!-- q: idea=token type=apply wrong="parentheses are not tokens" -->
**Task 1.** Write the token stream for `pos = (init + rate) * 60;` and the parser's view of it (token names only). How many tokens are there?

<details>
<summary>Answer</summary>

```text
<id,1> <=> <(> <id,2> <+> <id,3> <)> <*> <60> <;>
id = ( id + id ) * num ;
```

Ten tokens: the eight of the running statement plus the two parentheses, which are tokens like any other punctuation; each matches a one-character pattern. The symbol-table lines are unchanged, because the names appear in the same order. Leaving the parentheses out is the usual slip; the parser needs them, since they are exactly what changes the tree in task 2.

</details>

<!-- q: idea=compiler-phase type=predict wrong="the parentheses appear in the tree" -->
**Task 2.** For the same statement, draw the syntax tree and write the three-address code before and after optimization. The variables are floating-point, as in the textbook trace.

<details>
<summary>Answer</summary>

```text
          =
        /   \
   <id,1>     *
            /   \
           +     inttofloat
         /   \       |
    <id,2>  <id,3>   60
```

```text
t1 = id2 + id3
t2 = inttofloat(60)
t3 = t1 * t2
id1 = t3
```

```text
t1 = id2 + id3
id1 = t1 * 60.0
```

The `*` is now at the top under the `=`, because the parentheses forced the `+` to be done first; the operator nearest the root is done last. The parentheses themselves do not appear in the tree: their whole job was to decide its shape. The optimizer does the same two things as before: it folds the conversion of the constant and removes the copy through `t3`.

</details>

<!-- q: idea=compile-pipeline type=decide wrong="every error comes from the compiler" -->
**Task 3.** For each message, name the `gcc` station that produced it, and for those from the compiler proper, the phase: (a) `fatal error: stdllib.h: No such file or directory`; (b) `undefined reference to 'yylex'`; (c) `error: expected ';' before 'return'`; (d) `error: invalid suffix "abc" on integer constant`; (e) `error: 'count' undeclared (first use in this function)`.

<details>
<summary>Answer</summary>

(a) The preprocessor: an `#include` names a file that does not exist (`stdllib.h` for `stdlib.h`). (b) The linker: something calls `yylex` and no object file or library given to `gcc` defines it. You will see exactly this in chapter 8 if the scanner's file is left out of the build. (c) The compiler proper, syntax analysis: the tokens are fine but a `;` is missing before `return`, probably at the end of the line above. (d) The compiler proper, lexical analysis: something like `12abc` matches no pattern. (e) The compiler proper, semantic analysis: a well-formed name with no declaration. The trap is (b): it looks like a compiler error about your code, but no phase of the compiler ever complains about a missing definition; only the linker knows which names were never filled.

</details>

### On your own

<!-- q: idea=compiler-phase type=apply unseen="total-statement" wrong="only constants need inttofloat" -->
**Task 4.** Trace `total = price * qty + tax / 2;` through all six phases, as you would in the paper. `total`, `price` and `tax` are `double`; `qty` is `int`. Give the token stream, the symbol table after scanning, the tree after semantic analysis, the three-address code, the optimized code and the target code.

<details>
<summary>Answer</summary>

Tokens (ten), with the symbol table lines 1 `total`, 2 `price`, 3 `qty`, 4 `tax`:

```text
<id,1> <=> <id,2> <*> <id,3> <+> <id,4> </> <2> <;>
```

Tree after semantic analysis. Both `*` and `/` bind tighter than `+`, so they are its two children. There are two conversions: `qty` is an integer meeting the `double` `price`, and 2 is an integer meeting the `double` `tax`.

```text
              =
            /   \
       <id,1>     +
               /     \
              *        /
            /   \     /  \
       <id,2>  ITF  <id,4>  ITF
                |            |
              <id,3>         2
```

(ITF stands for `inttofloat`, shortened to fit.) Three-address code, left subtree first:

```text
t1 = inttofloat(id3)
t2 = id2 * t1
t3 = inttofloat(2)
t4 = id4 / t3
t5 = t2 + t4
id1 = t5
```

Optimized: the constant's conversion is folded into `2.0` and the final copy is removed. The conversion of `id3` stays: `qty` is a variable, so its value is not known until the program runs.

```text
t1 = inttofloat(id3)
t2 = id2 * t1
t3 = id4 / 2.0
id1 = t2 + t3
```

Target code, in the Dragon book's style. Its simple machine has no instruction for converting an integer, so write one and say what it means in a line: here `ITOF R1, R1` converts the integer in R1 to floating-point.

```text
LD   R1, id3
ITOF R1, R1
LDF  R2, id2
MULF R2, R2, R1
LDF  R3, id4
DIVF R3, R3, #2.0
ADDF R2, R2, R3
STF  id1, R2
```

The likely mistake is a single conversion, of the 2 only, because that is where the textbook example puts it. The rule is the same everywhere: a conversion goes wherever an integer value meets a floating-point operand, constant or not.

</details>

<!-- q: idea=cross-compiler type=decide unseen="engine-controller" wrong="small devices use interpreters" -->
**Task 5.** A team writes two programs on their x86-64 laptops. One runs on a car's engine controller, a small chip with little memory that must react within microseconds. The other is a calculator for the team's own use, where they type a formula and want the answer at once, with no build step. For each, should they use a native compiler, a cross compiler or an interpreter? Give the reason.

<details>
<summary>Answer</summary>

The engine controller: a cross compiler. The code must run on the controller's chip, not on the laptop, and the chip is too small to run a compiler itself; a compiled program also runs fast, with no analysis left for run time. The calculator: an interpreter. It runs each formula straight away on the machine where it was typed, and there is no target program to keep. The tempting wrong answer for the controller is "an interpreter, because it is small", but an interpreter has to live on the chip and redo the analysis while the engine waits.

</details>

<!-- q: idea=token type=explain wrong="the parser could just read characters" -->
**Task 6.** In the viva the examiner asks: "Why have a separate lexical analyzer at all? Couldn't the parser read the characters itself?" Answer in three or four sentences, using the running statement.

<details>
<summary>Answer</summary>

It could, but its grammar would then have to describe spelling as well as structure: how names are spelled, how numbers are spelled, and where spaces and comments may go, all mixed into the rules for expressions. With a scanner in front, the parser sees `id = id + id * num ;`, and one grammar rule for `id` covers `pos`, `init`, `rate` and every other name. The scanner does the simple, character-by-character work (which is also where a compiler spends much of its time reading input), and the parser does only the structure. Each half is smaller and easier to get right, and the scanner's patterns can be written and tested on their own, which is exactly what chapters 2 to 4 do.

</details>

### Chapter project: check your hand trace against gcc

<!-- q: idea=compiler-phase type=build -->
**Task 7.** In `~/cdlab/ch1`, write `total.c`: a complete C program with the globals `double total, price = 12.5, tax = 3.0;` and `int qty = 4;`, whose `main` runs the statement `total = price * qty + tax / 2;` and prints the result as `total = ` followed by the value with `%g`, then a newline. Build it with `gcc -Wall -Wextra` and no warnings. Then make GCC dump its three-address code (`-fdump-tree-gimple`, as in section 1.2) and compare it, line by line, with your answer to task 4: find where GCC puts the conversion of `qty`, and what it did with `tax / 2`.

You are done when your program prints exactly what the model answer prints, and you can point to the GIMPLE line that matches each line of your hand-written three-address code. To check the first part with no help, open a terminal in the understudy folder (the one that holds `book/`) and type `python3 scripts/us_practice.py check compiler-design 1.7 --in ~/cdlab/ch1`.

<details>
<summary>Answer</summary>

<!-- code: id=total check="gcc -Wall -Wextra total.c -o total && ./total" -->
```c
#include <stdio.h>

double total, price = 12.5, tax = 3.0;
int qty = 4;

int main(void)
{
    total = price * qty + tax / 2;
    printf("total = %g\n", total);
    return 0;
}
```
```text
total = 51.5
```

12.5 times 4 is 50, plus 3.0 divided by 2 is 1.5, which makes 51.5. Here is GCC's three-address code for the same program. The answer key is written to `total-key.c` so that it does not overwrite your own `total.c`:

<!-- code: id=total-gimple -->
```console
$ cat > total-key.c <<'END'
#include <stdio.h>

double total, price = 12.5, tax = 3.0;
int qty = 4;

int main(void)
{
    total = price * qty + tax / 2;
    printf("total = %g\n", total);
    return 0;
}
END
$ gcc -c -fdump-tree-gimple total-key.c
$ cat total-key.c.*.gimple
int main ()
{
  int D.3182;

  {
    qty.0_1 = qty;
    _2 = (double) qty.0_1;
    price.1_3 = price;
    _4 = _2 * price.1_3;
    tax.2_5 = tax;
    _6 = tax.2_5 / 2.0e+0;
    _7 = _4 + _6;
    total = _7;
    total.3_8 = total;
    printf ("total = %g\n", total.3_8);
    D.3182 = 0;
    return D.3182;
  }
  D.3182 = 0;
  return D.3182;
}

```

Match it against task 4. GCC copies each global into a temporary first, as it did for `pos.c`. The line `_2 = (double) qty.0_1;` is your `t1 = inttofloat(id3)`: `qty` is a variable, so the conversion has to happen at run time, and GCC keeps it. `_4 = _2 * price.1_3;` is your `t2 = id2 * t1`, with the operands swapped. The 2 never appears as an integer at all: `_6 = tax.2_5 / 2.0e+0;` divides by the floating-point constant 2.0, the same folding you did by hand in your optimized `t3 = id4 / 2.0`. Then `_7 = _4 + _6;` and `total = _7;` are your last two lines, one operation each. If your version has a separate `inttofloat(2)`, that is right for unoptimized three-address code in the paper; GCC simply folds constants earlier than the textbook does.

</details>
