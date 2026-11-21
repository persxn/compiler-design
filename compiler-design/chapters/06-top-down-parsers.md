# Chapter 6: Top-down parsers

<!-- contents -->
- [6.1 A recursive-descent parser in C](#61-a-recursive-descent-parser-in-c)
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
=>lm id T' E'   [6]
=>lm id E'   [6]
=>lm id + T E'   [2]
=>lm id + F T' E'   [4]
=>lm id + id T' E'   [8]
=>lm id + id * F T' E'   [5]
=>lm id + id * id T' E'   [8]
=>lm id + id * id E'   [6]
=>lm id + id   [3]
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
$ sed '/^int main/,$d' ../ch2/tiny-scan.c > scan.h
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
