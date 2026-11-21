# Chapter 8: Yacc and Bison programs

<!-- contents -->
- [8.1 The first Yacc program: the parts of a .y file and linking it with Lex](#81-the-first-yacc-program-the-parts-of-a-y-file-and-linking-it-with-lex)
- [8.2 Validating arithmetic expressions](#82-validating-arithmetic-expressions)
- [8.3 A calculator: values, precedence and associativity](#83-a-calculator-values-precedence-and-associativity)
- [8.4 Recognising languages: identifiers, a^n b^n and other classics](#84-recognising-languages-identifiers-an-bn-and-other-classics)
- [8.5 Reading Bison's conflict reports with -v and the .output file](#85-reading-bisons-conflict-reports-with--v-and-the-output-file)
- [8.6 Infix to postfix with semantic actions](#86-infix-to-postfix-with-semantic-actions)
- [8.7 Practice: the lab set of Yacc programs, written from the question alone](#87-practice-the-lab-set-of-yacc-programs-written-from-the-question-alone)
<!-- /contents -->

## 8.1 The first Yacc program: the parts of a .y file and linking it with Lex

### A table nobody should have to type

Section 7.7 ended with `lr.c`, an SLR parser whose ACTION and GOTO tables you built by hand and typed in, row by row. For the expression grammar that was twelve states and nine columns, and one wrong entry gave a parser that accepted nonsense or rejected good input. `lr1.h` showed that a program can build the tables instead, but it only printed them; turning them into a working parser was still your job. For a real language, with a few hundred states, nobody types the table.

Chapter 4 solved the same problem for the scanner. You wrote the patterns, Flex built the DFA and wrote the C. A parser generator does that for the parser: you write the grammar, and next to each production the C to run when the parser reduces by it. The generator builds the LALR(1) table of section 7.5 and writes a C file holding the table and the driver of section 7.7. The original, written at Bell Labs in the 1970s, is called Yacc, "yet another compiler-compiler"; the free rewrite on every Linux machine is GNU Bison, and on most machines the command `yacc` runs Bison in its Yacc mode (`bison -y`). On the book's machine that is Bison 3.8.2. The lab asks for "a Yacc program", and every program in this chapter builds with `yacc` as typed here; if a lab machine has no `yacc`, type `bison -y` in its place.
<!-- risk: Yacc by Stephen C. Johnson at Bell Labs, early 1970s; check the attribution and date -->

This section writes the smallest useful Yacc program twice: first with a scanner written by hand inside the Yacc file, so that you can see exactly what the parser asks of a scanner, and then with the scanner written in Lex, which is how every lab program in this chapter is built.

### The second station, built from a sheet

In the assembly-line picture of section 1.2, Flex built the first station from your specification sheet. Yacc builds the second one the same way: you hand it the grammar, it hands back the parser as C. The parser it builds is the LR parser of chapter 7, the stack of plates with a state written on every plate.

The picture breaks in the way chapter 1 warned. The stations do not run one after the other. The parser is in charge: whenever it needs the next token, it calls the scanner, which reads just far enough to find one token and returns it. So the scanner is not a station that finishes and passes on a box of tokens; it is a function the parser calls, dozens of times. That is why a Lex scanner for Yacc has a `return` in its actions (section 4.4), and why the two files must agree on one thing above all: the number that stands for each kind of token.

### The three parts of a .y file

<!-- teaches: yacc-structure = Yacc specification, %token, Yacc action -->
A Yacc program is a text file whose name ends in `.y`, called a *Yacc specification*. It has the same three parts as a Lex file, separated by `%%` lines:

```text
declarations
%%
rules
%%
user code
```

The *declarations* hold C between `%{` and `%}`, copied to the top of the output, and the declarations Yacc itself reads, each starting with `%`. The first of these is `%token`, which names the kinds of token that are more than one character: `%token ID NUM` says that the scanner will return the codes `ID` and `NUM`. A token that is a single character, such as `=` or `;`, needs no declaration: it is written in the grammar in single quotes, `'='`, and its code is the character's own code.

The *rules* are the grammar, one production per line, in a notation close to the one in chapter 5. `E -> E + T | T` is written

```text
e : e '+' t
  | t
  ;
```

with a colon for the arrow, a bar between the alternatives, and a semicolon after the last one. Nonterminals are plain names, written in lower case by custom so that they stand out from the token names in capitals. The left side of the first rule is the start symbol. Each production may end with an *action*: C code in braces, run when the parser reduces by that production. Inside an action, `$1`, `$2`, `$3` are the values of the first, second and third symbols of the right side, and `$$` is the value of the left side (section 8.3 uses that).

The *user code* is plain C, copied to the end of the output. It must provide two functions the parser calls but does not define: `yylex`, the scanner, and `yyerror`, which the parser calls with a message such as `syntax error` when the input cannot be parsed. A program that runs on its own also needs a `main`.

### hand.y: a parser with its own scanner

The first program reads lines of the form `name = number;`, the simplest Tiny assignment, and says the value of each. Make this chapter's folder; open a terminal in your home folder:

<!-- code: id=ch8-setup -->
```console
$ mkdir cdlab/ch8
$ cd cdlab/ch8
```

Save the program as `hand.y`:

<!-- code: id=hand-write -->
```console
$ cat > hand.y <<'END'
%{
/* hand.y: Tiny assignments of a number, with a scanner written by hand */
#include <stdio.h>
#include <ctype.h>
int yylex(void);
void yyerror(const char *s);
%}
%token ID NUM
%%
lines : /* nothing */
      | lines line
      ;
line  : ID '=' NUM ';' '\n'   { printf("assignment of %d\n", $3); }
      ;
%%
int yylex(void)
{
    int c = getchar();

    while (c == ' ' || c == '\t')
        c = getchar();
    if (isalpha(c)) {
        while (isalnum(c = getchar()))
            ;
        ungetc(c, stdin);
        return ID;
    }
    if (isdigit(c)) {
        yylval = c - '0';
        while (isdigit(c = getchar()))
            yylval = yylval * 10 + (c - '0');
        ungetc(c, stdin);
        return NUM;
    }
    if (c == EOF)
        return 0;
    return c;
}

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

Read it part by part.

**Declarations.** The `%{ ... %}` block includes two headers and declares `yylex` and `yyerror`, as habit 5 asks: the parser that Yacc writes calls both, and both are defined only further down, in the user code. Then `%token ID NUM`.

**Rules.** Two nonterminals. `lines` is a list of any number of `line`s, possibly none; the empty alternative is written as nothing at all, with a comment so that a reader knows the blank is meant. The list is left-recursive, `lines : lines line`, exactly the kind of production that sent chapter 6's parsers round in circles and that an LR parser handles with no trouble. `line` is one assignment: a name, `=`, a number, `;` and the newline. Its action prints `$3`, the value of the third symbol, `NUM`.

**User code.** `yylex` is a hand scanner in the style of chapter 2, made small: skip blanks, read a name or a number, and return anything else as the character itself, newline included. At the end of the input it returns 0. `yyerror` prints its message. `main` calls `yyparse` and passes on what it returns.

### Build it and run it

Building takes one command more than a C program and one fewer than a Lex program:

<!-- code: id=hand-build -->
```console
$ yacc hand.y
$ ls
hand.y  y.tab.c
$ gcc -Wall -Wextra y.tab.c -o hand
$ printf 'init = 10;\nrate = 25;\n' | ./hand
assignment of 10
assignment of 25
```

`yacc hand.y` printed nothing, so it found nothing wrong with the grammar. It wrote `y.tab.c`, a name as fixed as Flex's `lex.yy.c`: whatever the grammar file is called, Yacc writes `y.tab.c`. GCC compiled it, with your `yylex`, `yyerror` and `main` inside it, into `hand`, with no warnings. Two lines went in, and the action ran once for each, printing the number. Now an empty input and an input with a mistake, as habit 3 asks:

<!-- code: id=hand-errors -->
```console
$ ./hand < /dev/null; echo "exit $?"
exit 0
$ printf 'init = 10;\nrate = ;\n' | ./hand; echo "exit $?"
assignment of 10
syntax error
exit 1
```

The empty input is accepted: `lines` can be nothing, so the parser met the end of input in a state where it was allowed, and `yyparse` returned 0. On the second input the first line was parsed and acted on before the second was even read. On the second line the parser met `;` where a `NUM` had to come, called `yyerror("syntax error")` and returned 1, which `main` passed on as the exit status.

### What yyparse does with yylex

<!-- teaches: yyparse = yyparse, token code -->
The program never calls `yylex`. `yyparse`, the function Yacc wrote, does: it is the LR driver of section 7.7, with the table built from your grammar. Each time the driver needs a lookahead, it calls `yylex()` and uses the number it gets back as the token. That number has to be one the table knows:

- for a one-character token, it is the character's code: `'='` in the grammar is the number `'='` returned by `yylex`, which is why `return c;` is all `hand.y`'s scanner does for `=`, `;` and the newline;
- for a `%token`, it is a number that Yacc chose, starting at 258 so that it can never be mistaken for a character;
- for the end of input it is 0, the `$` of chapter 7. A scanner that returns 0 too early has told the parser that the input is over.

`yyparse` returns 0 when it accepts and 1 when it gives up on a syntax error, after calling `yyerror`. It calls `yyerror` with the text `syntax error` and nothing else; anything more, such as a line number, your `yyerror` must add itself (section 8.7 does).

You can see the numbers Yacc chose with `-d`, which tells it to write a second file, the header `y.tab.h`:

<!-- code: id=hand-header -->
```console
$ yacc -d hand.y
$ grep -n 'define ID\|define NUM\|extern YYSTYPE\|typedef int' y.tab.h
67:#define ID 258
68:#define NUM 259
72:typedef int YYSTYPE;
78:extern YYSTYPE yylval;
```

`ID` is 258 and `NUM` is 259, in the order of the `%token` line. Then the last line: a variable `yylval` of type `YYSTYPE`, which is `int` unless you say otherwise.

### yylval: the value that travels with a token

<!-- teaches: yylval = yylval, YYSTYPE -->
A token code says what kind of thing was found: a number. It does not say which number. The parser's table only needs the kind, but the actions need the value, so the scanner puts it in a global variable, `yylval`, before it returns. `hand.y`'s scanner does exactly that: it builds the number's value in `yylval` digit by digit, then returns `NUM`.

The parser copies `yylval` at the moment it shifts the token, and keeps the copy on its stack, beside the state, in a second stack that grows and shrinks with the first. When it later reduces by `line : ID '=' NUM ';' '\n'`, five symbols are on top of the stack, each with its value; `$1` is the value that came with `ID`, `$3` the one that came with `NUM`. So the picture of a plate with a state on it grows one more line: each plate carries a state, a symbol and a value. Here is the stack at that moment, for the line `init = 10;`:

| | `$1` | `$2` | `$3` | `$4` | `$5` |
|---|---|---|---|---|---|
| symbol | `ID` | `'='` | `NUM` | `';'` | `'\n'` |
| value | whatever `yylval` held | whatever `yylval` held | 10 | whatever `yylval` held | whatever `yylval` held |

Only `NUM` set `yylval`, so only `$3` means anything; for the other tokens the stack holds whatever value `yylval` happened to have when they were shifted. The value of a token is copied when it is *shifted*, not when the scanner finds it. That difference is why section 8.6 warns against reading `yytext` in an action.

### first.l and first.y: the scanner in Lex

In the lab, the scanner is a Lex program. Move `hand.y`'s scanner into its own file, `first.l`, and keep only the parser in `first.y`. The two must agree on the token codes, and `y.tab.h` is how they agree: the Yacc file declares the tokens, `yacc -d` writes their numbers into `y.tab.h`, and the Lex file includes it.

<!-- code: id=first-write -->
```console
$ cat > first.y <<'END'
%{
/* first.y: Tiny assignments of a number; the scanner is first.l */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
%}
%token ID NUM
%%
lines : /* nothing */
      | lines line
      ;
line  : ID '=' NUM ';' '\n'   { printf("assignment of %d\n", $3); }
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
$ cat > first.l <<'END'
%{
/* first.l: the scanner for first.y */
#include <stdlib.h>
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
[A-Za-z_][A-Za-z0-9_]*   { return ID; }
[0-9]+                   { yylval = atoi(yytext); return NUM; }
[ \t]+                   { /* blanks */ }
\n                       { return '\n'; }
.                        { return yytext[0]; }
%%
END
```

`first.y` is `hand.y` with the scanner taken out. `first.l` is the scanner written as patterns. Its definitions include `y.tab.h`, which gives it `ID`, `NUM` and `yylval`. Its rules return `ID` for a name and `NUM` for a number, after putting the number's value in `yylval`. A newline is returned as itself, because the grammar uses it. The last rule, `.`, returns every other character as its own code, `yytext[0]`, so `=` and `;` reach the parser as `'='` and `';'`. Its user code section is empty: `main` is in `first.y`, and there is no `yywrap` to write, because of `%option noyywrap`.

Now the build. The order matters, because each step makes what the next one needs:

<!-- code: id=first-build -->
```console
$ yacc -d first.y
$ flex first.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o first
$ printf 'init = 10;\nrate = 25;\n' | ./first
assignment of 10
assignment of 25
```

1. `yacc -d first.y` writes `y.tab.c`, the parser, and `y.tab.h`, the token codes and `yylval`.
2. `flex first.l` writes `lex.yy.c`, the scanner, which includes `y.tab.h`; so `yacc -d` must come first.
3. `gcc` compiles both C files into one program. The parser in `y.tab.c` calls `yylex`, defined in `lex.yy.c`; the scanner in `lex.yy.c` writes `yylval`, defined in `y.tab.c`; `main` is in `y.tab.c`. Neither file links without the other.

The output is the same as `hand`'s, which is the test that the move to Lex changed nothing.

### When it goes wrong

Each of the usual mistakes gives a message that names the missing piece; read it for the name, and you know which step was skipped.

**`yacc` without `-d`.** `y.tab.h` is never written, and compiling the scanner stops at its include:

<!-- code: id=first-no-d exit=1 -->
```console
$ mkdir nod
$ cp first.y first.l nod
$ cd nod
$ yacc first.y
$ flex first.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o first
first.l:4:10: fatal error: y.tab.h: No such file or directory
    4 | #include "y.tab.h"
      |          ^~~~~~~~~
compilation terminated.
```

The message names `first.l`, line 4, although GCC was compiling `lex.yy.c`: Flex marks every line it copies with the place it came from, so errors point at your file, not at the generated one. The cure is `yacc -d`. A `y.tab.h` left over from another program is worse than a missing one: it compiles, and the token numbers belong to a different grammar.

**Only one of the two C files given to GCC.** The linker lists what the other file would have supplied:

<!-- code: id=first-one-file exit=1 varies -->
```console
$ cd ..
$ gcc -Wall -Wextra y.tab.c -o first
/usr/bin/ld: /tmp/ccX3wvKQ.o: in function `yyparse':
y.tab.c:(.text+0x318): undefined reference to `yylex'
collect2: error: ld returned 1 exit status
```

`undefined reference to yylex` from `yyparse`: the scanner is missing, so `lex.yy.c` was left off the line. Leave off `y.tab.c` instead and the complaint is about `main` and `yylval`.

**The declarations of habit 5 left out.** GCC 13 warns `implicit declaration of function 'yylex'` and `'yyerror'` and still builds the program. GCC 14 makes the same thing an error, and the program does not build at all. Write the two declarations in every Yacc file and the question never comes up.

**A token the scanner never returns.** If `first.l` had no `\n` rule, the newline would not reach the parser at all: `.` does not match a newline, so no rule matches it, and a character no rule matches is copied to the output by Flex (section 4.2). The parser would wait for a `'\n'` that never comes and report `syntax error` on the next line's `ID`. Every character the grammar names must be returned by the scanner, and every character the scanner returns must appear somewhere in the grammar.
<!-- risk: not run: first.l without the \n rule; reasoning from flex's default ECHO rule and the grammar needing '\n' -->

**Two `main`s.** A Lex file copied from chapter 4 still has its own `main`, and the link fails with `multiple definition of 'main'`. When Yacc is in charge, `main` lives in the `.y` file and calls `yyparse`; the scanner's file has none.

### In one line each

1. A `.y` file has declarations (`%{ C %}`, `%token`), rules (`a : b c | d ;` with actions in braces) and user code (`yyerror`, `main`, and `yylex` if no Lex file).
2. `yacc -d file.y` writes `y.tab.c` (the parser, `yyparse`) and `y.tab.h` (token codes from 258, `yylval`); `flex` comes after it, then `gcc y.tab.c lex.yy.c`.
3. `yyparse` calls `yylex` for each token; one-character tokens are their own codes, 0 is the end of input.
4. The scanner puts a token's value in `yylval` before returning; the parser copies it on the shift and the action reads it as `$n`.
5. `yyparse` returns 0 on accept and 1 after calling `yyerror("syntax error")`.

**Check yourself**

<!-- q: idea=yacc-structure type=predict wrong="the first rule written is the last one used" -->
1. Swap the two rules of `first.y`, so that `line : ...` comes first and `lines : ...` second. Predict what `./first` does with the same two-line input, and why. Then try it in a copy.

<details>
<summary>Answer</summary>

The left side of the first rule is the start symbol, so now the start symbol is `line`: the whole input must be one line. The first line is parsed and its action prints `assignment of 10`; then the parser has a complete `line` and expects the end of input, but gets `ID` for `rate`: `syntax error`, exit status 1. `lines` is no longer reachable from the start symbol, and Bison says so with a warning when it builds the parser.

The `sed` lines below build the swapped copy in a folder `swap`: lines 1 to 8 of `first.y` (the declarations and the first `%%`), then the two lines of the `line` rule, then the three lines of the `lines` rule and everything after them.

<!-- code: id=first-swap -->
```console
$ mkdir swap
$ sed -n '1,8p' first.y > swap/first.y
$ sed -n '12,13p' first.y >> swap/first.y
$ sed -n '9,11p;14,$p' first.y >> swap/first.y
$ cp first.l swap
$ cd swap
$ yacc -d first.y
first.y: warning: 1 nonterminal useless in grammar [-Wother]
first.y: warning: 2 rules useless in grammar [-Wother]
first.y:11.1-5: warning: nonterminal useless in grammar: lines [-Wother]
   11 | lines : /* nothing */
      | ^~~~~
$ flex first.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o first
$ printf 'init = 10;\nrate = 25;\n' | ./first; echo "exit $?"
assignment of 10
syntax error
exit 1
$ cd ..
```

The rule `%start lines` in the declarations would name the start symbol and make the order of the rules not matter. Most lab programs simply put the start symbol's rule first.

</details>

<!-- q: idea=yylval type=fix wrong="the action can print the number as yytext" -->
2. A student writes the `NUM` rule of `first.l` as `[0-9]+ { return NUM; }`, without the `yylval` line. Everything builds with no warning. What does `./first` print for `init = 10;`, and why does no tool complain?

<details>
<summary>Answer</summary>

`assignment of 0`: nothing ever stored a value in `yylval`, a global `int`, which starts at 0, so the parser copies 0 onto its stack when it shifts `NUM`, and `$3` is 0. The program is legal C and the grammar is correct; only the meaning is lost. Had an earlier token set `yylval`, the number printed would be that token's value, which is worse, because it looks plausible. Every token whose value an action uses must set `yylval` in the scanner.

</details>

<!-- q: idea=yyparse type=decide wrong="any distinct numbers will do as token codes" -->
3. A student joins chapter 4's `tiny.l`, whose definitions still hold its own `enum { END, ID, NUM, IF, ... }`, to a Yacc grammar that declares `%token ID NUM ...`, and deletes the `#include "y.tab.h"` line because "the enum already defines the names". Everything compiles. What does the parser receive for the first name in the input, and what does it do?

<details>
<summary>Answer</summary>

The scanner returns its own `ID`, which in that enum is 1. The parser's `ID` is 258, from `y.tab.h`, and to the parser 1 is not `ID`: codes below 256 are characters, and 1 is the control character Ctrl-A, which no rule mentions. So the parser reports `syntax error` at the very first token of every input. Worse, the scanner's `END` is 0, which happens to be right, so the empty input is accepted and the program seems to half work. The two files must take the codes from one place, `y.tab.h`; section 8.7 makes exactly this change to `tiny.l`.

</details>

## 8.2 Validating arithmetic expressions

### The first question on the lab sheet

Almost every lab sheet's Yacc list starts the same way: "Write a program using Lex and Yacc to recognise a valid arithmetic expression that uses the operators +, -, * and /." The program says *valid* or *invalid*, and nothing else. It is the smallest program that needs a real grammar, and you already own that grammar: the expression grammar of section 5.2, `E -> E + T | T`, `T -> T * F | F`, `F -> ( E ) | id`, with `-` and `/` added beside `+` and `*`. Chapter 6 had to rewrite it before a top-down parser could use it, and chapter 7 built its LR tables by hand. Here you type it as it stands, and Yacc does the rest.

This section writes the program twice: first in the shape the lab usually expects, one expression per run; then in a shape that is easier to test, one expression per line, where a bad line is reported and the next line is still checked. Then it switches on Bison's own trace, so that you can watch the LR parser of chapter 7 do its moves on your input.

### valid.y: one expression, valid or invalid

The scanner returns a name or a number as one token each, and every other character as itself. Since the grammar does not care which name or which number it saw, it can use one token, `ID`, for both. Save `valid.l`, then `valid.y`:

<!-- code: id=valid-write -->
```console
$ cat > valid.l <<'END'
%{
/* valid.l: the scanner for valid.y and expr.y; names and numbers are ID */
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
[A-Za-z_][A-Za-z0-9_]*      { return ID; }
[0-9]+(\.[0-9]+)?           { return ID; }
[ \t]+                      { /* blanks */ }
\n                          { return '\n'; }
.                           { return yytext[0]; }
%%
END
$ cat > valid.y <<'END'
%{
/* valid.y: is the line a valid arithmetic expression? */
#include <stdio.h>
#include <stdlib.h>
int yylex(void);
void yyerror(const char *s);
%}
%token ID
%%
line : e '\n'
     ;
e    : e '+' t
     | e '-' t
     | t
     ;
t    : t '*' f
     | t '/' f
     | f
     ;
f    : '(' e ')'
     | ID
     ;
%%
void yyerror(const char *s)
{
    printf("invalid expression (%s)\n", s);
    exit(1);
}

int main(void)
{
    yyparse();
    printf("valid expression\n");
    return 0;
}
END
```

The rules are section 5.2's grammar letter for letter, with `e`, `t` and `f` for `E`, `T` and `F`, two more alternatives for `-` and `/`, and `ID` for `id`. The start symbol is `line`: one expression followed by the newline that ends the line. No production has an action. A validator does not need any: reaching the end of `yyparse` without an error *is* the answer.

That is how `main` and `yyerror` share the work. `yyerror` is called only on a syntax error; it prints `invalid expression` with the parser's message and ends the program with `exit(1)`, so it never returns to `yyparse`. If `yyparse` returns at all, the input was accepted, and `main` prints `valid expression`. Both messages go to standard output, as the lab expects them on the screen.

Build it and test it three ways, as habit 3 asks:

<!-- code: id=valid-run -->
```console
$ yacc -d valid.y
$ flex valid.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o valid
$ echo 'init + rate * 60' | ./valid
valid expression
$ echo '(a + b) * (c - 2.5) / d' | ./valid
valid expression
$ echo 'a + * b' | ./valid; echo "exit $?"
invalid expression (syntax error)
exit 1
$ echo '' | ./valid; echo "exit $?"
invalid expression (syntax error)
exit 1
```

The running statement's expression is valid, and so is a longer one with every operator and two levels of parentheses. `a + * b` is not: after `+` the parser needs the start of a `t`, which is `(` or `ID`, and finds `*`. The empty line is invalid too, because `line` needs an `e` before the newline. Whether an empty input counts as valid is a decision; this grammar says no, and question 2 below changes it.

### Where the parser stopped

`valid` says *that* the input is wrong, not *where*. In the LR parser of chapter 7, an error is a blank ACTION entry: the state on top of the stack has no action for the lookahead. For `a + * b`, the moves are:

| Stack | Input | Action |
|---|---|---|
| `$` | `a + * b \n` | shift `ID` |
| `$ ID` | `+ * b \n` | reduce f -> ID |
| `$ f` | `+ * b \n` | reduce t -> f |
| `$ t` | `+ * b \n` | reduce e -> t |
| `$ e` | `+ * b \n` | shift `+` |
| `$ e +` | `* b \n` | error: no action for `*` |

An LR parser finds the error at the first token that cannot continue any valid input, here the `*`. Everything before it, `a +`, is the start of many good expressions; nothing that starts `a + *` is. This *valid-prefix property* is why Yacc's errors are reported at the right token, and it is the reason the moves above stop where they do, without reading the `b`.

### expr.y: one line at a time, and the error token

<!-- teaches: yacc-validator = expression validator, error token, yyerrok -->
Testing `valid` takes a new run for every expression. A better tester reads many lines and judges each one, which needs two things: a list of lines in the grammar, as in `first.y`, and a way to carry on after a bad line instead of stopping.

Yacc's way to carry on is a special token called `error`. You put it in a production, and the parser uses that production when something goes wrong:

```text
line : e '\n'          { printf("valid\n"); }
     | error '\n'      { yyerrok; }
     ;
```

Read the second alternative as "a line can also be: some mistake, then a newline". When the parser meets a blank ACTION entry it calls `yyerror`, and then, instead of giving up:

1. it pops states off its stack until it uncovers one that can shift `error`: here, the state just after `lines`, which expects the start of a `line`;
2. it shifts `error` as if it were a token;
3. it throws away input tokens until it finds one that can follow `error`: here, `'\n'`;
4. it shifts the newline and reduces by `line : error '\n'`, and parsing goes on with the next line.

This is panic-mode recovery, the same idea as section 6.5's synchronising tokens, with the newline as the one synchronising token. After a recovery the parser stays quiet about new errors until it has shifted three tokens, so that one mistake does not produce a burst of messages. `yyerrok` in the action says "the recovery is over, report the next error at once"; without it, an error in the line right after a bad line can go unreported, as you will see.

Save `expr.y`. It uses the same scanner, `valid.l`:

<!-- code: id=expr-write -->
```console
$ cat > expr.y <<'END'
%{
/* expr.y: is each line a valid arithmetic expression? */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
%}
%token ID
%%
lines : /* nothing */
      | lines line
      ;
line  : e '\n'          { printf("valid\n"); }
      | error '\n'      { yyerrok; }
      ;
e     : e '+' t
      | e '-' t
      | t
      ;
t     : t '*' f
      | t '/' f
      | f
      ;
f     : '(' e ')'
      | ID
      ;
%%
void yyerror(const char *s)
{
    printf("invalid (%s)\n", s);
}

int main(void)
{
#if YYDEBUG
    yydebug = 1;        /* built with yacc -t: print every move */
#endif
    return yyparse();
}
END
```

Two things are new besides the list of lines and the `error` production. `yyerror` now only prints, without `exit`, because the parser is going to recover. And `main` has three lines between `#if YYDEBUG` and `#endif`, which do nothing yet; the next part uses them.

<!-- code: id=expr-run -->
```console
$ yacc -d expr.y
$ flex valid.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o expr
$ printf 'init + rate * 60\na +\n(a * b\n2.5 / (x - y)\n)a\nb\n' | ./expr
valid
invalid (syntax error)
invalid (syntax error)
valid
invalid (syntax error)
valid
```

Six lines, six answers, in order: the running expression; `a +`, which ends where a `t` must start; `(a * b`, a parenthesis never closed; a good one with a number and parentheses; `)a`, wrong at its first token; and a lone name, which is a valid expression. Each bad line was reported, and the next line was judged on its own.

To see what `yyerrok` is for, build a copy without it and give it two bad lines in a row:

<!-- code: id=expr-noerrok -->
```console
$ sed 's/{ yyerrok; }//' expr.y > noerrok.y
$ yacc -d noerrok.y
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o noerrok
$ printf 'a +\n)\nb * c\n' | ./noerrok
invalid (syntax error)
valid
```

Three lines, two answers. After the error in `a +`, the parser recovered and shifted `error` and the newline; then the `)` of the second line was a new error, but fewer than three tokens had been shifted since the last one, so the parser threw the `)` away *without calling `yyerror`*, and the second line simply vanished. `b * c` was reported as valid, and nothing said that line 2 was ever wrong. `flex valid.l` was not run again: `lex.yy.c` includes `y.tab.h`, and the new `y.tab.h` gives `ID` the same code, 258, because the `%token` line is the same, so the old `lex.yy.c` still fits.

### Watching the LR parser at work

The parser in `y.tab.c` is the LR driver of section 7.7, and Bison will show you its moves. `yacc -t` builds the parser with its trace code included, and the variable `yydebug`, when it is not 0, switches the trace on. `-t` also defines `YYDEBUG` as 1, which is what the three lines in `main` test, so the same `expr.y` builds into a silent program without `-t` and a talkative one with it. The trace goes to standard error; `2>&1` joins it to the output so that `grep` can pick out the moves:

<!-- code: id=expr-trace -->
```console
$ yacc -d -t expr.y
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o exprt
$ echo 'a+b*c' | ./exprt 2>&1 | grep -E '^(Shifting|Reducing|Next token)'
Reducing stack by rule 1 (line 9):
Next token is token ID ()
Shifting token ID ()
Reducing stack by rule 12 (line 24):
Reducing stack by rule 10 (line 21):
Next token is token '+' ()
Reducing stack by rule 7 (line 17):
Next token is token '+' ()
Shifting token '+' ()
Next token is token ID ()
Shifting token ID ()
Reducing stack by rule 12 (line 24):
Reducing stack by rule 10 (line 21):
Next token is token '*' ()
Shifting token '*' ()
Next token is token ID ()
Shifting token ID ()
Reducing stack by rule 12 (line 24):
Reducing stack by rule 8 (line 19):
Next token is token '\n' ()
Reducing stack by rule 5 (line 15):
Next token is token '\n' ()
Shifting token '\n' ()
Reducing stack by rule 3 (line 12):
Reducing stack by rule 2 (line 10):
Shifting token "end of file" ()
```

Bison numbers the rules in the order you wrote them, from 1, after its own rule 0 (`$accept : lines $end`, the augmented production S' -> S of section 7.3). Each alternative is a rule of its own, so `lines : /* nothing */` is rule 1, `f : ID` is rule 12, and `(line 24)` is where rule 12 stands in `expr.y`. The very first move is a reduction by rule 1, before any token is read: the list of lines starts empty, as section 7.7's parser did for B -> ε. The last move shifts `"end of file"`, Bison's name for the `$` of chapter 7, and the accept follows it. Between the two, read the trace against the moves of section 7.7 for `id + id * id`, and they are the same moves in the same order: reduce the first `ID` all the way to `e` (three reductions), because the next token is `+`; shift `+`; reduce the second `ID` only to `t`, because the next token is `*`, and shift the `*` instead of reducing `t` to `e`; reduce `t * f` to `t`; then `e + t` to `e` when the newline arrives.

One thing differs, and it is worth seeing. After each `ID` is shifted, the trace goes straight to `Reducing`, with no `Next token` line in between. Bison has noticed that the state after `ID` has only one thing it can do, reduce by `f : ID`, whatever comes next, so it reduces without reading the lookahead. This is called a *default reduction*. It does not change which moves are made, only when the next token is read, and section 8.6 shows a program whose output depends on exactly that.


The trace prints much more than these lines: the state entered after every move (`Entering state 4`) and the whole stack of states (`Stack now 0 1 8 14`), which is the stack column of chapter 7's traces. Run `./exprt` without the `grep` once to see them. The state numbers are Bison's own and will not match the numbers in a hand-built collection; section 8.5 shows where Bison prints its states.

### When it goes wrong

**Every line is invalid, even `a`.** The grammar has no `'\n'` but the scanner returns one, or the other way round. If `line : e '\n'` is written as `line : e`, the parser finishes an `e`, meets the newline, and reports an error on every line; if the scanner swallows newlines with the blanks rule, `'\n'` never comes. The grammar and the scanner must agree on what ends a line.

**Bison says `4 shift/reduce conflicts`.** The grammar was written as `e : e '+' e | e '*' e | ...`, the ambiguous grammar of section 5.4. The program still builds and seems to work, but it does not respect precedence; for a validator that is harmless, for anything that computes it is wrong. Section 8.3 fixes it with `%left`, and section 8.5 reads the report.

**`valid expression` printed after `invalid expression`.** `yyerror` printed its message and returned, and `main` printed its own message anyway. In the one-expression program either `yyerror` exits, or `main` tests what `yyparse` returned (`if (yyparse() == 0)`), never neither.

**`error` used, but the program still stops at the first mistake.** The `error` production is there, but nothing after `error` can ever be found: `line : error` with no `'\n'`, or a synchronising token the scanner never returns. Recovery needs a token after `error` that is sure to come.

**A character the grammar does not know.** `a % b` is reported invalid, as it should be: the scanner returns `'%'` as a character token, no rule mentions it, and the parser finds no action for it. That is the `.` rule doing its job; a scanner that silently skipped unknown characters would call `a % b` valid, since the parser would see `a b`, and then reject it for the wrong reason, or accept `a %+ b` as `a + b`.

### In one line each

1. A validator is the grammar with no actions: if `yyparse` returns, the input was valid.
2. Either `yyerror` exits, or `main` tests what `yyparse` returned; never print "valid" after "invalid".
3. An LR parser reports an error at the first token that cannot continue any valid input.
4. `line : error '\n' { yyerrok; }` reports a bad line, throws away tokens up to its newline and carries on; `yyerrok` keeps the next error from being swallowed.
5. `yacc -t` and `yydebug = 1` print Bison's shifts and reductions, the moves of section 7.7.

**Check yourself**

<!-- q: idea=yacc-validator type=predict wrong="the parser reports the error at the end of the line" -->
1. For the input `a * (b + ) - c`, at which token does `expr` find the error, and what is on the parser's stack, as grammar symbols, at that moment? Work it out, then check with `./exprt` (the line `Error: popping` and the lines before it).

<details>
<summary>Answer</summary>

At the `)`. Before it, `a * (b +` is the start of many valid expressions; after `+` the parser needs the start of a `t`, `(` or `ID`, and `)` is neither. The stack as symbols is `lines t * ( e +`: the first `a` has become `f` and then `t` (it stays a `t` because `*` followed), `*` and `(` were shifted, `b` became `f`, `t` and then `e` inside the parentheses, and `+` was shifted. The `- c` is never read before the error.

<!-- code: id=expr-where -->
```console
$ echo 'a * (b + ) - c' | ./exprt 2>&1 | grep -B3 -m1 'Error: popping'
Stack now 0 1 8 15 5 11 13
Reading a token
Next token is token ')' ()
Error: popping token '+' ()
```

`grep -m1` stops at the first matching line, and `-B3` prints the three lines before it as well. The trace shows the parser reading `)`, reporting the error, and then popping its states one by one, looking for one that can shift `error`, which is step 1 of the recovery above.

</details>

<!-- q: idea=yacc-validator type=apply wrong="an empty alternative needs the word empty" -->
2. Change `valid.y` so that an empty line counts as a valid (empty) expression, and say what you changed. Should it? Decide for a lab program that checks expressions typed by a user.

<details>
<summary>Answer</summary>

Add an alternative with nothing before the newline:

```text
line : e '\n'
     | '\n'
     ;
```

With it, the empty line is `valid expression`. Whether that is right is a decision about the language, not a fact of Yacc: an empty line is not an arithmetic expression, so a checker that reports it as valid is arguably wrong, and the lab answer usually rejects it. A calculator, on the other hand, should ignore an empty line rather than complain, and section 8.3's `calc.y` has exactly this alternative. Writing `| /* nothing */ '\n'` with a comment is a good habit; Yacc needs no keyword for an empty right side.

</details>

<!-- q: idea=yacc-validator type=fix wrong="the second line had no error" -->
3. A student's validator prints `invalid` for the first line of `a +` / `)` / `b * c` and `valid` for the third, and nothing for the second. They say the second line must have been fine. Name what is missing from their program and explain what happened to the `)`.

<details>
<summary>Answer</summary>

`yyerrok` is missing from the `error` production's action. After the first error the parser was still in its "recovering" mode when it met `)`: fewer than three tokens had been shifted since the last error, so it discarded `)` without calling `yyerror`, and the line was swallowed by the recovery. The second line was wrong; the program just never said so. Put `{ yyerrok; }` on `line : error '\n'`, as in `expr.y`.

</details>

## 8.3 A calculator: values, precedence and associativity

### From valid to how much

Chapter 1 traced `pos = init + rate * 60;` with `init` 10 and `rate` 2.5, and `pos.c` printed `pos = 160`. A validator can tell you that `10 + 2.5 * 60` is a well-formed expression; it cannot tell you that it is 160. The second program on every lab sheet does: "Write a Yacc program that evaluates an arithmetic expression", in other words a desk calculator. You type a line, it prints the value.

Two things are needed that `expr.y` did not have. Every part of an expression must carry a number up the parse tree, so that when `e '+' e` is reduced the parser knows what the two `e`s were worth. And the precedence of the operators must be right, so that `2.5 * 60` is computed before the `+`. The first comes from the values of section 8.1, now on nonterminals too; the second can come from the E, T, F grammar, but Yacc offers a shorter way, and it is the one the lab answer uses.

### Values on every plate

<!-- teaches: yacc-value = semantic value, default action -->
In section 8.1 only tokens had values: the scanner put a number in `yylval`, and the parser copied it onto its stack when it shifted the token. A nonterminal gets its value from the action of the production that made it. When the parser reduces by `e : e '+' e`, three symbols are on top of the stack, with values `$1`, `$2` and `$3`. The action computes `$$`, the value of the new `e`, and after the reduction that value sits on the stack beside the `e` that replaced the three symbols:

```text
e : e '+' e    { $$ = $1 + $3; }
```

So each plate carries a value as well as a state, and a reduction takes a few values off and puts one back. Here are the reductions for `10 + 2.5 * 60`, shown as the symbols on the stack with each value written after its symbol:

| Move | Stack (symbol:value) | Next | Action |
|---|---|---|---|
| 1 | | `10` | shift `NUM`, value 10 |
| 2 | `NUM:10` | `+` | reduce e -> NUM, `$$ = $1` = 10 |
| 3 | `e:10` | `+` | shift `+` |
| 4 | `e:10 +` | `2.5` | shift `NUM`, value 2.5 |
| 5 | `e:10 + NUM:2.5` | `*` | reduce e -> NUM, 2.5 |
| 6 | `e:10 + e:2.5` | `*` | shift `*` (it binds tighter than `+`) |
| 7 | `e:10 + e:2.5 *` | `60` | shift `NUM`, value 60 |
| 8 | `e:10 + e:2.5 * NUM:60` | newline | reduce e -> NUM, 60 |
| 9 | `e:10 + e:2.5 * e:60` | newline | reduce e -> e * e, `$$` = 2.5 × 60 = 150 |
| 10 | `e:10 + e:150` | newline | reduce e -> e + e, `$$` = 10 + 150 = 160 |

The value is computed as the tree is built, from the leaves up: the multiplication's node is finished at move 9, before the addition's at move 10, so the value of `rate * 60` exists before the `+` needs it. That order is the same as the order of the reductions, which is the rightmost derivation backwards (section 7.1). A production with no action, such as `e : NUM`, gets the *default action* `$$ = $1`: the value of the first symbol passes up unchanged. That is why `NUM`'s value reached the `e` above it in moves 2, 5 and 8 although no action was written for it.

The values here are fractions, so the value type must be `double`, not the `int` of section 8.1. The oldest way to say so, and the one most lab answers use, is a line in the declarations:

```c
#define YYSTYPE double
```

`YYSTYPE` is the type of `yylval` and of every value on the stack. Bison uses `int` only if `YYSTYPE` is not already defined when its code reaches that point, so this line, placed in the `%{ %}` block, wins. Remember that it is a C macro, seen only by the file that contains it; that matters in a moment.

### Precedence without the extra nonterminals

<!-- teaches: yacc-precedence = %left, %right, %prec -->
The E, T, F grammar gets precedence right by having one nonterminal per level (section 5.2). It works with Yacc, but it is long, and each extra level of operators (`^` for powers, unary minus) needs another nonterminal. The ambiguous grammar is much shorter:

```text
e : e '+' e | e '-' e | e '*' e | e '/' e | '(' e ')' | NUM ;
```

Section 7.6 showed what an LR table generator makes of it: states with a shift and a reduce in the same entry, because `e + e` with `*` next could be reduced (meaning `(e + e) * ...`) or extended by a shift (meaning `e + (e * ...)`). Section 7.6 also showed how to settle each conflict by hand, entry by entry, from the operators' precedence and associativity. Yacc lets you write those decisions as declarations and makes them itself:

```text
%left '+' '-'
%left '*' '/'
%right '^'
%right UMINUS
```

Each line declares a group of tokens of equal precedence; each line binds tighter than the lines above it. `%left` says the group is left-associative, `%right` right-associative, and `%nonassoc` (not used here) that two in a row are an error, as in `a < b < c` in some languages. A production gets the precedence of the last token in its right side, so `e : e '+' e` has the precedence of `'+'`.

When Bison meets a shift-reduce conflict, it compares the precedence of the production it could reduce by with the precedence of the lookahead token it could shift:

1. the token binds tighter: shift. With `e + e` on the stack and `*` next, shift `*`, so the multiplication is done first;
2. the production binds tighter: reduce. With `e * e` on the stack and `+` next, reduce, so `e * e` is finished before the `+`;
3. the same level: the associativity decides. `%left` reduces, so `2 - 3 - 4` is `(2 - 3) - 4`; `%right` shifts, so `2 ^ 3 ^ 2` is `2 ^ (3 ^ 2)`; `%nonassoc` makes the entry an error.

These are the same choices you made in section 7.6's table for `amb.g`, written once instead of entry by entry. Bison reports nothing for a conflict it settled this way, and section 8.5 shows where to see the decisions.
<!-- checked: how=run date=2026-11-21 ref="Bison 3.8.2 --report=all on calc.y: resolved as reduce (%left), shift (%right), by level otherwise; %nonassoc per Bison manual, Precedence" -->

One operator needs more. The minus sign is both binary (`5 - 2`) and unary (`-2`), but it is one token, `'-'`, with one precedence, the low one of `+`. The production `e : '-' e` would get that low precedence, and `-2 * 3` would be read as `-(2 * 3)`, which is the same number here but not for `-2 ^ 2`. `%prec` gives one production a precedence of its own:

```text
e : '-' e %prec UMINUS   { $$ = -$2; }
```

`UMINUS` is not a token the scanner ever returns. It is a name declared on the last precedence line only so that `%prec` can refer to its level, the tightest of all.

### calc.y and calc.l

<!-- code: id=calc-write -->
```console
$ cat > calc.y <<'END'
%{
/* calc.y: a desk calculator; one expression per line, value printed */
#include <stdio.h>
#include <math.h>
#define YYSTYPE double
int yylex(void);
void yyerror(const char *s);
%}
%token NUM
%left '+' '-'
%left '*' '/'
%right '^'
%right UMINUS
%%
lines : /* nothing */
      | lines line
      ;
line  : e '\n'              { printf("= %g\n", $1); }
      | '\n'
      | error '\n'          { yyerrok; }
      ;
e     : e '+' e             { $$ = $1 + $3; }
      | e '-' e             { $$ = $1 - $3; }
      | e '*' e             { $$ = $1 * $3; }
      | e '/' e             { if ($3 == 0) {
                                  yyerror("division by zero");
                                  $$ = 0;
                              } else
                                  $$ = $1 / $3;
                            }
      | e '^' e             { $$ = pow($1, $3); }
      | '-' e %prec UMINUS  { $$ = -$2; }
      | '(' e ')'           { $$ = $2; }
      | NUM
      ;
%%
void yyerror(const char *s)
{
    fprintf(stderr, "error: %s\n", s);
}

int main(void)
{
    return yyparse();
}
END
```

<!-- code: id=calc-write-l -->
```console
$ cat > calc.l <<'END'
%{
/* calc.l: the scanner for calc.y; a number carries its value in yylval */
#include <stdlib.h>
#define YYSTYPE double
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
[0-9]+(\.[0-9]+)?    { yylval = atof(yytext); return NUM; }
[ \t]+               { /* blanks */ }
\n                   { return '\n'; }
.                    { return yytext[0]; }
%%
END
```

**The declarations.** `#define YYSTYPE double` makes every value a `double`. `math.h` is there for `pow`. Then one token, `NUM`, and the four precedence lines, loosest first.

**The rules.** `lines` and the `error` recovery are `expr.y`'s. A line may also be just a newline, so that pressing Enter on an empty line is not an error. Each operator's action computes the new value from `$1` and `$3`. Division checks for zero, reports it through `yyerror` and gives 0, so that one bad line does not end the session. `( e )` passes up the value of the middle symbol, `$2`; without that action the default `$$ = $1` would pass up the value of `'('`, which is whatever `yylval` held when it was shifted. `e : NUM` has no action and relies on the default.

**The scanner.** `atof` turns the lexeme into a `double` in `yylval`. And the `#define YYSTYPE double` line appears again, *before* `#include "y.tab.h"`. `y.tab.h` declares `extern YYSTYPE yylval;`, with `YYSTYPE` as `int` unless it has been defined already, and the `#define` in `calc.y` is not seen here: `lex.yy.c` is compiled on its own, from its own text.

### Build it and use it

`pow` lives in the maths library, which GCC only links when asked: `-lm`, at the end of the line.

<!-- code: id=calc-run -->
```console
$ yacc -d calc.y
$ flex calc.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o calc -lm
$ printf '10 + 2.5 * 60\n2 - 3 - 4\n8 / 2 / 2\n2 ^ 3 ^ 2\n(1 + 2) * -3\n' | ./calc
= 160
= -5
= 2
= 512
= -9
```

The running statement's expression is 160, as `pos.c` printed in chapter 1: the `*` was done first. `2 - 3 - 4` is -5, so `-` grouped to the left, `(2 - 3) - 4`; grouped to the right it would be `2 - (-1)` = 3. `8 / 2 / 2` is 2, also left to right; the other way it would be 8. `2 ^ 3 ^ 2` is 512, which is `2 ^ 9`, so `^` grouped to the right as `%right` asked; to the left it would be `8 ^ 2` = 64. `(1 + 2) * -3` shows the parentheses and the unary minus together.

Then the inputs that must not give a number:

<!-- code: id=calc-errors -->
```console
$ printf '7 / 0\n1 + * 2\n\n4 ^ 0.5\n' | ./calc
error: division by zero
= 0
error: syntax error
= 2
```

`7 / 0` reported the division and went on with 0 as its value, so a `= 0` follows; the calculator might as well print nothing for that line, and question 3 below changes it. `1 + * 2` is a syntax error, recovered by the `error` production. The empty line printed nothing, as its production has no action. And the calculator still works after the two errors.

### Where the precedence went

The calculator's answers prove the precedence is right, but you can also see each decision Bison made. `yacc -v` writes a report of the parser's states to a file, `y.output`; with `--report=all` the report also says how each conflict was settled. Ask for it and pick out the decisions for rule 6, which is `e : e '+' e` (rule 1 is `lines : /* nothing */`, and the rules are numbered in the order they are written):

<!-- code: id=calc-report -->
```console
$ yacc -d -v --report=all calc.y
$ grep 'rule 6 and' y.output
    Conflict between rule 6 and token '+' resolved as reduce (%left '+').
    Conflict between rule 6 and token '-' resolved as reduce (%left '-').
    Conflict between rule 6 and token '*' resolved as shift ('+' < '*').
    Conflict between rule 6 and token '/' resolved as shift ('+' < '/').
    Conflict between rule 6 and token '^' resolved as shift ('+' < '^').
```

Each line is one entry of section 7.6's table, settled. With `e + e` on the stack: a second `+` or a `-` next means reduce, because they are on the same level as `+` and the level is `%left`; a `*`, `/` or `^` next means shift, because their level is tighter (`'+' < '*'` reads "`+` binds less tightly than `*`"). Bison does not warn about any of these conflicts, because the precedence lines settled every one; section 8.5 reads the report for a grammar where some are left over.

### When it goes wrong

**Every answer is nonsense, and nothing warned.** Leave the `#define YYSTYPE double` out of `calc.l` and see:

<!-- code: id=calc-bad -->
```console
$ grep -v 'define YYSTYPE' calc.l > calcbad.l
$ flex calcbad.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o calcbad -lm
$ echo '10 + 2.5 * 60' | ./calcbad
= 4.94066e-323
```

The scanner's `yylval` is an `int` in its own file and a `double` in the parser's file; they are the same eight bytes in memory, seen as two types. `atof` gives 10.0, the scanner stores it as the `int` 10 in the low four bytes, and the parser reads those bytes as a `double`, a number very close to zero. GCC compiles each file alone and the linker does not check types, so no tool complains. When a calculator prints numbers like `4.94066e-323` or `0` for everything, look at `YYSTYPE` in both files first. Section 8.6 shows `%union`, which declares the type inside `y.tab.h` and removes this mistake for good.

**`undefined reference to 'pow'`.** `-lm` was left off, or written before the source files. Put it last.

**`2 - 3 - 4` gives 3.** `'-'` was declared `%right`, or the grammar was written with `e '-' e` and no precedence at all, so Bison chose to shift. Check the precedence lines.

**`(2 + 3)` gives a strange number.** The action `$$ = $2;` is missing from `'(' e ')'`, so the default `$$ = $1` passed up the value of `'('`.

**Unary minus binds loosely.** Without `%prec UMINUS`, `-2 ^ 2` would use the precedence of `'-'`, the loosest, and be read as `-(2 ^ 2)`. That is in fact what mathematicians mean, and question 1 asks which you want.

**`$3` used in a rule with two symbols.** Bison stops with `error: integer out of range: '$3'` and points at the `$3`. Count the symbols of the right side, tokens included.
<!-- checked: how=run date=2026-11-21 ref="scratch calc.y with | '!' e { $$ = $3; }: Bison 3.8.2 error: integer out of range: '$3'" -->

### In one line each

1. `$$` is the value of the left side, `$1`, `$2` ... the values of the right side; with no action, `$$ = $1`.
2. `#define YYSTYPE double` sets the type of every value, in the `.y` file and again before `#include "y.tab.h"` in the `.l` file.
3. `%left`, `%right`, `%nonassoc` lines declare precedence, loosest first; a production takes the precedence of its last token, or the one named by `%prec`.
4. Bison settles a shift-reduce conflict by comparing the production's precedence with the lookahead's: tighter token, shift; tighter production, reduce; equal, associativity decides.
5. Test associativity with `2 - 3 - 4` and `2 ^ 3 ^ 2`, and link `pow` with `-lm` last.

**Check yourself**

<!-- q: idea=yacc-precedence type=predict wrong="unary minus always binds tighter than everything" -->
1. `calc` prints `= 4` for `-2 ^ 2`. Explain why from the precedence lines, then change one line so that it prints `= -4`, the usual mathematical meaning, without breaking `(1 + 2) * -3`.

<details>
<summary>Answer</summary>

`UMINUS` is on the last line, tighter than `'^'`, so with `- e` on the stack (the `e` is the 2) and `^` next, the production `e : '-' e %prec UMINUS` binds tighter than the token `^`: reduce first, giving `(-2) ^ 2` = 4. Move `%right UMINUS` above `%right '^'`:

```text
%left '+' '-'
%left '*' '/'
%right UMINUS
%right '^'
```

Now `^` binds tighter than unary minus: the parser shifts `^`, computes `2 ^ 2` first, and then negates: -4. `(1 + 2) * -3` is still -9, because unary minus is still tighter than `*`.

<!-- code: id=calc-uminus -->
```console
$ mkdir um
$ sed '/%right UMINUS/d; s/^%right .\^.$/%right UMINUS\n&/' calc.y > um/calc.y
$ cp calc.l um
$ cd um
$ yacc -d calc.y
$ flex calc.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o calc -lm
$ printf -- '-2 ^ 2\n(1 + 2) * -3\n' | ./calc
= -4
= -9
$ cd ..
```

</details>

<!-- q: idea=yacc-value type=fix wrong="the default action passes up the middle symbol" -->
2. A student's calculator prints `= 3` for `1 + 2`, but `= 0` for `(1 + 2)` and `= 0` for `(5)`. Their rules are those of `calc.y`, with one difference. Which one, and why exactly 0?

<details>
<summary>Answer</summary>

`'(' e ')'` has no action, so it gets the default `$$ = $1`, the value that came with the token `'('`. A one-character token's value is whatever `yylval` held when it was shifted. In `(1 + 2)` and `(5)` the `'('` is the first token of the line, and the scanner of `calc.l` only stores into `yylval` for a `NUM`; before any number has been read, `yylval` is still 0, the value every global variable starts with. (After an earlier line, it would be that line's last number, a plausible-looking wrong answer.) The fix is `{ $$ = $2; }`.

</details>

<!-- q: idea=yacc-value type=apply wrong="yyerror can only be called by the parser" -->
3. Change `calc.y` so that a line that divides by zero prints only the error, and no `= 0` after it. Keep the calculator working for the lines that follow.

<details>
<summary>Answer</summary>

The value of a line is printed by `line : e '\n'`, which cannot tell a real 0 from a failed division. Let the division set a flag that the line's action reads and clears:

```text
%{
...
int failed;              /* set by a division by zero in this line */
%}
...
line  : e '\n'      { if (!failed) printf("= %g\n", $1); failed = 0; }
...
      | e '/' e     { if ($3 == 0) {
                          yyerror("division by zero");
                          failed = 1;
                          $$ = 0;
                      } else
                          $$ = $1 / $3;
                    }
```

The value 0 still travels up, so that the rest of the line can be parsed, but nothing is printed for it. A second way is the special action `YYERROR;`, which makes the parser behave as if it had met a syntax error at that point and recover through `error '\n'`. It does not call `yyerror` itself, so write `{ yyerror("division by zero"); YYERROR; }`; with `YYERROR` alone, the bad line vanishes without a word.
<!-- checked: how=run date=2026-11-21 ref="scratch calc.y with { YYERROR; } in the division action: '7 / 0' printed nothing, next line '= 2'" -->

</details>

## 8.4 Recognising languages: identifiers, a^n b^n and other classics

### Questions about strings, not programs

The rest of the lab sheet's Yacc list does not look like compiler work at all: "Write a Yacc program to recognise strings of the form a^n b^n, n ≥ 1", "... the language a^n b, n ≥ 10", "... a valid variable name, a letter followed by any number of letters or digits". Each asks you to turn a description of a set of strings, a *language* in the sense of section 3.1, into a grammar, type the grammar into Yacc, and let the parser say yes or no for each line.

They are worth doing for two reasons. First, they are in the exam, and each has one small trick. Second, they show where the line falls between what Lex can do and what needs Yacc. Some of these languages could be recognised by a single regular expression in Lex; one of them cannot, and that one is the reason parsers exist.

### One scanner for the a-and-b languages

In these programs the scanner does almost nothing: it hands over one character at a time. A token per character makes the grammar do all the work, which is the point of the exercise. Save one scanner for the languages over `a` and `b`, and the first grammar:

<!-- code: id=ab-write -->
```console
$ cat > ab.l <<'END'
%{
/* ab.l: the scanner for the a-and-b languages */
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
a        { return A; }
b        { return B; }
[ \t]+   { /* blanks */ }
\n       { return '\n'; }
.        { return yytext[0]; }
%%
END
```

`a` and `b` become the tokens `A` and `B`, a newline is returned as itself, and any other character as itself too, so that `aaxbb` reaches the parser with an `'x'` in it, a token no rule mentions, and is rejected.

### a^n b^n: the language Lex cannot do

<!-- teaches: yacc-language = language recogniser, a^n b^n, counting with a grammar -->
The language a^n b^n, n ≥ 1, is `ab`, `aabb`, `aaabbb` and so on: some `a`s, then exactly as many `b`s. The grammar writes "as many" as nesting: a string of the language is either `ab`, or an `a`, a shorter string of the language, and a `b`:

```text
S -> a S b | a b
```

Each `a` is matched with its `b` the way an opening parenthesis is matched with its closing one. `aaabbb` is `a (a (a b) b) b`.

No regular expression can say this, and so no Lex pattern and no DFA. A DFA has a fixed number of squares on its board (chapter 3's picture), and to check `a^n b^n` it would have to remember n while it reads the `a`s; with only k squares it cannot tell `a^k` from `a^(k+1)` apart, so it must accept both `a^k b^k` and `a^(k+1) b^k` or neither. An LR parser can, because its stack has no fixed size: every `a` is a plate, and every `b` takes one plate's worth of the stack away. That is the difference between a scanner and a parser, in one language. `a*b*` is regular; `a^n b^n` is not.

<!-- code: id=anbn-write -->
```console
$ cat > anbn.y <<'END'
%{
/* anbn.y: is each line a^n b^n, n >= 1? */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
%}
%token A B
%%
lines : /* nothing */
      | lines line
      ;
line  : s '\n'         { printf("accepted\n"); }
      | error '\n'     { yyerrok; }
      ;
s     : A s B
      | A B
      ;
%%
void yyerror(const char *s)
{
    printf("rejected (%s)\n", s);
}

int main(void)
{
    return yyparse();
}
END
$ yacc -d anbn.y
$ flex ab.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o anbn
$ printf 'ab\naaabbb\naabbb\naaabb\nba\n\naabxb\n' | ./anbn
accepted
accepted
rejected (syntax error)
rejected (syntax error)
rejected (syntax error)
rejected (syntax error)
rejected (syntax error)
```

The grammar part is two lines; the rest is the frame you have now written four times, the list of lines with error recovery. `ab` and `aaabbb` are accepted. `aabbb` has one `b` too many: the parser has matched both `a`s, has a complete `s`, and wants the newline, but gets a `b`. `aaabb` has one too few: the newline comes while an `a` is still waiting for its `b`. `ba` starts wrong. The empty line is rejected, because n ≥ 1, and `aabxb` has a character outside the alphabet.

Follow the stack for `aaabbb` to see the counting. The parser shifts all three `A`s; it cannot reduce anything, since no right side ends in `A`. Then it shifts the first `B`: the top is `A B`, which is reduced to `s`. The next `B` makes `A s B` on top, reduced to `s`; and the last `B` the same. Each reduction takes away one `A` and one `B`. The `A`s waiting on the stack *are* the count. If a `B` comes when no `A` is waiting, or the newline comes while one is, the ACTION entry is blank.

### a^n b, n ≥ 10: counting with values

The next classic, "a^n b where n ≥ 10", is ten or more `a`s followed by one `b`. It *is* regular: `aaaaaaaaaaa*b` in Lex, or `a{10,}b`. In Yacc there are two ways to write it. The first is pure grammar: ten `A`s written out, then a list of more:

```text
s    : A A A A A A A A A A more B ;
more : /* nothing */ | more A ;
```

It works and it is the answer many lab manuals print, but it does not stretch: n ≥ 100 would need a hundred `A`s in the rule. The second way lets the grammar accept any run of `a`s followed by a `b`, and lets an action count them with values, as `calc.y` computed with them:

<!-- code: id=a10b-write -->
```console
$ cat > a10b.y <<'END'
%{
/* a10b.y: is each line a^n b with n >= 10? */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
%}
%token A B
%%
lines : /* nothing */
      | lines line
      ;
line  : as B '\n'      { if ($1 >= 10)
                             printf("accepted (n = %d)\n", $1);
                         else
                             printf("rejected (n = %d, fewer than 10)\n", $1);
                       }
      | error '\n'     { yyerrok; }
      ;
as    : A              { $$ = 1; }
      | as A           { $$ = $1 + 1; }
      ;
%%
void yyerror(const char *s)
{
    printf("rejected (%s)\n", s);
}

int main(void)
{
    return yyparse();
}
END
$ yacc -d a10b.y
$ flex ab.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o a10b
$ printf 'aaaaaaaaaab\naaaaaaaaab\naaaaaaaaaaaaaaab\naaaaaaaaaabb\nb\n' | ./a10b
accepted (n = 10)
rejected (n = 9, fewer than 10)
accepted (n = 15)
rejected (syntax error)
rejected (syntax error)
```

`as` is a list of `A`s, and its value is its length: the first `A` makes an `as` worth 1, and each further `A` makes an `as` worth one more. When the whole line has been read, `$1` of `line` is n, and the action decides. Ten `a`s are accepted, nine are not, fifteen are. Two `b`s, or a `b` with no `a`, are syntax errors, because the grammar itself demands at least one `A` and exactly one `B`.

Notice the two kinds of "no" in the output. A line rejected by the grammar comes through `yyerror`; a line rejected by the action is one the grammar accepted and the action turned down. The language a^n b, n ≥ 10, is the grammar's language with a condition added. That split, "the grammar says what shape, an action checks a fact the grammar does not", is exactly how a compiler checks types in section 9.6: the grammar accepts `x = y + 1;` whatever the types are, and an action complains.

### A valid identifier

"Write a Yacc program to check whether a given string is a valid identifier, a letter followed by letters or digits." Again the scanner hands over single characters, now of two kinds:

<!-- code: id=idy-write -->
```console
$ cat > idy.l <<'END'
%{
/* idy.l: one token per character for idy.y */
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
[A-Za-z_]   { return LETTER; }
[0-9]       { return DIGIT; }
\n          { return '\n'; }
.           { return yytext[0]; }
%%
END
$ cat > idy.y <<'END'
%{
/* idy.y: is each line a valid identifier (a letter, then letters or digits)? */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
%}
%token LETTER DIGIT
%%
lines : /* nothing */
      | lines line
      ;
line  : id '\n'        { printf("valid identifier\n"); }
      | error '\n'     { yyerrok; }
      ;
id    : LETTER
      | id LETTER
      | id DIGIT
      ;
%%
void yyerror(const char *s)
{
    printf("not an identifier (%s)\n", s);
}

int main(void)
{
    return yyparse();
}
END
$ yacc -d idy.y
$ flex idy.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o idy
$ printf 'rate\nx1\n_tmp9\n9lives\nmy-var\nmy var\n' | ./idy
valid identifier
valid identifier
valid identifier
not an identifier (syntax error)
not an identifier (syntax error)
not an identifier (syntax error)
```

The grammar is the regular definition of section 3.1, `letter (letter | digit)*`, written as a left-recursive list: an `id` starts with a `LETTER`, and any `id` followed by a `LETTER` or a `DIGIT` is a longer `id`. As in Tiny, the underscore counts as a letter. `9lives` starts with a digit; `my-var` has a `'-'`, which no rule mentions. `my var` is rejected too, and here is a choice made in the scanner: `idy.l` has no rule for blanks, so the blank reaches the parser as `' '` and is an error. A scanner that skipped blanks would call `my var` a valid identifier, because the parser would see `myvar`. Whether blanks are allowed is part of the language, and it is decided in the `.l` file, not the `.y` file.

### Lex or Yacc?

| Language | Regular? | Natural tool | Why |
|---|---|---|---|
| identifier, `letter (letter \| digit)*` | yes | Lex: one pattern | a DFA with two squares does it |
| a^n b, n ≥ 10 | yes | Lex: `a{10,}b` | a fixed count fits in a fixed number of squares |
| a^n b^n | no | Yacc | needs to remember an unbounded n: a stack |
| balanced parentheses | no | Yacc | the same matching as a^n b^n |
| a valid arithmetic expression | no | Yacc | parentheses nest without limit |

The lab asks for the first two in Yacc anyway, to see that you can write a grammar. In a compiler, the first two belong to the scanner and the last three to the parser, and the dividing line is the one in this table: anything that needs matching or nesting needs the parser's stack.
<!-- checked: how=run date=2026-11-21 ref="scratch flex 2.6.4 a{10,}b: matches 10 and 12 a then b, not 9" -->

### When it goes wrong

**Everything is rejected after the first line.** The `'\n'` is in the scanner but not in the grammar, or the other way round, as in section 8.2. These programs live on single characters, so check what the scanner returns for every character of a test line.

**a^n b^n accepts `aabbb` and the like.** The grammar was written as two separate lists, `s : as bs`, which is `a+ b+`: any number of `a`s, then any number of `b`s. It loses the matching. The matching only comes from a production that puts the `b` *after* the inner string and the `a` *before* it in the same right side: `s : A s B`.

**Bison warns about conflicts in a language grammar.** A grammar such as `s : A s B | A B | /* nothing */` lets the empty string be an `s` and makes `ab` reachable two ways, `A B` and `A s B` with an empty `s`. Ambiguity in a recognizer does not change yes or no, but it is a sign the grammar says more than it should. Take out the alternative that duplicates another; if n ≥ 0 is wanted, keep `A s B | /* nothing */` and drop `A B`.

**Counting with `$1` gives 1 every time.** The action of `as : as A` was written `{ $$ = $2 + 1; }`; `$2` is the value of the token `A`, which the scanner never set. The count lives in the first symbol, `$1`.

### In one line each

1. A language recogniser is a grammar with a line-list frame: the grammar decides, `yyerror` says no, the action says yes.
2. a^n b^n is `s : A s B | A B`: nesting matches each `a` with its `b`, and the parser's stack does the counting that no DFA can.
3. A condition the grammar cannot say neatly (n ≥ 10) is counted in values and checked in an action.
4. Whether blanks or other characters are allowed is decided in the scanner.
5. Regular languages belong to Lex, nested and matched ones to Yacc.

**Check yourself**

<!-- q: idea=yacc-language type=apply wrong="a^n b^m needs two counters in actions" -->
1. Write the rules (only the grammar part) for the language a^n b^m with n ≥ 1, m ≥ 1, and say whether it is regular. Then the language a^n b^n c^m, m ≥ 0.

<details>
<summary>Answer</summary>

a^n b^m, n, m ≥ 1, is `a+ b+`, regular:

```text
s  : as bs ;
as : A | as A ;
bs : B | bs B ;
```

a^n b^n c^m is a^n b^n followed by any number of `c`s, and it is not regular, because of its first part:

```text
s  : ab cs ;
ab : A ab B | A B ;
cs : /* nothing */ | cs C ;
```

The scanner needs a third token, `C`. The usual wrong answer counts both parts in actions; the grammar already does the counting where counting is needed, and a list does the rest.

</details>

<!-- q: idea=yacc-language type=predict wrong="the stack holds only n plates for a^n b^n" -->
2. While `anbn` parses `aaaabbbb`, what is the largest number of symbols on the parser's stack (not counting `lines`), and at which token is it reached?

<details>
<summary>Answer</summary>

Five: the four `A`s and the first `B`, just after the first `B` is shifted. At that moment the top is `A B`, which is reduced to `s`, leaving `A A A s`, four symbols. From then on, each `B` is shifted (five again) and the top three, `A s B`, are reduced to one `s`, so the stack never grows past five. In general, a^n b^n needs a stack of n + 1 symbols, which is why no fixed number of states can do it.

</details>

<!-- q: idea=yacc-language type=build wrong="balanced parentheses need a counter" -->
3. Write `paren.y`, which accepts lines of balanced parentheses such as `()`, `(())()` and the empty line, and rejects `(()` and `)(`. Use a scanner that returns `(`, `)` and the newline as themselves. Test it on those five lines.

<details>
<summary>Answer</summary>

A balanced string is empty, or a balanced string inside parentheses followed by a balanced string:

<!-- code: id=paren -->
```console
$ cat > paren.l <<'END'
%{
/* paren.l: parentheses and newlines, each as itself */
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
[ \t]+   { /* blanks */ }
\n       { return '\n'; }
.        { return yytext[0]; }
%%
END
$ cat > paren.y <<'END'
%{
/* paren.y: is each line a balanced string of parentheses? */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
%}
%%
lines : /* nothing */
      | lines line
      ;
line  : p '\n'          { printf("balanced\n"); }
      | error '\n'      { yyerrok; }
      ;
p     : /* nothing */
      | '(' p ')' p
      ;
%%
void yyerror(const char *s)
{
    printf("not balanced (%s)\n", s);
}

int main(void)
{
    return yyparse();
}
END
$ yacc -d paren.y
$ flex paren.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o paren
$ printf '()\n(())()\n\n(()\n)(\n' | ./paren
balanced
balanced
balanced
not balanced (syntax error)
not balanced (syntax error)
```

The grammar has no `%token` line: every token is a character. `p : '(' p ')' p` is the a^n b^n idea with something allowed after the match. Bison builds it without a conflict: when the parser wonders whether an empty `p` ends here, the lookahead (`(` or not) decides.

</details>

## 8.5 Reading Bison's conflict reports with -v and the .output file

### A warning that still builds

Every Yacc program so far has built in silence. Sooner or later one will not. You type a grammar, run `yacc`, and get

```text
calcnp.y: warning: 30 shift/reduce conflicts [-Wconflicts-sr]
```

and then, which is the dangerous part, Bison writes `y.tab.c` anyway, GCC builds it, and the program runs. Section 7.6 explained what a conflict is: an entry of the LR table with two actions in it, because the grammar is ambiguous or because the method's lookahead is too rough. A table with two actions in one entry is not a parser; Bison makes it one by choosing one of the two actions in every conflicting entry. This section shows what it chooses, how to read the report in which it tells you, and how to decide whether the choice is the one you wanted. The examiner asks this in the viva ("why does Yacc say shift/reduce conflict, and what does it do about it?"), and the paper asks it about the dangling `else`.

### What Bison does with a conflict

Bison settles a conflict that no precedence declaration covers by two fixed rules:

- a **shift-reduce** conflict is settled in favour of the **shift**;
- a **reduce-reduce** conflict is settled in favour of the production written **first** in the grammar.

It then counts the conflicts it settled this way and reports the number as a warning. The parser works, in the sense that it always does something; whether it does the right thing depends on the grammar.
<!-- checked: how=run date=2026-11-21 ref="Bison 3.8.2 y.output: amb.y and dangle.y keep the shift, rr.y reduces by rule 3 (first) and calls rule 4 useless" -->

Shift is often the right choice, which is why it is the default: it keeps the parser reading, and makes an operator group to the right. For `if ... else` (below) it is exactly right. For arithmetic it is wrong. Take `calc.y` and delete its four precedence lines, together with the `%prec UMINUS` that refers to one of them:

<!-- code: id=calcnp -->
```console
$ grep -v '^%left\|^%right' calc.y | sed 's/ %prec UMINUS//' > calcnp.y
$ yacc -d calcnp.y
calcnp.y: warning: 30 shift/reduce conflicts [-Wconflicts-sr]
calcnp.y: note: rerun with option '-Wcounterexamples' to generate conflict counterexamples
$ flex calc.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o calcnp -lm
$ printf '2 - 3 - 4\n2 * 3 + 4\n2 + 3 * 4\n' | ./calcnp
= 3
= 14
= 14
```

Thirty conflicts, settled by shifting every time. Shifting means "do not finish what is on the stack; take the next operator first", so every operator groups to the right, and every operator has the same precedence: `2 - 3 - 4` is `2 - (3 - 4)` = 3, and `2 * 3 + 4` is `2 * (3 + 4)` = 14. `2 + 3 * 4` happens to come out right, 14, because grouping to the right is what `*` needs there. A test on that line alone would have passed. The warning was the only sign.

### y.output: the parser, written out

<!-- teaches: yacc-conflict-report = y.output, -v, conflict report, counterexample, %expect -->
To see which entries conflict and what Bison chose, ask for its report with `-v`. It writes `y.output` (in Yacc mode; `bison -v file.y` writes `file.output`). Use the smallest grammar that has the problem, the ambiguous expression grammar of section 5.4 with two operators, as `amb.y`; to look at its report, it needs no actions, no `main` and no scanner:

<!-- code: id=amb-report -->
```console
$ cat > amb.y <<'END'
%token ID
%%
e : e '+' e
  | e '*' e
  | '(' e ')'
  | ID
  ;
END
$ yacc -v amb.y
amb.y: warning: 4 shift/reduce conflicts [-Wconflicts-sr]
amb.y: note: rerun with option '-Wcounterexamples' to generate conflict counterexamples
$ head -3 y.output
State 9 conflicts: 2 shift/reduce
State 10 conflicts: 2 shift/reduce
```

The report opens with a summary: which states have conflicts and how many. Then come four parts, in order:

1. **Grammar**: the productions, numbered. Rule 0 is Bison's own `$accept: e $end`, the augmented production S' -> e of section 7.3, with `$end` for `$`.
2. **Terminals** and **Nonterminals**: each symbol and the rules it appears in.
3. **States**: one block per state of the LALR(1) collection, which is what you read.

Here is the first conflicting state:

<!-- code: id=amb-state9 -->
```console
$ sed -n '/^State 9$/,/^State 10$/p' y.output
State 9

    1 e: e • '+' e
    1  | e '+' e •
    2  | e • '*' e

    '+'  shift, and go to state 6
    '*'  shift, and go to state 7

    '+'       [reduce using rule 1 (e)]
    '*'       [reduce using rule 1 (e)]
    $default  reduce using rule 1 (e)


State 10
```

Read it as one row of the LR table, with the set of items that made it:

- **The items.** The lines with `•` are the kernel items of the state, written with the dot as section 7.3 wrote them: `e: e • '+' e` is the item E -> E . + E. The number in front is the rule. The second line, `| e '+' e •`, is a complete item: `e + e` is on top of the stack and could be reduced. So this is the state the parser is in after `e + e`, the state section 7.6 called the one with the conflict.
- **The actions.** `'+'  shift, and go to state 6` is the entry ACTION[9, +] = s6. Each line is one non-blank entry of the row.
- **The losers.** The lines in square brackets are the actions Bison *did not* take. `'+' [reduce using rule 1 (e)]` says: under `+` there was also a reduction by rule 1, and it lost to the shift. These bracketed lines are the conflicts, and there are two, under `+` and under `*`, as the summary said.
- **`$default`.** Every other lookahead, here `)` and `$end`, means reduce by rule 1. Bison writes the most common reduction once as the default instead of filling every column with it; it is also why section 8.2's trace sometimes reduced without reading a token. A blank entry that the default covers is reported later than in the hand-built table: the parser reduces first and finds the error in the next state.

Now you can check each loser against what the grammar should mean. With `e + e` on the stack and `*` next, shifting is right: the `*` binds tighter. With `+` next, shifting is wrong: `a + b + c` should be `(a + b) + c`, a reduce. State 10, the state after `e * e`, has the mirror picture: there, both shifts are wrong. These are the same four decisions you made by hand for `amb.g` in section 7.6, and `%left '+'` and `%left '*'` (section 8.3) are how you tell Bison to make them.

### Counterexamples: the conflict as a sentence

The note at the end of the warning offers a second view. With `-Wcounterexamples`, Bison finds, for each conflict, an input on which the two actions lead to two different parse trees, and prints both. Ask for the first:

<!-- code: id=amb-cex -->
```console
$ yacc -Wcounterexamples amb.y 2>&1 | sed -n '2,11p'
amb.y: warning: shift/reduce conflict on token '+' [-Wcounterexamples]
  Example: e '+' e • '+' e
  Shift derivation
    e
    ↳ 1: e '+' e
               ↳ 1: e • '+' e
  Reduce derivation
    e
    ↳ 1: e              '+' e
         ↳ 1: e '+' e •
```

`Example: e '+' e • '+' e` is the sentential form, with the dot where the parser stands: two `+` signs, and the parser between the second `e` and the second `+`. The shift derivation builds `e + (e + e)`, the reduce derivation `(e + e) + e`. That is the ambiguity of section 5.4 in Bison's words, and the fix follows from it: say which tree you want, here with `%left '+'`. Counterexamples are a recent addition to Bison (older Bisons and Berkeley Yacc do not have them); `y.output` is everywhere.
<!-- risk: -Wcounterexamples first appeared in Bison 3.7 (2020); Berkeley yacc has no counterexamples; check Bison NEWS -->

### The dangling else: one conflict to keep

The `if` statement of section 5.4 is the paper's favourite conflict. Write it as `dangle.y`, with single tokens standing in for the expression and the other statements:

<!-- code: id=dangle -->
```console
$ cat > dangle.y <<'END'
%token IF THEN ELSE EXPR OTHER
%%
s : IF EXPR THEN s
  | IF EXPR THEN s ELSE s
  | OTHER
  ;
END
$ yacc -v dangle.y
dangle.y: warning: 1 shift/reduce conflict [-Wconflicts-sr]
dangle.y: note: rerun with option '-Wcounterexamples' to generate conflict counterexamples
$ sed -n '/^State 7$/,/^State 8$/p' y.output
State 7

    1 s: IF EXPR THEN s •
    2  | IF EXPR THEN s • ELSE s

    ELSE  shift, and go to state 8

    ELSE      [reduce using rule 1 (s)]
    $default  reduce using rule 1 (s)


State 8
```

State 7 is the parser after `if E then S`, with `else` next: it can reduce the `if` without an `else` (rule 1), so that the `else` belongs to an outer `if`, or shift the `else`, so that it belongs to this, the nearest, `if`. Bison shifted. Every C-like language wants exactly that: an `else` belongs to the nearest `if` that has none. So the default is right, and the warning only says that the grammar is ambiguous, which you knew.

A warning you have decided to keep should still not appear on every build, because a new conflict added later would hide among the old ones. There are two ways to accept this one.

**Say how many you expect.** `%expect 1` in the declarations tells Bison that exactly one shift-reduce conflict is expected. With one, it is silent; with any other number it is an error, and the build stops, which is what you want when a change adds a conflict. In Yacc mode Bison adds a warning that POSIX Yacc has no `%expect`; with `bison` instead of `yacc` there is none.

**Give the reduction a precedence.** Precedence declarations work on any shift-reduce conflict, not only on operators. Give the `else`-less production a level below `ELSE`:

<!-- code: id=dangle-prec -->
```console
$ cat > dangle2.y <<'END'
%token IF THEN ELSE EXPR OTHER
%nonassoc LOWER_THAN_ELSE
%nonassoc ELSE
%%
s : IF EXPR THEN s          %prec LOWER_THAN_ELSE
  | IF EXPR THEN s ELSE s
  | OTHER
  ;
END
$ yacc -v dangle2.y
$ head -3 y.output
Grammar

    0 $accept: s $end
```

No warning, and the report starts straight with the grammar: there is no summary line of conflicting states, because there are none. With `if E then S` on the stack and `ELSE` next, the production's level (`LOWER_THAN_ELSE`) is below the token's (`ELSE`), so the parser shifts, as before; but now the choice is declared, not defaulted, and Bison says nothing. `LOWER_THAN_ELSE`, like `UMINUS` in section 8.3, is a name only `%prec` uses. This is the form most real grammars use, and section 9.5 uses it for Tiny's `if`.

### Reduce-reduce: always a mistake

A reduce-reduce conflict means that the parser has two different right sides complete on top of the stack, or the same one under two names, and the lookahead cannot choose. Here is the smallest case: a name that is an `a` in one rule and a `b` in another, both followed by `x`.

<!-- code: id=rr -->
```console
$ cat > rr.y <<'END'
%token ID
%%
s : a 'x'
  | b 'x'
  ;
a : ID ;
b : ID ;
END
$ yacc -v rr.y
rr.y: warning: 1 reduce/reduce conflict [-Wconflicts-rr]
rr.y: note: rerun with option '-Wcounterexamples' to generate conflict counterexamples
rr.y:7.5-6: warning: rule useless in parser due to conflicts [-Wother]
    7 | b : ID ;
      |     ^~
$ sed -n '/^State 1$/,/^State 2$/p' y.output
State 1

    3 a: ID •
    4 b: ID •

    'x'       reduce using rule 3 (a)
    'x'       [reduce using rule 4 (b)]
    $default  reduce using rule 3 (a)


State 2
```

Bison chose rule 3, the one written first, and then saw that rule 4 can now never be used, and said so: `rule useless in parser due to conflicts`. Whatever `b` was meant to do, the parser will never do it. No default can be right here, because the two reductions build different trees with different actions; one of the two is lost for every input. A reduce-reduce conflict is a fault in the grammar, never a thing to `%expect`. The cure is to rewrite: usually the two nonterminals are really one, or the scanner must tell the two cases apart by returning different tokens (a type name and a variable name, in C), or the decision must be put off by merging the rules until a later token can decide. Section 7.6 traced where such conflicts come from, including the one LALR merging can add.

### When it goes wrong

**The warning is ignored because the tests pass.** `calcnp` passed `2 + 3 * 4`. A grammar with conflicts needs a test chosen *from the conflict*: read the counterexample, and test that exact shape (`a + b + c`, `a * b + c`).

**A conflict is "fixed" with `%expect`.** `%expect` is for conflicts you have read and decided to keep, such as the dangling `else`. Used on the thirty conflicts of `calcnp.y`, it would silence a calculator that gets `2 - 3 - 4` wrong.

**The state numbers do not match your hand-built collection.** They are not meant to. Bison numbers states in the order it creates them, and it builds LALR(1) states directly; match states by their items, not by their numbers.

**`y.output` was not written.** `-v` was not given, or the report is in `file.output` because `bison` was run without `-y`.

**A precedence line changed nothing.** The production that needs it has no token in it (`e : e e` for implicit multiplication), so it has no precedence; give it one with `%prec`. Or the conflict is reduce-reduce, which precedence never settles.
<!-- risk: claim that precedence declarations never resolve reduce/reduce conflicts in Bison 3.8; check the Bison manual -->

### In one line each

1. Bison settles a shift-reduce conflict by shifting and a reduce-reduce conflict by the rule written first, counts them, warns, and builds the parser anyway.
2. `yacc -v` writes `y.output`: a summary, the numbered grammar, then every state's items (`•` is the dot) and actions.
3. A bracketed action in a state is the one that lost; `$default` is the reduction used for every other lookahead.
4. `-Wcounterexamples` prints an input where the two choices give two trees; test that input.
5. Keep a conflict only when the default is right (the dangling `else`) and say so with `%expect` or `%prec`; a reduce-reduce conflict is a fault in the grammar.

**Check yourself**

<!-- q: idea=yacc-conflict-report type=predict wrong="Bison refuses to build a grammar with conflicts" -->
1. A student adds the rule `e : e e` (implicit multiplication, as in `2 x`) to `calc.y` without any `%prec`. Bison warns about conflicts. Without running anything, say which input shape the conflict is about, what Bison will do by default, and why `%left '*'` does not help.

<details>
<summary>Answer</summary>

After `e e` on the stack, a token that can start an `e` (a number, `(` or `-`) can either be shifted, making the second `e` longer, or the parser can first reduce `e e` to `e`. With `2 3 4`, the shift gives `2 (3 4)` and the reduce gives `(2 3) 4`; with `2 - 3`, the conflict is worse, because `-` can be binary (`2 - 3`) or the start of `e`, a unary minus (`2 (-3)`). Bison shifts. `%left '*'` does not help, because `e : e e` contains no `*`: it has no token, so it has no precedence. The fix is a precedence of its own, `e : e e %prec '*'`, which still leaves the question of what `2 - 3` means. That second question is a language decision, and it is why calculators rarely allow implicit multiplication.

</details>

<!-- q: idea=yacc-conflict-report type=apply wrong="the empty alternative only adds the empty line to the language" -->
2. Section 8.4 warned that `s : A s B | A B | /* nothing */` has a conflict. Build its report and find the state with the conflict. Which two actions are in it, under which token, and what input shows the two trees?

<details>
<summary>Answer</summary>

<!-- code: id=anbn-conflict -->
```console
$ cat > anbn0.y <<'END'
%token A B
%%
s : A s B
  | A B
  | /* nothing */
  ;
END
$ yacc -v anbn0.y
anbn0.y: warning: 1 shift/reduce conflict [-Wconflicts-sr]
anbn0.y: note: rerun with option '-Wcounterexamples' to generate conflict counterexamples
$ head -1 y.output
State 1 conflicts: 1 shift/reduce
$ yacc -Wcounterexamples anbn0.y 2>&1 | grep Example
  Example: A • B
```

After an `A`, with `B` next, the parser can shift the `B` (making `A B`, rule 2) or reduce an empty `s` (rule 3, so that the `B` closes `A s B`). Bison shifts. The input is `ab`: it is `A B` by rule 2, and also `A s B` with an empty `s`. Both mean the same here, which is why the program still works, but the grammar has two trees for one sentence. Drop `A B` if n ≥ 0 is wanted, or the empty alternative if n ≥ 1.

</details>

<!-- q: idea=yacc-conflict-report type=decide wrong="%expect is the right fix for any conflict you understand" -->
3. In each case, decide: keep the conflict with `%expect`, settle it with precedence, or rewrite the grammar. (a) The dangling `else` in a C-like language. (b) `e : e '-' e` with no precedence. (c) A reduce-reduce conflict between `type_name : ID` and `var : ID`.

<details>
<summary>Answer</summary>

(a) Keep or declare: shifting is the meaning every C-like language wants; use `%expect 1` or the `%prec LOWER_THAN_ELSE` form. (b) Precedence: `%left '-'`, because the default shift makes subtraction group to the right, which is wrong arithmetic. (c) Rewrite: no default can be right, since one of the two rules becomes useless; real C compilers have the scanner look the name up and return a different token for a type name, so that the parser never has to choose.

</details>

## 8.6 Infix to postfix with semantic actions

### Doing something as you parse

Every action so far has either printed a verdict at the end of a line or computed a number. The last classic of the lab sheet, "convert an infix expression to postfix using Yacc", needs something in between: the program must *translate* the input, writing a new text as it parses. `init + rate * 60` becomes `init rate 60 * +`, the order in which a stack machine (or a person with a calculator that has no brackets) would do the work.

It is a small program, but it is the first real translation in the book, and the pattern it uses is the one chapter 9 builds on: put a piece of output in the action of each production, and let the order in which the parser reduces put the pieces in the right order. It also has the most common trap of Yacc programming in it, and you will fall into it on purpose.

### The order of reductions is postfix order

<!-- teaches: semantic-action = semantic action, translation during parsing, mid-rule action, %union -->
A *semantic action* is the C code attached to a production, run when the parser reduces by it. Since section 8.1 you have used actions; what matters now is *when* each one runs. Look again at the reductions of the calculator for `10 + 2.5 * 60` (section 8.3):

```text
e -> NUM (10)     e -> NUM (2.5)     e -> NUM (60)     e -> e * e     e -> e + e
```

Each operand is reduced when it is read, and each operator's production is reduced only after both of its operands are complete. That is exactly the order of postfix notation: operands first, operator after its two operands. It is no accident. An LR parser builds the tree from the leaves up, and finishes a node only when all its children are finished; that is a *postorder* walk of the parse tree, and postfix is the postorder of the expression tree. So the translation needs no stack of its own and no reordering: print an operand in the action that reduces it, print an operator in the action that reduces its production, and the output is postfix.

```text
e : e '+' e    { printf("+ "); }
  | e '*' e    { printf("* "); }
  | ID         { printf("%s ", ...the name...); }
```

The only question is where `...the name...` comes from.

### postbad.y: the name from yytext

The obvious answer is `yytext`, the text of the token the scanner matched last. It is a global, declared in `lex.yy.c`, so the Yacc file declares it `extern` and prints it:

<!-- code: id=postbad-write -->
```console
$ cat > postbad.y <<'END'
%{
/* postbad.y: infix to postfix, printing the name from yytext */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
extern char *yytext;
%}
%token ID
%left '+' '-'
%left '*' '/'
%%
lines : /* nothing */
      | lines line
      ;
line  : e '\n'          { printf("\n"); }
      ;
e     : e '+' e         { printf("+ "); }
      | e '-' e         { printf("- "); }
      | e '*' e         { printf("* "); }
      | e '/' e         { printf("/ "); }
      | '(' e ')'
      | ID              { printf("%s ", yytext); }
      ;
%%
void yyerror(const char *s)
{
    fprintf(stderr, "error: %s\n", s);
}

int main(void)
{
    return yyparse();
}
END
$ yacc -d postbad.y
$ flex valid.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o postbad
$ printf 'init + rate * 60\n(a + b) * c\n' | ./postbad
init rate 60 * +
a b + c *
```

It works. `init + rate * 60` came out as `init rate 60 * +`, and `(a + b) * c` as `a b + c *`. The grammar is the calculator's: the ambiguous one with `%left` lines, so that `*` binds tighter and both operators group to the left. The parentheses production has no action, because parentheses do not appear in postfix: the order they forced is already in the order of the reductions. `valid.l`, the scanner of section 8.2, returns names and numbers alike as `ID`.

Now add one production that every real language has, an assignment inside an expression, `pos = init + rate * 60`, with `=` the loosest operator and grouped to the right:

<!-- code: id=postbad2 -->
```console
$ sed -e "s/^%left '+' '-'/%right '='\n&/" -e "s/^      | ID .*/&\n      | ID '=' e        { printf(\"= \"); }/" postbad.y > postbad2.y
$ grep -n "'='" postbad2.y
9:%right '='
24:      | ID '=' e        { printf("= "); }
$ yacc -d postbad2.y
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o postbad2
$ printf 'init + rate * 60\n(a + b) * c\n' | ./postbad2
+ *
 * +
+ ) +
 *
```

The names are gone. The same two lines that worked a moment ago now come out as four broken ones. For the first expression, `postbad2` printed `+` where `init` should be, `*` where `rate` should be, and a newline where `60` should be, so the translation breaks off in the middle and the operators `* +` land on the next line. The second expression is as bad: `+ ) +`, then a newline for `c`, then `*`. Nothing about the expressions changed; one production was added *elsewhere* in the grammar.

### What yytext holds when the action runs

`yytext` holds the text of the last token the scanner returned, and the parser does not always reduce `e : ID` straight after reading the `ID`. To decide what to do after an `ID`, the parser in `postbad2` must look at the next token: `=` means this `ID` is the start of an assignment, anything else means it is an operand to reduce to `e`. So it calls `yylex` once more *before* reducing, and by the time the action of `e : ID` runs, `yytext` holds the *next* token's text: `+` after `init`, `*` after `rate`, the newline after `60`.

`postbad` printed the right names only by luck. In its grammar, the state after `ID` had a single action, reduce by `e : ID`, whatever came next, and Bison, as section 8.2's trace showed, reduces in such a state without reading the lookahead. Add any rule that makes the lookahead matter, and the luck runs out. A program that depends on whether the parser happened to need a lookahead is wrong, even while it passes its tests.

The cure is the one section 8.1 gave for numbers: a token's value must travel *with the token*, in `yylval`, which the parser copies onto its stack at the shift. Then it does not matter how many tokens are read before the reduction. For a name the value is a string, and the scanner must hand over a *copy* of it, because `yytext` is Flex's buffer and is overwritten by the next match.

### %union: values of more than one type

A number's value is a `double` and a name's value is a string. `#define YYSTYPE` gives every value one type, and section 8.3 showed what happens when the two files disagree about it. `%union` is the better way: it declares all the types a value can have, as the members of a C `union`, and Bison writes that union into `y.tab.h`, so the scanner gets the same type with no `#define` to forget.

```text
%union {
    char *s;
}
%token <s> ID
```

`%token <s> ID` says that the value of an `ID` is the member `s`, a `char *`. In an action, `$1` of an `ID` then means `yylval.s` as it was at the shift, with the right C type, so `printf("%s ", $1)` needs no cast. A nonterminal whose value is used is declared with `%type <member> name`; `postfix` uses no values on `e`, so it needs none. In the scanner, the value is set through the member: `yylval.s = strdup(yytext);`.

### post.y: the translator done right

<!-- code: id=post-write -->
```console
$ cat > post.l <<'END'
%{
/* post.l: the scanner for post.y; a name or number goes out as a copy */
#include <string.h>
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
[A-Za-z_][A-Za-z0-9_]*      { yylval.s = strdup(yytext); return ID; }
[0-9]+(\.[0-9]+)?           { yylval.s = strdup(yytext); return ID; }
[ \t]+                      { /* blanks */ }
\n                          { return '\n'; }
.                           { return yytext[0]; }
%%
END
$ cat > post.y <<'END'
%{
/* post.y: infix to postfix; each name travels in yylval as a string */
#include <stdio.h>
#include <stdlib.h>
int yylex(void);
void yyerror(const char *s);
%}
%union {
    char *s;
}
%token <s> ID
%right '='
%left '+' '-'
%left '*' '/'
%right UMINUS
%%
lines : /* nothing */
      | lines line
      ;
line  : e '\n'                 { printf("\n"); }
      | error '\n'             { yyerrok; }
      ;
e     : e '+' e                { printf("+ "); }
      | e '-' e                { printf("- "); }
      | e '*' e                { printf("* "); }
      | e '/' e                { printf("/ "); }
      | '-' e %prec UMINUS     { printf("neg "); }
      | '(' e ')'
      | ID                     { printf("%s ", $1); free($1); }
      | ID                     { printf("%s ", $1); free($1); }
        '=' e                  { printf("= "); }
      ;
%%
void yyerror(const char *s)
{
    fflush(stdout);
    fprintf(stderr, "<- error: %s\n", s);
}

int main(void)
{
    return yyparse();
}
END
```

**The scanner.** Both rules for names and numbers copy the lexeme with `strdup`, which allocates fresh memory and copies the string into it, and put the pointer in `yylval.s`. `y.tab.h` now declares `yylval` as the union, so `yylval.s` is a `char *` in this file too.

**The operand.** `e : ID` prints `$1`, the copy made when this very `ID` was read, and then frees it, since nothing needs it after it is printed.

**Unary minus.** Postfix cannot tell a unary minus from a binary one by its symbol (`a - b` is `a b -`; what would `-a` be?), so the unary one is printed as `neg`.

**The assignment, and an action in the middle.** In postfix the target of an assignment comes first: `pos = init + rate * 60` is `pos init rate 60 * + =`. But an action at the end of `ID '=' e` runs after the whole `e` has been translated, too late to print `pos` first. So the production has a second action *in the middle*, right after the `ID`. A *mid-rule action* runs when the parser has got that far in the production: here, as soon as it has decided that this `ID` is followed by `=`. Bison makes it work by inventing an empty nonterminal for the action and reducing it at that point; it counts as a symbol of the right side, so in the final action the `ID` is still `$1`, the mid-rule action is `$2`, `'='` is `$3` and `e` is `$4`. Of the two productions that start with `ID`, the lookahead decides: `=` next means the mid-rule action, anything else means `e : ID`. That lookahead is exactly the one that broke `postbad2`; here it does no harm, because nothing reads `yytext`.

**`yyerror`.** The output of a line is printed piece by piece, without a newline until the end, so when an error is found, part of the line is still waiting in `stdout`'s buffer. `fflush(stdout)` writes it out first, so that the error message appears after what was translated, pointing at where the parser stopped.

Build it and translate:

<!-- code: id=post-run -->
```console
$ yacc -d post.y
$ flex post.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o post
$ printf 'pos = init + rate * 60\n(a + b) * c\na - b - c\na = b = c\n-a * b\n' | ./post
pos init rate 60 * + =
a b + c *
a b - c -
a b c = =
a neg b *
```

The running statement first: `pos init rate 60 * + =`, which reads as "push the address of `pos`, push `init`, `rate` and 60, multiply, add, store". Chapter 9 turns exactly this order into three-address code, with a temporary for each operator. Then the parentheses, gone from the output with their work done. `a - b - c` subtracts left to right; `a = b = c` assigns right to left, as `%right '='` asked (first `b = c`, then `a =` that value). And `neg` binds tighter than `*`.

Then an empty input and a bad line:

<!-- code: id=post-errors -->
```console
$ ./post < /dev/null
$ printf 'a + * b\nx * y\n' | ./post
a <- error: syntax error
x y *
```

The empty input prints nothing and is accepted. In `a + * b`, the `a` was already translated when the parser met `*`; `fflush` put it on the screen before the message, so the message points at the place where the translation stopped. Then the next line is translated as usual.

### Walking the translation

Here is `pos = init + rate * 60` once more as the parser sees it, with each action as it runs; `M` stands for the empty nonterminal that holds the mid-rule action.

| Moves | What the parser does | Printed |
|---|---|---|
| shift `ID` (pos), read `=` | `=` next: this `ID` starts an assignment; reduce `M -> ε` | `pos ` |
| shift `=`, shift `ID` (init), read `+` | reduce `e -> ID` | `init ` |
| shift `+`, shift `ID` (rate), read `*` | reduce `e -> ID` | `rate ` |
| shift `*` (tighter than `+`), shift `ID` (60), read `\n` | reduce `e -> ID` | `60 ` |
| | reduce `e -> e * e` | `* ` |
| | reduce `e -> e + e` | `+ ` |
| | reduce `e -> ID M = e` | `= ` |
| shift `\n` | reduce `line -> e \n` | newline |

The left column has a read of the next token before every reduction of an `ID`. That is the read that overwrote `yytext` in `postbad2`, and that `$1` does not care about.

### When it goes wrong

**Names come out as operators or blanks.** The action reads `yytext`. Pass the name in `yylval` and read `$n`.

**Every name comes out as the last name of the line.** The scanner wrote `yylval.s = yytext;` without `strdup`: every value on the stack points into Flex's one buffer, which by the time the actions run holds later text. Copy the string.

**`error: $1 of 'e' has no declared type`.** A `%union` is declared, and an action uses the value of a symbol whose type was never given: here the `ID` of `e : ID`, declared with a plain `%token ID` instead of `%token <s> ID`. A nonterminal whose value is used needs a `%type` line in the same way. Every symbol whose value an action reads needs its member named.
<!-- checked: how=run date=2026-11-21 ref="scratch post.y with %token ID: Bison 3.8.2 error: $1 of 'e' has no declared type" -->

**`yylval` is an `int` in the scanner.** The scanner includes a `y.tab.h` from before the `%union` was added (`yacc` was run without `-d`). With `%union` both files read the type from the same header, so remake it.

**The mid-rule action causes a conflict.** A mid-rule action is a new empty nonterminal, and it must be reduced *before* the parser knows how the production ends. If another production starts the same way and needs the same lookahead to be told apart, Bison reports a conflict that the grammar without the action did not have. Move the action to the end if it can wait, or give both productions the same action.

### In one line each

1. A semantic action runs when its production is reduced; reductions come in postorder, so printing operands at their reduction and operators at theirs gives postfix.
2. Never read `yytext` in a Yacc action: the parser may have read the next token already.
3. A token's value travels in `yylval`, set by the scanner and copied at the shift; a string must be copied with `strdup`.
4. `%union` declares the value types once, in `y.tab.h`; `%token <member>` and `%type <member>` give each symbol its type.
5. A mid-rule action runs part-way through a production and counts as a symbol in the `$n` numbering.

**Check yourself**

<!-- q: idea=semantic-action type=predict wrong="the actions run in the order the symbols appear in the input" -->
1. Predict what `post` prints for `a * (b - c) / d` and for `x = -y + z`. Then check.

<details>
<summary>Answer</summary>

`a b c - * d /` and `x y neg z + =`. In the first, the parentheses make `b - c` finish before the `*` can reduce, and `/` groups with the result of the `*` because they are on one level and `%left`. In the second, `neg` comes right after `y`, since unary minus is reduced before the `+` is even shifted, and `x` comes first through the mid-rule action.

<!-- code: id=post-predict -->
```console
$ printf 'a * (b - c) / d\nx = -y + z\n' | ./post
a b c - * d /
x y neg z + =
```

</details>

<!-- q: idea=semantic-action type=apply wrong="prefix can be printed as you go, like postfix" -->
2. Infix to *prefix* (`+ init * rate 60`) cannot be done by printing in the actions as `post.y` does. Why not? Say what each `e` must carry instead.

<details>
<summary>Answer</summary>

In prefix the operator comes *before* its operands, but the parser only knows which operator a subexpression has when it reduces it, after both operands have been reduced (and, if printed as you go, already printed). So the output cannot be written in reduction order. Instead each `e` carries its translation as a string value: `%union { char *s; }`, `%type <s> e`, and the action for `e '+' e` builds a new string `"+ " $1 " " $3` (with `malloc` and `sprintf`), freeing the two old ones; `e : ID` passes the name up. The line's action prints `$1` of `e`. The same method gives postfix, infix with full brackets, or any other order, and it is the synthesised attribute of section 9.1.

</details>

<!-- q: idea=semantic-action type=fix wrong="strdup is only needed for numbers" -->
3. A student's `post.l` has `yylval.s = yytext; return ID;` for names. Predict what `post` prints for `a + b`, and explain. What would `free($1)` in `post.y` then do?

<details>
<summary>Answer</summary>

Every `yylval.s` points at the same place, Flex's buffer, where `yytext` lives. By the time `e : ID` runs for `a`, the scanner has already matched the `+` (the parser read it as the lookahead), so the string at that pointer is no longer `a`, and the name would come out wrong. But the program does not get that far in one piece: `free($1)` frees memory that `malloc` never gave, a pointer into Flex's buffer. That is undefined behaviour. On the book's machine the C library noticed and stopped the program at once, with `double free or corruption (!prev)` and exit status 134, before any of the translated text left the buffer, so nothing at all was printed. `strdup` gives each value its own memory, which is safe to keep and to free.
<!-- checked: how=run date=2026-11-21 ref="scratch post.l with yylval.s = yytext on 'a + b': glibc 'double free or corruption (!prev)', exit 134, no output" -->

</details>

## 8.7 Practice: the lab set of Yacc programs, written from the question alone

You can now write the three parts of a `.y` file and build it with a Lex scanner (8.1), validate expressions and recover from errors (8.2), compute values with precedence and associativity (8.3), recognise the classic languages (8.4), read Bison's conflict report (8.5) and translate as you parse with `%union` and mid-rule actions (8.6). The lab exam gives you one question from a list like this one, a blank folder and about an hour, and the examiner asks you to explain any line. These tasks climb from a variation on a program you have written, through two problems the chapter has not shown, to the Yacc parser of the mini compiler. Work in `~/cdlab/ch8`. For each one, write the `.y` and `.l` files from the question, build them with `yacc -d`, `flex` and `gcc`, and test them on a good input, an empty input and a bad one before you open the answer.

### Guided

<!-- q: idea=yacc-value type=apply wrong="one #define YYSTYPE can hold both a number and a variable" -->
**1. A calculator with variables.** Extend `calc` to `calcvar`: 26 variables named `a` to `z`, all 0 at the start; a line `x = e` stores the value of `e` in `x` and prints `x = value`; a line that is only an expression prints `= value` as before; a variable in an expression stands for its value. Leave out `^` and the division check. Test it on `i = 10`, `r = 2.5`, `i + r * 60`, `p = i + r * 60`, `p / 4`, `q` and `x = = 1`.

Hint: a number's value is a `double`, but a variable's token must say *which* variable, an `int`. That is two types, so use `%union`.

<details>
<summary>Answer</summary>

<!-- code: id=calcvar -->
```console
$ cat > calcvar.l <<'END'
%{
/* calcvar.l: numbers, one-letter variables, and single characters */
#include <stdlib.h>
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
[0-9]+(\.[0-9]+)?    { yylval.d = atof(yytext); return NUM; }
[a-z]                { yylval.i = yytext[0] - 'a'; return VAR; }
[ \t]+               { /* blanks */ }
\n                   { return '\n'; }
.                    { return yytext[0]; }
%%
END
```

<!-- code: id=calcvar-y -->
```console
$ cat > calcvar.y <<'END'
%{
/* calcvar.y: a calculator with 26 variables, a to z */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
double var[26];          /* the value of each variable, 0 at the start */
%}
%union {
    double d;            /* the value of a number or an expression */
    int i;               /* the index of a variable, 0 for a */
}
%token <d> NUM
%token <i> VAR
%type  <d> e
%left '+' '-'
%left '*' '/'
%right UMINUS
%%
lines : /* nothing */
      | lines line
      ;
line  : e '\n'            { printf("= %g\n", $1); }
      | VAR '=' e '\n'    { var[$1] = $3; printf("%c = %g\n", 'a' + $1, $3); }
      | '\n'
      | error '\n'        { yyerrok; }
      ;
e     : e '+' e           { $$ = $1 + $3; }
      | e '-' e           { $$ = $1 - $3; }
      | e '*' e           { $$ = $1 * $3; }
      | e '/' e           { $$ = $1 / $3; }
      | '-' e %prec UMINUS  { $$ = -$2; }
      | '(' e ')'         { $$ = $2; }
      | NUM
      | VAR               { $$ = var[$1]; }
      ;
%%
void yyerror(const char *s)
{
    fprintf(stderr, "error: %s\n", s);
}

int main(void)
{
    return yyparse();
}
END
```

<!-- code: id=calcvar-run -->
```console
$ yacc -d calcvar.y
$ flex calcvar.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o calcvar
$ printf 'i = 10\nr = 2.5\ni + r * 60\np = i + r * 60\np / 4\nq\nx = = 1\n' | ./calcvar
i = 10
r = 2.5
= 160
p = 160
= 40
= 0
error: syntax error
```

`%type <d> e` gives every `e` a `double` value, `NUM` carries a `double` and `VAR` an `int`, the variable's index. In `VAR '=' e '\n'`, `$1` is that index and `$3` the value. `q` was never assigned, so it is 0. A `VAR` at the start of a line is either the target of an assignment or the start of an expression; the parser decides when it sees the next token, `=` or not, and Bison builds it without a conflict. The usual wrong answer keeps `#define YYSTYPE double` and stores the index in a `double`; it works by luck in the parser but needs casts everywhere, and the scanner's file must repeat the `#define`. With `%union`, `y.tab.h` carries the type to both files.

</details>

<!-- q: idea=yacc-validator type=apply unseen="c-declaration" wrong="a comma after the last name is allowed" -->
**2. Is it a C declaration?** Write `decl.y` and `decl.l`. Each input line must be a declaration of the form `type name, name, ... ;`, where the type is `int`, `float`, `char` or `double`, and a name may be an array with a size, `avg[10]`. Print `valid declaration` or `invalid declaration` for each line. Test it on `int a, b, c;`, `float rate, avg[10];`, `int;`, `int a b;`, `char a,;`, `double x[n];` and `int int;`.

<details>
<summary>Answer</summary>

<!-- code: id=decl -->
```console
$ cat > decl.l <<'END'
%{
/* decl.l: the scanner for decl.y */
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
int|float|char|double       { return TYPE; }
[A-Za-z_][A-Za-z0-9_]*      { return ID; }
[0-9]+                      { return NUM; }
[ \t]+                      { /* blanks */ }
\n                          { return '\n'; }
.                           { return yytext[0]; }
%%
END
$ cat > decl.y <<'END'
%{
/* decl.y: is each line a valid C declaration such as int a, b[10], c; ? */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
%}
%token TYPE ID NUM
%%
lines : /* nothing */
      | lines line
      ;
line  : decl '\n'       { printf("valid declaration\n"); }
      | error '\n'      { yyerrok; }
      ;
decl  : TYPE list ';'
      ;
list  : item
      | list ',' item
      ;
item  : ID
      | ID '[' NUM ']'
      ;
%%
void yyerror(const char *s)
{
    printf("invalid declaration (%s)\n", s);
}

int main(void)
{
    return yyparse();
}
END
$ yacc -d decl.y
$ flex decl.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o decl
$ printf 'int a, b, c;\nfloat rate, avg[10];\nint;\nint a b;\nchar a,;\ndouble x[n];\nint int;\n' | ./decl
valid declaration
valid declaration
invalid declaration (syntax error)
invalid declaration (syntax error)
invalid declaration (syntax error)
invalid declaration (syntax error)
invalid declaration (syntax error)
```

The list is the comma-separated list of section 5.5, written left-recursively, as an LR parser likes it: one `item`, or a list, a comma and one more `item`. It cannot end in a comma, so `char a,;` is invalid; it cannot be empty, so `int;` is invalid. The array size must be a `NUM`, so `x[n]` is rejected. `int int` is rejected by the scanner's rule order (section 4.2): the keyword rule comes before the name rule, so the second `int` is a `TYPE`, not an `ID`. The keywords are one token, `TYPE`, because the grammar treats them all alike; a compiler would give the type a value, which is the next chapter's business.

</details>

### On your own

<!-- q: idea=yacc-language type=apply unseen="wcwr" wrong="w c w^R needs the parser to compare two halves in an action" -->
**3. A mirror language.** Write a Yacc program that accepts the lines of the language w c w^R, where w is any string of `a`s and `b`s (possibly empty) and w^R is w backwards: `c`, `abcba`, `aabcbaa`, but not `abcab` or `ab`. Say whether the language is regular, and why the `c` makes the grammar easy for an LR parser.

<details>
<summary>Answer</summary>

Each character before the `c` must be matched by the same character after it, in mirror order, which is the matching of a^n b^n with two kinds of pair:

```text
s : A s A
  | B s B
  | C
  ;
```

<!-- code: id=wcwr -->
```console
$ cat > wcw.l <<'END'
%{
/* wcw.l: a, b, c and newline for wcw.y */
#include "y.tab.h"
%}
%option noyywrap nounput noinput
%%
a        { return A; }
b        { return B; }
c        { return C; }
\n       { return '\n'; }
.        { return yytext[0]; }
%%
END
$ cat > wcw.y <<'END'
%{
/* wcw.y: is each line w c w-reversed, w over {a, b}? */
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
%}
%token A B C
%%
lines : /* nothing */
      | lines line
      ;
line  : s '\n'          { printf("accepted\n"); }
      | error '\n'      { yyerrok; }
      ;
s     : A s A
      | B s B
      | C
      ;
%%
void yyerror(const char *s)
{
    printf("rejected (%s)\n", s);
}

int main(void)
{
    return yyparse();
}
END
$ yacc -d wcw.y
$ flex wcw.l
$ gcc -Wall -Wextra y.tab.c lex.yy.c -o wcw
$ printf 'c\nabcba\naabcbaa\nabcab\nab\n' | ./wcw
accepted
accepted
accepted
rejected (syntax error)
rejected (syntax error)
```

Not regular: like a^n b^n, it needs to remember an unbounded first half. The `c` makes it easy: the parser shifts every `a` and `b` until the `c`, reduces `C` to `s` there, and from then on each incoming character must match the one on the stack below the `s`. Without the marker (the even palindromes w w^R), the parser could not know where the middle is, and Bison reports conflicts: that language needs a parser that can guess, which no LR(1) parser can.
<!-- checked: how=run date=2026-11-21 ref="Bison 3.8.2 on s : A s A | B s B | (empty): 2 shift/reduce conflicts" -->

</details>

<!-- q: idea=yacc-conflict-report type=decide unseen="comma-list" wrong="any list grammar is ambiguous" -->
**4. A list with a conflict.** A student writes a list of names as `s : ID | s ',' s`. Bison warns `1 shift/reduce conflict`. Build the report, find the state with the conflict and the counterexample, and explain the two trees. Then give two fixes: one that changes the grammar, and one that keeps the grammar and adds a declaration. Which would you hand in?

<details>
<summary>Answer</summary>

<!-- code: id=clist -->
```console
$ cat > clist.y <<'END'
%token ID
%%
s : ID
  | s ',' s
  ;
END
$ yacc -v clist.y
clist.y: warning: 1 shift/reduce conflict [-Wconflicts-sr]
clist.y: note: rerun with option '-Wcounterexamples' to generate conflict counterexamples
$ head -1 y.output
State 5 conflicts: 1 shift/reduce
$ sed -n '/^State 5$/,$p' y.output
State 5

    2 s: s • ',' s
    2  | s ',' s •

    ','  shift, and go to state 4

    ','       [reduce using rule 2 (s)]
    $default  reduce using rule 2 (s)
$ yacc -Wcounterexamples clist.y 2>&1 | grep Example
  Example: s ',' s • ',' s
```

After `s , s`, with another `,` next, the parser can reduce (the first two items form a list, which the third joins: `(a, b), c`) or shift (the comma starts a list `b, c` inside: `a, (b, c)`). Both trees describe the same list of three names, which is why the program would work either way; the grammar is ambiguous all the same. Fix 1, the grammar: `s : ID | s ',' ID`, the left-recursive list of section 5.5 and of `decl.y`, with one tree per list and no conflict. Fix 2, a declaration: `%left ','`, which settles the conflict as a reduce. Hand in fix 1: it says what a list is, and it leaves nothing for a reader to wonder about.

</details>

### A lab program from a blank file

<!-- q: idea=semantic-action type=build wrong="the scanner's own enum can be kept beside y.tab.h" -->
**5. tiny.y, the parser of the mini compiler.** Write the fourth file of the mini compiler, `tiny.y`, with its scanner `tiny.l`. Start the scanner from chapter 4's (`cp ../ch4/tiny.l .`) and change only what Yacc needs. The parser reads a Tiny program of assignments: `ID = expr ;`, any number of them, with `+ - * /` and parentheses, `*` and `/` binding tighter and all four grouping to the left. For each statement it prints one line: the target, ` =`, and the expression in postfix, each item preceded by a blank (`pos = init rate 60 * +`). At the end it prints `N statement(s), no syntax errors`, as `tiny-rd` did in section 6.1. On a syntax error it ends the line already started, prints `line L: syntax error` to standard error, where L is the scanner's line, and exits with status 1. A lexical error is reported by the scanner and counted; if there were any, the program prints `N lexical error(s)` to standard error at the end instead of the count of statements, and exits with status 1.

**Done when** `./tiny` with this input:

```text
/* area of a ring */
pi = 3.14159;
r1 = 2; r2 = 5;
area = pi * (r2 * r2 - r1 * r1);   // 65.97
ring = area / 2
     - 1;
```

prints exactly the model's output below. You can test that without anyone marking it: in the understudy folder (the one that holds `book/`), in the terminal, run `python3 scripts/us_practice.py check compiler-design 8.7 --in ~/cdlab/ch8`. It runs `yacc -d tiny.y`, `flex tiny.l`, builds the two C files and compares what `./tiny` prints for this input with the model. Then test it, as habit 3 asks, on an empty input, which gives `0 statement(s)`, on `pos = init + ;`, and on a program with a `while` in its second line, which must stop at line 2.

<details>
<summary>Answer</summary>

The changes to `tiny.l` are the ones section 8.1 named. The `enum` of token codes and the `name` table go: the codes come from `y.tab.h`, which the definitions now include. The names and numbers are passed on as copies in `yylval.s`, as in `post.l`. The unclosed comment returns 0 instead of `END`, the end of input. And `main` goes, since the parser's file has it. Everything else, every pattern and the error function, is chapter 4's.

The model, as a script that writes both files, builds them and runs the parser on the input above:

<!-- code: id=tiny check="yacc -d tiny.y && flex tiny.l && gcc -Wall -Wextra y.tab.c lex.yy.c -o tiny && ./tiny" stdin="/* area of a ring */\npi = 3.14159;\nr1 = 2; r2 = 5;\narea = pi * (r2 * r2 - r1 * r1);   // 65.97\nring = area / 2\n     - 1;\n" -->
```bash
cat > tiny.l <<'END'
%{
/* tiny.l for tiny.y: chapter 4's scanner, returning y.tab.h's codes;
   a name or number goes to the parser as a copy in yylval.s */
#include <stdio.h>
#include <string.h>
#include "y.tab.h"
int errors;                    /* lexical errors so far */
void error(const char *msg);
%}
%option noyywrap nounput noinput yylineno

letter   [A-Za-z_]
digit    [0-9]
id       {letter}({letter}|{digit})*
num      {digit}+(\.{digit}+)?

%%
[ \t\r\n]+                     { /* blanks: nothing to return */ }
"//".*                         { /* a line comment */ }
"/*"([^*]|"*"+[^*/])*"*"+"/"   { /* a block comment */ }
"/*"                           { error("comment not closed"); return 0; }

"if"          { return IF; }
"while"       { return WHILE; }
{id}          { yylval.s = strdup(yytext); return ID; }
{num}         { yylval.s = strdup(yytext); return NUM; }
{digit}+"."   { error("digit expected after '.'"); }
{num}{id}     { error("letters after a number"); }

"=="          { return EQ; }
"!="          { return NE; }
"<="          { return LE; }
">="          { return GE; }
"="           { return ASSIGN; }
"<"           { return LT; }
">"           { return GT; }
"+"           { return PLUS; }
"-"           { return MINUS; }
"*"           { return TIMES; }
"/"           { return DIVIDE; }
"("           { return LPAREN; }
")"           { return RPAREN; }
";"           { return SEMI; }
"!"           { error("'!' without '='"); }
.             { error("unexpected character"); }
%%
void error(const char *msg)
{
    fprintf(stderr, "line %d: error: %s: %s\n", yylineno, msg, yytext);
    errors++;
}
END
cat > tiny.y <<'END'
%{
/* tiny.y: a Yacc parser for Tiny's assignments; prints each one with
   its expression in postfix. The scanner is tiny.l. */
#include <stdio.h>
#include <stdlib.h>
int yylex(void);
void yyerror(const char *s);
extern int yylineno;           /* the scanner's line, from %option yylineno */
extern int errors;             /* lexical errors, counted in tiny.l */
int stmts;                     /* statements parsed */
int open;                      /* 1 while a statement's line is unfinished */
%}
%union {
    char *s;                   /* the text of a name or a number */
}
%token <s> ID NUM
%token IF WHILE ASSIGN EQ NE LT LE GT GE
%token PLUS MINUS TIMES DIVIDE LPAREN RPAREN SEMI
%left PLUS MINUS
%left TIMES DIVIDE
%%
program : /* nothing */
        | program stmt         { stmts++; }
        ;
stmt    : ID ASSIGN            { printf("%s =", $1); free($1); open = 1; }
          expr SEMI            { printf("\n"); open = 0; }
        ;
expr    : expr PLUS expr       { printf(" +"); }
        | expr MINUS expr      { printf(" -"); }
        | expr TIMES expr      { printf(" *"); }
        | expr DIVIDE expr     { printf(" /"); }
        | LPAREN expr RPAREN
        | ID                   { printf(" %s", $1); free($1); }
        | NUM                  { printf(" %s", $1); free($1); }
        ;
%%
void yyerror(const char *s)
{
    if (open)
        printf("\n");          /* end the statement's line first */
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
    printf("%d statement(s), no syntax errors\n", stmts);
    return 0;
}
END
yacc -d tiny.y && flex tiny.l && gcc -Wall -Wextra y.tab.c lex.yy.c -o tiny && ./tiny
```
```text
pi = 3.14159
r1 = 2
r2 = 5
area = pi r2 r2 * r1 r1 * - *
ring = area 2 / 1 -
5 statement(s), no syntax errors
```

The parser in a few lines. Every token of chapter 4 is declared, including `IF`, `WHILE` and the comparisons that no rule uses yet, so that the scanner can return them and the parser reports them as syntax errors instead of failing to compile; chapter 9 gives them rules. `ID` and `NUM` carry `<s>`. The expression grammar is the ambiguous one with two `%left` lines, as in `calc.y`. The statement uses a mid-rule action to print the target before the expression, as `post.y` did. `yyerror` exits, so `main` only gets past `yyparse` on success, and `open` makes sure the half-printed line is ended before the message. The tests of habit 3:

<!-- code: id=tiny-tests exit=1 -->
```console
$ ./tiny < /dev/null
0 statement(s), no syntax errors
$ printf 'pos = init + ;\n' | ./tiny; echo "exit $?"
pos = init
line 1: syntax error
exit 1
$ printf 'x = 9lives;\ny = 2;\n' | ./tiny; echo "exit $?"
line 1: error: letters after a number: 9lives
x =
line 1: syntax error
exit 1
$ printf 'a = 1;\nwhile (a) b = 2;\n' | ./tiny
a = 1
line 2: syntax error
```

`pos = init + ;` stops at the `;`, after `pos = init` was printed. In `x = 9lives;` the scanner reports the bad number and drops it, so the parser sees `x = ;` and stops too: one lexical message, then one syntax message, as `tiny-rd` gave. The `while` program translates its first line and stops at line 2. Set `tiny-rd` from section 6.1 beside it: the same language, a third of the code, and the precedence in two lines instead of two functions.

</details>
