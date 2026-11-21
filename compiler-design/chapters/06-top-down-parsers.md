# Chapter 6: Top-down parsers

<!-- contents -->
- [6.1 A recursive-descent parser in C](#61-a-recursive-descent-parser-in-c)
- [6.2 Building the LL(1) parsing table](#62-building-the-ll1-parsing-table)
- [6.3 Is a grammar LL(1)? Finding the conflict](#63-is-a-grammar-ll1-finding-the-conflict)
- [6.4 A table-driven predictive parser in C, with its stack trace](#64-a-table-driven-predictive-parser-in-c-with-its-stack-trace)
- [6.5 Error recovery: panic mode and synch entries](#65-error-recovery-panic-mode-and-synch-entries)
- [6.6 Practice: grammar to table to parser for a new language](#66-practice-grammar-to-table-to-parser-for-a-new-language)
<!-- /contents -->

## 6.1 A recursive-descent parser in C

### Tokens that nobody has checked yet

Feed `tiny-scan` the line `pos = init + ;` and it prints five tokens without a word of complaint: `id`, `=`, `id`, `+`, `;`. Every lexeme is a good lexeme. The scanner of chapter 2 knows patterns, not order, so it cannot see that a `+` with nothing after it is not an expression. Chapter 5 wrote down what the order must be, as a grammar, and showed with `derive` that `pos = init + rate * 60;` has a parse tree. But `derive` needed you to give it the production numbers. Nobody has yet written the program that finds the tree by itself, from the tokens alone, and says "no" to `pos = init + ;`.

That program is the parser, the second station of the assembly line in section 1.2, and this chapter writes it twice. This section writes it the way you would by hand, one C function per nonterminal: a *recursive-descent* parser. It is the parser you will write most often in the lab, because it needs no table and no tool, and it is the third file of the mini compiler, `tiny-rd.c`.

### One worker per nonterminal

Picture the parser station staffed by one worker per nonterminal. The `E` worker knows only the productions for `E`; the `T` worker only those for `T`. When the `E` worker's production says "a `T`, then an `E'`", it does not build the `T` itself: it hands the job to the `T` worker and waits. The `T` worker may in turn hand a job to the `F` worker. Only when the `F` worker finishes does the `T` worker carry on, and only when the `T` worker finishes does `E` move to its `E'`.

The waiting workers form the book's stack of plates. Each worker who hands a job on puts a plate on the pile, saying where it was in its production; the worker at the top of the pile is the only one working; when it finishes, its plate comes off, and the worker below picks up exactly where it stopped. In C, each worker is a function, a call puts a plate on the pile, and a return takes it off. The pile is the C call stack, and you do not have to write it.

Two places where the picture stops being true. First, the assembly line in section 1.2 showed the scanner handing a finished token list to the parser. A real parser does not wait for a list: whenever a worker needs to see the next token, the parser calls `gettoken()` for one more. The two stations run interleaved, one token at a time. Second, you cannot look at this pile of plates: the call stack belongs to the C runtime, and the program cannot print it. Section 6.4 builds the same parser with a pile you can see, an array, and prints it at every step.

### How each function decides

<!-- teaches: recursive-descent = recursive-descent parser, lookahead token, match -->
A *recursive-descent parser* is a set of functions, one for each nonterminal, each of which reads the part of the input that its nonterminal derives. They are recursive because the grammar is: `F -> ( E )` makes the `F` function call the `E` function, which can call `F` again for the inner parentheses. The parser keeps one token in hand, the *lookahead token* (here a global variable `tok`): the next token of the input, read but not yet used. The functions are built from four rules.

1. **A terminal in a production is matched.** A function `match(t)` checks that the lookahead is `t`, then reads the next token into `tok`. If the lookahead is anything else, the input is wrong here, and `match` reports it.
2. **A nonterminal in a production is called.** For `T -> F T'`, the body of `T` is a call of `F`, then a call of `T'`.
3. **A choice between alternatives is made by the lookahead.** For `F -> ( E ) | id`, look at `tok`: if it is in FIRST(`( E )`) = { ( }, take the first alternative; if it is in FIRST(`id`) = { id }, take the second. That is the use section 5.5 promised for FIRST sets.
4. **An ε-alternative is taken when the lookahead can follow the nonterminal.** For `T' -> * F T' | ε`, take `* F T'` on `*`; take ε (do nothing and return) only on a token in FOLLOW(T') = { +, ), $ }, the rule section 5.6 ended on. On any other token, report an error at once.

Rule 3 only works if the alternatives of a nonterminal cannot both start with the token in hand. That is why chapter 5 prepared the grammar first. A left-recursive production would have the `E` function call `E` as its first act, with the same token in hand, forever: section 5.3's `leftloop` ran until the stack overflowed and the program died with `Segmentation fault`. Two alternatives with a common prefix, such as `A -> a b c | a b d`, would both be chosen by `a`, so section 5.4 left-factored them. So the grammar this parser follows is `exprll.g`, the expression grammar with left recursion removed:

```text
E  -> T E'          E' -> + T E' | eps
T  -> F T'          T' -> * F T' | eps
F  -> ( E ) | id
```

### The expression parser, rd.c

Lab sheets that ask for "a recursive-descent parser for the expression grammar" usually give the input as a short string with `i` standing for `id`, such as `i+i*i`, the same shorthand as the lab-sheet format of section 5.7. This first parser does exactly that, so that you can see the method without a scanner in the way: every character except a blank is one token, and the end of the line is `$`. It prints each production the moment it chooses it, indented by how many functions are running, so that you can watch the pile of plates grow and shrink.

Make the chapter's folder. Open a terminal in your home folder:

<!-- code: id=ch6-setup -->
```console
$ mkdir cdlab/ch6
$ cd cdlab/ch6
```

Save this as `rd.c`:

<!-- code: id=rd mode=compile -->
```c
/* rd.c: a recursive-descent parser for the expression grammar
       E  -> T E'          E' -> + T E' | eps
       T  -> F T'          T' -> * F T' | eps
       F  -> ( E ) | i
   Reads one line. Every character but a blank is one token, i stands
   for id, and the end of the line is $. Prints each production as it
   is chosen, indented by depth, then "accepted" or an error. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char line[256];    /* the input */
int at;            /* where the next token starts in line */
int col;           /* the column of the lookahead, counted from 1 */
int tok;           /* the lookahead token; '$' at the end of the line */
int depth;         /* how many parsing functions are running */

void next(void)    /* make the next token the lookahead */
{
    while (line[at] == ' ' || line[at] == '\t')
        at++;
    col = at + 1;
    if (line[at] == '\0')
        tok = '$';
    else
        tok = line[at++];
}

void error(const char *expected)
{
    if (tok == '$')
        printf("error at column %d: expected %s, found end of input\n",
               col, expected);
    else
        printf("error at column %d: expected %s, found %c\n",
               col, expected, tok);
    printf("rejected\n");
    exit(1);
}

void match(int t)  /* the lookahead must be t: use it up */
{
    char want[2] = { (char) t, '\0' };

    if (tok != t)
        error(want);
    next();
}

void say(const char *production)   /* one line of the trace */
{
    printf("%*s%s\n", 2 * (depth - 1), "", production);
}

void E(void);

void F(void)                       /* F -> ( E ) | i */
{
    depth++;
    if (tok == '(') {
        say("F -> ( E )");
        match('(');
        E();
        match(')');
    } else if (tok == 'i') {
        say("F -> i");
        match('i');
    } else
        error("( or i");
    depth--;
}

void Tp(void)                      /* T' -> * F T' | eps */
{
    depth++;
    if (tok == '*') {
        say("T' -> * F T'");
        match('*');
        F();
        Tp();
    } else if (tok == '+' || tok == ')' || tok == '$')   /* FOLLOW(T') */
        say("T' -> eps");
    else
        error("*, +, ) or end of input");
    depth--;
}

void T(void)                       /* T -> F T' */
{
    depth++;
    say("T -> F T'");
    F();
    Tp();
    depth--;
}

void Ep(void)                      /* E' -> + T E' | eps */
{
    depth++;
    if (tok == '+') {
        say("E' -> + T E'");
        match('+');
        T();
        Ep();
    } else if (tok == ')' || tok == '$')                 /* FOLLOW(E') */
        say("E' -> eps");
    else
        error("+, ) or end of input");
    depth--;
}

void E(void)                       /* E -> T E' */
{
    depth++;
    say("E -> T E'");
    T();
    Ep();
    depth--;
}

int main(void)
{
    if (fgets(line, sizeof line, stdin) == NULL) {
        fprintf(stderr, "rd: no input\n");
        return 1;
    }
    line[strcspn(line, "\r\n")] = '\0';
    next();                        /* the first lookahead */
    E();
    if (tok != '$')
        error("end of input");
    printf("accepted\n");
    return 0;
}
```

Read it against the four rules. `next` is the parser's whole contact with the input: it skips blanks, remembers the column, and puts one token in `tok`. `match` is rule 1. The five parsing functions are the five nonterminals; `E'` and `T'` become `Ep` and `Tp`, because a C name cannot hold a prime. `E` and `T` have one production each, so they make no decision at all: they announce it and make the calls of rule 2. `F` makes a rule 3 decision between `(` and `i`, and its last `else` is the case where the lookahead starts neither alternative. `Ep` and `Tp` each make a rule 3 decision for the non-empty alternative and a rule 4 decision for ε, with the FOLLOW set written out in the condition, and an error for everything else.

`main` reads the first lookahead before calling anything, calls the function of the start symbol, and then checks that nothing is left. That last check matters: `E` returning means that an expression was found at the *start* of the input, not that the input was only an expression.

`depth` counts the running functions. Each function adds one on entry and takes one away on return, so `say` indents each production by two spaces for every plate under it.

### The running statement's expression, traced

The expression on the right of `pos = init + rate * 60;` is `id + id * num`. This grammar has no `num`, so stand an `i` in for every operand:

<!-- code: id=rd-run -->
```console
$ gcc -Wall -Wextra rd.c -o rd
$ echo 'i + i * i' | ./rd
E -> T E'
  T -> F T'
    F -> i
    T' -> eps
  E' -> + T E'
    T -> F T'
      F -> i
      T' -> * F T'
        F -> i
        T' -> eps
    E' -> eps
accepted
$ echo '(i+i)*i' | ./rd
E -> T E'
  T -> F T'
    F -> ( E )
      E -> T E'
        T -> F T'
          F -> i
          T' -> eps
        E' -> + T E'
          T -> F T'
            F -> i
            T' -> eps
          E' -> eps
    T' -> * F T'
      F -> i
      T' -> eps
  E' -> eps
accepted
```

Follow the first run with the lookahead and the pile of plates side by side. The pile is written bottom first, so the function running now is the last one named.

| Step | Lookahead | Pile of plates | What happens |
|---|---|---|---|
| 1 | `i` (column 1) | E | `E` has one production: prints `E -> T E'`, calls `T` |
| 2 | `i` | E T | `T` has one production: prints `T -> F T'`, calls `F` |
| 3 | `i` | E T F | `i` is in FIRST(`i`): prints `F -> i`, matches `i`; the lookahead becomes `+` |
| 4 | `+` | E T T' | `F` returned; `T` calls `Tp`. `+` is not `*`, but it is in FOLLOW(T'): prints `T' -> eps`, returns |
| 5 | `+` | E E' | `T` returned; `E` calls `Ep`. `+` chooses `E' -> + T E'`: matches `+`; the lookahead becomes `i` |
| 6 | `i` | E E' T F | as steps 2 and 3, one level deeper; after the match the lookahead is `*` |
| 7 | `*` | E E' T T' | `*` chooses `T' -> * F T'`: matches `*`, calls `F`, which matches the last `i`; the lookahead becomes `$` |
| 8 | `$` | E E' T T' T' | the inner `Tp`: `$` is in FOLLOW(T'): ε |
| 9 | `$` | E E' E' | the inner `Ep`: `$` is in FOLLOW(E'): ε. Every function returns |
| 10 | `$` | (empty) | `main` finds `$` and prints `accepted` |

Two things to notice. In step 4 the `T'` that ends the first term is empty, and the parser knew it was safe to make it empty only because `+` can follow a `T'`. And at no step did any function guess and come back: each choice was made once, by one token, and was right. That is what preparing the grammar bought.

### The output is a leftmost derivation and a parse tree

Read the productions in the order they were printed: `E -> T E'`, `T -> F T'`, `F -> i`, `T' -> eps`, `E' -> + T E'`, and so on. Each one rewrites the leftmost nonterminal still waiting, because the functions run left to right and finish everything to the left before they start anything to the right. So the trace is the leftmost derivation of the input. Section 5.1's `derive` can confirm it: number the productions of `exprll.g` as `derive` lists them (1 `E -> T E'`, 2 `E' -> + T E'`, 3 `E' -> eps`, 4 `T -> F T'`, 5 `T' -> * F T'`, 6 `T' -> eps`, 7 `F -> ( E )`, 8 `F -> id`) and give it the trace of the first run as numbers:

<!-- code: id=rd-derive -->
```console
$ ../ch5/derive ../ch5/exprll.g 1 4 8 6 2 4 8 5 8 6 3 | head -12
     E
=>lm T E'   [1]
=>lm F T' E'   [4]
=>lm id T' E'   [8]
=>lm id E'   [6]
=>lm id + T E'   [2]
=>lm id + F T' E'   [4]
=>lm id + id T' E'   [8]
=>lm id + id * F T' E'   [5]
=>lm id + id * id T' E'   [8]
=>lm id + id * id E'   [6]
=>lm id + id * id   [3]
```

`derive` accepted every step, and its last line is `id + id * id`. In chapter 5 you had to work out those numbers yourself; the parser worked them out from the tokens. The indentation of the trace is the parse tree as well, drawn the way `derive` draws trees: each production's children are the lines indented under it.

### When the input is wrong

Now the inputs that must be rejected. A `for` loop in the shell runs `rd` once for each string, so one command tests five inputs:

<!-- code: id=rd-errors exit=1 -->
```console
$ for s in 'i+*i' 'i i' '(i+i' 'i)' ''; do echo "== $s"; echo "$s" | ./rd; done
== i+*i
E -> T E'
  T -> F T'
    F -> i
    T' -> eps
  E' -> + T E'
    T -> F T'
error at column 3: expected ( or i, found *
rejected
== i i
E -> T E'
  T -> F T'
    F -> i
error at column 3: expected *, +, ) or end of input, found i
rejected
== (i+i
E -> T E'
  T -> F T'
    F -> ( E )
      E -> T E'
        T -> F T'
          F -> i
          T' -> eps
        E' -> + T E'
          T -> F T'
            F -> i
            T' -> eps
          E' -> eps
error at column 5: expected ), found end of input
rejected
== i)
E -> T E'
  T -> F T'
    F -> i
    T' -> eps
  E' -> eps
error at column 2: expected end of input, found )
rejected
==
E -> T E'
  T -> F T'
error at column 1: expected ( or i, found end of input
rejected
```

Each error is found at the first token that cannot continue a correct expression, and the message names what would have been allowed there.

- `i+*i`: after `+`, `E'` called `T`, `T` called `F`, and `F` holds `*`, which starts neither `( E )` nor `i`.
- `i i`: after the first `i`, `Tp` holds `i`. It is not `*`, and it is not in FOLLOW(T'), so rule 4 refuses ε at once. The message lists both: what would continue the term (`*`) and what may come after it (`+`, `)`, the end).
- `(i+i`: the inner expression is complete, both ε-choices are allowed at `$` (which is in FOLLOW(E') and FOLLOW(T')), and the error comes from `match(')')` in `F`, the one place that was waiting for the parenthesis.
- `i)`: `)` can follow an `E'`, so the parser takes ε twice and returns all the way to `main`. Only `main`'s final check finds that nothing was waiting for this `)`. Without that check, `rd` would have printed `accepted`, because `E` did find an expression, `i`, at the start of the input.
- The empty line: the very first decision, in `F`, finds the end of the input.

And the input with no line at all, which `fgets` reports as end of file:

<!-- code: id=rd-empty exit=1 -->
```console
$ ./rd < /dev/null
rd: no input
```

### Recursive descent with backtracking

Exam papers also ask about the other kind of recursive descent, the kind that guesses. Take `S -> c A d`, `A -> a b | a`, and the input `c a d`. A parser that cannot decide by the lookahead tries the alternatives in order: `S` matches `c`; `A` tries `a b`, matches `a`, and then finds `d` where it wanted `b`. So it *backtracks*: it puts the input pointer back to where `A` started, at `a`, and tries `A -> a`, which matches; then `S` matches `d`, and the input is accepted.

Backtracking works, but it is slow (every failure re-reads input, and the failures multiply with nesting), it makes error messages almost useless (which of the failed attempts was the real mistake?), and it cannot be done at all if a production has already done something that cannot be undone, such as printing a line of the trace. Section 5.4 is the cure: left-factor `A -> a b | a` into `A -> a A'`, `A' -> b | ε`, and the lookahead decides. Every parser in this chapter is recursive descent *without* backtracking.

### From recursion to loops: tiny-rd.c

Look at `Ep` again. Its non-empty alternative ends by calling `Ep`, and nothing happens after that call except returning. A function whose last act is to call itself is a loop in disguise: "match `+`, call `T`, and do it all again while the lookahead is `+`". So `E -> T E'` with `E' -> + T E' | ε` can be written as one function:

<!-- code: id=e-loop mode=fragment -->
```c
void E(void)                 /* E -> T { + T } */
{
    T();
    while (tok == '+') {
        match('+');
        T();
    }
}
```

The braces in the comment are the usual shorthand for "zero or more times", so `E -> T { + T }` reads "a `T`, then any number of `+ T`". It is section 5.3's `T (+ T)*` again, written straight into C. The loop form is shorter, it needs no `E'`, and it keeps every operand of one expression in one function, which is what chapter 9 needs when it makes the parser compute values and generate code. One thing is lost: the loop stops on any token that is not `+`, without checking FOLLOW. That is safe, because the token is not used; the caller looks at it next and complains if it is wrong. The error is found at the same token, but by the caller, so the message names what the caller wanted.

Tiny's statements are assignments, with `+ - * /` and parentheses. In the same shorthand:

```text
program -> { stmt } end
stmt    -> id = expr ;
expr    -> term { ( + | - ) term }
term    -> factor { ( * | / ) factor }
factor  -> ( expr ) | id | num
```

`end` is the scanner's `END` token, the `$` of the grammar. A program is zero or more statements, so an empty file is a correct (if useless) program.

The parser needs a scanner, and it already exists: `tiny-scan.c`, from section 2.7. Its `gettoken` is exactly the function a parser calls for one more token. Only its `main`, which prints the token list, is in the way. One `sed` command copies the file without it: `/^int main/,$d` means "from the line that starts with `int main` to the last line, delete", and the result goes to a new file, `scan.h`. The original is not touched.

<!-- code: id=scan-h -->
```console
$ sed '/^int main/,$d' ../ch2/tiny-scan.c > scan.h
```

Save the parser as `tiny-rd.c`:

<!-- code: id=tiny-rd mode=compile -->
```c
/* tiny-rd.c: a recursive-descent parser for Tiny's assignments.
       program -> { stmt } end
       stmt    -> id = expr ;
       expr    -> term { ( + | - ) term }
       term    -> factor { ( * | / ) factor }
       factor  -> ( expr ) | id | num
   The scanner is tiny-scan.c without its main, in scan.h.
   usage: ./tiny-rd [-t] < file     (-t prints the parse tree) */
#include <stdlib.h>
#include "scan.h"

int tok;           /* the lookahead token */
int depth;         /* the tree's indentation level */
int show;          /* 1 with -t */

void advance(void) /* the next token; the scanner reports bad ones */
{
    do
        tok = gettoken();
    while (tok == ERROR);
}

void syntax(const char *expected)
{
    if (tok == END)
        fprintf(stderr, "line %d: syntax error: expected %s, "
                "found end of input\n", tokline, expected);
    else
        fprintf(stderr, "line %d: syntax error: expected %s, "
                "found '%s'\n", tokline, expected, lexeme);
    exit(1);
}

void node(const char *what)        /* a nonterminal's line in the tree */
{
    if (show)
        printf("%*s%s\n", 2 * depth, "", what);
}

void expect(int t)                 /* match token t, show it, move on */
{
    if (tok != t)
        syntax(name[t]);
    if (show && (t == ID || t == NUM))
        printf("%*s%s %s\n", 2 * depth, "", name[t], lexeme);
    else if (show)
        printf("%*s%s\n", 2 * depth, "", name[t]);
    advance();
}

void expr(void);

void factor(void)                  /* factor -> ( expr ) | id | num */
{
    node("factor");
    depth++;
    if (tok == LPAREN) {
        expect(LPAREN);
        expr();
        expect(RPAREN);
    } else if (tok == ID)
        expect(ID);
    else if (tok == NUM)
        expect(NUM);
    else
        syntax("(, id or num");
    depth--;
}

void term(void)                    /* term -> factor { ( * | / ) factor } */
{
    node("term");
    depth++;
    factor();
    while (tok == TIMES || tok == DIVIDE) {
        expect(tok);
        factor();
    }
    depth--;
}

void expr(void)                    /* expr -> term { ( + | - ) term } */
{
    node("expr");
    depth++;
    term();
    while (tok == PLUS || tok == MINUS) {
        expect(tok);
        term();
    }
    depth--;
}

void stmt(void)                    /* stmt -> id = expr ; */
{
    node("stmt");
    depth++;
    expect(ID);
    expect(ASSIGN);
    expr();
    expect(SEMI);
    depth--;
}

int main(int argc, char *argv[])
{
    int n = 0;

    show = argc > 1 && strcmp(argv[1], "-t") == 0;
    advance();                     /* the first lookahead */
    while (tok != END) {           /* program -> { stmt } end */
        stmt();
        n++;
    }
    if (errors > 0) {
        fprintf(stderr, "%d lexical error(s)\n", errors);
        return 1;
    }
    printf("%d statement(s), no syntax errors\n", n);
    return 0;
}
```

The shape is `rd.c`'s, with three differences. The tokens come from `gettoken`, by way of `advance`, which also skips the `ERROR` tokens: `tiny-scan` has already printed a message for each one and counted it in `errors`, so the parser carries on with the next good token, and `main` reports the lexical errors at the end. The terminals are matched by `expect`, which uses the scanner's `name` table for its messages and `tokline` for the line number. And `expr` and `term` are loops. `factor` is the one function with a real choice, made by FIRST: `(`, `id` or `num`. `main` is the `{ stmt }` loop: while the lookahead is not `END`, there must be another statement.

With `-t`, each function prints its nonterminal on entry and `expect` prints each terminal it matches, with the lexeme for `id` and `num`. Because `depth` goes up after a node is printed, a node's children are indented under it.

Build it and parse the running statement:

<!-- code: id=tiny-rd-run -->
```console
$ gcc -Wall -Wextra tiny-rd.c -o tiny-rd
$ echo 'pos = init + rate * 60;' | ./tiny-rd -t
stmt
  id pos
  =
  expr
    term
      factor
        id init
    +
    term
      factor
        id rate
      *
      factor
        num 60
  ;
1 statement(s), no syntax errors
```

This is the tree of section 1.2, found by a program. Under `expr` there are two terms joined by `+`, and the `*` sits inside the second term, so `rate * 60` is one unit before the addition happens: the precedence that the grammar of section 5.2 built in. The tree is flatter than Figure 5.1a because the loops put all the terms of one expression side by side under one `expr`, instead of in a chain of `E` and `E'` nodes.

Now a real file. `prog.tiny` from chapter 2 starts with three assignments and then has a `while` loop, which Tiny's grammar will only get in chapter 9. The first five lines of the file hold the three assignments:

<!-- code: id=tiny-rd-files exit=1 -->
```console
$ cp ../ch2/prog.tiny .
$ head -5 prog.tiny > assign.tiny
$ ./tiny-rd < assign.tiny
3 statement(s), no syntax errors
$ ./tiny-rd < prog.tiny
line 6: syntax error: expected id, found 'while'
```

`assign.tiny` has a comment, three statements, a statement split over two lines and a `//` comment; the scanner removed the comments and the line break, and the parser saw three statements. In `prog.tiny`, the parser met `while` where a statement must start, and a statement must start with `id`. The scanner had no complaint (`while` is a keyword it knows); the parser has one, because its grammar has no statement that begins with `while` yet.

Then the empty input, and the inputs that must be rejected:

<!-- code: id=tiny-rd-errors exit=1 -->
```console
$ ./tiny-rd < /dev/null
0 statement(s), no syntax errors
$ for s in 'pos = init + ;' 'pos = (init + rate * 60;' 'pos = init rate;' 'x = 9lives;'; do echo "$s" | ./tiny-rd; done
line 1: syntax error: expected (, id or num, found ';'
line 1: syntax error: expected ), found ';'
line 1: syntax error: expected ;, found 'rate'
line 1: error: letters after a number: 9lives
line 1: syntax error: expected (, id or num, found ';'
```

The first is the line that opened this section: after `+`, `expr`'s loop called `term`, `term` called `factor`, and `factor` found `;`. The second is a parenthesis never closed: the inner `expr` stopped at `;`, and `factor`'s `expect(RPAREN)` found it. The third shows the loop form at work: after `init`, the loops in `term` and `expr` stopped at `rate` (neither an operator), and `stmt`, which wanted the `;`, reported it. The last line shows both stations: the scanner reported `9lives` and dropped it, so the parser saw `x = ;` and reported the missing operand. Notice that the first error stops the parser: one syntax error, one message, exit. Section 6.5 shows how a parser can carry on after an error.

### When it goes wrong

**The program crashes with `Segmentation fault` on every input.** The grammar still has left recursion, and some function calls itself before matching anything; GCC usually warns first, as it did for section 5.3's `leftloop` (`infinite recursion detected`). Remove the left recursion, or use the loop form.

**Two alternatives both run.** Writing `if (tok == '(') { ... }` and then a separate `if (tok == 'i') { ... } else error(...);` without the `else` in between: after a parenthesised `F` succeeds, the second `if` looks at the new lookahead, does not find `i`, and reports an error on correct input. The choices for one nonterminal are one `if ... else if ... else` chain.

**`accepted` for input with something left over.** The check after the start function was left out, so `i)` or `pos = 1; )` passes. The start function only promises that the input *begins* with a sentence.

**Nothing works from the first token.** The first lookahead was never read: `main` called `E()` with `tok` still 0, so every decision fails at once. Call `next()` (or `advance()`) before the start function.

**The trace shows a production chosen on a token that is not in its FIRST set.** A mistake copied from a wrong FIRST or FOLLOW set. Run `../ch5/follow` on the grammar and compare each condition in your code with the sets it prints, as habit 7 asks.

### In one line each

1. One function per nonterminal; a terminal is matched, a nonterminal is called, in the order of the production.
2. The lookahead chooses the alternative by FIRST; an ε-alternative is taken on a token in FOLLOW, and anything else is an error.
3. The grammar must be free of left recursion and left-factored, or one token cannot choose.
4. The functions run in the order of a leftmost derivation; the C call stack is the parser's pile of plates.
5. A tail-recursive `E'` is a `while` loop; the loop leaves the error to the caller.
6. After the start function returns, the lookahead must be the end of the input.

**Check yourself**

<!-- q: idea=recursive-descent type=predict wrong="the deepest call prints first" -->
1. Predict every line `rd` prints for `(i)`, with its indentation, then run it.

<details>
<summary>Answer</summary>

`E`, `T` and `F` each have one choice at `(`; inside the parentheses the same happens for `i`; then the two ε-choices inside, at `)`, which is in both FOLLOW sets; then `F` matches `)` and the two outer ε-choices happen at `$`. Each line is printed when its function *starts*, so the outermost production comes first, not the deepest.

<!-- code: id=rd-paren -->
```console
$ echo '(i)' | ./rd
E -> T E'
  T -> F T'
    F -> ( E )
      E -> T E'
        T -> F T'
          F -> i
          T' -> eps
        E' -> eps
    T' -> eps
  E' -> eps
accepted
```

</details>

<!-- q: idea=recursive-descent type=explain wrong="the scanner is at fault when the message names a token that is present" -->
2. Find the fault, or explain it away. A student runs `echo 'pos = init rate;' | ./tiny-rd` and says the parser is wrong: "there *is* a semicolon on the line, yet it says `expected ;`". Explain which function printed the message and why `;` is the right thing to ask for.

<details>
<summary>Answer</summary>

The parser is right. After `init`, `factor` returns, and the loops in `term` and `expr` look at the lookahead, `rate`: it is not `*`, `/`, `+` or `-`, so both loops end and both functions return without using it. Back in `stmt`, the next symbol of `stmt -> id = expr ;` is `;`, and `expect(SEMI)` finds `rate` instead. From the parser's point of view the expression ended after `init`, and the only thing that may follow an expression in a statement is `;`. The semicolon later in the line is never reached; the parser stops at the first error.

</details>

<!-- q: idea=recursive-descent type=apply wrong="each alternative needs its own function" -->
3. Write the recursive-descent function, in the style of `rd.c`, for `S -> a S b | c`. Which tokens choose which alternative, and what happens on any other token?

<details>
<summary>Answer</summary>

FIRST(`a S b`) = { a } and FIRST(`c`) = { c }; there is no ε-alternative, so every other token is an error. One function holds both alternatives:

<!-- code: id=s-func mode=fragment -->
```c
void S(void)                       /* S -> a S b | c */
{
    if (tok == 'a') {
        match('a');
        S();
        match('b');
    } else if (tok == 'c')
        match('c');
    else
        error("a or c");
}
```

The call `S()` in the middle is the recursion the grammar asks for; it is safe because `match('a')` has used a token before it, so each call starts further along the input.

</details>

<!-- q: idea=recursive-descent type=decide wrong="a recursive-descent parser needs the whole statement to choose" -->
4. Decide. Tiny is to get procedure calls, `stmt -> id = expr ; | id ( ) ;`. Can `stmt` still choose with one lookahead token? If not, what do you change, and how many tokens does the new version look at before it chooses?

<details>
<summary>Answer</summary>

Not as written: both alternatives start with `id`, so FIRST sets overlap and the `id` in hand cannot choose. Left-factor it (section 5.4): `stmt -> id rest`, `rest -> = expr ; | ( ) ;`. Now `stmt` matches `id` with no decision, and `rest` chooses by the *next* token, `=` or `(`. The parser never looks more than one token ahead; it simply makes the decision one token later, after the common prefix has been used.

</details>

## 6.2 Building the LL(1) parsing table

### The decisions rd.c made, in one grid

Look again at how `rd.c` decides. The set FIRST(`( E )`) = { ( } is hidden in the condition `tok == '('` inside `F`. FOLLOW(T') = { +, ), $ } is spread over the condition `tok == '+' || tok == ')' || tok == '$'` inside `Tp`. Every other token falls into an `else` that calls `error`. The sets of sections 5.5 and 5.6 are all there, but scattered through five functions. To check them you have to read the whole program, and to change the grammar you have to rewrite it.

The written paper asks for the same decisions in a different form: "construct the predictive parsing table for the grammar". That table is every decision of every parsing function, collected in one grid, with a row for each nonterminal and a column for each terminal. Once you have it, two things follow. Section 6.4 writes a parser that consults the grid instead of a chain of `if`s, so a new grammar needs a new grid and not a new program. And section 6.3 uses the grid to answer the question every exam paper asks next: can this grammar be parsed this way at all?

### The board game, with plates on the squares

Chapter 3 pictured a DFA as a board game: each square says where each input card sends you, and the transition table is the rule book, one row per square and one column per card. The parsing table looks like that rule book. Its rows are the nonterminals, its columns are the tokens, and the entry where row `T'` meets column `+` says what to do when the parser is about to read a `T'` and the card in its hand is `+`.

The picture stops being true in what the entry tells you. A DFA's entry names one square, and you move there. A parsing table's entry names a *production*, and the parser replaces the nonterminal with that production's whole right side: one plate taken off the pile, several put on. A pile that can grow without limit is exactly what a finite automaton does not have, so this is not a DFA, and the parser of section 6.4 needs its stack. The other part of the picture holds: a blank square means the card you hold cannot be played here, which in a DFA table was the dead state and here is a syntax error.

### Two rules, one production at a time

<!-- teaches: ll1-table = LL(1) parsing table, predictive parsing table, M[A, a] -->
The table is called the *LL(1) parsing table*, or the *predictive parsing table*, and it is written M. The entry in row A and column a is M[A, a]: the production to use when the parser is about to read an A and the lookahead token is a. The columns are the terminals and `$`, never ε, because ε is not something the lookahead can ever be. The table is filled by visiting every production A -> α of the grammar once and applying two rules to it:

1. **For every terminal a in FIRST(α), put A -> α in M[A, a].**
2. **If ε is in FIRST(α), then for every terminal b in FOLLOW(A), put A -> α in M[A, b]. If `$` is in FOLLOW(A), put A -> α in M[A, $] as well.**

Every entry that neither rule fills stays blank, and a blank entry means "error".

These are the two decisions of section 6.1, nothing more. Rule 1 is how `F` chose: an alternative is taken on any token that can begin it. Rule 2 is how `Tp` chose ε: an alternative that can vanish is taken on any token that can come after the nonterminal, because when α vanishes, the token in hand must belong to whatever follows A. The blank entries are the `error` calls.

Two details in the wording matter, and both cost marks when they are missed. Rule 2 asks whether ε is in FIRST(α), not whether α *is* ε. A production such as `D -> E F`, where both E and F can vanish, has ε in FIRST(`E F`), so rule 2 applies to it too. And rule 2 uses FOLLOW of A, the left side, not FOLLOW of any symbol inside α: when α vanishes, the parser has finished A, and what comes next is whatever follows A.

### The expression grammar's table, by hand

Take `exprll.g`, the grammar `rd.c` follows:

```text
E  -> T E'          E' -> + T E' | eps
T  -> F T'          T' -> * F T' | eps
F  -> ( E ) | id
```

The rules need FIRST of each right side and FOLLOW of each left side. You computed both in chapter 5; `first` printed the right sides' FIRST sets in section 5.5 and `follow` printed the FOLLOW sets in section 5.6. Copy them down beside the grammar, because every entry comes from one of them:

| Nonterminal | FOLLOW |
|---|---|
| E | { ), $ } |
| E' | { ), $ } |
| T | { +, ), $ } |
| T' | { +, ), $ } |
| F | { +, *, ), $ } |

Now visit the eight productions in order. For each, write FIRST of its right side, decide whether rule 2 applies, and list the entries it makes:

| Production | FIRST(α) | ε in it? | Entries |
|---|---|---|---|
| `E -> T E'` | { (, id } | no | M[E, (], M[E, id] |
| `E' -> + T E'` | { + } | no | M[E', +] |
| `E' -> eps` | { ε } | yes: FOLLOW(E') = { ), $ } | M[E', )], M[E', $] |
| `T -> F T'` | { (, id } | no | M[T, (], M[T, id] |
| `T' -> * F T'` | { * } | no | M[T', *] |
| `T' -> eps` | { ε } | yes: FOLLOW(T') = { +, ), $ } | M[T', +], M[T', )], M[T', $] |
| `F -> ( E )` | { ( } | no | M[F, (] |
| `F -> id` | { id } | no | M[F, id] |

Look at the third row. FIRST(ε) is { ε }, which holds no terminal, so rule 1 puts `E' -> eps` nowhere; rule 2 then puts it under every token of FOLLOW(E'). An ε-production lives only in the FOLLOW columns of its left side.

Fourteen entries in all. Put them in the grid, columns in the order the tokens appear in the grammar, `$` last:

| | + | * | ( | ) | id | $ |
|---|---|---|---|---|---|---|
| **E** | | | E -> T E' | | E -> T E' | |
| **E'** | E' -> + T E' | | | E' -> ε | | E' -> ε |
| **T** | | | T -> F T' | | T -> F T' | |
| **T'** | T' -> ε | T' -> * F T' | | T' -> ε | | T' -> ε |
| **F** | | | F -> ( E ) | | F -> id | |

That is the answer the paper wants, and it is worth reading back against `rd.c` row by row. Row F has two entries, `(` and `id`: the two branches of `F`'s `if`, and the blanks in that row are its `error("( or i")`. Row T' has `* F T'` under `*` and ε under `+`, `)` and `$`: exactly `Tp`'s two conditions. Rows E and T have the same production in both their entries, because those functions make no choice; they only check, through the functions they call, that the token can begin an E or a T at all.

The table even explains `rd.c`'s error messages. When `Tp` met `i` in `i i`, it printed `expected *, +, ) or end of input`: the non-blank columns of row T'. A message that lists the tokens that would have been accepted is the row of the table read aloud.

### Change one thing: a nullable symbol in the middle

The expression grammar's ε-entries all come from productions that are ε. Section 5.5's `null.g` has a production that is not ε but whose FIRST walk passes through two nullable symbols:

```text
S -> A B c
A -> a | eps
B -> b | eps
```

From chapter 5: FIRST(`A B c`) = { a, b, c } (the walk goes past A and B, which can both vanish, to `c`, which cannot), FOLLOW(A) = { b, c } and FOLLOW(B) = { c }.

| Production | FIRST(α) | ε in it? | Entries |
|---|---|---|---|
| `S -> A B c` | { a, b, c } | no | M[S, a], M[S, b], M[S, c] |
| `A -> a` | { a } | no | M[A, a] |
| `A -> eps` | { ε } | yes: FOLLOW(A) = { b, c } | M[A, b], M[A, c] |
| `B -> b` | { b } | no | M[B, b] |
| `B -> eps` | { ε } | yes: FOLLOW(B) = { c } | M[B, c] |

| | c | a | b | $ |
|---|---|---|---|---|
| **S** | S -> A B c | S -> A B c | S -> A B c | |
| **A** | A -> ε | A -> a | A -> ε | |
| **B** | B -> ε | | B -> b | |

Read row S with an input in mind. On `b c`, the parser must choose S's production while holding `b`, a token that does not belong to S directly at all; it belongs to B, two symbols in. Rule 1 still gets it right, because FIRST(`A B c`) walked through A to reach B's `b`. Then, at A with `b` still in hand, M[A, b] says vanish, because `b` can follow A. The `$` column of every row is blank: `$` is in no FIRST and, here, only in FOLLOW(S), whose one production cannot vanish. So an empty input is refused on the very first move, which is right, since every sentence of this grammar ends in `c`.

### The table in C: ll1.c

Building the table by hand is where the paper's marks are, and checking it by program is habit 7. The program needs FIRST and FOLLOW, which chapter 5 already wrote: `grammar.h` reads the grammar, `first.h` computes FIRST, and `computefollow` in `follow.c` computes FOLLOW. Copy the two headers into this chapter's folder, and take `follow.c` without its `main`, with the same `sed` command that made `scan.h` in section 6.1:

<!-- code: id=ll1-setup -->
```console
$ cp ../ch5/grammar.h ../ch5/first.h .
$ sed '/^int main/,$d' ../ch5/follow.c > follow.h
```

`follow.h` now starts with `#include "grammar.h"` and `#include "first.h"`, finds both in this folder, and ends just before `follow.c`'s `main`. Save the table builder as `ll1.c`:

<!-- code: id=ll1 mode=compile -->
```c
/* ll1.c: the LL(1) parsing table of a grammar, and its conflicts.
   Reads a grammar on standard input (grammar.h's form). Needs
   follow.h, which is ../ch5/follow.c without its main. */
#include "follow.h"

#define MAXC 4                      /* productions kept per entry */
int entry[MAXSYM][MAXSYM][MAXC];    /* entry[A][a][k]: a production */
int count[MAXSYM][MAXSYM];          /* how many are in M[A, a] */
int conflicts;                      /* entries with more than one */

void enter(int a, int t, int p)     /* add production p to M[a, t] */
{
    int k;

    for (k = 0; k < count[a][t]; k++)
        if (entry[a][t][k] == p)
            return;                 /* there already */
    if (count[a][t] == MAXC)
        die("more than %d productions in one entry", MAXC);
    entry[a][t][count[a][t]++] = p;
    if (count[a][t] == 2)
        conflicts++;
}

void buildtable(void)               /* the two rules, for every production */
{
    int p, t;
    set f;

    computefirst(0);
    computefollow();
    for (p = 0; p < np; p++) {
        f = firstof(rhs[p], len[p]);
        for (t = 1; t < nsym; t++) {
            if (isnt[t])
                continue;
            if (f & bit(t))                         /* rule 1 */
                enter(lhs[p], t, p);
            if ((f & bit(EPS)) && (follow[lhs[p]] & bit(t)))
                enter(lhs[p], t, p);                /* rule 2 */
        }
    }
}

int column[MAXSYM], ncol;           /* the terminals in grammar order, then $ */

void columns(void)
{
    int s;

    for (s = 2; s < nsym; s++)
        if (!isnt[s])
            column[ncol++] = s;
    column[ncol++] = END;
}

char *cell(int a, int t)            /* M[a, t] in short form: "E->TE'" */
{
    static char buf[MAXC * (MAXR + 2) * MAXLEN];
    int k, i, p;

    buf[0] = '\0';
    for (k = 0; k < count[a][t]; k++) {
        p = entry[a][t][k];
        if (k > 0)
            strcat(buf, " ");
        strcat(buf, name[a]);
        strcat(buf, "->");
        if (len[p] == 0)
            strcat(buf, "eps");
        for (i = 0; i < len[p]; i++)
            strcat(buf, name[rhs[p][i]]);
    }
    return buf;
}

int main(void)
{
    int i, j, a, n, w[MAXSYM];

    prog = "ll1";
    readgrammar();
    buildtable();
    columns();
    for (j = 0; j < ncol; j++) {     /* each column as wide as its widest */
        w[j] = strlen(name[column[j]]);
        for (i = 0; i < nnt; i++) {
            n = strlen(cell(ntorder[i], column[j]));
            if (n > w[j])
                w[j] = n;
        }
    }
    printf("%-4s", "");
    for (j = 0; j < ncol; j++)
        printf("| %-*s ", w[j], name[column[j]]);
    printf("\n");
    for (i = 0; i < nnt; i++) {
        a = ntorder[i];
        printf("%-4s", name[a]);
        for (j = 0; j < ncol; j++)
            printf("| %-*s ", w[j], cell(a, column[j]));
        printf("\n");
    }
    if (conflicts == 0) {
        printf("LL(1): yes, no entry has two productions\n");
        return 0;
    }
    for (i = 0; i < nnt; i++)
        for (j = 0; j < ncol; j++)
            if (count[ntorder[i]][column[j]] > 1)
                printf("conflict: M[%s, %s] = %s\n", name[ntorder[i]],
                       name[column[j]], cell(ntorder[i], column[j]));
    printf("LL(1): no, %d entr%s with two or more productions\n",
           conflicts, conflicts == 1 ? "y" : "ies");
    return 0;
}
```

`buildtable` is the two rules and nothing else. It computes FIRST and FOLLOW with chapter 5's functions, then visits every production `p`, takes `f` = FIRST of its right side from `firstof`, and walks every terminal `t`, `$` included (symbol 1 is `$`, and the loop starts at 1 to take it in; symbol 0, ε, is skipped). If `t` is in `f`, rule 1 enters the production. If ε is in `f` and `t` is in FOLLOW of the left side, rule 2 enters it. Since `$` is never in a FIRST set, only rule 2 can ever fill a `$` column.

An entry is a small list, not a single number: `entry[A][a]` holds up to four productions and `count[A][a]` says how many. In a grammar that suits this kind of parser, every count is 0 or 1. A count of 2 or more is called a conflict; `enter` counts each such entry once, and `main` lists them at the end. Section 6.3 is about what they mean. `enter` also refuses to put the same production into the same entry twice. That can happen when α can vanish and one of its first tokens can also follow A; both rules then reach the same entry with the same production, which is one choice, not two, and must not be counted as a conflict.

The rest is printing. `cell` writes an entry in the short form exam answers use, `E->TE'`, the symbols run together; a conflict entry shows all its productions side by side. `main` makes each column as wide as its widest entry, so that the table fits a terminal.

Build it and run it on the two grammars you just did by hand:

<!-- code: id=ll1-run -->
```console
$ gcc -Wall -Wextra ll1.c -o ll1
$ ./ll1 < ../ch5/exprll.g
    | +        | *        | (      | )       | id     | $
E   |          |          | E->TE' |         | E->TE' |
E'  | E'->+TE' |          |        | E'->eps |        | E'->eps
T   |          |          | T->FT' |         | T->FT' |
T'  | T'->eps  | T'->*FT' |        | T'->eps |        | T'->eps
F   |          |          | F->(E) |         | F->id  |
LL(1): yes, no entry has two productions
$ ./ll1 < ../ch5/null.g
    | c      | a      | b      | $
S   | S->ABc | S->ABc | S->ABc |
A   | A->eps | A->a   | A->eps |
B   | B->eps |        | B->b   |
LL(1): yes, no entry has two productions
```

Both are the hand tables, entry for entry and blank for blank. The column order is the order in which the tokens first appear in the grammar file, as in `first`'s sets; a table's columns have no required order, and in the paper you may use any order you like, as long as `$` is there.

The whole of chapter 5 and this section now fit in one line, from the left-recursive grammar to the table: chapter 5's `leftrec` and `leftfac`, run from this folder as `../ch5/leftrec` and `../ch5/leftfac`, feed `ll1`. Then the inputs that must be refused:

<!-- code: id=ll1-more exit=1 -->
```console
$ ../ch5/leftrec < ../ch5/expr.g | ../ch5/leftfac | ./ll1
    | +        | *        | (      | )       | id     | $
E   |          |          | E->TE' |         | E->TE' |
E'  | E'->+TE' |          |        | E'->eps |        | E'->eps
T   |          |          | T->FT' |         | T->FT' |
T'  | T'->eps  | T'->*FT' |        | T'->eps |        | T'->eps
F   |          |          | F->(E) |         | F->id  |
LL(1): yes, no entry has two productions
$ ./ll1 < /dev/null; echo "exit status $?"
ll1: no grammar on input
exit status 1
$ printf 'S -> a |\n' | ./ll1
ll1: line 1: empty alternative (write eps)
```

The messages come from `grammar.h`'s reader, which `ll1` shares with every tool of chapter 5.

### When it goes wrong

**An ε column in your table.** ε is never a column: the lookahead is always a real token or `$`. An ε-production goes in the FOLLOW columns of its left side. If your table has a column headed ε, every entry in it belongs in the FOLLOW columns instead.

**ε-productions under FIRST.** Writing `T' -> ε` under `*` because "FIRST(T') is { *, ε }". Rule 1 uses FIRST of the production's own right side, which for `T' -> ε` holds no terminal at all. FIRST of the nonterminal is the union over all its productions and is the wrong set here.

**A FOLLOW entry for a production that cannot vanish.** Putting `T' -> * F T'` under `+` because T' is nullable. Rule 2 is about whether *this* α can vanish, not whether A can. A nullable nonterminal usually has one production that can vanish and others that cannot; only the one that can goes in the FOLLOW columns.

**A missing `$` column.** Without it, the ε-entries at the end of the input are lost, and the parser of section 6.4 rejects every correct input on its last move, with the input used up. `$` is a column in every LL(1) table.

**Your table and `ll1`'s disagree.** First check that the grammar file says what your paper says, with blanks between the symbols. Then run `../ch5/follow` on the same file: a wrong entry nearly always comes from a wrong FOLLOW set, and the program prints the sets your entries came from.

### In one line each

1. M[A, a] is the production to use when the parser is about to read A and the lookahead is a; a blank is an error.
2. Rule 1: A -> α goes under every terminal in FIRST(α).
3. Rule 2: if α can vanish, A -> α also goes under every terminal of FOLLOW(A), and under `$` if `$` is in FOLLOW(A).
4. The rules work one production at a time; the columns are terminals and `$`, never ε.
5. The table is `rd.c`'s decisions in one grid; `ll1` builds it from chapter 5's FIRST and FOLLOW code.

**Check yourself**

<!-- q: idea=ll1-table type=apply wrong="an epsilon-production goes under FIRST of its nonterminal" -->
1. Build by hand the LL(1) table for `S -> a S b | eps`, then check it with `ll1`.

<details>
<summary>Answer</summary>

FIRST(`a S b`) = { a }, so `S -> a S b` goes in M[S, a]. FOLLOW(S) = { b, $ }: `$` because S is the start symbol, `b` because of the `b` after S in `S -> a S b`. So `S -> eps` goes in M[S, b] and M[S, $].

<!-- code: id=ll1-anbn -->
```console
$ printf 'S -> a S b | eps\n' | ./ll1
    | a      | b      | $
S   | S->aSb | S->eps | S->eps
LL(1): yes, no entry has two productions
```

</details>

<!-- q: idea=ll1-table type=fix wrong="rule 2 uses FOLLOW of the symbols inside the production" -->
2. Find the fault. For `adh.g` of section 5.7 (`S -> a B D h`, `B -> c C`, `C -> b C | eps`, `D -> E F`, `E -> g | eps`, `F -> f | eps`), a student puts `D -> E F` under `g` and `f` only, reasoning "FIRST(E F) is { g, f, ε }, and ε has no column". What is missing, and why?

<details>
<summary>Answer</summary>

`D -> E F` can vanish (both E and F can), so rule 2 applies to it: it also goes under every terminal of FOLLOW(D) = { h }. The student dropped the ε and with it rule 2. Without M[D, h], the input `a c h` is rejected at D, although D should simply vanish there. `./ll1 < ../ch5/adh.g` shows `D->EF` in three columns: `h`, `g` and `f`.

<!-- code: id=ll1-adh -->
```console
$ ./ll1 < ../ch5/adh.g
    | a       | h      | c     | b     | g      | f      | $
S   | S->aBDh |        |       |       |        |        |
B   |         |        | B->cC |       |        |        |
C   |         | C->eps |       | C->bC | C->eps | C->eps |
D   |         | D->EF  |       |       | D->EF  | D->EF  |
E   |         | E->eps |       |       | E->g   | E->eps |
F   |         | F->eps |       |       |        | F->f   |
LL(1): yes, no entry has two productions
```

</details>

<!-- q: idea=ll1-table type=predict wrong="a blank entry means the parser does nothing" -->
3. Predict. In the expression grammar's table, the parser is about to read an `F` and the lookahead is `+`. What does the table say, and which input that `rd` rejected in section 6.1 reaches exactly this entry?

<details>
<summary>Answer</summary>

M[F, +] is blank: `+` can neither begin an F nor let it vanish, since F has no ε-production. A blank is a syntax error. `rd` reached the same decision on `i+*i`, but with `*`: after the `+`, `F` held `*` and printed `expected ( or i, found *`. An input like `i*+i` reaches M[F, +] itself: after `*`, the parser needs an F and finds `+`.

</details>

## 6.3 Is a grammar LL(1)? Finding the conflict

### Two productions in one square

Every table in section 6.2 ended with the line `LL(1): yes, no entry has two productions`. The rules do not promise that. They put a production into every entry its FIRST or FOLLOW set reaches, and nothing stops two productions of the same nonterminal from reaching the same entry. Section 5.6 has already shown you one: in the left-factored dangling-else grammar, the token `e` is in FIRST(`e S`) and also in FOLLOW(S'), so M[S', e] gets both `S' -> e S` and `S' -> ε`.

In the board-game picture, that is a square whose rule says two different things for the same card. A player who meets it cannot move without guessing, and a parser that guesses is the backtracking parser of section 6.1, which this chapter set out to avoid. The table is the place where you find out, before writing a line of the parser, whether one token will always be enough. The exam asks it in three forms: "construct the table and say whether the grammar is LL(1)", "show that the grammar is not LL(1)", and "check, without constructing the table, whether the grammar is LL(1)". This section answers all three.

### What LL(1) means

<!-- teaches: ll1-conflict = LL(1) grammar, LL(1) conflict, FIRST/FIRST conflict, FIRST/FOLLOW conflict -->
A grammar is *LL(1)* when its LL(1) parsing table has at most one production in every entry. An entry with two or more is an *LL(1) conflict*, and a grammar with one conflict is not LL(1), however many entries are fine. The name spells out the parser this table drives. The first L: it reads the input from Left to right. The second L: it produces a Leftmost derivation, as `rd`'s trace was. The 1: it decides with one token of lookahead.

A conflict always comes from one of two causes, and it helps to name them by the rules that made it.

- A *FIRST/FIRST conflict*: rule 1 puts two productions A -> α and A -> β in the same entry, because some terminal is in both FIRST(α) and FIRST(β). Both alternatives can begin with the token in hand.
- A *FIRST/FOLLOW conflict*: rule 1 puts A -> α in an entry and rule 2 puts A -> β there too, because β can vanish and some terminal is in both FIRST(α) and FOLLOW(A). The token in hand could begin α, or it could be what comes after an A that has vanished.

Rule 2 can also clash with itself: if two alternatives can both vanish, rule 2 puts both under every token of FOLLOW(A), and under `$`. That is a conflict even when FOLLOW(A) holds only `$`.

### Checking without the table: three conditions

The table makes conflicts easy to see, but for a quick answer you can check the same thing directly. A grammar is LL(1) exactly when, for every nonterminal A and every two of its alternatives A -> α | β:

1. FIRST(α) and FIRST(β) have no terminal in common. (No FIRST/FIRST conflict.)
2. At most one of α and β can derive ε. (Rule 2 does not clash with itself.)
3. If β can derive ε, then FIRST(α) and FOLLOW(A) have no terminal in common; and the same with α and β swapped. (No FIRST/FOLLOW conflict.)

Each condition is one way that two productions could land in the same entry, so together they say that none can. A nonterminal with one production passes trivially: there is no pair. So in practice you only look at the nonterminals with alternatives, which in an exam grammar is usually two or three of them.

Try it on `exprll.g`. Only E', T' and F have alternatives. E': FIRST(`+ T E'`) = { + } and FIRST(ε) = { ε }; no common terminal; only ε can vanish; and FIRST(`+ T E'`) = { + } against FOLLOW(E') = { ), $ }: nothing in common. T' is the same with `*` against { +, ), $ }. F: { ( } against { id }, and neither can vanish. All three pass, and the grammar is LL(1), as `ll1` said. Notice that being nullable did not hurt E' or T'. A nullable nonterminal is fine; what matters is whether its FOLLOW set collides with the FIRST of its other alternatives.

### Four grammars that fail, and why

Each of chapter 5's problem grammars fails in a way that tells you what to do about it. Run them through `ll1`:

<!-- code: id=ll1-expr -->
```console
$ ./ll1 < ../ch5/expr.g
    | + | * | (           | ) | id          | $
E   |   |   | E->E+T E->T |   | E->E+T E->T |
T   |   |   | T->T*F T->F |   | T->T*F T->F |
F   |   |   | F->(E)      |   | F->id       |
conflict: M[E, (] = E->E+T E->T
conflict: M[E, id] = E->E+T E->T
conflict: M[T, (] = T->T*F T->F
conflict: M[T, id] = T->T*F T->F
LL(1): no, 4 entries with two or more productions
```

**Left recursion gives FIRST/FIRST conflicts.** FIRST(`E + T`) is FIRST(E), which includes FIRST(T), so `E -> E + T` and `E -> T` share every token that can begin a T. This always happens: for A -> A α | β, everything in FIRST(β) is in FIRST(A α) too. So a left-recursive grammar is never LL(1), and section 5.3's transformation is not optional. It is the same fact as section 6.1's `leftloop` crash, seen from the table instead of the call stack.

<!-- code: id=ll1-prefix -->
```console
$ ./ll1 < ../ch5/prefix.g
    | a                   | b | c | d | e | $
A   | A->abc A->abd A->ae |   |   |   |   |
conflict: M[A, a] = A->abc A->abd A->ae
LL(1): no, 1 entry with two or more productions
$ ../ch5/leftfac < ../ch5/prefix.g | ./ll1
    | a       | b        | e      | c     | d     | $
A   | A->aA'' |          |        |       |       |
A'' |         | A''->bA' | A''->e |       |       |
A'  |         |          |        | A'->c | A'->d |
LL(1): yes, no entry has two productions
```

**A common prefix gives a FIRST/FIRST conflict.** All three alternatives of `A -> a b c | a b d | a e` begin with `a`, so all three land in M[A, a]: an entry with three productions, counted once. After left factoring, each decision is made one token later, after the shared part has been read, and every entry holds one production. That was the promise of section 5.4; the table is the proof.

<!-- code: id=ll1-dangle -->
```console
$ ../ch5/leftfac < ../ch5/dangle.g | ./ll1
    | i         | t | a    | e              | b    | $
S   | S->iEtSS' |   | S->a |                |      |
S'  |           |   |      | S'->eS S'->eps |      | S'->eps
E   |           |   |      |                | E->b |
conflict: M[S', e] = S'->eS S'->eps
LL(1): no, 1 entry with two or more productions
```

**The dangling else gives a FIRST/FOLLOW conflict that no transformation removes.** Left factoring cured the FIRST/FIRST conflict at M[S, i] (both `if` forms began with `i E t S`). What is left is condition 3 failing for S': `S' -> ε` can vanish, and `e` is in both FIRST(`e S`) and FOLLOW(S'). Why is `e` in FOLLOW(S')? In `S -> i E t S S'`, the inner S is followed by S', whose FIRST holds `e`, so `e` is in FOLLOW(S). And S' ends the production for S, so by rule 3 of section 5.6, FOLLOW(S) is in FOLLOW(S'). An `else` can follow a whole `if` statement that is itself inside a `then` part. That is the ambiguity of section 5.2, exactly: in `i b t i b t a e a`, the `e` could close the inner `if` or the outer one.

<!-- code: id=ll1-amb -->
```console
$ ./ll1 < ../ch5/amb.g
    | + | * | (                    | ) | id                  | $
E   |   |   | E->E+E E->E*E E->(E) |   | E->E+E E->E*E E->id |
conflict: M[E, (] = E->E+E E->E*E E->(E)
conflict: M[E, id] = E->E+E E->E*E E->id
LL(1): no, 2 entries with two or more productions
```

**An ambiguous grammar is never LL(1).** An LL(1) parser makes every choice by one token, with no alternatives, so for each sentence it can follow only one leftmost derivation, and a grammar with two parse trees for some sentence has two leftmost derivations for it. Somewhere along the second one, a choice was made differently on the same token, and that token's entry must hold both productions. `amb.g` fails three ways at once: it is ambiguous, left-recursive, and its alternatives share prefixes.

### The converse is false

So: left-recursive, not left-factored, or ambiguous means not LL(1). It is tempting to read that backwards, and the exam sometimes tempts you: "the grammar is unambiguous and has no left recursion, so it is LL(1)". It does not follow. Take a language whose sentences are some `a`s followed by as many `b`s or by as many `c`s, such as `a a b b` and `a c`:

<!-- code: id=ll1-abc -->
```console
$ cat > abc.g <<'END'
S -> A | B
A -> a A b | a b
B -> a B c | a c
END
$ ../ch5/leftfac < abc.g | ./ll1
    | a         | b     | c     | $
S   | S->A S->B |       |       |
A   | A->aA'    |       |       |
A'  | A'->Ab    | A'->b |       |
B   | B->aB'    |       |       |
B'  | B'->Bc    |       | B'->c |
conflict: M[S, a] = S->A S->B
LL(1): no, 1 entry with two or more productions
```

The grammar is not ambiguous (a sentence ends in `b` or in `c`, never both, and each has one tree), it has no left recursion, and `leftfac` factored A and B as far as they go. But it could not touch S, because `A` and `B` are different nonterminals: their common prefix `a` is not written in any one production. At S, the parser must choose A or B while holding the first `a`, and what decides it is the first `b` or `c`, after any number of `a`s. No rewriting helps with one token of lookahead, or with any fixed number: for this language, no LL grammar exists at all.<!-- risk: the claim that this language has no LL(k) grammar for any k is a theorem stated without proof here --> The bottom-up parsers of chapter 7 handle it, because they choose between A and B only after the `a`s have been read and the `b` or `c` is in hand.

### Settling the dangling else by hand

The dangling-else conflict is the one conflict you are allowed to keep. C, Java and Pascal all say that an `else` belongs to the nearest `then` that has no `else` yet. In the table, that rule is one choice: in M[S', e], keep `S' -> e S` and delete `S' -> ε`. When the parser is inside the inner `if` and sees `e`, it takes the `else` at once, for the inner statement; it never lets the inner S' vanish so that an outer one can have it.

| | i | t | a | e | b | $ |
|---|---|---|---|---|---|---|
| **S** | S -> i E t S S' | | S -> a | | | |
| **S'** | | | | S' -> e S | | S' -> ε |
| **E** | | | | | E -> b | |

`rd.c` makes the same choice without being told. A recursive-descent function for S' would test `if (tok == 'e')` first and take ε only otherwise, so the inner `if` always takes the `else`. This works because the conflict was a choice between two meanings, both of them sentences of the language, and the rule picks one. Deleting an entry to remove any other conflict would make the parser reject correct inputs; that is why the cure for left recursion and common prefixes is to change the grammar, and only the dangling else is settled by editing the table.

### Answering "show that the grammar is not LL(1)"

One conflicting entry is a complete proof, and the answer is three lines: the entry, the two productions in it, and the sets that put them there. For the left-factored dangling else:

> M[S', e] holds both S' -> e S (rule 1, since e ∈ FIRST(e S)) and S' -> ε (rule 2, since ε ∈ FIRST(ε) and e ∈ FOLLOW(S') = { e, $ }). An entry with two productions means the grammar is not LL(1); this is a FIRST/FOLLOW conflict.

You do not need the whole table for this, only the sets for the one nonterminal that fails. When the question says "without constructing the table", give the three conditions and the one that fails, with the same sets.

### When it goes wrong

**"Not LL(1), because T' is nullable."** Nullable nonterminals are normal in LL(1) grammars; `exprll.g` has two. The question is condition 3: whether FIRST of the other alternative meets FOLLOW of the nonterminal.

**Condition 3 checked with the wrong FOLLOW.** The set is FOLLOW of the left side A, not FOLLOW of the symbols in α or β. The check fails with the wrong set in either direction: a real conflict is missed, or one is invented.

**Counting a conflict once per production.** `prefix.g` has one conflicting entry with three productions. "The grammar is not LL(1)" needs only one entry; if the question asks how many conflicts, count entries, as `ll1` does, and say how many productions are in each.

**Settling a conflict that is not the dangling else.** Deleting `E -> E + T` from M[E, id] makes the table look LL(1), and then the parser rejects `id + id`, because no remaining production can produce a `+` at that level. Change the grammar instead.

**`LL(1): yes` for the grammar in your file, not the one on your paper.** A missing blank (`T'->eps`, or `E'` typed as `E`) gives a different grammar, which may well be LL(1). Compare `ll1`'s rows with your paper's nonterminals before you trust its last line.

### In one line each

1. A grammar is LL(1) when no entry of its table holds two productions.
2. FIRST/FIRST: two alternatives can begin with the same token. FIRST/FOLLOW: one alternative can begin with a token that may follow A when another alternative vanishes.
3. Without the table: for each pair of alternatives, FIRSTs disjoint, at most one nullable, and the non-nullable one's FIRST disjoint from FOLLOW(A).
4. Left recursion, a common prefix or ambiguity each make a grammar non-LL(1); the reverse does not hold.
5. The dangling else is settled by keeping S' -> e S in M[S', e]; every other conflict is cured by changing the grammar.

**Check yourself**

<!-- q: idea=ll1-conflict type=decide wrong="a grammar with an epsilon-production cannot be LL(1)" -->
1. Decide, without the table, whether `S -> A a A b | B b B a`, `A -> eps`, `B -> eps` is LL(1). Then check with `ll1`.

<details>
<summary>Answer</summary>

Only S has alternatives. FIRST(`A a A b`): A vanishes, so the walk reaches `a`; the set is { a }. FIRST(`B b B a`) is { b }. Disjoint, and neither can vanish (each contains a terminal), so conditions 1 to 3 hold: LL(1). A and B have only ε, so they are fine too. FOLLOW(A) = { a, b } puts `A -> eps` in two entries; one production per entry.

<!-- code: id=ll1-aabb -->
```console
$ printf 'S -> A a A b | B b B a\nA -> eps\nB -> eps\n' | ./ll1
    | a       | b       | $
S   | S->AaAb | S->BbBa |
A   | A->eps  | A->eps  |
B   | B->eps  | B->eps  |
LL(1): yes, no entry has two productions
```

</details>

<!-- q: idea=ll1-conflict type=explain wrong="every conflict means the grammar is ambiguous" -->
2. Explain. `abc.g` is not LL(1), and neither is `amb.g`. Which of the two can a different parser still handle with one parse tree per sentence, and what does the difference tell you about what a conflict means?

<details>
<summary>Answer</summary>

`abc.g` is unambiguous: every sentence has exactly one parse tree, and a parser that delays its decision until it has seen the first `b` or `c` can find it. Its conflicts are about *when* the parser must choose, not about two meanings. `amb.g` is ambiguous: `id + id * id` has two trees, and no parser can choose between them without an outside rule such as precedence. So a conflict means only that one token is not enough to choose at that point; sometimes the grammar is ambiguous, sometimes the choice simply comes too early for top-down parsing.

</details>

<!-- q: idea=ll1-conflict type=apply wrong="removing left recursion is enough to make a grammar LL(1)" -->
3. Apply. Section 6.1's procedure-call grammar was `stmt -> id = expr ; | id ( ) ;`. Write it with single letters, `S -> i = e ; | i ( ) ;`, run it through `ll1`, name the conflict, then left factor it and check again.

<details>
<summary>Answer</summary>

Both alternatives begin with `i`: a FIRST/FIRST conflict in M[S, i]. After `leftfac`, S matches `i` and the new S' chooses by `=` or `(`.

<!-- code: id=ll1-call -->
```console
$ printf 'S -> i = e ; | i ( ) ;\n' | ./ll1
    | i               | = | e | ; | ( | ) | $
S   | S->i=e; S->i(); |   |   |   |   |   |
conflict: M[S, i] = S->i=e; S->i();
LL(1): no, 1 entry with two or more productions
$ printf 'S -> i = e ; | i ( ) ;\n' | ../ch5/leftfac | ./ll1
    | i      | =       | e | ; | (       | ) | $
S   | S->iS' |         |   |   |         |   |
S'  |        | S'->=e; |   |   | S'->(); |   |
LL(1): yes, no entry has two productions
```

</details>

## 6.4 A table-driven predictive parser in C, with its stack trace

### A pile of plates you can see

`ll1` prints a table, and nothing reads it yet. `rd.c` makes the same decisions, but in code, with its pile of plates hidden in the C call stack, where no program can print it. The written paper asks for exactly what `rd` cannot show: "using the table, show the moves of the predictive parser on the input `id + id * id`", meaning the stack, the remaining input and the action, one line per move. And the lab asks for the program that prints those lines: "implement a predictive (non-recursive) parser for the expression grammar".

Both are the same thing: a parser that keeps its own stack, an ordinary array, and consults the table at every move. It has one loop and no recursion, and it does not change when the grammar does; only the table changes. This section writes it twice: first `pp.c`, the lab program, with the expression grammar's table typed in, then `llparse.c`, which takes its table from `ll1` and parses with any LL(1) grammar, to check the traces you do by hand.

The pile of plates is back, and this time it is in plain view. Each plate carries one grammar symbol: something the parser still expects to find in the input, in order, the top plate first. A terminal plate is a promise that this exact token comes next. A nonterminal plate is a promise that something derived from it comes next, and the table says how to replace it with smaller promises.

The picture breaks in a way worth knowing. In `rd.c`, a plate was a running function, and it said where that function was in its production. Here a plate is a single symbol, and a production's right side is put on the pile all at once, as several plates; the parser does not remember which production a plate came from. That is why the trace can be printed, and why nothing is left over to remember: the pile says everything about what is still to come.

### The driver: one loop, four cases

<!-- teaches: predictive-parser = table-driven predictive parser, parsing stack, predictive parsing driver -->
A *table-driven predictive parser* has three parts: the input, with `$` added at its end; a *parsing stack* of grammar symbols, which starts as `$` with the start symbol on top of it; and the table M. A loop, the *driver*, looks at two things each time round: X, the symbol on top of the stack, and a, the current input token. Then exactly one of four cases applies.

1. **X is `$` and a is `$`**: both are used up together. Accept.
2. **X is a terminal (or `$`)**: if X equals a, pop X and move to the next token; this move is called a *match*. If X is not a, report an error: the stack promised a token that is not there.
3. **X is a nonterminal and M[X, a] is a production X -> Y1 Y2 ... Yk**: pop X and push the right side so that Y1 ends on top, which means pushing Yk first and Y1 last. Output the production. For X -> ε, pop X and push nothing.
4. **X is a nonterminal and M[X, a] is blank**: report an error.

Why push the right side backwards? Because Y1 is what the input must show first, and the top of the stack is what the parser looks at next. If you push Y1 first, it ends at the bottom of the new plates, and the parser would look for Yk before Y1.

One fact makes the whole trace checkable. At every move, the tokens matched so far, followed by the stack read from the top down, form a sentential form of the leftmost derivation. Section 6.1 showed that `rd`'s productions come out in leftmost-derivation order; the stack is what is left of that derivation's current line, still to be matched.

### pp.c: the expression grammar, table typed in

Lab sheets want single-character symbols, so `pp.c` uses the names of section 5.7's lab-sheet format: `R` for E', `Y` for T', `i` for `id`, and `#` for ε. The input is one line, as in `rd.c`, and every character but a blank is one token. Save it as `pp.c`:

<!-- code: id=pp mode=compile -->
```c
/* pp.c: a table-driven predictive parser for the expression grammar
       E -> T R        R -> + T R | #        (R is E')
       T -> F Y        Y -> * F Y | #        (Y is T')
       F -> ( E ) | i
   # is the empty string, i stands for id. Reads one line; every
   character but a blank is one token, and $ is added at the end.
   Prints the stack, the rest of the input and the action at every move. */
#include <stdio.h>
#include <string.h>

const char *nts = "ERTYF";           /* the rows of the table */
const char *ts = "i+*()$";           /* its columns */
const char *M[5][6] = {              /* right sides; NULL is a blank */
    /*         i      +      *      (      )      $   */
    /* E */ { "TR",  NULL,  NULL,  "TR",  NULL,  NULL },
    /* R */ { NULL,  "+TR", NULL,  NULL,  "#",   "#"  },
    /* T */ { "FY",  NULL,  NULL,  "FY",  NULL,  NULL },
    /* Y */ { NULL,  "#",   "*FY", NULL,  "#",   "#"  },
    /* F */ { "i",   NULL,  NULL,  "(E)", NULL,  NULL },
};

char stack[100] = "$E";              /* the bottom is stack[0]; top at the end */
int top = 1;                         /* index of the top symbol */
char input[100];                     /* the tokens, then $ */

int moves;                           /* lines printed so far */

void show(int ip, const char *action)
{
    printf("%3d  %-14s%16s   %s\n", ++moves, stack, input + ip, action);
}

int main(void)
{
    char line[100], act[40];
    const char *r, *row, *col;
    int ip = 0, n = 0, i, k;
    char X, a;

    if (fgets(line, sizeof line, stdin) == NULL) {
        fprintf(stderr, "pp: no input\n");
        return 1;
    }
    for (i = 0; line[i] != '\0' && line[i] != '\n'; i++)
        if (line[i] != ' ' && line[i] != '\t' && n < 98)
            input[n++] = line[i];
    input[n] = '$';
    printf("%3s  %-14s%16s   %s\n", "", "STACK", "INPUT", "ACTION");
    for (;;) {
        X = stack[top];
        a = input[ip];
        if (X == '$' && a == '$') {
            show(ip, "accept");
            return 0;
        }
        row = strchr(nts, X);
        if (row == NULL) {                       /* X is a terminal or $ */
            if (X != a) {
                sprintf(act, "error: expected %c, found %c", X, a);
                show(ip, act);
                printf("rejected\n");
                return 1;
            }
            sprintf(act, "match %c", a);
            show(ip, act);
            stack[top--] = '\0';                 /* pop X ... */
            ip++;                                /* ... and use up a */
            continue;
        }
        col = strchr(ts, a);
        r = (col == NULL) ? NULL : M[row - nts][col - ts];
        if (r == NULL) {                         /* a blank entry */
            sprintf(act, "error: M[%c,%c] is blank", X, a);
            show(ip, act);
            printf("rejected\n");
            return 1;
        }
        sprintf(act, "%c->%s", X, r);
        show(ip, act);
        stack[top--] = '\0';                     /* pop X */
        if (strcmp(r, "#") != 0)                 /* push the right side, */
            for (k = strlen(r) - 1; k >= 0; k--) /* last symbol first */
                stack[++top] = r[k];
        stack[top + 1] = '\0';
    }
}
```

The table is section 6.2's grid, typed in. `nts` and `ts` name the rows and the columns, and an entry is the right side of its production as a string, `NULL` for a blank, `"#"` for ε. The left side is not stored: it is the row. To look up M[X, a], `strchr` finds X in `nts` and a in `ts`, and the two pointer differences, `row - nts` and `col - ts`, are the row and column numbers. A character found in neither is not a symbol of the grammar; `strchr` returns `NULL` for it, which the code treats as a blank entry, so a stray `x` in the input is a syntax error, not a crash.

The stack is a C string. `stack[0]` is `$`, the top is `stack[top]`, and a `'\0'` always follows the top, so `printf("%s", stack)` prints the whole pile, bottom first. Popping writes `'\0'` over the top and moves `top` down. Pushing writes the right side from its last character to its first, then puts the `'\0'` back after the new top.

The loop is the four cases in order. Case 1 is the first `if`. If X is not in `nts`, it is a terminal or `$`: case 2, a match or an error. Otherwise X is a nonterminal: case 4 if the entry is blank, case 3 if not. Each move prints one line before it changes anything, so a line shows the stack and input *as they were* when the action was chosen. That is the layout of the textbook trace. `show` also numbers the moves, so that a trace can be discussed as "move 5"; the paper's trace has no numbers, and you can leave them out there.

### The running expression, move by move

Build it and give it the expression of the running statement, `init + rate * 60` with an `i` for every operand:

<!-- code: id=pp-run -->
```console
$ gcc -Wall -Wextra pp.c -o pp
$ echo 'i + i * i' | ./pp
     STACK                    INPUT   ACTION
  1  $E                      i+i*i$   E->TR
  2  $RT                     i+i*i$   T->FY
  3  $RYF                    i+i*i$   F->i
  4  $RYi                    i+i*i$   match i
  5  $RY                      +i*i$   Y->#
  6  $R                       +i*i$   R->+TR
  7  $RT+                     +i*i$   match +
  8  $RT                       i*i$   T->FY
  9  $RYF                      i*i$   F->i
 10  $RYi                      i*i$   match i
 11  $RY                        *i$   Y->*FY
 12  $RYF*                      *i$   match *
 13  $RYF                        i$   F->i
 14  $RYi                        i$   match i
 15  $RY                          $   Y->#
 16  $R                           $   R->#
 17  $                            $   accept
```

Seventeen lines, and every one is a case of the driver. Walk the first five. The stack starts as `$E`, with `E` on top, and the input is `i+i*i$`. M[E, i] is `TR`: `E` is popped and `R`, then `T`, pushed, so the stack reads `$RT` with `T` on top. M[T, i] is `FY`: `T` becomes `Y`, `F`. M[F, i] is `i`: now a terminal is on top. It equals the input token, so the fourth move matches it, pops it and moves the input on; the input column loses its `i`. With `+` in hand and `Y` on top, M[Y, +] is `#`: `Y` is popped and nothing is pushed. That is the ε-choice `rd` made at step 4 of its trace in section 6.1, and here you can see its effect: the `Y` just disappears from the pile.

Figure 6.4a shows the pile at those moments.

<!-- fig: id=pp-stack
title: pp's stack for i+i*i, bottom on the left
kind: memory
index: off
frame m1: before move 1: start, E on $
row s1: $ E | hot=1
frame m2: before move 2: E replaced by T R, pushed R first
row s2: $ R T | hot=2
frame m4: before move 4: the terminal i on top, i in hand
row s4: $ R Y i | hot=3
frame m5: before move 5: i matched and popped
row s5: $ R Y | hot=2
-->
![Memory. before move 1: start, E on $. It holds s1. s1 holds $ and E. s1\[1\] is highlighted. before move 2: E replaced by T R, pushed R first. It holds s2. s2 holds $, R and T. s2\[2\] is highlighted. before move 4: the terminal i on top, i in hand. It holds s4. s4 holds $, R, Y and i. s4\[3\] is highlighted. before move 5: i matched and popped. It holds s5. s5 holds $, R and Y. s5\[2\] is highlighted.](fig/06-top-down-parsers/pp-stack.svg)

*Figure 6.4a: `pp`'s stack before moves 1, 2, 4 and 5 on `i+i*i`, bottom (`$`) on the left; the top symbol, the one the driver looks at, is highlighted. R is E' and Y is T'.*

Now the fact from the driver section, on one line. Take move 12, `$RYF*  *i$  match *`. The tokens matched before it are `i + i`. The stack from the top down is `* F Y R`, which in the grammar's names is `* F T' E'`. Together: `i + i * F T' E'`, which is the line `=>lm id + id * F T' E'` of the leftmost derivation `derive` printed in section 6.1, with `id` for `i`. Every line of the trace is like that. And read only the actions that are productions, skipping the matches: `E->TR`, `T->FY`, `F->i`, `Y->#`, `R->+TR`, ... These are `rd`'s productions for the same input, in the same order. The two parsers are one algorithm, with the stack in two different places.

A note for the exam paper. This book's traces put the bottom of the stack on the left, `$E`, because that is how the array is stored. The second edition of the Dragon book writes the stack the other way round, top on the left, `E$`, and adds a column of the input matched so far; the first edition has the `$` on the left, as here.<!-- risk: the claim about how the two editions of the Dragon book lay out the stack column is from memory of the books, not checked --> Either is correct if you say which end is the top.

### When the input is wrong

The same five inputs that `rd` rejected:

<!-- code: id=pp-errors exit=1 -->
```console
$ for s in 'i+*i' 'i i' '(i+i' 'i)' ''; do echo "== $s"; echo "$s" | ./pp; done
== i+*i
     STACK                    INPUT   ACTION
  1  $E                       i+*i$   E->TR
  2  $RT                      i+*i$   T->FY
  3  $RYF                     i+*i$   F->i
  4  $RYi                     i+*i$   match i
  5  $RY                       +*i$   Y->#
  6  $R                        +*i$   R->+TR
  7  $RT+                      +*i$   match +
  8  $RT                        *i$   error: M[T,*] is blank
rejected
== i i
     STACK                    INPUT   ACTION
  1  $E                         ii$   E->TR
  2  $RT                        ii$   T->FY
  3  $RYF                       ii$   F->i
  4  $RYi                       ii$   match i
  5  $RY                         i$   error: M[Y,i] is blank
rejected
== (i+i
     STACK                    INPUT   ACTION
  1  $E                       (i+i$   E->TR
  2  $RT                      (i+i$   T->FY
  3  $RYF                     (i+i$   F->(E)
  4  $RY)E(                   (i+i$   match (
  5  $RY)E                     i+i$   E->TR
  6  $RY)RT                    i+i$   T->FY
  7  $RY)RYF                   i+i$   F->i
  8  $RY)RYi                   i+i$   match i
  9  $RY)RY                     +i$   Y->#
 10  $RY)R                      +i$   R->+TR
 11  $RY)RT+                    +i$   match +
 12  $RY)RT                      i$   T->FY
 13  $RY)RYF                     i$   F->i
 14  $RY)RYi                     i$   match i
 15  $RY)RY                       $   Y->#
 16  $RY)R                        $   R->#
 17  $RY)                         $   error: expected ), found $
rejected
== i)
     STACK                    INPUT   ACTION
  1  $E                         i)$   E->TR
  2  $RT                        i)$   T->FY
  3  $RYF                       i)$   F->i
  4  $RYi                       i)$   match i
  5  $RY                         )$   Y->#
  6  $R                          )$   R->#
  7  $                           )$   error: expected $, found )
rejected
==
     STACK                    INPUT   ACTION
  1  $E                           $   error: M[E,$] is blank
rejected
```

Every error is at the same token as `rd`'s, and now you can see why from the stack. In `i+*i`, `T` is on top with `*` in hand, and row T has nothing under `*`; `rd`'s `F` was one call deeper when it noticed, because `T` there had no decision of its own. In `(i+i`, the `)` that `rd`'s `F` was waiting for with `match(')')` is pushed by move 3 and waits on the stack until move 17, when it reaches the top and the input has run out. In `i)`, the stack is down to `$` while the input still holds `)`: case 2 with X = `$`, the same thing `rd`'s final check in `main` found. In this parser that check needs no extra code, because `$` on the stack is a terminal like any other.

And no line at all:

<!-- code: id=pp-empty exit=1 -->
```console
$ ./pp < /dev/null
pp: no input
```

### llparse.c: the same driver, any LL(1) grammar

`pp.c` is the program to write in the lab. For the paper, you want a program that takes *any* grammar you have worked by hand, builds its table and prints the moves, so that you can check your trace line by line. `ll1.c` already builds the table in `buildtable`; take `ll1.c` without its `main` as a header, once more with `sed`:

<!-- code: id=llparse-setup -->
```console
$ sed '/^int main/,$d' ll1.c > ll1.h
```

Save the driver as `llparse.c`:

<!-- code: id=llparse mode=compile -->
```c
/* llparse.c: the predictive parser for any LL(1) grammar.
   usage: ./llparse 'id + id * id' < grammar
   The grammar is read as by ll1 and its table built by ll1.h (ll1.c
   without its main); the sentence's tokens are separated by blanks. */
#include "ll1.h"

int stack[200], top;                 /* symbol numbers; stack[0] is $ */
int in[100], nin, ip;                /* the tokens, then $ */
int moves;                           /* lines printed so far */

void show(const char *action)
{
    char st[400] = "", rest[400] = "";
    int i;

    for (i = 0; i <= top; i++) {
        strcat(st, name[stack[i]]);
        strcat(st, " ");
    }
    for (i = ip; i < nin; i++) {
        strcat(rest, name[in[i]]);
        strcat(rest, " ");
    }
    printf("%3d  %-22s%22s  %s\n", ++moves, st, rest, action);
}

int main(int argc, char *argv[])
{
    char words[400], *w, act[200];
    int X, a, p, k, s;

    prog = "llparse";
    if (argc != 2)
        die("usage: llparse 'tokens separated by blanks' < grammar");
    readgrammar();
    buildtable();
    if (conflicts > 0)
        die("the grammar is not LL(1): run ll1 to see the conflicts");
    snprintf(words, sizeof words, "%s", argv[1]);
    for (w = strtok(words, " \t"); w != NULL; w = strtok(NULL, " \t")) {
        for (s = 2; s < nsym && strcmp(name[s], w) != 0; s++)
            ;
        if (s == nsym || isnt[s])
            die("%s is not a terminal of the grammar", w);
        if (nin == 99)
            die("too many tokens");
        in[nin++] = s;
    }
    in[nin++] = END;
    stack[0] = END;
    stack[top = 1] = ntorder[0];             /* $ and the start symbol */
    printf("%3s  %-22s%22s  %s\n", "", "STACK", "INPUT", "ACTION");
    for (;;) {
        X = stack[top];
        a = in[ip];
        if (X == END && a == END) {
            show("accept");
            return 0;
        }
        if (!isnt[X]) {                      /* a terminal, or $ */
            if (X != a) {
                sprintf(act, "error: expected %s", name[X]);
                show(act);
                return 1;
            }
            sprintf(act, "match %s", name[a]);
            show(act);
            top--;
            ip++;
        } else if (count[X][a] == 0) {       /* a blank entry */
            sprintf(act, "error: M[%s, %s] is blank", name[X], name[a]);
            show(act);
            return 1;
        } else {
            p = entry[X][a][0];
            sprintf(act, "%s", prodtext(p));
            show(act);
            top--;                           /* pop X, push the right */
            for (k = len[p] - 1; k >= 0; k--)  /* side, last symbol first */
                stack[++top] = rhs[p][k];
        }
    }
}
```

The loop is `pp.c`'s, case for case, with symbol numbers in place of characters. The table is `ll1`'s `entry` and `count`; a nonterminal's row is its symbol number, and an entry's production gives both the right side to push and, through `prodtext` from `grammar.h`, the text to print. The input is the program's argument, cut into words by `strtok`, each word looked up among the grammar's terminals; a word that is not one is refused before parsing starts. The stack and the input are printed with blanks between the symbols, because a symbol can now be `id` or `E'`. And it refuses a grammar with conflicts: an entry with two productions is a guess, and this parser does not guess.

Build it and parse the paper's usual sentence with the paper's grammar:

<!-- code: id=llparse-run -->
```console
$ gcc -Wall -Wextra llparse.c -o llparse
$ ./llparse 'id + id * id' < ../ch5/exprll.g
     STACK                                  INPUT  ACTION
  1  $ E                          id + id * id $   E -> T E'
  2  $ E' T                       id + id * id $   T -> F T'
  3  $ E' T' F                    id + id * id $   F -> id
  4  $ E' T' id                   id + id * id $   match id
  5  $ E' T'                         + id * id $   T' -> eps
  6  $ E'                            + id * id $   E' -> + T E'
  7  $ E' T +                        + id * id $   match +
  8  $ E' T                            id * id $   T -> F T'
  9  $ E' T' F                         id * id $   F -> id
 10  $ E' T' id                        id * id $   match id
 11  $ E' T'                              * id $   T' -> * F T'
 12  $ E' T' F *                          * id $   match *
 13  $ E' T' F                              id $   F -> id
 14  $ E' T' id                             id $   match id
 15  $ E' T'                                   $   T' -> eps
 16  $ E'                                      $   E' -> eps
 17  $                                         $   accept
```

That is `pp`'s trace in the grammar's own names, which is the form the paper's answer takes. Then the nullable grammar of section 6.2, whose table said "vanish" at A when `b` is in hand, and the inputs that must be refused:

<!-- code: id=llparse-more exit=1 -->
```console
$ ./llparse 'b c' < ../ch5/null.g
     STACK                                  INPUT  ACTION
  1  $ S                                   b c $   S -> A B c
  2  $ c B A                               b c $   A -> eps
  3  $ c B                                 b c $   B -> b
  4  $ c b                                 b c $   match b
  5  $ c                                     c $   match c
  6  $                                         $   accept
$ ./llparse 'id' < ../ch5/expr.g; echo "exit status $?"
llparse: the grammar is not LL(1): run ll1 to see the conflicts
exit status 1
$ ./llparse 'id + x' < ../ch5/exprll.g; echo "exit status $?"
llparse: x is not a terminal of the grammar
exit status 1
$ ./llparse 'id + * id' < ../ch5/exprll.g
     STACK                                  INPUT  ACTION
  1  $ E                             id + * id $   E -> T E'
  2  $ E' T                          id + * id $   T -> F T'
  3  $ E' T' F                       id + * id $   F -> id
  4  $ E' T' id                      id + * id $   match id
  5  $ E' T'                            + * id $   T' -> eps
  6  $ E'                               + * id $   E' -> + T E'
  7  $ E' T +                           + * id $   match +
  8  $ E' T                               * id $   error: M[T, *] is blank
```

### Recursive descent or a table?

You now have the same parser twice, and the exam asks you to compare them. They make the same decisions in the same order, because both read the same FIRST and FOLLOW sets; they differ in where the decisions and the stack live.

| | Recursive descent (`rd.c`, `tiny-rd.c`) | Table-driven (`pp.c`, `llparse.c`) |
|---|---|---|
| The decisions | written as code, one function per nonterminal | entries in a table; one driver for every grammar |
| The stack | the C call stack, invisible | an array you can print |
| A new grammar | rewrite the functions | rebuild the table; the driver stays |
| Actions while parsing | easy: any C code in any function | needs extra machinery |
| Error messages | each function can say what it wanted | the blank entry's row says what was allowed |

That is why the mini compiler's parser, `tiny-rd.c`, is recursive descent: chapter 9 will make it do things as it parses, and code in functions is the easy place for that. The table-driven parser is the one to reach for when the grammar is generated or changes often, and it is the model for the bottom-up parsers of chapter 7, which are always table-driven.

### When it goes wrong

**The right side pushed in the wrong order.** A loop that pushes from the first symbol to the last leaves the *last* symbol on top. Here is that mistake made on purpose, in a copy of `pp.c`, and its first moves:

<!-- code: id=pp-forward exit=1 -->
```console
$ sed 's/k = strlen(r) - 1; k >= 0; k--/k = 0; k < (int) strlen(r); k++/' pp.c > ppfwd.c
$ gcc -Wall -Wextra ppfwd.c -o ppfwd
$ echo 'i+i*i' | ./ppfwd
     STACK                    INPUT   ACTION
  1  $E                      i+i*i$   E->TR
  2  $TR                     i+i*i$   error: M[R,i] is blank
rejected
```

The symptom is an error on the second move of every correct input, with the right side of the first production printed in its written order on the stack, `$TR`, where it should be reversed, `$RT`.

**`#` pushed as a symbol.** Without the test `strcmp(r, "#") != 0`, an ε-entry pushes the character `#`. It is not in `nts`, so the driver takes it for a terminal and wants to match it:

<!-- code: id=pp-hash exit=1 -->
```console
$ sed 's/if (strcmp(r, "#") != 0)/if (1)/' pp.c > pphash.c
$ gcc -Wall -Wextra pphash.c -o pphash
$ echo 'i' | ./pphash
     STACK                    INPUT   ACTION
  1  $E                          i$   E->TR
  2  $RT                         i$   T->FY
  3  $RYF                        i$   F->i
  4  $RYi                        i$   match i
  5  $RY                          $   Y->#
  6  $R#                          $   error: expected #, found $
rejected
```

The parser stops at the first ε-move, on input that is correct. An ε-production pushes nothing.

**The trace shows the stack after the move, not before.** If `show` is called after the push, every line pairs an action with the stack it produced, and the trace no longer matches the textbook's layout or the hand method. Print first, then change.

**Your hand trace differs from `llparse`'s.** Find the first line that differs. If the stack is different, you pushed in the wrong order or used the wrong entry; if the input is different, a match was skipped or done twice. Then look up that entry in `ll1`'s table.

### In one line each

1. The stack starts as `$` and the start symbol; the input ends in `$`.
2. Terminal on top: match it with the input or report an error. Nonterminal on top: replace it with M[X, a]'s right side, pushed last symbol first, or report a blank entry.
3. `$` on top and `$` in hand: accept.
4. Matched input plus the stack, top down, is always a line of the leftmost derivation.
5. `pp.c` is the lab program, table typed in; `llparse` builds the table from a grammar file to check any hand trace.

**Check yourself**

<!-- q: idea=predictive-parser type=predict wrong="the stack holds the input already read" -->
1. Predict the stack column of `pp`'s trace for `(i)` line by line, then run it.

<details>
<summary>Answer</summary>

`(` brings in `F->(E)`, which pushes `)`, `E`, `(`; the `(` is matched; inside, `E`, `T`, `F` and the `i` go as before, then `Y` and `R` vanish at `)`, which is then matched against the `)` that has waited on the stack since the third move; the outer `Y` and `R` vanish at `$`.

<!-- code: id=pp-paren -->
```console
$ echo '(i)' | ./pp
     STACK                    INPUT   ACTION
  1  $E                        (i)$   E->TR
  2  $RT                       (i)$   T->FY
  3  $RYF                      (i)$   F->(E)
  4  $RY)E(                    (i)$   match (
  5  $RY)E                      i)$   E->TR
  6  $RY)RT                     i)$   T->FY
  7  $RY)RYF                    i)$   F->i
  8  $RY)RYi                    i)$   match i
  9  $RY)RY                      )$   Y->#
 10  $RY)R                       )$   R->#
 11  $RY)                        )$   match )
 12  $RY                          $   Y->#
 13  $R                           $   R->#
 14  $                            $   accept
```

</details>

<!-- q: idea=predictive-parser type=explain wrong="the predictive parser needs a final check for leftover input" -->
2. Explain. `rd.c` needs the test `if (tok != '$')` after `E()` returns, or it accepts `i)`. `pp.c` has no such test, yet rejects `i)`. Which line of `pp.c` does that test's job?

<details>
<summary>Answer</summary>

The `$` at the bottom of the stack. When the expression is finished, the stack is down to `$` and the input still holds `)`. `$` is not in `nts`, so case 2 compares it with the token, finds `)`, and prints `error: expected $, found )`. Only when both are `$` does the first `if` accept. The bottom `$` is the end-of-input check, built into the stack.

</details>

<!-- q: idea=predictive-parser type=apply wrong="the driver must change when the grammar changes" -->
3. Apply. Change `pp.c` into a parser for `S -> a S b | c` (one nonterminal, three terminals). Which lines change, and which stay the same?

<details>
<summary>Answer</summary>

Only the table and the stack's starting contents change: `nts = "S"`, `ts = "abc$"`, `M[1][4] = { { "aSb", NULL, "c", NULL } }`, and `stack[100] = "$S"`. The driver loop, `show` and the input reading stay exactly as they are, because they never mention a particular grammar. That is the point of a table-driven parser. FIRST(`a S b`) = { a } and FIRST(`c`) = { c } give the two entries; S has no ε-production, so FOLLOW(S) adds nothing.

</details>

## 6.5 Error recovery: panic mode and synch entries

`pp` stops at the first error. A real compiler that did that would make you fix one mistake per build. The exam asks how a predictive parser can report an error and carry on, and the usual answer is *panic mode*: throw away input, or give up on part of the stack, until parsing can resume.

<!-- teaches: panic-mode = panic-mode recovery, synchronizing token, synch entry -->
The method needs one change to the table. In every blank entry M[A, b] where b is in FOLLOW(A), write `synch`. These tokens are *synchronizing tokens*: if one of them is in hand, the A on the stack can be abandoned, because the token is something that may come after an A. Then the driver gets three error moves in place of "stop":

1. **M[A, a] is blank**: report an error and skip a, the input token.
2. **M[A, a] is `synch`**: report an error and pop A, as if A had been found.
3. **A terminal on top does not match a**: report an error and pop the terminal, as if it had been there.

For the expression grammar, FOLLOW(E) = { ), $ }, FOLLOW(T) = { +, ), $ } and FOLLOW(F) = { +, *, ), $ } add synch entries to rows E, T and F; rows E' and T' have every FOLLOW column filled already, by their ε-productions.

| | + | * | ( | ) | id | $ |
|---|---|---|---|---|---|---|
| **E** | | | E -> T E' | synch | E -> T E' | synch |
| **E'** | E' -> + T E' | | | E' -> ε | | E' -> ε |
| **T** | synch | | T -> F T' | synch | T -> F T' | synch |
| **T'** | T' -> ε | T' -> * F T' | | T' -> ε | | T' -> ε |
| **F** | synch | synch | F -> ( E ) | synch | F -> id | synch |

Trace the input `* i * + i`, with two mistakes: a `*` before anything, and a `*` with nothing after it. The stack is written as `pp` writes it, `R` for E' and `Y` for T':

| Stack | Input | Action |
|---|---|---|
| `$E` | `*i*+i$` | M[E, *] is blank: error, skip `*` |
| `$E` | `i*+i$` | E -> T R |
| `$RT` | `i*+i$` | T -> F Y |
| `$RYF` | `i*+i$` | F -> i |
| `$RYi` | `i*+i$` | match i |
| `$RY` | `*+i$` | Y -> * F Y |
| `$RYF*` | `*+i$` | match * |
| `$RYF` | `+i$` | M[F, +] is synch: error, pop F |
| `$RY` | `+i$` | Y -> ε |
| `$R` | `+i$` | R -> + T R |
| ... | ... | the rest as for `i+i` |

Two errors reported, at the first `*` and at the `+`, and parsing reaches `$` with an empty stack. The first was blank: nothing on the stack could use a `*`, so the token went. The second was synch: an F was due, but `+` can follow an F, so the parser gave up on the F and carried on with what comes after it, the way a reader skips a word that makes no sense and goes on with the sentence. Task 5 of section 6.6 asks you to add these moves to `pp.c` and check this trace with it.

The idea is the same for a recursive-descent parser: on an error, a function skips tokens until one in its FOLLOW set appears, and returns. Choosing the synchronizing set is a matter of judgement, and FOLLOW is only the usual first choice; compilers often add the keywords that begin statements, so that one bad statement does not swallow the next.

### The usual mistake: synch on the start symbol

With only the three moves, an input that begins with `)` meets M[E, )] = synch before anything else. The parser pops E, and the stack is `$`. Every remaining token is now "input left over" and is skipped with an error of its own, so one stray `)` produces an error for every token in the input. Task 5 of section 6.6 shows it happen. The cure is to treat synch as blank when the nonterminal is the only thing on the stack above `$`: skip the token instead, and keep E.<!-- risk: the effect of the cure was reasoned, not run -->

**Check yourself**

<!-- q: idea=panic-mode type=predict wrong="synch skips the input token" -->
1. With the synch table, the parser has `$RYF` on the stack and `)` in hand (as in the input `i * )`). Which move does it make, and what does it do next?

<details>
<summary>Answer</summary>

M[F, )] is synch, because `)` is in FOLLOW(F): report an error and pop F, keeping the `)`. Then `Y` is on top with `)` in hand, and M[Y, )] is Y -> ε, so parsing goes on normally: `R` also vanishes at `)`. Then `$` is on top with `)` still in hand, which no entry can repair: an error, and the `)` is skipped. The input has two errors, and both are reported.

</details>

<!-- q: idea=panic-mode type=explain wrong="synch entries are needed in rows that already have epsilon entries" -->
2. Why do rows E' and T' get no synch entries at all?

<details>
<summary>Answer</summary>

Synch goes only in *blank* entries under FOLLOW(A). E' and T' have ε-productions, and rule 2 of section 6.2 already put those under every token of their FOLLOW sets. On such a token, they simply vanish, which is what synch would have done, without an error.

</details>

## 6.6 Practice: grammar to table to parser for a new language

You can now write a recursive-descent parser by hand (6.1), build an LL(1) table from FIRST and FOLLOW (6.2), say whether a grammar is LL(1) and prove it is not with one entry (6.3), run the table-driven driver by hand and in C (6.4), and add panic-mode recovery (6.5). The written paper puts all of it in one question: "for the grammar below, remove left recursion, compute FIRST and FOLLOW, construct the predictive parsing table and show the moves of the parser on the given input". The lab asks for the parser. These tasks climb from reading a trace, through a broken table and a grammar that cannot be fixed, to a new grammar worked alone, and end with a parser for a language this chapter has not used, written from a blank file. Work in `~/cdlab/ch6`. Do every hand answer on paper first, then check it with the tools: `../ch5/follow` for the sets, `./ll1` for the table, `./llparse` for the moves.

### Guided

<!-- q: idea=predictive-parser type=predict wrong="the stack is tallest at the deepest parenthesis only by one symbol" -->
**1. A trace the paper asks for.** Write `pp`'s full trace for `i*(i+i)` by hand: stack, input and action on every line. Before you run it, say how many lines it has (count the moves, the accept included) and what the tallest stack is. Then run `pp` and compare line by line.

<details>
<summary>Answer</summary>

Count first, without tracing: every token is matched once (7 matches), every production used is one move, and the accept is one more. The productions are the leftmost derivation's: E, T, F, Y and R for the outer expression and the term after `*` (`E->TR`, `T->FY`, `F->i`, `Y->*FY`, `F->(E)`, and at the end `Y->#`, `R->#`), and inside the parentheses another `E->TR`, `T->FY`, `F->i`, `Y->#`, `R->+TR`, `T->FY`, `F->i`, `Y->#`, `R->#`: 16 in all. So 7 + 16 + 1 = 24 moves. The stack is tallest inside the parentheses, where the outer `RY` and the waiting `)` sit under a whole new expression: `$RY)RYF`, seven symbols.

<!-- code: id=practice-pp -->
```console
$ echo 'i*(i+i)' | ./pp
     STACK                    INPUT   ACTION
  1  $E                    i*(i+i)$   E->TR
  2  $RT                   i*(i+i)$   T->FY
  3  $RYF                  i*(i+i)$   F->i
  4  $RYi                  i*(i+i)$   match i
  5  $RY                    *(i+i)$   Y->*FY
  6  $RYF*                  *(i+i)$   match *
  7  $RYF                    (i+i)$   F->(E)
  8  $RY)E(                  (i+i)$   match (
  9  $RY)E                    i+i)$   E->TR
 10  $RY)RT                   i+i)$   T->FY
 11  $RY)RYF                  i+i)$   F->i
 12  $RY)RYi                  i+i)$   match i
 13  $RY)RY                    +i)$   Y->#
 14  $RY)R                     +i)$   R->+TR
 15  $RY)RT+                   +i)$   match +
 16  $RY)RT                     i)$   T->FY
 17  $RY)RYF                    i)$   F->i
 18  $RY)RYi                    i)$   match i
 19  $RY)RY                      )$   Y->#
 20  $RY)R                       )$   R->#
 21  $RY)                        )$   match )
 22  $RY                          $   Y->#
 23  $R                           $   R->#
 24  $                            $   accept
```

The usual slip is at the third line inside the parentheses: forgetting that `)` is still on the stack under the inner expression, and so not knowing what `Y` and `R` vanish *for*: they vanish because `)` is in their FOLLOW sets.

</details>

<!-- q: idea=ll1-table type=fix wrong="a missing table entry makes every input fail" -->
**2. A table with a hole.** A student's `pp.c` has row Y typed as `{ NULL, NULL, "*FY", NULL, "#", "#" }`. Which of `i*i`, `(i)` and `i+i` are still accepted? For the one that fails, at which move, and which rule of section 6.2 did the student forget? Predict, then make the faulty copy with `sed` and run it.

<details>
<summary>Answer</summary>

The hole is M[Y, +]: `T' -> ε` belongs under every token of FOLLOW(T') = { +, ), $ }, and the student left out `+`, so it is rule 2 applied to only part of FOLLOW. Inputs where no `+` ever meets a `Y` on top are untouched: `i*i` and `(i)` are accepted. `i+i` fails on its fifth move, the first time `Y` is on top with `+` in hand.

<!-- code: id=practice-hole exit=1 -->
```console
$ sed 's|/\* Y \*/ { NULL,  "#",|/* Y */ { NULL,  NULL,|' pp.c > ppbad.c
$ gcc -Wall -Wextra ppbad.c -o ppbad
$ for s in 'i*i' '(i)' 'i+i'; do echo "$s" | ./ppbad | tail -1; done
 11  $                            $   accept
 14  $                            $   accept
rejected
$ echo 'i+i' | ./ppbad
     STACK                    INPUT   ACTION
  1  $E                        i+i$   E->TR
  2  $RT                       i+i$   T->FY
  3  $RYF                      i+i$   F->i
  4  $RYi                      i+i$   match i
  5  $RY                        +i$   error: M[Y,+] is blank
rejected
```

A hole in a table shows up only on inputs that reach it, which is why habit 3 asks for more than one test input, and why habit 7 asks you to compare a typed table with `ll1`'s.

</details>

<!-- q: idea=ll1-conflict type=decide wrong="every LL(1) conflict can be settled by choosing the non-epsilon production" -->
**3. A conflict that cannot be settled.** Section 5.7's `abab.g` is `S -> a S b S | b S a S | eps`. Is it LL(1)? If not, name the conflicts and their kind. Then decide: could you settle them as the dangling else was settled, by keeping the non-ε production in each conflicting entry?

<details>
<summary>Answer</summary>

FOLLOW(S) = { b, a, $ }: `$` as the start symbol, `b` from `a S b S`, `a` from `b S a S`. `S -> eps` therefore goes under `a`, `b` and `$`, where rule 1 has already put `S -> a S b S` under `a` and `S -> b S a S` under `b`: two FIRST/FOLLOW conflicts.

<!-- code: id=practice-abab -->
```console
$ ./ll1 < ../ch5/abab.g
    | a              | b              | $
S   | S->aSbS S->eps | S->bSaS S->eps | S->eps
conflict: M[S, a] = S->aSbS S->eps
conflict: M[S, b] = S->bSaS S->eps
LL(1): no, 2 entries with two or more productions
```

Keeping the non-ε productions does not work. Trace `a b`, which is in the language (`S -> a S b S` with both inner S empty): S on top, `a` in hand, take `S -> a S b S`; match `a`. Now the inner S is on top with `b` in hand, and the edited entry says `S -> b S a S`, so the parser starts a new `b ... a` pair it will never finish, and rejects at `$` with `a` still expected. The inner S had to vanish here, but at the outer level, on another input, the same entry must start a pair. The dangling else could be settled because either choice gave a sentence and the rule picked one meaning; here one choice loses correct sentences. The grammar is ambiguous, and the cure is a different grammar, not an edited table.

</details>

### On your own

<!-- q: idea=ll1-table type=apply unseen="declaration-grammar" wrong="the semicolon is in FOLLOW of T" -->
**4. A declaration grammar, the whole exam question.** C declarations such as `int a, b;` follow this grammar:

```text
D -> T L ;
T -> int | float
L -> L , id | id
```

Remove the left recursion, compute FIRST and FOLLOW of every nonterminal, build the LL(1) table, say whether the grammar is LL(1), and write the moves of the predictive parser on `int id , id ;`. Then check each of the four steps with a tool.

<details>
<summary>Answer</summary>

Left recursion is in L only: `L -> id L'`, `L' -> , id L' | ε`. FIRST(D) = FIRST(T) = { int, float }, FIRST(L) = { id }, FIRST(L') = { ,, ε }. FOLLOW(D) = { $ }; FOLLOW(T) = FIRST(L) = { id }; FOLLOW(L) = { ; }, the token after L in `D -> T L ;`; FOLLOW(L') = FOLLOW(L) = { ; }, since L' ends L's production. The `;` belongs to L, not T, which is the usual slip: T is followed by L, and L starts with `id`.

Table: `D -> T L ;` under `int` and `float`; `T -> int` under `int`, `T -> float` under `float`; `L -> id L'` under `id`; `L' -> , id L'` under `,`; `L' -> ε` under `;`. One production per entry: LL(1).

<!-- code: id=practice-decl -->
```console
$ cat > decl.g <<'END'
D -> T L ;
T -> int | float
L -> L , id | id
END
$ ../ch5/leftrec < decl.g > declll.g
$ ../ch5/follow < declll.g
FIRST(D) = { int, float }
FIRST(T) = { int, float }
FIRST(L) = { id }
FIRST(L') = { ,, eps }
FOLLOW(D) = { $ }
FOLLOW(T) = { id }
FOLLOW(L) = { ; }
FOLLOW(L') = { ; }
(FIRST: 3 passes, FOLLOW: 2 passes)
$ ./ll1 < declll.g
    | ;       | int    | float    | id      | ,         | $
D   |         | D->TL; | D->TL;   |         |           |
T   |         | T->int | T->float |         |           |
L   |         |        |          | L->idL' |           |
L'  | L'->eps |        |          |         | L'->,idL' |
LL(1): yes, no entry has two productions
$ ./llparse 'int id , id ;' < declll.g
     STACK                                  INPUT  ACTION
  1  $ D                         int id , id ; $   D -> T L ;
  2  $ ; L T                     int id , id ; $   T -> int
  3  $ ; L int                   int id , id ; $   match int
  4  $ ; L                           id , id ; $   L -> id L'
  5  $ ; L' id                       id , id ; $   match id
  6  $ ; L'                             , id ; $   L' -> , id L'
  7  $ ; L' id ,                        , id ; $   match ,
  8  $ ; L' id                            id ; $   match id
  9  $ ; L'                                  ; $   L' -> eps
 10  $ ;                                     ; $   match ;
 11  $                                         $   accept
```

`FIRST(L') = { ,, eps }` is `printset` writing the comma token followed by its own separator; the set is { `,`, ε }.

</details>

<!-- q: idea=panic-mode type=build wrong="after an error the parser must stop" -->
**5. Panic mode in C.** Copy `pp.c` to `pprec.c` and give it section 6.5's recovery: synch entries in the table, and the three error moves in place of stopping. Count the errors; at the end print `accept` if there were none, otherwise `end` and the number of errors, and exit with status 1. Check it on `*i*+i`, whose trace section 6.5 worked by hand, on `i*)` from that section's first question, and on `)i*+i`.

<details>
<summary>Answer</summary>

The table gets an `"s"` for synch in the blank FOLLOW entries of rows E, T and F; the loop no longer returns on an error. Two cases of section 6.5's list need care in C: when `$` is on top and the input is not used up, there is no terminal to pop, so the token is skipped; and the loop ends only when both are `$`.

<!-- code: id=pprec mode=compile -->
```c
/* pprec.c: pp.c with panic-mode error recovery and synch entries.
   Grammar, table and input as in pp.c; "s" in the table is synch. */
#include <stdio.h>
#include <string.h>

const char *nts = "ERTYF";
const char *ts = "i+*()$";
const char *M[5][6] = {              /* "s" is synch, NULL is blank */
    /*         i      +      *      (      )      $   */
    /* E */ { "TR",  NULL,  NULL,  "TR",  "s",   "s"  },
    /* R */ { NULL,  "+TR", NULL,  NULL,  "#",   "#"  },
    /* T */ { "FY",  "s",   NULL,  "FY",  "s",   "s"  },
    /* Y */ { NULL,  "#",   "*FY", NULL,  "#",   "#"  },
    /* F */ { "i",   "s",   "s",   "(E)", "s",   "s"  },
};

char stack[100] = "$E";
int top = 1;
char input[100];

int moves;                           /* lines printed so far */

void show(int ip, const char *action)
{
    printf("%3d  %-14s%16s   %s\n", ++moves, stack, input + ip, action);
}

int main(void)
{
    char line[100], act[40];
    const char *r, *row, *col;
    int ip = 0, n = 0, i, k, errors = 0;
    char X, a;

    if (fgets(line, sizeof line, stdin) == NULL) {
        fprintf(stderr, "pprec: no input\n");
        return 1;
    }
    for (i = 0; line[i] != '\0' && line[i] != '\n'; i++)
        if (line[i] != ' ' && line[i] != '\t' && n < 98)
            input[n++] = line[i];
    input[n] = '$';
    printf("%3s  %-14s%16s   %s\n", "", "STACK", "INPUT", "ACTION");
    for (;;) {
        X = stack[top];
        a = input[ip];
        if (X == '$' && a == '$')
            break;
        row = strchr(nts, X);
        if (row == NULL) {                       /* X is a terminal or $ */
            if (X == a) {
                sprintf(act, "match %c", a);
                show(ip, act);
                stack[top--] = '\0';
                ip++;
            } else if (X == '$') {               /* input left over */
                sprintf(act, "error: skip %c", a);
                show(ip, act);
                errors++;
                ip++;
            } else {                             /* pretend X was there */
                sprintf(act, "error: pop %c, missing", X);
                show(ip, act);
                errors++;
                stack[top--] = '\0';
            }
            continue;
        }
        col = strchr(ts, a);
        r = (col == NULL) ? NULL : M[row - nts][col - ts];
        if (r == NULL) {                         /* blank: skip the token */
            sprintf(act, "error: skip %c", a);
            show(ip, act);
            errors++;
            ip++;
        } else if (strcmp(r, "s") == 0) {        /* synch: give up on X */
            sprintf(act, "error: synch, pop %c", X);
            show(ip, act);
            errors++;
            stack[top--] = '\0';
        } else {
            sprintf(act, "%c->%s", X, r);
            show(ip, act);
            stack[top--] = '\0';
            if (strcmp(r, "#") != 0)
                for (k = strlen(r) - 1; k >= 0; k--)
                    stack[++top] = r[k];
            stack[top + 1] = '\0';
        }
    }
    if (errors == 0) {
        show(ip, "accept");
        return 0;
    }
    show(ip, "end");
    printf("%d error(s)\n", errors);
    return 1;
}
```

<!-- code: id=pprec-run exit=1 -->
```console
$ gcc -Wall -Wextra pprec.c -o pprec
$ echo '*i*+i' | ./pprec; echo "exit status $?"
     STACK                    INPUT   ACTION
  1  $E                      *i*+i$   error: skip *
  2  $E                       i*+i$   E->TR
  3  $RT                      i*+i$   T->FY
  4  $RYF                     i*+i$   F->i
  5  $RYi                     i*+i$   match i
  6  $RY                       *+i$   Y->*FY
  7  $RYF*                     *+i$   match *
  8  $RYF                       +i$   error: synch, pop F
  9  $RY                        +i$   Y->#
 10  $R                         +i$   R->+TR
 11  $RT+                       +i$   match +
 12  $RT                         i$   T->FY
 13  $RYF                        i$   F->i
 14  $RYi                        i$   match i
 15  $RY                          $   Y->#
 16  $R                           $   R->#
 17  $                            $   end
2 error(s)
exit status 1
$ echo 'i*)' | ./pprec | tail -6
  7  $RYF                        )$   error: synch, pop F
  8  $RY                         )$   Y->#
  9  $R                          )$   R->#
 10  $                           )$   error: skip )
 11  $                            $   end
2 error(s)
$ echo ')i*+i' | ./pprec
     STACK                    INPUT   ACTION
  1  $E                      )i*+i$   error: synch, pop E
  2  $                       )i*+i$   error: skip )
  3  $                        i*+i$   error: skip i
  4  $                         *+i$   error: skip *
  5  $                          +i$   error: skip +
  6  $                           i$   error: skip i
  7  $                            $   end
6 error(s)
```

The first run is section 6.5's hand trace, line for line. The second is that section's first question: synch pops F, Y and R vanish normally at `)`, and the stray `)` is finally skipped at the bottom. The third is the mistake at the end of section 6.5: M[E, )] is synch, so E is popped before anything was parsed, and the five good tokens that follow are each skipped as leftover input. One mistake, six errors. If you add the cure, the first line becomes `error: skip )` and E stays to parse `i*+i`.<!-- risk: the effect of the cure was reasoned, not run -->

</details>

### A lab program from a blank file

<!-- q: idea=predictive-parser type=build unseen="list-language" wrong="the left-recursive list grammar can be put in the table as it is" -->
**6. A parser for nested lists.** A list is a single atom `a`, or a parenthesised list of lists separated by commas: `a`, `(a)`, `(a,a)` and `(a,(a,a))` are all lists. The grammar is

```text
S -> ( L ) | a
L -> L , S | S
```

Remove the left recursion by hand, calling the new nonterminal `M` (for L'), compute FIRST and FOLLOW, build the LL(1) table, and check it with `ll1`. Then write `lp.c` from a blank file: a table-driven predictive parser for the transformed grammar in the style of `pp.c`. It reads one line, every character but a blank is one token, and `$` is added at the end. It prints the header line and one numbered line per move with exactly `pp.c`'s `printf` formats: the move number, the stack bottom first with `$` at the bottom and the start symbol `S` on top at the start, the rest of the input, and the action, which is a production written like `S->(L)` (with `#` for ε), `match c` for a token c, `accept`, `error: M[X,c] is blank`, or `error: expected X, found c`. After an error it prints `rejected` and exits with status 1. With no input line at all it prints `lp: no input` to standard error and exits with status 1.

**Done when** `./lp` on the input `(a,(a,a))` prints exactly the model's output below. You can test that without anyone marking it: in the understudy folder (the one that holds `book/`), in the terminal, run `python3 scripts/us_practice.py check compiler-design 6.6 --in ~/cdlab/ch6`. It builds your `lp.c` and compares what it prints for this input with the model. Then test it, as habit 3 asks, on an empty line and on `(a,)`, which must be rejected, and on `a`, which must be accepted.

<details>
<summary>Answer</summary>

The hand work first. `L -> L , S | S` is immediate left recursion with α = `, S` and β = `S`, so `L -> S M`, `M -> , S M | ε`. FIRST(S) = { (, a } = FIRST(L); FIRST(M) = { ,, ε }. FOLLOW(S) = { $ } (start), plus `,` (from `, S M`, where S is followed by M, which can begin with `,`), plus FOLLOW(M) and FOLLOW(L) (S ends `L -> S M` when M vanishes, and ends `M -> , S M` the same way): { ), ,, $ }. FOLLOW(L) = { ) }; FOLLOW(M) = FOLLOW(L) = { ) }.

| | a | ( | ) | , | $ |
|---|---|---|---|---|---|
| **S** | S -> a | S -> ( L ) | | | |
| **L** | L -> S M | L -> S M | | | |
| **M** | | | M -> ε | M -> , S M | |

<!-- code: id=practice-lists -->
```console
$ cat > lists.g <<'END'
S -> ( L ) | a
L -> S M
M -> , S M | eps
END
$ ./ll1 < lists.g
    | (      | )      | a     | ,      | $
S   | S->(L) |        | S->a  |        |
L   | L->SM  |        | L->SM |        |
M   |        | M->eps |       | M->,SM |
LL(1): yes, no entry has two productions
```

`../ch5/leftrec` on the original grammar gives a different but equally correct answer, `L -> ( L ) L' | a L'`, because its method for indirect recursion first substitutes S's productions into L. Both are LL(1); use the one you derived by hand, and say which in the paper.

The program is `pp.c` with a new table, new names in `nts` and `ts`, and `$S` as the starting stack; the driver does not change, which is the point of section 6.4.

<!-- code: id=lp check="gcc -Wall -Wextra lp.c -o lp && ./lp" stdin="(a,(a,a))\n" -->
```c
/* lp.c: a table-driven predictive parser for nested lists
       S -> ( L ) | a
       L -> S M            M -> , S M | #        (M is L')
   # is the empty string. Reads one line; every character but a blank
   is one token, and $ is added at the end. Prints the stack, the rest
   of the input and the action at every move. */
#include <stdio.h>
#include <string.h>

const char *nts = "SLM";             /* the rows of the table */
const char *ts = "a(),$";            /* its columns */
const char *M[3][5] = {              /* right sides; NULL is a blank */
    /*         a      (      )      ,      $   */
    /* S */ { "a",   "(L)", NULL,  NULL,  NULL },
    /* L */ { "SM",  "SM",  NULL,  NULL,  NULL },
    /* M */ { NULL,  NULL,  "#",   ",SM", NULL },
};

char stack[100] = "$S";              /* the bottom is stack[0]; top at the end */
int top = 1;                         /* index of the top symbol */
char input[100];                     /* the tokens, then $ */

int moves;                           /* lines printed so far */

void show(int ip, const char *action)
{
    printf("%3d  %-14s%16s   %s\n", ++moves, stack, input + ip, action);
}

int main(void)
{
    char line[100], act[40];
    const char *r, *row, *col;
    int ip = 0, n = 0, i, k;
    char X, a;

    if (fgets(line, sizeof line, stdin) == NULL) {
        fprintf(stderr, "lp: no input\n");
        return 1;
    }
    for (i = 0; line[i] != '\0' && line[i] != '\n'; i++)
        if (line[i] != ' ' && line[i] != '\t' && n < 98)
            input[n++] = line[i];
    input[n] = '$';
    printf("%3s  %-14s%16s   %s\n", "", "STACK", "INPUT", "ACTION");
    for (;;) {
        X = stack[top];
        a = input[ip];
        if (X == '$' && a == '$') {
            show(ip, "accept");
            return 0;
        }
        row = strchr(nts, X);
        if (row == NULL) {                       /* X is a terminal or $ */
            if (X != a) {
                sprintf(act, "error: expected %c, found %c", X, a);
                show(ip, act);
                printf("rejected\n");
                return 1;
            }
            sprintf(act, "match %c", a);
            show(ip, act);
            stack[top--] = '\0';
            ip++;
            continue;
        }
        col = strchr(ts, a);
        r = (col == NULL) ? NULL : M[row - nts][col - ts];
        if (r == NULL) {                         /* a blank entry */
            sprintf(act, "error: M[%c,%c] is blank", X, a);
            show(ip, act);
            printf("rejected\n");
            return 1;
        }
        sprintf(act, "%c->%s", X, r);
        show(ip, act);
        stack[top--] = '\0';                     /* pop X */
        if (strcmp(r, "#") != 0)                 /* push the right side, */
            for (k = strlen(r) - 1; k >= 0; k--) /* last symbol first */
                stack[++top] = r[k];
        stack[top + 1] = '\0';
    }
}
```
```text
     STACK                    INPUT   ACTION
  1  $S                  (a,(a,a))$   S->(L)
  2  $)L(                (a,(a,a))$   match (
  3  $)L                  a,(a,a))$   L->SM
  4  $)MS                 a,(a,a))$   S->a
  5  $)Ma                 a,(a,a))$   match a
  6  $)M                   ,(a,a))$   M->,SM
  7  $)MS,                 ,(a,a))$   match ,
  8  $)MS                   (a,a))$   S->(L)
  9  $)M)L(                 (a,a))$   match (
 10  $)M)L                   a,a))$   L->SM
 11  $)M)MS                  a,a))$   S->a
 12  $)M)Ma                  a,a))$   match a
 13  $)M)M                    ,a))$   M->,SM
 14  $)M)MS,                  ,a))$   match ,
 15  $)M)MS                    a))$   S->a
 16  $)M)Ma                    a))$   match a
 17  $)M)M                      ))$   M->#
 18  $)M)                       ))$   match )
 19  $)M                         )$   M->#
 20  $)                          )$   match )
 21  $                            $   accept
```

The tests that must be rejected, the one-atom list, and no input:

<!-- code: id=lp-tests -->
```console
$ gcc -Wall -Wextra lp.c -o lp
$ echo 'a' | ./lp
     STACK                    INPUT   ACTION
  1  $S                          a$   S->a
  2  $a                          a$   match a
  3  $                            $   accept
$ echo '(a,)' | ./lp | tail -2
  8  $)MS                        )$   error: M[S,)] is blank
rejected
$ echo '' | ./lp; echo "exit status $?"
     STACK                    INPUT   ACTION
  1  $S                           $   error: M[S,$] is blank
rejected
exit status 1
$ ./lp < /dev/null; echo "exit status $?"
lp: no input
exit status 1
$ echo '(a a)' | ./lp | tail -2
  6  $)M                        a)$   error: M[M,a] is blank
rejected
```

In `(a,)` the comma promised another list, so after it an S is due and `)` cannot begin one. In `(a a)` the first list is finished and M must either see a `,` for another list or vanish at `)`; an `a` is neither. The likeliest mistake in this task is the table, not the code: putting `M -> ε` under `,` or `$` as well as `)`. FOLLOW(M) is only { ) }, because a list of lists is always closed by its parenthesis.

</details>
