# Chapter 7: Bottom-up parsers

<!-- contents -->
- [7.1 Shift-reduce parsing and handles, with a C program](#71-shift-reduce-parsing-and-handles-with-a-c-program)
- [7.2 Operator-precedence parsing: relations, table and a C parser](#72-operator-precedence-parsing-relations-table-and-a-c-parser)
- [7.3 LR(0) items, closure and goto: the canonical collection](#73-lr0-items-closure-and-goto-the-canonical-collection)
- [7.4 The SLR(1) parsing table](#74-the-slr1-parsing-table)
- [7.5 CLR(1) and LALR(1) tables by hand](#75-clr1-and-lalr1-tables-by-hand)
- [7.6 Shift-reduce and reduce-reduce conflicts](#76-shift-reduce-and-reduce-reduce-conflicts)
- [7.7 The LR driver in C: one table, one stack](#77-the-lr-driver-in-c-one-table-one-stack)
- [7.8 Practice: the full bottom-up question set](#78-practice-the-full-bottom-up-question-set)
<!-- /contents -->

## 7.1 Shift-reduce parsing and handles, with a C program

### The grammar you had to take apart

Chapter 6 parsed `i + i * i`, but not with the grammar you would write first. The expression grammar `E -> E + T | T`, `T -> T * F | F`, `F -> ( E ) | id` had to be rebuilt as `exprll.g`, with `E'` and `T'` and two ε-productions, before a top-down parser could use it, because a top-down parser decides on a production *before* it has seen what the production covers, and `E -> E + T` sent it round in circles. The rebuilt grammar works, but its parse trees no longer look like the expressions: `i + i` hangs off an `E'` that means nothing to a person.

There is another way to read the input. Instead of starting at `E` and guessing downward, start at the tokens and build upward: see an `i`, decide it is an `F`; see that `F`, decide it is a `T`; and so on, until the whole line has become one `E`. A parser that works this way only decides on a production *after* it has seen everything the production covers, so `E -> E + T` gives it no trouble at all. It is called a *bottom-up* parser, and it is the kind that Yacc and Bison build for you in chapter 8, which is the main reason the paper keeps asking about it.

This chapter builds bottom-up parsers three ways. This section and the next write two simple ones by hand. Sections 7.3 to 7.6 build the tables of the LR parsers, the family Yacc belongs to, by hand and by program. Section 7.7 writes the one driver that reads all of those tables, and section 7.8 is the question set.

### Plates that become one plate

The book's stack of plates comes back, used the other way round. In section 6.4 the parser began with the start symbol on its pile and broke it down into smaller promises. Here the parser begins with an empty pile and has two moves:

- **Shift**: take the next token from the input and put it on the pile as a new plate.
- **Reduce**: when the top few plates spell out the right side of a production, take them off and put down one plate for its left side. Three plates `E`, `+`, `T` become one plate `E`.

When the input is used up and the pile holds exactly one plate, the start symbol, the parser accepts. If it gets stuck before that, the input was wrong.

The picture breaks at the point that matters most. The plates tell you *what* is on top, but not *whether now is the moment* to reduce. A pile reading `E + T` may be the whole of `E -> E + T`, ready to be reduced, or it may be the start of `E + T * F`, where the `T` is about to grow and reducing now would be a mistake. The plates look the same in both cases. This whole chapter is about how a parser tells those cases apart, and the LR parsers of section 7.3 do it by writing extra information on every plate. For now, the parser will look at one more thing beside the plates: the next token of the input.

### The four actions, and the derivation they undo

<!-- teaches: shift-reduce = shift-reduce parser, shift, reduce, accept -->
A *shift-reduce parser* has a stack of grammar symbols, with `$` at the bottom, and the input, with `$` at its end. At every move it takes exactly one of four actions:

1. **shift**: move the next input token onto the top of the stack;
2. **reduce**: the top of the stack is the right side β of a production A -> β; pop β and push A;
3. **accept**: the stack is `$` with the start symbol on it, and the input is `$`;
4. **error**: none of the above can be done.

Here is the run on the expression of the running statement. `pos = init + rate * 60;` has the expression `init + rate * 60`, which the scanner turns into `id + id * num`, and with an `id` standing in for every operand that is `id + id * id`. The table shows the stack (bottom on the left, as in section 6.4), the input still to be read, and the action. Do not worry yet about *why* each reduction was chosen; look at what the moves do.

| Move | Stack | Input | Action |
|---|---|---|---|
| 1 | `$` | `id + id * id $` | shift |
| 2 | `$ id` | `+ id * id $` | reduce by F -> id |
| 3 | `$ F` | `+ id * id $` | reduce by T -> F |
| 4 | `$ T` | `+ id * id $` | reduce by E -> T |
| 5 | `$ E` | `+ id * id $` | shift |
| 6 | `$ E +` | `id * id $` | shift |
| 7 | `$ E + id` | `* id $` | reduce by F -> id |
| 8 | `$ E + F` | `* id $` | reduce by T -> F |
| 9 | `$ E + T` | `* id $` | shift |
| 10 | `$ E + T *` | `id $` | shift |
| 11 | `$ E + T * id` | `$` | reduce by F -> id |
| 12 | `$ E + T * F` | `$` | reduce by T -> T * F |
| 13 | `$ E + T` | `$` | reduce by E -> E + T |
| 14 | `$ E` | `$` | accept |

Now read only the reductions, from the last to the first: E -> E + T, T -> T * F, F -> id, T -> F, F -> id, E -> T, T -> F, F -> id. Apply them in that order starting from `E`, always to the rightmost nonterminal, and you get

```text
E  =>rm  E + T  =>rm  E + T * F  =>rm  E + T * id  =>rm  E + F * id
   =>rm  E + id * id  =>rm  T + id * id  =>rm  F + id * id  =>rm  id + id * id
```

which is a rightmost derivation of the input. That is the central fact about bottom-up parsing: **a shift-reduce parser traces out a rightmost derivation in reverse**. A top-down parser produces the leftmost derivation from the front (section 6.1); a bottom-up parser produces the rightmost one from the back. Each line of the table, if you put the stack (without `$`) in front of the remaining input, is one line of that derivation: at move 9, `E + T` and `* id` make `E + T * id`, the derivation's fourth line.

And the parse tree is built from the leaves up. Move 2 makes the node `F` over the first `id`; move 3 puts a `T` over that; move 12 joins a `T`, a `*` and an `F` under a new `T`. The last reduction makes the root. That is where the name "bottom-up" comes from.

### Handles: what gets reduced

Look at move 9 again. The stack holds `E + T`, and that is the right side of `E -> E + T`. The parser shifted anyway. Had it reduced, the stack would hold `E` with `* id` still to come, and `E * ...` cannot be finished: no production has `E *` in it. The `E + T` on the stack was the right shape at the wrong moment.

<!-- teaches: handle = handle, right-sentential form, handle pruning -->
The thing a correct parser reduces has a name. A *right-sentential form* is any line of a rightmost derivation. Its *handle* is the right side β of a production A -> β, at a particular place in the line, such that replacing that β by A gives the line before it in the rightmost derivation. In `E + T * id`, the handle is the `id` at the end, by F -> id, because the line before it in the derivation is `E + T * F`. The `E + T` at the front is not a handle, even though it matches a right side, because replacing it gives `E * id`, which is not a line of any rightmost derivation of this grammar. Reducing exactly the handle, line after line, until only the start symbol is left, is called *handle pruning*, and it is what every shift-reduce parser is trying to do.

The paper asks this directly: "for the right-sentential form below, find the handle". The method is to find the line before it. Here are the right-sentential forms of `id + id * id`, with each one's handle:

| Right-sentential form | Handle | Production |
|---|---|---|
| `id + id * id` | the first `id` | F -> id |
| `F + id * id` | `F` | T -> F |
| `T + id * id` | `T` | E -> T |
| `E + id * id` | the `id` after `+` | F -> id |
| `E + F * id` | `F` | T -> F |
| `E + T * id` | the last `id` | F -> id |
| `E + T * F` | `T * F` | T -> T * F |
| `E + T` | `E + T` | E -> E + T |

Two things to see. First, in `E + T * id` the handle is the `id`, not the `T` that also matches `E -> T`, and not the `E + T`. A form can contain several right sides; only one of them is the handle. Second, every handle in this table sits at the right-hand end of what the parser has seen so far: it is on top of the stack, never buried under other symbols. This holds for every shift-reduce parse, and the reason is the rightmost derivation. Everything to the right of a handle is still raw input, never yet rewritten, because a rightmost derivation rewrites the rightmost nonterminal first, so undoing it finds the most recent rewrite at the right end of what has been built. A shift-reduce parser therefore never has to dig in its stack: it only ever looks at the top. That is why a stack is enough.

So the parser's whole difficulty is one question asked at every move: *is there a handle on top of the stack right now, and which one?* If yes, reduce it; if no, shift.

### sr.c: a shift-reduce parser in C

A lab sheet that says "implement a shift-reduce parser" wants a program that prints the table above. The program needs a rule for the question, and here is one that works for this grammar. Reduce by A -> β when two things are true: β is on top of the stack, and the next token is one that can follow A, that is, it is in FOLLOW(A). Section 5.6 computed these sets for exactly this grammar: FOLLOW(E) = { +, ), $ }, FOLLOW(T) = FOLLOW(F) = { +, *, ), $ }. If more than one production qualifies, take the one with the longest right side. Otherwise shift.

Check the rule at move 9. `E + T` is on top, but the next token is `*`, and `*` is not in FOLLOW(E): an `E` is never followed by `*`. So `E -> E + T` is not allowed, and neither is `E -> T`, for the same reason. Shift. At move 13 the same `E + T` is on top with `$` next; `$` is in FOLLOW(E), and the longest right side that fits is `E + T`. Reduce.

The program also keeps a switch, `-g`, that drops the FOLLOW test and reduces greedily, whenever any right side is on top. It is there to show you what happens without the test. The input is a line in the lab-sheet style of section 6.1, every character but a blank one token, `i` for `id`.

Make the chapter's folder. Open a terminal in your home folder:

<!-- code: id=ch7-setup -->
```console
$ mkdir cdlab/ch7
$ cd cdlab/ch7
```

Save this as `sr.c`:

<!-- code: id=sr mode=compile -->
```c
/* sr.c: a shift-reduce parser for the expression grammar
       E -> E + T | T      T -> T * F | F      F -> ( E ) | i
   i stands for id. Reads one line; every character but a blank is one
   token, and $ is added at the end. At every move it reduces if the top
   of the stack is a right side whose left side may be followed by the
   lookahead, choosing the longest such right side; otherwise it shifts.
   With -g it reduces greedily, by any right side on top, whatever the
   lookahead. Prints the stack, the rest of the input and the action. */
#include <stdio.h>
#include <string.h>

struct production {
    char lhs;               /* the left side */
    const char *rhs;        /* the right side */
    const char *follow;     /* FOLLOW(lhs): when a reduction is allowed */
} P[] = {
    { 'E', "E+T", "+)$"  },
    { 'E', "T",   "+)$"  },
    { 'T', "T*F", "+*)$" },
    { 'T', "F",   "+*)$" },
    { 'F', "(E)", "+*)$" },
    { 'F', "i",   "+*)$" },
};
#define NP ((int) (sizeof P / sizeof P[0]))

char stack[100] = "$";      /* the bottom is stack[0]; the top at the end */
int top = 0;                /* index of the top symbol */
char input[100];            /* the tokens, then $ */
int moves;                  /* lines printed so far */

void show(int ip, const char *action)
{
    printf("%3d  %-14s%16s   %s\n", ++moves, stack, input + ip, action);
}

/* the production to reduce by, or -1: its right side must be on top */
int handle(char a, int greedy)
{
    int p, n, best = -1, bestlen = 0;

    for (p = 0; p < NP; p++) {
        n = strlen(P[p].rhs);
        if (n > top || strcmp(stack + top + 1 - n, P[p].rhs) != 0)
            continue;                       /* not on top of the stack */
        if (!greedy && strchr(P[p].follow, a) == NULL)
            continue;                       /* a cannot follow the lhs */
        if (n > bestlen) {
            best = p;
            bestlen = n;
        }
    }
    return best;
}

int main(int argc, char *argv[])
{
    char line[100], act[40];
    int greedy = argc > 1 && strcmp(argv[1], "-g") == 0;
    int ip = 0, n = 0, i, p;

    if (fgets(line, sizeof line, stdin) == NULL) {
        fprintf(stderr, "sr: no input\n");
        return 1;
    }
    for (i = 0; line[i] != '\0' && line[i] != '\n'; i++)
        if (line[i] != ' ' && line[i] != '\t' && n < 98)
            input[n++] = line[i];
    input[n] = '$';
    printf("%3s  %-14s%16s   %s\n", "", "STACK", "INPUT", "ACTION");
    for (;;) {
        if (strcmp(stack, "$E") == 0 && input[ip] == '$') {
            show(ip, "accept");
            return 0;
        }
        p = handle(input[ip], greedy);
        if (p >= 0) {                       /* reduce */
            sprintf(act, "reduce %c->%s", P[p].lhs, P[p].rhs);
            show(ip, act);
            top -= strlen(P[p].rhs);        /* pop the handle ... */
            stack[++top] = P[p].lhs;        /* ... push the left side */
            stack[top + 1] = '\0';
        } else if (input[ip] != '$' && top < 98) {
            sprintf(act, "shift %c", input[ip]);
            show(ip, act);
            stack[++top] = input[ip++];     /* shift */
            stack[top + 1] = '\0';
        } else {
            show(ip, "error: no handle, nothing to shift");
            printf("rejected\n");
            return 1;
        }
    }
}
```

The grammar is a table of three columns: the left side, the right side as a string, and the FOLLOW set of the left side as a string of the tokens that may come next. A reduction is allowed only when the lookahead is in that string, which `strchr` tests in one call.

The stack is a C string, exactly as in `pp.c`: `stack[0]` is `$`, `stack[top]` is the top, and a `'\0'` always follows it, so `show` can print the whole pile with `%s`. That also makes the question "is β on top?" one `strcmp`: `stack + top + 1 - n` points at the last `n` characters of the stack, and they are compared with the right side. The test `n > top` stops a right side longer than the stack from reaching below `stack[0]`.

`handle` answers the parser's one question. It walks every production, skips those whose right side is not on top, skips (unless `-g` was given) those whose left side cannot be followed by the lookahead, and keeps the longest right side among the rest. It returns -1 when there is no handle.

The loop in `main` is the four actions. Accept comes first: stack `$E`, input at `$`. Then reduce, if `handle` found one: `top` moves down by the length of the right side, so it points at the symbol just under the handle, and the left side is written above it. Then shift, if there is a token left to shift. If none of the three is possible, the input is wrong: an error, `rejected`, and exit status 1.

### The running expression, move by move

Build it and give it the expression:

<!-- code: id=sr-run -->
```console
$ gcc -Wall -Wextra sr.c -o sr
$ echo 'i + i * i' | ./sr
     STACK                    INPUT   ACTION
  1  $                       i+i*i$   shift i
  2  $i                       +i*i$   reduce F->i
  3  $F                       +i*i$   reduce T->F
  4  $T                       +i*i$   reduce E->T
  5  $E                       +i*i$   shift +
  6  $E+                       i*i$   shift i
  7  $E+i                       *i$   reduce F->i
  8  $E+F                       *i$   reduce T->F
  9  $E+T                       *i$   shift *
 10  $E+T*                       i$   shift i
 11  $E+T*i                       $   reduce F->i
 12  $E+T*F                       $   reduce T->T*F
 13  $E+T                         $   reduce E->E+T
 14  $E                           $   accept
```

Fourteen moves, and they are the hand table's moves, one for one. The stack column is the pile of plates; the input column loses a token on every shift and stays the same on every reduce. A reduce never reads input. It only rewrites the top of the stack.

Section 5.1's `derive` can confirm that the reductions are a rightmost derivation backwards. Number `expr.g`'s productions the way `derive` lists them (1 E -> E + T, 2 E -> T, 3 T -> T * F, 4 T -> F, 5 F -> ( E ), 6 F -> id), take the reductions from the last to the first, and ask for a rightmost derivation with `-r`:

<!-- code: id=sr-derive -->
```console
$ ../ch5/derive -r ../ch5/expr.g 1 3 6 4 6 2 4 6 | head -9
     E
=>rm E + T   [1]
=>rm E + T * F   [3]
=>rm E + T * id   [6]
=>rm E + F * id   [4]
=>rm E + id * id   [6]
=>rm T + id * id   [2]
=>rm F + id * id   [4]
=>rm id + id * id   [6]
```

`derive` accepted every step and ended on `id + id * id`. Each line of its derivation is a line of `sr`'s trace with the stack and the input joined: line 4, `E + T * id`, is move 9's `$E+T` followed by `*i$`.

### Without the lookahead

Now the same input with `-g`, which reduces whenever any right side is on top:

<!-- code: id=sr-greedy exit=1 -->
```console
$ echo 'i + i * i' | ./sr -g
     STACK                    INPUT   ACTION
  1  $                       i+i*i$   shift i
  2  $i                       +i*i$   reduce F->i
  3  $F                       +i*i$   reduce T->F
  4  $T                       +i*i$   reduce E->T
  5  $E                       +i*i$   shift +
  6  $E+                       i*i$   shift i
  7  $E+i                       *i$   reduce F->i
  8  $E+F                       *i$   reduce T->F
  9  $E+T                       *i$   reduce E->E+T
 10  $E                         *i$   shift *
 11  $E*                         i$   shift i
 12  $E*i                         $   reduce F->i
 13  $E*F                         $   reduce T->F
 14  $E*T                         $   reduce E->T
 15  $E*E                         $   error: no handle, nothing to shift
rejected
```

Up to move 8 the two runs are the same. At move 9 the greedy parser sees `E+T` on top and reduces it, because nothing tells it not to. From there it is lost: it shifts `*` and `i`, reduces the `i` to `F`, `T` and then `E` (each was a right side on top), and ends with `$E*E` on the stack and nothing to shift. No production has `E * E` as its right side, so it rejects a correct expression. The greedy parser reduced something that matched a right side but was not the handle, exactly the mistake the handle table warned about. The one token of lookahead, checked against FOLLOW, is what made the first run right.

### When the input is wrong

The rejected inputs of chapter 6:

<!-- code: id=sr-errors exit=1 -->
```console
$ for s in 'i+*i' 'i i' '(i' ''; do echo "== $s"; echo "$s" | ./sr; done
== i+*i
     STACK                    INPUT   ACTION
  1  $                        i+*i$   shift i
  2  $i                        +*i$   reduce F->i
  3  $F                        +*i$   reduce T->F
  4  $T                        +*i$   reduce E->T
  5  $E                        +*i$   shift +
  6  $E+                        *i$   shift *
  7  $E+*                        i$   shift i
  8  $E+*i                        $   reduce F->i
  9  $E+*F                        $   reduce T->F
 10  $E+*T                        $   reduce E->T
 11  $E+*E                        $   error: no handle, nothing to shift
rejected
== i i
     STACK                    INPUT   ACTION
  1  $                          ii$   shift i
  2  $i                          i$   shift i
  3  $ii                          $   reduce F->i
  4  $iF                          $   reduce T->F
  5  $iT                          $   reduce E->T
  6  $iE                          $   error: no handle, nothing to shift
rejected
== (i
     STACK                    INPUT   ACTION
  1  $                          (i$   shift (
  2  $(                          i$   shift i
  3  $(i                          $   reduce F->i
  4  $(F                          $   reduce T->F
  5  $(T                          $   reduce E->T
  6  $(E                          $   error: no handle, nothing to shift
rejected
==
     STACK                    INPUT   ACTION
  1  $                            $   error: no handle, nothing to shift
rejected
```

All four are rejected, and you should notice *when*. In `i+*i` the mistake is the `*` straight after the `+`, but `sr` shifts the `*` without complaint (no reduction is allowed with `*` next to `E+`, so it shifts) and only gives up at the very end, with `$E+*E` on the stack. In `i i` it shifts the second `i` on top of the first and again only finds out at `$`. The FOLLOW test can stop a wrong *reduction*, but nothing in `sr` can say that a *shift* is wrong: it shifts whatever comes. So the error is reported late, and the message can only say that nothing fits.

That is one weakness of this simple rule; section 7.4 shows the other, a grammar where FOLLOW cannot decide. Section 7.7's LR parser has a table entry for every pair of state and token, so it refuses `*` the moment it is read: no correct expression has `+ *` in it, and the table knows it.

### When it goes wrong

**Reducing whenever a right side is on top.** The `-g` run above: correct inputs are rejected, or, with other grammars, accepted with the wrong tree. A right side on top is a candidate; it is a handle only if the lookahead allows it.

**Reading the reductions forwards as the derivation.** The first reduction, F -> id, is the *last* step of the rightmost derivation, not the first. If the paper asks for the derivation the parser found, write the reductions from the bottom of the trace up.

**Popping the wrong number of symbols.** A reduce by T -> T * F pops three symbols, the whole right side, and pushes one. Popping only the `F`, or pushing `T` on top of `T *`, leaves a stack that no later move can fix. In code it is `top -= strlen(rhs)` before the push, never after.

**Forgetting `$`.** Without `$` at the end of the input, there is no lookahead after the last token, so the last reductions (moves 11 to 13 above) have nothing to check against and the parser cannot decide to finish.

**The handle of a form, by eye.** "The leftmost right side" is not the handle in general: in `E + T * id` the leftmost right side is `E + T`, and the handle is the `id`. Find the line before it in the rightmost derivation; the part that changed is the handle.

### In one line each

1. A shift-reduce parser shifts tokens onto a stack and reduces right sides on top of it to their left sides, accepting when only the start symbol is left.
2. Its reductions, read backwards, are a rightmost derivation; the tree is built from the leaves up.
3. A handle is the right side whose reduction gives the previous line of the rightmost derivation; it is always on top of the stack.
4. A right side on top is not always a handle; `sr.c` decides with the lookahead and FOLLOW of the left side, and `-g` shows what happens without it.
5. This simple parser finds errors late, because it can never refuse a shift.

**Check yourself**

<!-- q: idea=handle type=apply wrong="the leftmost right side is the handle" -->
1. For `( id + id ) * id`, write the right-sentential forms of its rightmost derivation from the input up to `E`, and mark the handle of each. Then check your list against `sr`'s trace for `(i+i)*i`.

<details>
<summary>Answer</summary>

| Right-sentential form | Handle |
|---|---|
| `( id + id ) * id` | first `id`, F -> id |
| `( F + id ) * id` | `F`, T -> F |
| `( T + id ) * id` | `T`, E -> T |
| `( E + id ) * id` | `id`, F -> id |
| `( E + F ) * id` | `F`, T -> F |
| `( E + T ) * id` | `E + T`, E -> E + T |
| `( E ) * id` | `( E )`, F -> ( E ) |
| `F * id` | `F`, T -> F |
| `T * id` | last `id`, F -> id |
| `T * F` | `T * F`, T -> T * F |
| `T` | `T`, E -> T |

The usual slip is in `T * id`: `T` matches E -> T, but reducing it gives `E * id`, a dead end; the handle is the `id`. In `sr`'s trace this is move 14, `$T` with `*` next, where the parser shifts.

<!-- code: id=sr-paren -->
```console
$ echo '(i+i)*i' | ./sr | grep reduce
  3  $(i                     +i)*i$   reduce F->i
  4  $(F                     +i)*i$   reduce T->F
  5  $(T                     +i)*i$   reduce E->T
  8  $(E+i                     )*i$   reduce F->i
  9  $(E+F                     )*i$   reduce T->F
 10  $(E+T                     )*i$   reduce E->E+T
 12  $(E)                       *i$   reduce F->(E)
 13  $F                         *i$   reduce T->F
 16  $T*i                         $   reduce F->i
 17  $T*F                         $   reduce T->T*F
 18  $T                           $   reduce E->T
```

</details>

<!-- q: idea=shift-reduce type=predict wrong="the greedy parser fails on every input with an operator" -->
2. Predict which of `i+i`, `i*i` and `(i)` the greedy parser `./sr -g` accepts. Then run it.

<details>
<summary>Answer</summary>

`i+i` and `(i)` are accepted; `i*i` is rejected. Greedy reduction goes wrong only when it reduces a `T` to `E` (or `E + T` to `E`) that a `*` was about to extend. In `i+i` no `*` ever comes, and in `(i)` the reductions inside the parentheses are all wanted. In `i*i` the first `i` becomes `F`, `T`, then `E` before the `*` is seen, and `E * ...` can never be reduced.

<!-- code: id=sr-greedy-three -->
```console
$ for s in 'i+i' 'i*i' '(i)'; do echo "$s" | ./sr -g | tail -1; done
 10  $E                           $   accept
rejected
 10  $E                           $   accept
```

</details>

<!-- q: idea=shift-reduce type=fix wrong="the push and the pop can be done in either order" -->
3. Find the fault. A student's reduce step has the push and the pop the other way round, and the line that writes the `'\0'` after them is unchanged:

```c
stack[++top] = P[p].lhs;            /* ... push the left side */
top -= strlen(P[p].rhs);            /* pop the handle ... */
stack[top + 1] = '\0';
```

Predict what the program prints for the input `i`. Then make the faulty copy and run it (the `head` stops it).

<details>
<summary>Answer</summary>

On `$i`, the reduction F -> i first writes `F` above the `i` (`top` is now 2), then moves `top` back down by 1, to the `i`, and the third line writes `'\0'` over the `F`. The stack is `$i` again, exactly as before the reduction, with `i` on top and `$` next, so the parser reduces F -> i again, and again, for ever. The handle must be popped first and the left side written at the new top: the order in `sr.c`.

<!-- code: id=sr-swap -->
```console
$ sed '/pop the handle/{h;d;};/push the left side/G' sr.c > srbad.c
$ gcc -Wall -Wextra srbad.c -o srbad
$ echo 'i' | ./srbad | head -5
     STACK                    INPUT   ACTION
  1  $                           i$   shift i
  2  $i                           $   reduce F->i
  3  $i                           $   reduce F->i
  4  $i                           $   reduce F->i
```

The `sed` command holds the "pop" line (`h`), deletes it where it was (`d`), and appends it after the "push" line (`G`). A parser that loops without reading input is the usual sign of a reduce step that does not change the stack.

</details>

## 7.2 Operator-precedence parsing: relations, table and a C parser

### A parser that only looks at operators

`sr.c` needed the whole grammar, FOLLOW sets included, to decide its handles. But when the grammar is an expression grammar, you already know a quicker way to find what to work out first: you look at the operators. In `id + id * id` you do the `*` before the `+` because `*` binds tighter, and in `id - id - id` you do the left `-` first because `-` groups to the left. Neither decision needs to know the names `E`, `T` and `F`. It only compares two operators.

The *operator-precedence parser* turns exactly that into a parsing method. It finds handles from a small table of relations between pairs of terminals, and it ignores nonterminals altogether. It is older and weaker than the LR parsers of the coming sections, but it is easy to build by hand, its table is easy to work in the written paper, and "implement an operator-precedence parser" is one of the standard lab questions. The paper's question comes in four parts, and this section covers all four: build the table, compute LEADING and TRAILING, parse a string showing the stack, and find the precedence functions.

### Invisible brackets

Write `$` at both ends of `id + id * id` and put a mark between every two neighbouring terminals: `<·` where the right one starts something to be worked out first, `·>` where the left one ends it.

```text
$  <·  id  ·>  +  <·  id  ·>  *  <·  id  ·>  $
```

Each `<· ... ·>` pair now encloses a handle: each `id` is one, to be reduced to an expression. Reduce them, drop the nonterminals from view (they do not take part in the comparisons), and mark the terminals that are left:

```text
$  <·  +  <·  *  ·>  $
```

Now the only `<· ... ·>` pair encloses the `*`, so the next handle is `E * E`, the multiplication, with its two operands. After that only `$ <· + ·> $` is left, and the last handle is the addition. The marks behaved like brackets that someone had written into the expression invisibly: `(id + (id * id))`. The parser's job is to read those brackets off a table.

The picture breaks in one important way. The marks look like "less than" and "greater than", but they are not an order. `+ ·> +` holds (the left `+` is worked out first, because `+` groups to the left), so a terminal can be "greater" than itself; and `( <· (` holds while `) ·> )` also holds. They are three separate relations, written in a table, and only sometimes can they be replaced by numbers (the precedence functions at the end of this section).

### Operator grammars and the three relations

<!-- teaches: operator-precedence = operator grammar, operator-precedence relation, operator-precedence parser, precedence function -->
The method only works on an *operator grammar*: a grammar in which no right side is ε and no right side has two nonterminals side by side. The expression grammar `expr.g` is one, and so is the ambiguous `amb.g` (`E -> E + E | E * E | ( E ) | id`). `exprll.g` is not, because of its ε-productions, and `S -> A B` is not, because `A B` are neighbours. The condition guarantees that between any two terminals of a sentential form there is at most one nonterminal, so comparing neighbouring terminals tells you everything.

The three *operator-precedence relations*, for terminals a and b:

| Relation | Read | Meaning for a handle |
|---|---|---|
| a <· b | a yields precedence to b | a handle starts at b |
| a ≐ b | a has the same precedence as b | a and b are in the same handle |
| a ·> b | a takes precedence over b | a handle ends at a |

The `$` at each end is a terminal for this purpose: every relation with `$` says that the expression starts or ends there.

### The table from precedence and associativity

For an expression grammar, the quickest way to fill the table is straight from the rules you know. Write θ for an operator.

1. If θ1 has higher precedence than θ2: θ1 ·> θ2 and θ2 <· θ1. So `* ·> +` and `+ <· *`.
2. If θ1 and θ2 have equal precedence (θ1 may be θ2) and group to the left: θ1 ·> θ2 and θ2 ·> θ1. So `+ ·> +` and `* ·> *`. If they group to the right, use `<·` both ways instead.
3. `id` is worked out before any operator next to it: θ <· id and id ·> θ.
4. Parentheses: ( ≐ ); θ <· ( and ( <· θ; ) ·> θ and θ ·> ); also ( <· (, ) ·> ), ( <· id, id ·> ).
5. The ends: $ <· θ, $ <· id, $ <· (; θ ·> $, id ·> $, ) ·> $.

For `+` and `*`, `*` above `+`, both left-associative, the rows are the terminal on the stack and the columns the next input token:

| | id | + | * | ( | ) | $ |
|---|---|---|---|---|---|---|
| **id** | | ·> | ·> | | ·> | ·> |
| **+** | <· | ·> | <· | <· | ·> | ·> |
| **\*** | <· | ·> | ·> | <· | ·> | ·> |
| **(** | <· | <· | <· | <· | ≐ | |
| **)** | | ·> | ·> | | ·> | ·> |
| **$** | <· | <· | <· | <· | | |

Each blank is an error: no correct expression puts those two terminals next to each other with at most one nonterminal between them. `id` followed by `id`, `)` followed by `(`, `(` followed by `$`.

### The table from the grammar: LEADING and TRAILING

The paper often gives the grammar, not the precedence rules, and asks for the table "by computing LEADING and TRAILING". These two sets do for an operator grammar what FIRST and FOLLOW do for a top-down parser.

LEADING(A) is the set of terminals that can be the first *terminal* of a string derived from A, where a single nonterminal may stand in front of it. Two rules, repeated until nothing changes:

- if A -> a γ or A -> B a γ, then a is in LEADING(A);
- if A -> B γ, then everything in LEADING(B) is in LEADING(A).

TRAILING(A) is the same from the right end: the last terminal of a string derived from A, with at most one nonterminal after it.

- if A -> γ a or A -> γ a B, then a is in TRAILING(A);
- if A -> γ B, then everything in TRAILING(B) is in TRAILING(A).

For `expr.g`, start at the bottom, where the answers are direct. `F -> ( E ) | id`: LEADING(F) = { (, id }, TRAILING(F) = { ), id }. `T -> T * F | F`: `T * F` gives `*` to both LEADING(T) (as `B a γ` with B = T) and TRAILING(T) (as `γ a B` with B = F); `T * F` starts with T and ends with F, and `T -> F` is both, so LEADING(T) also takes LEADING(F) and TRAILING(T) takes TRAILING(F). LEADING(T) = { *, (, id }, TRAILING(T) = { *, ), id }. In the same way LEADING(E) = { +, *, (, id } and TRAILING(E) = { +, *, ), id }.

Then the relations, from every right side X1 X2 ... Xn:

1. **≐**: if Xi and Xi+1 are both terminals, Xi ≐ Xi+1; if Xi and Xi+2 are terminals with a nonterminal between, Xi ≐ Xi+2.
2. **<·**: if Xi is a terminal and Xi+1 a nonterminal, Xi <· every terminal in LEADING(Xi+1).
3. **·>**: if Xi is a nonterminal and Xi+1 a terminal, every terminal in TRAILING(Xi) ·> Xi+1.
4. **$**: $ <· every terminal in LEADING(S), and every terminal in TRAILING(S) ·> $, S the start symbol.

Apply them to `expr.g`:

- `E -> E + T`: rule 3, TRAILING(E) ·> +, so `+ ·> +`, `* ·> +`, `) ·> +`, `id ·> +`. Rule 2, + <· LEADING(T), so `+ <· *`, `+ <· (`, `+ <· id`.
- `T -> T * F`: TRAILING(T) ·> *, so `* ·> *`, `) ·> *`, `id ·> *`; * <· LEADING(F), so `* <· (`, `* <· id`.
- `F -> ( E )`: rule 1, `( ≐ )`. Rule 2, ( <· LEADING(E): `( <· +`, `( <· *`, `( <· (`, `( <· id`. Rule 3, TRAILING(E) ·> ): `+ ·> )`, `* ·> )`, `) ·> )`, `id ·> )`.
- Rule 4 with S = E: `$ <· +`, `$ <· *`, `$ <· (`, `$ <· id`, and `+ ·> $`, `* ·> $`, `) ·> $`, `id ·> $`.

Put them in the grid and it is the table above, entry for entry. That is no accident: `expr.g` builds precedence into its structure (section 5.2), and LEADING and TRAILING read it back out. The precedence comes from `E -> E + T` putting a `T` after `+`, so `+ <· *`; the left associativity from `E + T` putting an `E` before the `+`, so `+ ·> +`.

### The parsing algorithm

The parser keeps a stack of terminals and nonterminals, `$` at the bottom, and the input with `$` at its end. Every nonterminal is written `E`: an operator-precedence parser does not know which nonterminal a handle reduces to, and does not need to. Let a be the *topmost terminal* on the stack (skip any `E` on top), and b the next input token.

1. If a and b are both `$`: accept if the stack is `$E`, otherwise report an error.
2. If a <· b or a ≐ b: shift b.
3. If a ·> b: reduce. The handle ends at the top of the stack. To find where it starts, walk down the stack from terminal to terminal: as long as the terminal below is ≐ the one above it, keep going; at the first terminal below that is <· the one above it, stop. The handle is everything above that terminal, including an `E` just above it. Pop the handle and push one `E`. If the handle is not the right side of any production (with every nonterminal read as `E`), report an error.
4. If the table entry is blank: report an error.

Run it by hand on `id + id * id`. The relation column is the one between the topmost terminal and the next token.

| Move | Stack | Relation | Input | Action |
|---|---|---|---|---|
| 1 | `$` | <· | `id + id * id $` | shift |
| 2 | `$ id` | ·> | `+ id * id $` | reduce E -> id |
| 3 | `$ E` | <· | `+ id * id $` | shift (topmost terminal is `$`) |
| 4 | `$ E +` | <· | `id * id $` | shift |
| 5 | `$ E + id` | ·> | `* id $` | reduce E -> id |
| 6 | `$ E + E` | <· | `* id $` | shift (topmost terminal `+`, and `+ <· *`) |
| 7 | `$ E + E *` | <· | `id $` | shift |
| 8 | `$ E + E * id` | ·> | `$` | reduce E -> id |
| 9 | `$ E + E * E` | ·> | `$` | reduce E -> E * E |
| 10 | `$ E + E` | ·> | `$` | reduce E -> E + E |
| 11 | `$ E` | | `$` | accept |

Move 6 is the decision `sr.c` needed FOLLOW for. Here it is one entry: the topmost terminal is `+`, the next is `*`, and `+ <· *` says a new handle starts at the `*`, so shift. Move 9 shows how the handle is found. The topmost terminal is `*`; the terminal below it is `+`, and `+ <· *`, so the handle starts just above the `+`: it is `E * E`. At move 10 the topmost terminal is `+` and the one below is `$`, with `$ <· +`, so the handle is `E + E`.

### opp.c: the parser in C

The table goes into the program as six strings, one per row, one character per column: `<`, `=`, `>`, or a blank for an error. Save this as `opp.c`:

<!-- code: id=opp mode=compile -->
```c
/* opp.c: an operator-precedence parser for expressions over
   i (id), + , * and parentheses, with * above + and both left
   associative. Reads one line; every character but a blank is one
   token, and $ is added at the end. Every nonterminal is written E.
   Prints the stack, the relation, the rest of the input and the action. */
#include <stdio.h>
#include <string.h>

const char *ts = "i+*()$";          /* the terminals: rows and columns */
const char *R[6] = {                /* R[a][b]: a on the stack, b the input */
    /*        i+*()$  */
    /* i */  " >> >>",
    /* + */  "<><<>>",
    /* * */  "<>><>>",
    /* ( */  "<<<<= ",
    /* ) */  " >> >>",
    /* $ */  "<<<<  ",
};
const char *rhs[] = { "E+E", "E*E", "(E)", "i" };  /* the handles allowed */

char stack[100] = "$";              /* the bottom is stack[0]; top at the end */
int top = 0;
char input[100];
int moves;

void show(int ip, char rel, const char *action)
{
    char r[3] = "  ";

    if (rel == '<')
        strcpy(r, "<.");
    else if (rel == '>')
        strcpy(r, ".>");
    else if (rel == '=')
        strcpy(r, "=");
    printf("%3d  %-14s %-2s %14s   %s\n", ++moves, stack, r, input + ip,
           action);
}

int rel(char a, char b)             /* the relation between a and b */
{
    const char *x = strchr(ts, a), *y = strchr(ts, b);

    if (x == NULL || y == NULL || a == '\0' || b == '\0')
        return ' ';
    return R[x - ts][y - ts];
}

int topterm(int k)                  /* the topmost terminal at or below k */
{
    while (stack[k] == 'E')
        k--;
    return k;
}

int main(void)
{
    char line[100], act[128], handle[100];
    int ip = 0, n = 0, i, t, below, ok;
    char r;

    if (fgets(line, sizeof line, stdin) == NULL) {
        fprintf(stderr, "opp: no input\n");
        return 1;
    }
    for (i = 0; line[i] != '\0' && line[i] != '\n'; i++)
        if (line[i] != ' ' && line[i] != '\t' && n < 98)
            input[n++] = line[i];
    input[n] = '$';
    printf("%3s  %-14s %-2s %14s   %s\n", "", "STACK", "", "INPUT", "ACTION");
    for (;;) {
        t = topterm(top);
        if (stack[t] == '$' && input[ip] == '$') {
            if (strcmp(stack, "$E") == 0) {
                show(ip, ' ', "accept");
                return 0;
            }
            show(ip, ' ', "error: no expression");
            break;
        }
        r = rel(stack[t], input[ip]);
        if (r == '<' || r == '=') {         /* shift */
            sprintf(act, "shift %c", input[ip]);
            show(ip, r, act);
            if (top == 98)
                break;
            stack[++top] = input[ip++];
            stack[top + 1] = '\0';
        } else if (r == '>') {              /* reduce: find the <. */
            for (;;) {
                below = topterm(t - 1);
                if (rel(stack[below], stack[t]) == '<')
                    break;
                t = below;                  /* = : the handle goes on */
            }
            strcpy(handle, stack + below + 1);
            for (ok = i = 0; i < 4; i++)
                if (strcmp(handle, rhs[i]) == 0)
                    ok = 1;
            if (!ok) {
                sprintf(act, "error: no production E->%s", handle);
                show(ip, r, act);
                break;
            }
            sprintf(act, "reduce E->%s", handle);
            show(ip, r, act);
            top = below + 1;                /* pop the handle, push E */
            stack[top] = 'E';
            stack[top + 1] = '\0';
        } else {
            sprintf(act, "error: no relation %c %c", stack[t], input[ip]);
            show(ip, r, act);
            break;
        }
    }
    printf("rejected\n");
    return 1;
}
```

`rel` looks up a relation the way `pp.c` looked up its table: `strchr` finds each terminal in `ts`, and the two pointer differences are the row and the column. A character that is not a terminal gives a blank, an error. (The extra test for `'\0'` is needed because `strchr` finds the string's own terminating `'\0'` and would return a pointer to it.)

`topterm(k)` walks down from position `k` past any `E` and returns where the topmost terminal is. Since `$` is never an `E`, it always stops.

The reduce branch is rule 3 of the algorithm. `t` starts at the topmost terminal. The inner loop finds the terminal `below` it; if `below <· t`, the handle starts just above `below`, and the loop stops; otherwise (the relation is ≐) `t` moves down to `below` and the walk goes on. The handle is then the string from `stack + below + 1` to the top, which `strcpy` copies out so that it can be compared with the four allowed right sides. Notice that the grammar appears in this program only in that list, and only to catch errors: the relations alone decided every shift and every reduce.

### The running expression, and the brackets it finds

<!-- code: id=opp-run -->
```console
$ gcc -Wall -Wextra opp.c -o opp
$ echo 'i + i * i' | ./opp
     STACK                      INPUT   ACTION
  1  $              <.         i+i*i$   shift i
  2  $i             .>          +i*i$   reduce E->i
  3  $E             <.          +i*i$   shift +
  4  $E+            <.           i*i$   shift i
  5  $E+i           .>            *i$   reduce E->i
  6  $E+E           <.            *i$   shift *
  7  $E+E*          <.             i$   shift i
  8  $E+E*i         .>              $   reduce E->i
  9  $E+E*E         .>              $   reduce E->E*E
 10  $E+E           .>              $   reduce E->E+E
 11  $E                             $   accept
```

Eleven moves, the hand table's. Compare it with `sr`'s fourteen: the three moves `sr` spent on `F -> id`, `T -> F` and `E -> T` for one operand are one move here, `E->i`, because nonterminals are not told apart. That is exactly the operator-precedence parser's trade. It is shorter and needs no grammar analysis, but the tree it builds has only operators and operands in it, a syntax tree like the one in section 1.2 rather than a parse tree, and it cannot tell a `T` from an `E`.

Parentheses use the ≐ relation:

<!-- code: id=opp-paren -->
```console
$ echo '(i+i)*i' | ./opp
     STACK                      INPUT   ACTION
  1  $              <.       (i+i)*i$   shift (
  2  $(             <.        i+i)*i$   shift i
  3  $(i            .>         +i)*i$   reduce E->i
  4  $(E            <.         +i)*i$   shift +
  5  $(E+           <.          i)*i$   shift i
  6  $(E+i          .>           )*i$   reduce E->i
  7  $(E+E          .>           )*i$   reduce E->E+E
  8  $(E            =            )*i$   shift )
  9  $(E)           .>            *i$   reduce E->(E)
 10  $E             <.            *i$   shift *
 11  $E*            <.             i$   shift i
 12  $E*i           .>              $   reduce E->i
 13  $E*E           .>              $   reduce E->E*E
 14  $E                             $   accept
```

At move 8 the topmost terminal is `(` and the next token is `)`, with `( ≐ )`: shift, because the two belong to the same handle. At move 9, `)` on top and `*` next, with `) ·> *`: reduce. The walk down from `)` finds `(` with `( ≐ )`, so the handle goes on; below `(` is `$`, with `$ <· (`, so the handle starts just above `$` and is `(E)`.

### Checking a table with a program

Typing the table into `opp.c` is where a lab program goes wrong, and working it by hand is where the paper's marks are, so habit 7 applies: check it with a program. `optable` reads a grammar in `grammar.h`'s form, checks that it is an operator grammar, computes LEADING and TRAILING with the rules above (repeating until nothing changes, as `first` did), fills the relations, and lists every entry that gets two different relations, a *clash*. If there is none, it also computes the precedence functions of the next part. It uses `grammar.h` and `first.h` from chapter 5 (for the reading and the sets), which chapter 6 already copied into its folder. Copy them, with `follow.h`, which section 7.3 needs:

<!-- code: id=optable-setup -->
```console
$ cp ../ch6/grammar.h ../ch6/first.h ../ch6/follow.h .
```

Save this as `optable.c`:

<!-- code: id=optable mode=compile -->
```c
/* optable.c: LEADING and TRAILING of every nonterminal, the
   operator-precedence relations, and precedence functions if they exist.
   Reads an operator grammar on standard input (grammar.h's form). */
#include "grammar.h"
#include "first.h"

set lead[MAXSYM], trail[MAXSYM];
char rel[MAXSYM][MAXSYM];           /* '<', '=', '>', '*' (two) or 0 */
char why[MAXSYM * 4][40];           /* one line per entry given two relations */
int clash;

void checkoperator(void)            /* no eps, no two nonterminals side by side */
{
    int p, i;

    for (p = 0; p < np; p++) {
        if (len[p] == 0)
            die("not an operator grammar: %s", prodtext(p));
        for (i = 0; i + 1 < len[p]; i++)
            if (isnt[rhs[p][i]] && isnt[rhs[p][i + 1]])
                die("not an operator grammar: %s", prodtext(p));
    }
}

void leadtrail(void)                /* both, until a pass changes nothing */
{
    int p, n, changed;
    set old;

    do {
        changed = 0;
        for (p = 0; p < np; p++) {
            int *x = rhs[p], a = lhs[p];

            n = len[p];
            old = lead[a];          /* A -> a ...  or  A -> B a ... */
            if (!isnt[x[0]])
                lead[a] |= bit(x[0]);
            else {
                lead[a] |= lead[x[0]];          /* A -> B ... */
                if (n > 1)
                    lead[a] |= bit(x[1]);
            }
            changed |= lead[a] != old;
            old = trail[a];         /* A -> ... a  or  A -> ... a B */
            if (!isnt[x[n - 1]])
                trail[a] |= bit(x[n - 1]);
            else {
                trail[a] |= trail[x[n - 1]];    /* A -> ... B */
                if (n > 1)
                    trail[a] |= bit(x[n - 2]);
            }
            changed |= trail[a] != old;
        }
    } while (changed);
}

void setrel(int a, int b, char r)
{
    if (rel[a][b] == 0)
        rel[a][b] = r;
    else if (rel[a][b] != r && rel[a][b] != '*') {
        if (clash < MAXSYM * 4)
            sprintf(why[clash++], "clash: %s %s and %s %s", name[a],
                    rel[a][b] == '<' ? "<." : rel[a][b] == '>' ? ".>" : "=",
                    r == '<' ? "<." : r == '>' ? ".>" : "=", name[b]);
        rel[a][b] = '*';
    }
}

void relations(void)
{
    int p, i, b, *x;

    for (p = 0; p < np; p++) {
        x = rhs[p];
        for (i = 0; i + 1 < len[p]; i++) {
            if (!isnt[x[i]] && !isnt[x[i + 1]])
                setrel(x[i], x[i + 1], '=');    /* a b */
            if (i + 2 < len[p] && !isnt[x[i]] && isnt[x[i + 1]]
                && !isnt[x[i + 2]])
                setrel(x[i], x[i + 2], '=');    /* a B b */
            if (!isnt[x[i]] && isnt[x[i + 1]])  /* a B: a <. LEADING(B) */
                for (b = 2; b < nsym; b++)
                    if (lead[x[i + 1]] & bit(b))
                        setrel(x[i], b, '<');
            if (isnt[x[i]] && !isnt[x[i + 1]])  /* B a: TRAILING(B) .> a */
                for (b = 2; b < nsym; b++)
                    if (trail[x[i]] & bit(b))
                        setrel(b, x[i + 1], '>');
        }
    }
    for (b = 2; b < nsym; b++) {    /* $ <. LEADING(S), TRAILING(S) .> $ */
        if (lead[ntorder[0]] & bit(b))
            setrel(END, b, '<');
        if (trail[ntorder[0]] & bit(b))
            setrel(b, END, '>');
    }
}

int col[MAXSYM], ncol;              /* the terminals, then $ */

/* precedence functions: f(a) and g(b) are longest paths in a graph with
   an edge fa -> gb for a .> b and gb -> fa for a <. b; a = b joins fa
   and gb into one node. Node k < ncol is f, k >= ncol is g. */
int grp[2 * MAXSYM], memo[2 * MAXSYM], busy[2 * MAXSYM];
int edge[2 * MAXSYM][2 * MAXSYM];

int find(int k)
{
    return grp[k] == k ? k : (grp[k] = find(grp[k]));
}

int longest(int k)                  /* -1 if a cycle is reached */
{
    int j, d, best = 0;

    if (busy[k])
        return -1;
    if (memo[k] >= 0)
        return memo[k];
    busy[k] = 1;
    for (j = 0; j < 2 * ncol; j++)
        if (edge[k][j]) {
            d = longest(j);
            if (d < 0)
                return -1;
            if (d + 1 > best)
                best = d + 1;
        }
    busy[k] = 0;
    return memo[k] = best;
}

void functions(void)
{
    int i, j, a, b;

    for (i = 0; i < 2 * ncol; i++) {
        grp[i] = i;
        memo[i] = -1;
    }
    for (i = 0; i < ncol; i++)
        for (j = 0; j < ncol; j++)
            if (rel[col[i]][col[j]] == '=')
                grp[find(i)] = find(ncol + j);
    for (i = 0; i < ncol; i++)
        for (j = 0; j < ncol; j++) {
            a = find(i);
            b = find(ncol + j);
            if (rel[col[i]][col[j]] == '>')
                edge[a][b] = 1;
            else if (rel[col[i]][col[j]] == '<')
                edge[b][a] = 1;
        }
    for (i = 0; i < 2 * ncol; i++)
        if (longest(find(i)) < 0) {
            printf("no precedence functions: the graph has a cycle\n");
            return;
        }
    printf("%-4s", "");
    for (i = 0; i < ncol; i++)
        printf(" %4s", name[col[i]]);
    printf("\n%-4s", "f");
    for (i = 0; i < ncol; i++)
        printf(" %4d", longest(find(i)));
    printf("\n%-4s", "g");
    for (i = 0; i < ncol; i++)
        printf(" %4d", longest(find(ncol + i)));
    printf("\n");
}

int main(void)
{
    int i, j, s;
    char r[3];

    prog = "optable";
    readgrammar();
    checkoperator();
    leadtrail();
    for (i = 0; i < nnt; i++) {
        printf("LEADING(%s) = ", name[ntorder[i]]);
        printset(lead[ntorder[i]]);
        printf("TRAILING(%s) = ", name[ntorder[i]]);
        printset(trail[ntorder[i]]);
    }
    relations();
    for (s = 2; s < nsym; s++)
        if (!isnt[s])
            col[ncol++] = s;
    col[ncol++] = END;
    printf("%-4s", "");
    for (j = 0; j < ncol; j++)
        printf(" %3s", name[col[j]]);
    printf("\n");
    for (i = 0; i < ncol; i++) {
        printf("%3s ", name[col[i]]);  /* right-aligned: " $", not "$ " */
        for (j = 0; j < ncol; j++) {
            s = rel[col[i]][col[j]];
            strcpy(r, s == '<' ? "<." : s == '>' ? ".>" : s == '=' ? "="
                   : s == '*' ? "**" : "");
            printf(" %3s", r);
        }
        printf("\n");
    }
    if (clash) {
        for (i = 0; i < clash; i++)
            printf("%s\n", why[i]);
        printf("operator precedence: no, %d clash%s\n", clash,
               clash == 1 ? "" : "es");
        return 0;
    }
    printf("operator precedence: yes, one relation per entry\n");
    functions();
    return 0;
}
```

`leadtrail` is the two pairs of rules, applied to every production until a pass changes nothing, with the sets kept as bit sets from `first.h`. `relations` is the four relation rules: it walks every right side with `i` at each position and looks at `x[i]`, `x[i+1]` and, for the `a B b` case, `x[i+2]`. `setrel` records a relation and notices when an entry already holds a different one; such an entry is printed as `**` and listed under the table. The functions part is explained below. Build it and give it `expr.g`:

<!-- code: id=optable-run -->
```console
$ gcc -Wall -Wextra optable.c -o optable
$ ./optable < ../ch5/expr.g
LEADING(E) = { +, *, (, id }
TRAILING(E) = { +, *, ), id }
LEADING(T) = { *, (, id }
TRAILING(T) = { *, ), id }
LEADING(F) = { (, id }
TRAILING(F) = { ), id }
       +   *   (   )  id   $
  +   .>  <.  <.  .>  <.  .>
  *   .>  .>  <.  .>  <.  .>
  (   <.  <.  <.   =  <.
  )   .>  .>      .>      .>
 id   .>  .>      .>      .>
  $   <.  <.  <.      <.
operator precedence: yes, one relation per entry
        +    *    (    )   id    $
f       2    4    0    4    4    0
g       1    3    5    0    5    0
```

The six sets are the hand sets, and the table is the hand table with the columns in grammar order (as in `ll1`), `id` after the brackets. Every entry has one relation, so `expr.g` can be parsed this way.

Now the ambiguous grammar, which says nothing about precedence:

<!-- code: id=optable-amb -->
```console
$ ./optable < ../ch5/amb.g
LEADING(E) = { +, *, (, id }
TRAILING(E) = { +, *, ), id }
       +   *   (   )  id   $
  +   **  **  <.  .>  <.  .>
  *   **  **  <.  .>  <.  .>
  (   <.  <.  <.   =  <.
  )   .>  .>      .>      .>
 id   .>  .>      .>      .>
  $   <.  <.  <.      <.
clash: + .> and <. +
clash: + <. and .> *
clash: * .> and <. +
clash: * .> and <. *
operator precedence: no, 4 clashes
```

Four clashes, all among `+` and `*`. In `E -> E + E`, the `E` after `+` can begin with `+` (giving `+ <· +`) and the `E` before it can end with `+` (giving `+ ·> +`). The grammar leaves both readings of `id + id + id` open, which is section 5.2's ambiguity showing up as two relations in one entry. Choosing one relation in each clashing entry is choosing a precedence and an associativity; the table you built from the rules does exactly that, and Yacc's `%left` lines in section 8.3 do the same for an LR parser.

### Precedence functions

A table of n terminals has n² entries. The paper often asks you to replace it with two functions, f and g, one number for each terminal, such that for every pair:

- f(a) < g(b) whenever a <· b,
- f(a) = g(b) whenever a ≐ b,
- f(a) > g(b) whenever a ·> b.

Then the parser compares two numbers instead of looking in a table: f for the terminal on the stack, g for the one in the input. The table shrinks from n² to 2n entries. The price is the blanks: f(id) and g(id) are two numbers, so they compare *somehow*, and the error "`id` next to `id`" is no longer seen when it happens. It is found later, when the handle that results is not a right side.

The numbers come from a graph. Make a node f_a and a node g_a for every terminal a. Draw an edge from f_a to g_b if a ·> b, and from g_b to f_a if a <· b; if a ≐ b, merge f_a and g_b into one node. If the graph has a cycle, there are no functions. Otherwise f(a) is the length of the longest path starting at f_a, and g(b) the length of the longest path starting at g_b.

Work it for the four terminals `id`, `+`, `*`, `$` (the table above without the brackets). The edges are:

- from `id ·> +`, `id ·> *`, `id ·> $`: f_id → g_+, f_id → g_*, f_id → g_$;
- from `+ ·> +`, `+ ·> $`: f_+ → g_+, f_+ → g_$; from `+ <· *`, `+ <· id`: g_* → f_+, g_id → f_+;
- from `* ·> +`, `* ·> *`, `* ·> $`: f_* → g_+, f_* → g_*, f_* → g_$; from `* <· id`: g_id → f_*;
- from `$ <· +`, `$ <· *`, `$ <· id`: g_+ → f_$, g_* → f_$, g_id → f_$.

Now the longest paths, starting from the nodes with no edges out. f_$ and g_$ have none: 0. g_+ only reaches f_$: 1. f_+ reaches g_+ (1), so 2. g_* reaches f_+ (2), so 3. f_* reaches g_* (3), so 4. f_id also reaches g_*, so 4. g_id reaches f_* (4), so 5.

| | id | + | * | $ |
|---|---|---|---|---|
| f | 4 | 2 | 4 | 0 |
| g | 5 | 1 | 3 | 0 |

Check two entries against the table: `+ ·> +` needs f(+) > g(+), and 2 > 1; `+ <· *` needs f(+) < g(*), and 2 < 3. And look at `id` and `id`: f(id) = 4 < g(id) = 5 says `id <· id`, a relation the table never had. That is the lost blank.

`optable` builds the same graph (the `functions` part of the program). `find` keeps the merged nodes together, a small union-find; `longest` is a depth-first search that remembers each answer in `memo` and returns -1 if it ever comes back to a node it is still inside, `busy`, which means a cycle. Give it the grammar without brackets to check the hand values, then the full table:

<!-- code: id=optable-func -->
```console
$ printf 'E -> E + T | T\nT -> T * F | F\nF -> id\n' | ./optable | tail -3
        +    *   id    $
f       2    4    4    0
g       1    3    5    0
$ ./optable < ../ch5/expr.g | tail -3
        +    *    (    )   id    $
f       2    4    0    4    4    0
g       1    3    5    0    5    0
```

The first table is the hand one. In the second, `(` and `)` are added: f(`(`) = g(`)`) = 0 because `( ≐ )` merged their nodes, and g(`(`) = 5 is the largest g, because every terminal yields to an opening bracket.

### When the input is wrong

<!-- code: id=opp-errors exit=1 -->
```console
$ for s in 'i+*i' 'i i' '(i' ''; do echo "== $s"; echo "$s" | ./opp; done
== i+*i
     STACK                      INPUT   ACTION
  1  $              <.          i+*i$   shift i
  2  $i             .>           +*i$   reduce E->i
  3  $E             <.           +*i$   shift +
  4  $E+            <.            *i$   shift *
  5  $E+*           <.             i$   shift i
  6  $E+*i          .>              $   reduce E->i
  7  $E+*E          .>              $   error: no production E->*E
rejected
== i i
     STACK                      INPUT   ACTION
  1  $              <.            ii$   shift i
  2  $i                            i$   error: no relation i i
rejected
== (i
     STACK                      INPUT   ACTION
  1  $              <.            (i$   shift (
  2  $(             <.             i$   shift i
  3  $(i            .>              $   reduce E->i
  4  $(E                            $   error: no relation ( $
rejected
==
     STACK                      INPUT   ACTION
  1  $                              $   error: no expression
rejected
```

`i i` is caught at once, by the blank entry for `id` and `id`. `(i` is caught at the end, where `(` meets `$`, another blank. But `i+*i` is caught late, as in `sr`: the table has `+ <· *`, because `+` really can be followed by a `*` with an operand between them, and an operator-precedence parser does not see that the operand is missing, since it ignores nonterminals. It only finds out when the handle `*E` is not a right side.

That is the method's real weakness, along with one more: a token that is both unary and binary, like the minus in `-a - b`, needs two different rows and columns, and the scanner has to tell them apart before the parser sees them. The relations cannot do it, because they never look at what is on the left of an operator.

### When it goes wrong

**Rows and columns swapped.** The row is the terminal on the stack, the column the input token. Swap them and `+` and `*` change places in precedence: `i+i*i` reduces `E+E` first.<!-- risk: not run: no copy of opp.c with the table transposed was built; reasoning from the algorithm --> If a trace multiplies last, check the orientation before the entries.

**Comparing with the top of the stack instead of the topmost terminal.** At move 3 the top is `E`; the comparison uses `$`, the terminal under it. An `E` has no row in the table, and `opp.c` would report "no relation" at every operand.

**A handle that stops one terminal too early.** In `(E)`, walking down from `)` you meet `(`, with `( ≐ )`. ≐ means "same handle, keep going"; only <· stops the walk. Stopping at the ≐ gives the handle `)`, and the parser reports no production.

**Associativity the wrong way round.** Left-associative means θ ·> θ (the one on the stack is done first). Writing `+ <· +` makes `i+i+i` group as `i+(i+i)`: with `E+E` on the stack and the second `+` next, the trace shifts instead of reducing, and both additions are reduced only at the end.<!-- risk: not run: no copy of opp.c with + <. + was built; reasoning from the algorithm -->

**Functions from a table with a cycle.** If the graph has a cycle, no functions exist, and a "solution" found by guessing will break at least one relation. Check each of your numbers against every non-blank entry, or let `optable` say so.

### In one line each

1. An operator grammar has no ε-productions and no two nonterminals side by side.
2. The relations <·, ≐ and ·> between neighbouring terminals mark where handles start, continue and end.
3. Build the table from precedence and associativity, or from LEADING and TRAILING with the four rules; `optable` does the second and lists clashes.
4. The parser compares the topmost terminal with the input: shift on <· or ≐, reduce on ·> back to the nearest <·.
5. Precedence functions f and g replace the table by 2n numbers, if the relation graph has no cycle, at the cost of the blank entries' early errors.

**Check yourself**

<!-- q: idea=operator-precedence type=apply wrong="right-associative operators use the same relations as left-associative ones" -->
1. Add `^`, exponentiation, above `*` and right-associative. Write the new row and column for `^` by the precedence rules, then check them with `optable` on `E -> E + T | T`, `T -> T * P | P`, `P -> F ^ P | F`, `F -> ( E ) | id`.

<details>
<summary>Answer</summary>

Row `^` (on the stack): `^ ·> +`, `^ ·> *` (higher precedence); `^ <· ^` (right-associative: the one on the stack waits); `^ <· id`, `^ <· (`; `^ ·> )`, `^ ·> $`. Column `^` (in the input): `+ <· ^`, `* <· ^`, `^ <· ^`, `id ·> ^`, `) ·> ^`, `( <· ^`, `$ <· ^`.

<!-- code: id=optable-pow -->
```console
$ printf 'E -> E + T | T\nT -> T * P | P\nP -> F ^ P | F\nF -> ( E ) | id\n' | ./optable | sed -n '9,17p'
       +   *   ^   (   )  id   $
  +   .>  <.  <.  <.  .>  <.  .>
  *   .>  .>  <.  <.  .>  <.  .>
  ^   .>  .>  <.  <.  .>  <.  .>
  (   <.  <.  <.  <.   =  <.
  )   .>  .>  .>      .>      .>
 id   .>  .>  .>      .>      .>
  $   <.  <.  <.  <.      <.
operator precedence: yes, one relation per entry
```

The `^` row and column are as above, and `^ <· ^` is the right associativity: `P -> F ^ P` puts the recursion on the right.

</details>

<!-- q: idea=operator-precedence type=predict wrong="the parser compares the top of the stack, which may be E" -->
2. Write the moves of the operator-precedence parser on `i*(i+i)`, with the relation used at each move, before running `opp`. How many moves are there?

<details>
<summary>Answer</summary>

Fourteen: five shifts of operands and operators, `(` and `)` shifted too (seven shifts in all), six reductions (three `E->i`, `E->E+E`, `E->(E)`, `E->E*E`) and the accept. Inside the brackets the topmost terminal is `(` until the `+` is shifted; after `E->E+E`, the topmost is `(` and `( ≐ )` shifts the `)`.

<!-- code: id=opp-three -->
```console
$ echo 'i*(i+i)' | ./opp
     STACK                      INPUT   ACTION
  1  $              <.       i*(i+i)$   shift i
  2  $i             .>        *(i+i)$   reduce E->i
  3  $E             <.        *(i+i)$   shift *
  4  $E*            <.         (i+i)$   shift (
  5  $E*(           <.          i+i)$   shift i
  6  $E*(i          .>           +i)$   reduce E->i
  7  $E*(E          <.           +i)$   shift +
  8  $E*(E+         <.            i)$   shift i
  9  $E*(E+i        .>             )$   reduce E->i
 10  $E*(E+E        .>             )$   reduce E->E+E
 11  $E*(E          =              )$   shift )
 12  $E*(E)         .>              $   reduce E->(E)
 13  $E*E           .>              $   reduce E->E*E
 14  $E                             $   accept
```

</details>

<!-- q: idea=operator-precedence type=fix wrong="any set of numbers that fits the table's diagonal is a precedence function" -->
3. Find the fault. A student gives f(+) = 1, g(+) = 2, f(*) = 3, g(*) = 4 for the `+`, `*` part of the table. Which relation of the table is broken, and what does the parser do with `i+i+i` using these numbers?

<details>
<summary>Answer</summary>

f(+) = 1 < g(+) = 2 says `+ <· +`, but the table has `+ ·> +` (left-associative). `* ·> *` is broken the same way (3 < 4). With these functions the parser shifts the second `+` instead of reducing `E+E`, so `i+i+i` is grouped `i+(i+i)`: the parse is accepted but with the wrong tree.<!-- risk: not run: no program in this book parses with precedence functions; reasoning from the algorithm --> The hand values, f(+) = 2 > g(+) = 1 and f(*) = 4 > g(*) = 3, keep both operators left-associative.

</details>

## 7.3 LR(0) items, closure and goto: the canonical collection

### What the parser should remember

`sr.c` decided every move by looking at the top of its stack afresh, comparing it with every right side, and checking the lookahead against a FOLLOW set. That worked for the expression grammar, but it is guesswork dressed up: the parser had no idea *where it was* in the grammar. When `E + T` was on top with `*` next, it shifted only because `*` happened not to be in FOLLOW(E). It did not know the thing a person reading the line knows at that moment: "I am in the middle of `E + T`, and the `T` I have so far could still be the `T` of `T * F`."

The LR parsers, the family that Yacc builds, keep exactly that knowledge, written down in a precise form. At every moment the parser holds a set of "where I might be" notes, one note per production that could be in progress, each saying how much of the production has been seen. This section defines the notes, *items*, and builds every set of them that the parser can ever be in, for the expression grammar: twelve sets, called the *canonical collection*. Section 7.4 turns the collection into a parsing table, and section 7.7 runs the table.

### A board game whose squares are plans

Chapter 3's picture of a DFA was a board game: each square says where each card sends you. The sets of items form exactly such a board. Each square is one set of items, a plan of what the parser may be in the middle of; each card is a grammar symbol; and the square you land on after a card is the plan after that symbol has been seen. The parser plays the game with the symbols on its stack, from the bottom up, and the square it ends on says what to do next: shift, or reduce by a particular production.

The picture is true in two ways and breaks in two. True: it is a real DFA, with one start square and one move per square and card. Broken, first: some cards are nonterminals, which never arrive from the input; the parser plays an `E` card only after it has *made* an `E` by a reduction. Broken, second: after a reduction, the parser does not continue from the square it was on. It goes back to the square it was on *before* the right side was read and plays the left side from there. To be able to go back, the LR parser writes the square's number on every plate of its stack. This is where the book's stack of plates stops being the whole truth: an LR parser's plates carry states, not just symbols. Section 7.7 builds that stack.

### Items

<!-- teaches: lr0-item = LR(0) item, augmented grammar, kernel item, viable prefix -->
An *LR(0) item* is a production with a dot somewhere in its right side. The production `E -> E + T` gives four items:

```text
E -> . E + T      nothing of the right side seen yet; hoping for E + T
E -> E . + T      an E has been seen (reduced); hoping for + T
E -> E + . T      E + seen; hoping for a T
E -> E + T .      all seen: E + T is on top of the stack, ready to reduce
```

The symbols before the dot are on the stack already; the symbols after it are still to come. An item with the dot at the end is *complete*: its right side has been seen in full, and it is a candidate for reduction. The production `A -> ε` gives one item, `A -> .`, which is complete at once. The "(0)" means that an item carries no lookahead; section 7.5 adds one.

Before building anything, the grammar gets one extra production. The *augmented grammar* adds a new start symbol S' and the production S' -> S. For the expression grammar that is `E' -> E`, numbered 0, with the rest numbered 1 to 6 in the order of `expr.g`:

```text
0  E' -> E
1  E -> E + T        2  E -> T
3  T -> T * F        4  T -> F
5  F -> ( E )        6  F -> id
```

Why bother? Because `E` also appears inside `F -> ( E )`, so reducing something to `E` does not by itself mean the parse is over. With S', the parse is over exactly when the parser could reduce by `E' -> E` with `$` next, and that production is used nowhere else. These numbers are the ones the exam table uses: "r2" will mean "reduce by production 2, E -> T".

### Closure: everything that could start here

The parser begins hoping for an `E`: that is the item `E' -> . E`. But if it hopes for an `E`, it is also at the very beginning of every way of writing an `E`: of `E + T` and of `T`. And being at the start of a `T`, it is at the start of `T * F` and `F`; and so of `( E )` and `id`. Those are all items with the dot at the left end, and they belong in the same set.

<!-- teaches: closure = closure of a set of items, closure item -->
The *closure* of a set of items I is I with this rule applied until nothing new is added: if an item `A -> α . B β` is in the set, with a nonterminal B right after the dot, then add `B -> . γ` for every production `B -> γ`. Items with a terminal after the dot, and complete items, add nothing.

Here is closure({ E' -> . E }) by hand, one item at a time, in the order they join:

| Item | Symbol after the dot | Adds |
|---|---|---|
| `E' -> . E` | E | `E -> . E + T`, `E -> . T` |
| `E -> . E + T` | E | nothing new: E's items are in already |
| `E -> . T` | T | `T -> . T * F`, `T -> . F` |
| `T -> . T * F` | T | nothing new |
| `T -> . F` | F | `F -> . ( E )`, `F -> . id` |
| `F -> . ( E )` | ( | nothing: a terminal |
| `F -> . id` | id | nothing: a terminal |

Seven items. That set is I0, the start square. The first item, `E' -> . E`, is the one you started with; the other six were added by the closure. Items of the first kind, S' -> . S together with every item whose dot is not at the left end, are called *kernel items*; items with the dot at the left end, added by a closure, are *closure items*. Since the closure items can always be worked out from the kernel, a set is fully described by its kernel, and two sets with the same kernel are the same set.

### Goto: moving over one symbol

<!-- teaches: goto-function = goto function, canonical collection of LR(0) items, LR(0) automaton -->
The *goto function* says where a card sends you. goto(I, X), for a set I and a grammar symbol X, is found in two steps: take every item in I with X right after the dot and move the dot over X; then take the closure of the result.

goto(I0, E): two items of I0 have `E` after the dot, `E' -> . E` and `E -> . E + T`. Moving the dot gives `E' -> E .` and `E -> E . + T`. The first is complete; the second has a terminal after the dot; the closure adds nothing. That is I1.

goto(I0, T): `E -> . T` and `T -> . T * F` give `E -> T .` and `T -> T . * F`. Closure adds nothing. That is I2.

goto(I0, `(`): only `F -> . ( E )` has `(` after the dot, giving `F -> ( . E )`. Now there is a nonterminal after the dot, and the closure adds the same six items it added to I0: every way to begin an `E`. That is I4: one kernel item, six closure items. It looks a lot like I0, but its kernel is different, so it is a different set: inside brackets, a finished `E` must be followed by `)`, not by the end.

Symbols with no item to move over give nothing: goto(I0, `+`) is empty, a square with no exit for that card. An empty goto will become an error entry in the table.

The *canonical collection of LR(0) items* is every set the parser can reach from I0. Build it with a list: start with I0; take the sets one at a time in the order they joined the list; for each one, compute goto on each symbol that appears after a dot, in the order those symbols appear in the set's items; if the result is not empty and not already on the list, add it with the next number. Stop when every set on the list has been done. The sets with the goto function between them form a DFA, the *LR(0) automaton*.

### The expression grammar's collection, by hand

Here is the whole run. Each row is one goto that was computed; "new" means a set that joined the list then.

| From | Symbol | Kernel of the result | Set |
|---|---|---|---|
| I0 | E | `E' -> E .`, `E -> E . + T` | I1, new |
| I0 | T | `E -> T .`, `T -> T . * F` | I2, new |
| I0 | F | `T -> F .` | I3, new |
| I0 | ( | `F -> ( . E )` | I4, new |
| I0 | id | `F -> id .` | I5, new |
| I1 | + | `E -> E + . T` | I6, new |
| I2 | * | `T -> T * . F` | I7, new |
| I4 | E | `F -> ( E . )`, `E -> E . + T` | I8, new |
| I4 | T, F, (, id | same kernels as from I0 | I2, I3, I4, I5 |
| I6 | T | `E -> E + T .`, `T -> T . * F` | I9, new |
| I6 | F, (, id | | I3, I4, I5 |
| I7 | F | `T -> T * F .` | I10, new |
| I7 | (, id | | I4, I5 |
| I8 | ) | `F -> ( E ) .` | I11, new |
| I8 | + | `E -> E + . T` | I6 |
| I9 | * | `T -> T * . F` | I7 |

I3, I5, I10 and I11 hold only a complete item, so nothing goes out of them. The list stops at twelve sets, I0 to I11. In full, with the closure items:

```text
I0:  E' -> . E          I4:  F -> ( . E )        I7:  T -> T * . F
     E  -> . E + T           E -> . E + T             F -> . ( E )
     E  -> . T               E -> . T                 F -> . id
     T  -> . T * F           T -> . T * F
     T  -> . F               T -> . F            I8:  F -> ( E . )
     F  -> . ( E )           F -> . ( E )             E -> E . + T
     F  -> . id              F -> . id
                                                 I9:  E -> E + T .
I1:  E' -> E .          I5:  F -> id .                T -> T . * F
     E  -> E . + T
                        I6:  E -> E + . T        I10: T -> T * F .
I2:  E -> T .                T -> . T * F
     T -> T . * F            T -> . F            I11: F -> ( E ) .
                             F -> . ( E )
I3:  T -> F .                F -> . id
```

Figure 7.3a draws the automaton. To keep it readable, each square shows only its kernel, and only the eleven arrows that first reached each set are drawn; the other eleven gotos are the "same kernels" rows of the table.

<!-- fig: id=lr0-automaton
title: The LR(0) automaton of the expression grammar
kind: tree
box I0: I0\nE' -> .E
box I1: I1\nE' -> E.\nE -> E.+T
box I2: I2\nE -> T.\nT -> T.*F
box I3: I3\nT -> F.
box I4: I4\nF -> (.E)
box I5: I5\nF -> id.
box I6: I6\nE -> E+.T
box I7: I7\nT -> T*.F
box I8: I8\nF -> (E.)\nE -> E.+T
box I9: I9\nE -> E+T.\nT -> T.*F
box I10: I10\nT -> T*F.
box I11: I11\nF -> (E).
hot: I0
I0 -> I1: E
I0 -> I2: T
I0 -> I3: F
I0 -> I4: (
I0 -> I5: id
I1 -> I6: +
I2 -> I7: *
I4 -> I8: E
I6 -> I9: T
I7 -> I10: F
I8 -> I11: )
-->
![Tree, top to bottom. I0 E' -> .E leads to I1 E' -> E. E -> E.+T (E). I0 E' -> .E leads to I2 E -> T. T -> T.*F (T). I0 E' -> .E leads to I3 T -> F. (F). I0 E' -> .E leads to I4 F -> (.E) ((). I0 E' -> .E leads to I5 F -> id. (id). I1 E' -> E. E -> E.+T leads to I6 E -> E+.T (+). I2 E -> T. T -> T.*F leads to I7 T -> T*.F (*). I4 F -> (.E) leads to I8 F -> (E.) E -> E.+T (E). I6 E -> E+.T leads to I9 E -> E+T. T -> T.*F (T). I7 T -> T*.F leads to I10 T -> T*F. (F). I8 F -> (E.) E -> E.+T leads to I11 F -> (E). ()). I0 E' -> .E is highlighted.](fig/07-bottom-up-parsers/lr0-automaton.svg)

*Figure 7.3a: The canonical collection of LR(0) items for the expression grammar, kernels only. Each arrow is the goto that first made a set. The gotos not drawn lead back to sets already made: from I4 on T, F, ( and id to I2, I3, I4 and I5; from I6 on F, ( and id to I3, I4 and I5; from I7 on ( and id to I4 and I5; from I8 on + to I6; from I9 on * to I7.*

### What a set says to the parser

Read the sets as instructions. An item with a terminal after the dot, like `T -> T . * F` in I2, says "if `*` comes next, shift it and go to I7". A complete item, like `E -> T .` in I2, says "a handle may be on top: `T`, reducible to `E`". A nonterminal after the dot, like the `T` in `E -> E + . T` of I6, says "after a `T` has been made here, go to I9".

Now look at I2 again. It holds both kinds: shift on `*`, or reduce by `E -> T`. That is precisely the decision `sr.c` made at move 9 with `E + T` on top, and the set does not settle it by itself. I1 has the same mixture (accept, or shift `+`), and so does I9 (reduce by `E -> E + T`, or shift `*`). A grammar whose sets never mix a complete item with anything else can be parsed with the sets alone; it is called LR(0), and almost no useful grammar is. The expression grammar needs one token of lookahead in I1, I2 and I9, and section 7.4 supplies it from FOLLOW, which is the rule `sr.c` used, now applied in the right place.

The paths through the automaton have a name the paper uses. Any sequence of symbols that leads from I0 to some set, such as `E + T` (I0, I1, I6, I9) or `( ( id` (I0, I4, I4, I5), is a *viable prefix*: a string that can be on a shift-reduce parser's stack in some correct parse, with nothing on it that runs past a handle. `E + T *` is one; `E + T + ` is not, because no correct parse ever has `E + T` on the stack with a `+` shifted over it; that `E + T` should have been reduced first, and the automaton has no arrow for `+` out of I9. The set a viable prefix leads to holds exactly the items that are *valid* for it, the productions the parser can be in the middle of after reading that prefix. That is the knowledge `sr.c` lacked.

### lr0.h: the collection in C

The construction is long by hand, so check it by program, as always. The code goes in a header, `lr0.h`, because sections 7.4, 7.5 and 7.7 build on it. It needs `grammar.h` for the grammar and `follow.h` (with `first.h`) for the sets; you copied all three in section 7.2. Save this as `lr0.h`:

<!-- code: id=lr0-h mode=fragment file=lr0.h -->
```c
/* lr0.h: the augmented grammar and its canonical collection of LR(0)
   items. Needs follow.h (with grammar.h and first.h). */
#include "follow.h"

#define MAXST 100                   /* states */
#define MAXI  64                    /* items in one state */

/* an item is a production and a dot position, kept in one int */
#define ITEM(p, d) ((p) * (MAXR + 1) + (d))
#define PROD(it)   ((it) / (MAXR + 1))
#define DOT(it)    ((it) % (MAXR + 1))

typedef struct {
    int n, nk;                      /* items; the first nk are the kernel */
    int item[MAXI];
} state;

state st[MAXST];                    /* the collection: I0, I1, ... */
int nst;
int go[MAXST][MAXSYM];              /* goto(Ii, X), or -1 */

void augment(void)                  /* S' -> S becomes production 0 */
{
    int s = ntorder[0], a, k, p;

    a = primed(s);                  /* S', placed after S ... */
    for (k = 0; ntorder[k] != a; k++)
        ;
    for (; k > 0; k--)              /* ... moved to the front */
        ntorder[k] = ntorder[k - 1];
    ntorder[0] = a;
    put(a, &s, 1, NULL, 0);
    for (p = 0; p < np; p++)
        put(lhs[p], rhs[p], len[p], NULL, 0);
    install();
}

int after(int it)                   /* the symbol after the dot, or -1 */
{
    int p = PROD(it), d = DOT(it);

    return d < len[p] ? rhs[p][d] : -1;
}

void additem(state *s, int it)
{
    int k;

    for (k = 0; k < s->n; k++)
        if (s->item[k] == it)
            return;
    if (s->n == MAXI)
        die("more than %d items in a state", MAXI);
    s->item[s->n++] = it;
}

void closure(state *s)              /* s->n grows while the loop runs */
{
    int k, b, p;

    for (k = 0; k < s->n; k++) {
        b = after(s->item[k]);
        if (b < 0 || !isnt[b])
            continue;               /* dot at the end, or before a terminal */
        for (p = 0; p < np; p++)
            if (lhs[p] == b)
                additem(s, ITEM(p, 0));     /* B -> . gamma */
    }
}

void gotoset(const state *from, int x, state *to)
{
    int k;

    to->n = 0;
    for (k = 0; k < from->n; k++)
        if (after(from->item[k]) == x)
            additem(to, from->item[k] + 1); /* move the dot over x */
    to->nk = to->n;
    closure(to);
}

int samekernel(const state *a, const state *b)
{
    int i, j;

    if (a->nk != b->nk)
        return 0;
    for (i = 0; i < a->nk; i++) {
        for (j = 0; j < b->nk && b->item[j] != a->item[i]; j++)
            ;
        if (j == b->nk)
            return 0;
    }
    return 1;
}

int addstate(const state *s)        /* its number, new or old */
{
    int i;

    for (i = 0; i < nst; i++)
        if (samekernel(&st[i], s))
            return i;
    if (nst == MAXST)
        die("more than %d states", MAXST);
    st[nst] = *s;
    return nst++;
}

void collection(void)
{
    state t;
    int i, k, x;

    memset(go, -1, sizeof go);
    t.n = 0;
    additem(&t, ITEM(0, 0));        /* I0 = closure({ S' -> . S }) */
    t.nk = 1;
    closure(&t);
    addstate(&t);
    for (i = 0; i < nst; i++)       /* nst grows while the loop runs */
        for (k = 0; k < st[i].n; k++) {
            x = after(st[i].item[k]);
            if (x < 0 || go[i][x] >= 0)
                continue;           /* nothing to move over, or done */
            gotoset(&st[i], x, &t);
            go[i][x] = addstate(&t);
        }
}

char *itemtext(int it)              /* "E -> E . + T" */
{
    static char buf[(MAXR + 3) * (MAXLEN + 1) + 8];
    int p = PROD(it), d = DOT(it), k;

    sprintf(buf, "%s ->", name[lhs[p]]);
    for (k = 0; k <= len[p]; k++) {
        if (k == d)
            strcat(buf, " .");
        if (k < len[p]) {
            strcat(buf, " ");
            strcat(buf, name[rhs[p][k]]);
        }
    }
    return buf;
}
```

An item is two small numbers, a production and a dot position, and `ITEM` packs them into one `int`: the production times `MAXR + 1`, plus the dot. Since a right side has at most `MAXR` symbols, the dot is between 0 and `MAXR`, and `PROD` and `DOT` take the int apart again with `/` and `%`. Moving the dot one place right is then just adding 1, which is what `gotoset` does with `from->item[k] + 1`.

`augment` makes S' with `primed` from `grammar.h`, the function sections 5.3 and 5.4 used to make `E'`, and moves it to the front of `ntorder` so that it is the start symbol. It then rebuilds the productions with `put` and `install`, S' -> S first, so that the grammar's own productions are numbered from 1, as in the exam.

A state keeps its items in an array, kernel first: `nk` says how many of them are the kernel. `closure` is the hand table above in a loop. It walks the items with `k` while `additem` may append new ones at the end, so `s->n` grows as the loop runs, and every item added is itself visited later. That is how one pass reaches `F -> . id` from `E' -> . E` through `E` and `T`: no repeat-until-no-change is needed, because nothing visited ever changes. `additem` refuses duplicates, which is what stops the loop.

`gotoset` is the two steps of goto: move the dot over `x` in every item that has `x` after it, call that the kernel, close it. `samekernel` compares two kernels as sets, in any order, and `addstate` returns the number of the state with the same kernel if there is one, or adds the new one. `collection` is the list method: `i` walks the states in the order they were added while `nst` grows, and within each state the items are visited in order, so the symbols are tried in the order they appear after dots, which is what gives the Dragon book's numbering. `go[i][x]` is filled the first time a symbol is met in a state, and the `go[i][x] >= 0` test skips it the second time (the second `E` of I0, in `E -> . E + T`).

Now `lr0.c`, which prints the augmented grammar and every state with its gotos. Save it as `lr0.c`:

<!-- code: id=lr0 mode=compile -->
```c
/* lr0.c: the canonical collection of LR(0) items and the goto function.
   Reads a grammar on standard input (grammar.h's form). */
#include "lr0.h"

int main(void)
{
    int i, k, x, p, shown[MAXSYM];

    prog = "lr0";
    readgrammar();
    augment();
    printf("augmented grammar:\n");
    for (p = 0; p < np; p++)
        printf("%3d  %s\n", p, prodtext(p));
    collection();
    for (i = 0; i < nst; i++) {
        printf("\nI%d:\n", i);
        for (k = 0; k < st[i].n; k++)
            printf("  %s %s\n", k < st[i].nk ? " " : "+", itemtext(st[i].item[k]));
        memset(shown, 0, sizeof shown);
        for (k = 0; k < st[i].n; k++) {     /* in the order of the items */
            x = after(st[i].item[k]);
            if (x >= 0 && !shown[x]) {
                printf("  goto(I%d, %s) = I%d\n", i, name[x], go[i][x]);
                shown[x] = 1;
            }
        }
    }
    printf("\n%d states\n", nst);
    return 0;
}
```

A `+` in front of an item marks a closure item; kernel items have a blank there. Build it and run it on the expression grammar from chapter 5:

<!-- code: id=lr0-run -->
```console
$ gcc -Wall -Wextra lr0.c -o lr0
$ ./lr0 < ../ch5/expr.g | sed '/^I5:/,$d'
augmented grammar:
  0  E' -> E
  1  E -> E + T
  2  E -> T
  3  T -> T * F
  4  T -> F
  5  F -> ( E )
  6  F -> id

I0:
    E' -> . E
  + E -> . E + T
  + E -> . T
  + T -> . T * F
  + T -> . F
  + F -> . ( E )
  + F -> . id
  goto(I0, E) = I1
  goto(I0, T) = I2
  goto(I0, F) = I3
  goto(I0, () = I4
  goto(I0, id) = I5

I1:
    E' -> E .
    E -> E . + T
  goto(I1, +) = I6

I2:
    E -> T .
    T -> T . * F
  goto(I2, *) = I7

I3:
    T -> F .

I4:
    F -> ( . E )
  + E -> . E + T
  + E -> . T
  + T -> . T * F
  + T -> . F
  + F -> . ( E )
  + F -> . id
  goto(I4, E) = I8
  goto(I4, T) = I2
  goto(I4, F) = I3
  goto(I4, () = I4
  goto(I4, id) = I5
```

That is the grammar and the first five states. The rest:

<!-- code: id=lr0-run2 -->
```console
$ ./lr0 < ../ch5/expr.g | sed -n '/^I5:/,$p'
I5:
    F -> id .

I6:
    E -> E + . T
  + T -> . T * F
  + T -> . F
  + F -> . ( E )
  + F -> . id
  goto(I6, T) = I9
  goto(I6, F) = I3
  goto(I6, () = I4
  goto(I6, id) = I5

I7:
    T -> T * . F
  + F -> . ( E )
  + F -> . id
  goto(I7, F) = I10
  goto(I7, () = I4
  goto(I7, id) = I5

I8:
    F -> ( E . )
    E -> E . + T
  goto(I8, )) = I11
  goto(I8, +) = I6

I9:
    E -> E + T .
    T -> T . * F
  goto(I9, *) = I7

I10:
    T -> T * F .

I11:
    F -> ( E ) .

12 states
```

Twelve states, numbered as in the hand table, with the same items in the same order, and every goto of the table, including the eleven that lead back to old states. If your hand collection numbers the states differently, it is not wrong: the numbering depends on the order in which you tried the symbols. But the exam expects the Dragon numbering, which is this one, and following the list method strictly (states in order, symbols in the order of the items) gets it every time.

### When it goes wrong

**A closure that stops after one level.** Writing I0 as `E' -> . E`, `E -> . E + T`, `E -> . T` and stopping. The rule applies to the items it adds, too: `E -> . T` has `T` after the dot, which brings in T's items, and `T -> . F` brings in F's. A closure is finished only when every item's next symbol has been looked at.

**No closure after goto.** goto(I1, +) moves the dot to give `E -> E + . T`, and then the closure must add T's and F's items. Leaving them out makes I6 a dead end on `id`, and the table built from it rejects `id + id`.

**Moving the dot in every item.** goto(I, X) only takes the items with X *right after the dot*. Taking `E -> . T` into goto(I0, E) because "it is an E production" gives a wrong kernel, and the collection grows sets that do not exist.

**The same set twice.** I4's gotos on `T`, `F`, `(` and `id` give kernels already met from I0. If you do not compare kernels, you number them I12 to I15 and the collection never ends. Compare the kernel, not the whole set and not the order of the items.

**Forgetting to augment.** Without `E' -> E`, there is no item that says "the whole input is an `E`", and I1 cannot tell a finished parse from an `E` that is about to get a `+`. Every LR construction starts by adding S' -> S.

### In one line each

1. An LR(0) item is a production with a dot: before the dot is on the stack, after it is still to come.
2. The augmented grammar adds S' -> S so that the parse ends exactly at the reduction by production 0.
3. closure(I) adds B -> . γ for every B right after a dot, until nothing new comes in.
4. goto(I, X) moves the dot over X in the items that allow it, then takes the closure.
5. The canonical collection is every set reachable from I0 = closure({ S' -> . S }); for the expression grammar it is twelve sets, I0 to I11, and `lr0` prints it.

**Check yourself**

<!-- q: idea=closure type=apply wrong="closure only adds items for the nonterminal in the kernel" -->
1. Work out closure({ E -> E + . T }) by hand. Which set of the collection is it, and how many items does it have?

<details>
<summary>Answer</summary>

`T` is after the dot: add `T -> . T * F` and `T -> . F`. The first has `T` after the dot again (nothing new); the second has `F`: add `F -> . ( E )` and `F -> . id`, which have terminals after the dot. Five items in all: I6. The slip is to stop after T's items, which leaves I6 with no way to shift `id`.

</details>

<!-- q: idea=goto-function type=predict wrong="every goto makes a new set" -->
2. Before looking at `lr0`'s output, say what goto(I8, +) and goto(I9, *) are, and why they are not new sets.

<details>
<summary>Answer</summary>

goto(I8, +): only `E -> E . + T` has `+` after the dot; moving it gives the kernel `E -> E + . T`, the kernel of I6, so it is I6. goto(I9, *): only `T -> T . * F` qualifies; `T -> T * . F` is I7's kernel, so it is I7. A goto produces a new set only when its kernel has never been seen; here the same "middle of `E + T`" or "middle of `T * F`" situation was reached before, from I1 and I2.

</details>

<!-- q: idea=lr0-item type=apply wrong="the list grammar needs its left recursion removed first" -->
3. Build the canonical LR(0) collection for the list grammar of section 6.6 as it was first written, `S -> ( L ) | a`, `L -> L , S | S`, with its left recursion left in. How many sets are there? Which ones mix a complete item with a shift? Check with `lr0`.

<details>
<summary>Answer</summary>

Nine sets, I0 to I8. I0 = { S' -> . S, S -> . ( L ), S -> . a }; I1 = { S' -> S . }; I2 = goto(I0, `(`) = { S -> ( . L ), L -> . L , S, L -> . S, S -> . ( L ), S -> . a }; I3 = { S -> a . }; I4 = goto(I2, L) = { S -> ( L . ), L -> L . , S }; I5 = { L -> S . }; I6 = { S -> ( L ) . }; I7 = goto(I4, `,`) = { L -> L , . S, S -> . ( L ), S -> . a }; I8 = { L -> L , S . }. No set mixes a complete item with anything else, so this grammar is even LR(0). Left recursion is no trouble for a bottom-up parser: `L -> L , S` only makes I2 hold both `L -> . L , S` and `L -> . S`, and nothing loops.

<!-- code: id=lr0-list -->
```console
$ printf 'S -> ( L ) | a\nL -> L , S | S\n' | ./lr0 | tail -24
I4:
    S -> ( L . )
    L -> L . , S
  goto(I4, )) = I6
  goto(I4, ,) = I7

I5:
    L -> S .

I6:
    S -> ( L ) .

I7:
    L -> L , . S
  + S -> . ( L )
  + S -> . a
  goto(I7, S) = I8
  goto(I7, () = I2
  goto(I7, a) = I3

I8:
    L -> L , S .

9 states
```

</details>

## 7.4 The SLR(1) parsing table

### From plans to orders

The collection of section 7.3 says, for every state, what the parser *might* do: I2 says "shift `*`, or reduce by E -> T". A parser cannot run on "might". It needs one order for each state and each next token, and that is what a parsing table holds. The written paper asks for it in almost every year's question on bottom-up parsing, "construct the SLR parsing table for the grammar", and section 7.7's driver runs on it.

This section builds the table for the expression grammar by hand, writes the program that builds it for any grammar, and then shows a grammar for which it fails: not because the grammar is ambiguous, but because the method's lookahead is too rough.

### Two tables in one

<!-- teaches: slr-table = LR parsing table, ACTION, GOTO, SLR(1) table, SLR(1) grammar -->
An *LR parsing table* has one row per state and two parts.

- **ACTION**, with one column per terminal and one for `$`. ACTION[i, a] tells the parser what to do in state i when the next token is a: *shift j* (written `sj`: shift a and go to state j), *reduce by production n* (written `rn`), *accept* (`acc`), or *error* (a blank).
- **GOTO**, with one column per nonterminal (except S'). GOTO[i, A] = j says that after a reduction to A, uncovering state i, the parser goes to state j.

GOTO is the goto function on nonterminals, copied straight from the collection. ACTION is where the decisions are made. The *SLR(1)* way of filling it ("simple LR, one token of lookahead") uses the collection and the FOLLOW sets of section 5.6:

1. If `A -> α . a β` is in Ii, with a terminal a after the dot, and goto(Ii, a) = Ij: ACTION[i, a] = shift j.
2. If `A -> α .` is in Ii, complete, and A is not S': ACTION[i, a] = reduce by A -> α, for every a in FOLLOW(A) (including `$` if it is there).
3. If `S' -> S .` is in Ii: ACTION[i, $] = accept.
4. If goto(Ii, A) = Ij for a nonterminal A: GOTO[i, A] = j.
5. Every entry not filled by these rules is an error.

If any ACTION entry is given two different actions, the method has failed for this grammar, and the grammar is not *SLR(1)*. Section 7.6 is about those collisions; for now, look at why rule 2 is the interesting one.

Rule 2 is where FOLLOW comes in, and it is the rule `sr.c` used: reduce A -> α only if the next token can follow A. But `sr.c` applied it to any right side it found on top of its stack. Here it is applied only to the complete items of the current state, the productions the parser can actually be finishing at this point, which the collection has worked out. The difference matters more in a moment.

### The expression grammar's table, by hand

You need the twelve sets from section 7.3, the production numbers (1 E -> E + T, 2 E -> T, 3 T -> T * F, 4 T -> F, 5 F -> ( E ), 6 F -> id), and the FOLLOW sets of the left-recursive grammar, which section 5.6 worked out by hand for exactly this use:

```text
FOLLOW(E) = { +, ), $ }      FOLLOW(T) = { +, *, ), $ }      FOLLOW(F) = { +, *, ), $ }
```

Go through the sets one at a time, and through each set's items one at a time.

**I0.** `F -> . ( E )` has the terminal `(` after the dot and goto(I0, `(`) = I4: ACTION[0, (] = s4. `F -> . id`, goto(I0, id) = I5: ACTION[0, id] = s5. The items with E, T, F after the dot give no actions; the gotos give GOTO[0, E] = 1, GOTO[0, T] = 2, GOTO[0, F] = 3. No complete items.

**I1.** `E' -> E .` is the accept item: ACTION[1, $] = acc. `E -> E . + T`, goto on `+` is I6: ACTION[1, +] = s6.

**I2.** `E -> T .` is complete: reduce by production 2 under every token of FOLLOW(E), so ACTION[2, +] = ACTION[2, )] = ACTION[2, $] = r2. `T -> T . * F`: ACTION[2, *] = s7. The reduce went under `+`, `)` and `$`, the shift under `*`; they do not meet, because `*` is not in FOLLOW(E). This is the entry section 7.3 left open, and FOLLOW settled it.

**I3.** `T -> F .`: r4 under `+`, `*`, `)`, `$`.

**I4.** The same as I0 for the shifts (s4, s5), with GOTO[4, E] = 8, GOTO[4, T] = 2, GOTO[4, F] = 3.

**I5.** `F -> id .`: r6 under `+`, `*`, `)`, `$`.

**I6.** Shifts s4 and s5 from the F items; GOTO[6, T] = 9, GOTO[6, F] = 3.

**I7.** s4, s5; GOTO[7, F] = 10.

**I8.** `F -> ( E . )`: ACTION[8, )] = s11. `E -> E . + T`: ACTION[8, +] = s6.

**I9.** `E -> E + T .`: r1 under `+`, `)`, `$`. `T -> T . * F`: s7 under `*`. As in I2, FOLLOW(E) keeps them apart.

**I10.** `T -> T * F .`: r3 under `+`, `*`, `)`, `$`.

**I11.** `F -> ( E ) .`: r5 under `+`, `*`, `)`, `$`.

The table, columns in the Dragon book's order:

| State | id | + | * | ( | ) | $ | E | T | F |
|---|---|---|---|---|---|---|---|---|---|
| 0 | s5 | | | s4 | | | 1 | 2 | 3 |
| 1 | | s6 | | | | acc | | | |
| 2 | | r2 | s7 | | r2 | r2 | | | |
| 3 | | r4 | r4 | | r4 | r4 | | | |
| 4 | s5 | | | s4 | | | 8 | 2 | 3 |
| 5 | | r6 | r6 | | r6 | r6 | | | |
| 6 | s5 | | | s4 | | | | 9 | 3 |
| 7 | s5 | | | s4 | | | | | 10 |
| 8 | | s6 | | | s11 | | | | |
| 9 | | r1 | s7 | | r1 | r1 | | | |
| 10 | | r3 | r3 | | r3 | r3 | | | |
| 11 | | r5 | r5 | | r5 | r5 | | | |

Every entry has at most one action, so the expression grammar is SLR(1). Notice the patterns, because they let you check a hand table quickly. The rows of the states with only a complete item (3, 5, 10, 11) are a reduce under every token of one FOLLOW set and nothing else. The shift entries are under `id` and `(` in exactly the states whose closure brought in F's items (0, 4, 6, 7). And a shift number in a column is always the same state: every `s5` is under `id`, because every goto on `id` leads to I5.

### Why not reduce everywhere?

Rule 2 could have been simpler: "if a set has a complete item, reduce by it, whatever the next token". That is the *LR(0)* table, and it needs no FOLLOW sets. For the expression grammar it fails at once. Row 2 would have r2 in every column, `*` included, where rule 1 has already put s7. The state would not know whether `id + id * id` with `id + T` parsed so far should reduce the `T` or wait for the `*`. FOLLOW(E) is the one fact that settles it, which is why this grammar is SLR(1) but not LR(0).

### slr.h: the table in C

The program goes in a header, `slr.h`, because section 7.5's tables and section 7.7's driver use the same table and the same printing. It includes `lr0.h` for the collection. Save this as `slr.h`:

<!-- code: id=slr-h mode=fragment file=slr.h -->
```c
/* slr.h: an LR parsing table (ACTION and GOTO), its printing, and the
   SLR(1) way of filling it from lr0.h's collection and FOLLOW. */
#include "lr0.h"

#define MAXC 4                      /* actions kept per entry */
enum { SHIFT = 1, REDUCE, ACCEPT }; /* an action is kind + 4 * number */
#define KIND(a) ((a) & 3)
#define NUM(a)  ((a) >> 2)

int act[MAXST][MAXSYM][MAXC];       /* ACTION[i, a]: up to MAXC actions */
int nact[MAXST][MAXSYM];            /* how many are in ACTION[i, a] */
int gt[MAXST][MAXSYM];              /* GOTO[i, A], or -1 */
int nrow;                           /* rows in the table */
char rowname[MAXST][24];            /* "5", or "36" for merged states */
int conflicts;                      /* entries with two or more actions */

void addact(int i, int t, int a)
{
    int k;

    for (k = 0; k < nact[i][t]; k++)
        if (act[i][t][k] == a)
            return;                 /* there already */
    if (nact[i][t] == MAXC)
        die("more than %d actions in one entry", MAXC);
    for (k = nact[i][t]; k > 0 && act[i][t][k - 1] > a; k--)
        act[i][t][k] = act[i][t][k - 1];    /* keep them in order */
    act[i][t][k] = a;
    nact[i][t]++;
    if (nact[i][t] == 2)
        conflicts++;
}

void buildslr(void)
{
    int i, k, it, x, p, t;

    computefirst(0);
    computefollow();
    memset(gt, -1, sizeof gt);
    for (i = 0; i < nst; i++) {
        sprintf(rowname[i], "%d", i);
        for (x = 0; x < nsym; x++)
            if (isnt[x])
                gt[i][x] = go[i][x];
        for (k = 0; k < st[i].n; k++) {
            it = st[i].item[k];
            x = after(it);
            p = PROD(it);
            if (x >= 0 && !isnt[x])         /* A -> alpha . a beta */
                addact(i, x, SHIFT + 4 * go[i][x]);
            else if (x < 0 && p == 0)       /* S' -> S . */
                addact(i, END, ACCEPT);
            else if (x < 0)                 /* A -> alpha . */
                for (t = 1; t < nsym; t++)
                    if (!isnt[t] && (follow[lhs[p]] & bit(t)))
                        addact(i, t, REDUCE + 4 * p);
        }
    }
    nrow = nst;
}

char *acttext(int i, int t)         /* "s5", "r2", "acc", "s7/r2" */
{
    static char buf[MAXC * 30];
    char one[30];
    int kind, k, a;

    buf[0] = '\0';
    for (kind = SHIFT; kind <= ACCEPT; kind++)  /* shifts first */
        for (k = 0; k < nact[i][t]; k++) {
            a = act[i][t][k];
            if (KIND(a) != kind)
                continue;
            if (kind == SHIFT)
                sprintf(one, "s%s", rowname[NUM(a)]);
            else if (kind == REDUCE)
                sprintf(one, "r%d", NUM(a));
            else
                strcpy(one, "acc");
            if (buf[0] != '\0')
                strcat(buf, "/");
            strcat(buf, one);
        }
    return buf;
}

int tcol[MAXSYM], ntcol;            /* ACTION columns: terminals, then $ */
int gcol[MAXSYM], nncol;            /* GOTO columns: nonterminals but S' */

void printtable(void)
{
    int i, j, s, w, gw, n;

    ntcol = nncol = 0;
    for (s = 2; s < nsym; s++)
        if (!isnt[s])
            tcol[ntcol++] = s;
    tcol[ntcol++] = END;
    for (i = 1; i < nnt; i++)
        gcol[nncol++] = ntorder[i];
    w = 3;                          /* every column as wide as the widest */
    for (i = 0; i < nrow; i++)
        for (j = 0; j < ntcol; j++)
            if ((n = strlen(acttext(i, tcol[j]))) > w)
                w = n;
    gw = 2;
    for (i = 0; i < nrow; i++)
        if ((n = strlen(rowname[i])) > gw)
            gw = n;
    printf("%-6s", "");
    for (j = 0; j < ntcol; j++)
        printf("| %-*s ", w, name[tcol[j]]);
    for (j = 0; j < nncol; j++)
        printf("| %-*s ", gw, name[gcol[j]]);
    printf("\n");
    for (i = 0; i < nrow; i++) {
        printf("%-6s", rowname[i]);
        for (j = 0; j < ntcol; j++)
            printf("| %-*s ", w, acttext(i, tcol[j]));
        for (j = 0; j < nncol; j++)
            printf("| %-*s ", gw,
                   gt[i][gcol[j]] < 0 ? "" : rowname[gt[i][gcol[j]]]);
        printf("\n");
    }
}

void printconflicts(const char *kind)
{
    int i, j;

    for (i = 0; i < nrow; i++)
        for (j = 0; j < ntcol; j++)
            if (nact[i][tcol[j]] > 1)
                printf("conflict: ACTION[%s, %s] = %s\n", rowname[i],
                       name[tcol[j]], acttext(i, tcol[j]));
    if (conflicts == 0)
        printf("%s: yes, no conflicts\n", kind);
    else
        printf("%s: no, %d entr%s with two or more actions\n", kind,
               conflicts, conflicts == 1 ? "y" : "ies");
}
```

An action is one `int`: its kind (1 shift, 2 reduce, 3 accept) in the two lowest bits and its number (the state to shift to, or the production to reduce by) above them, so `SHIFT + 4 * 5` is "s5". `KIND` and `NUM` take it apart. As in `ll1.c`, an entry is a small list rather than one value, so that a collision can be kept and shown instead of one action silently overwriting the other: `nact[i][a]` counts the actions in ACTION[i, a], and `addact` refuses a duplicate, keeps the list in order (shifts before reduces, lower numbers first), and counts the entry as a conflict the moment it gets its second action.

`buildslr` is rules 1 to 4, item by item. It first computes FIRST and FOLLOW with chapter 5's functions, which `follow.h` brought in. The GOTO part is copied from `go` for every nonterminal. Then, for each item: a terminal after the dot is rule 1; a complete production 0 is rule 3; any other complete item is rule 2, with a loop over every terminal and `$` (symbol 1) testing the FOLLOW set of the item's left side. `rowname` is just the state's number here; section 7.5 gives merged states longer names, which is why the table prints names and not numbers.

`printtable` lays out the table, ACTION columns first (terminals in grammar order, then `$`) and GOTO columns after, every ACTION column as wide as its widest entry. `printconflicts` lists every entry with two or more actions and gives the verdict. The table program itself is short. Save it as `slr.c`:

<!-- code: id=slr mode=compile -->
```c
/* slr.c: the SLR(1) parsing table of a grammar, and its conflicts.
   Reads a grammar on standard input (grammar.h's form). */
#include "slr.h"

int main(void)
{
    int p;

    prog = "slr";
    readgrammar();
    augment();
    for (p = 0; p < np; p++)
        printf("%3d  %s\n", p, prodtext(p));
    collection();
    buildslr();
    printtable();
    printconflicts("SLR(1)");
    return 0;
}
```

<!-- code: id=slr-run -->
```console
$ gcc -Wall -Wextra slr.c -o slr
$ ./slr < ../ch5/expr.g
  0  E' -> E
  1  E -> E + T
  2  E -> T
  3  T -> T * F
  4  T -> F
  5  F -> ( E )
  6  F -> id
      | +   | *   | (   | )   | id  | $   | E  | T  | F
0     |     |     | s4  |     | s5  |     | 1  | 2  | 3
1     | s6  |     |     |     |     | acc |    |    |
2     | r2  | s7  |     | r2  |     | r2  |    |    |
3     | r4  | r4  |     | r4  |     | r4  |    |    |
4     |     |     | s4  |     | s5  |     | 8  | 2  | 3
5     | r6  | r6  |     | r6  |     | r6  |    |    |
6     |     |     | s4  |     | s5  |     |    | 9  | 3
7     |     |     | s4  |     | s5  |     |    |    | 10
8     | s6  |     |     | s11 |     |     |    |    |
9     | r1  | s7  |     | r1  |     | r1  |    |    |
10    | r3  | r3  |     | r3  |     | r3  |    |    |
11    | r5  | r5  |     | r5  |     | r5  |    |    |
SLR(1): yes, no conflicts
```

The hand table, entry for entry, with the columns in `expr.g`'s order: `+ * ( ) id` instead of the Dragon book's `id + * ( )`. As with the LL(1) table, the columns have no required order; keep the production numbers the same as the paper's, because those are what the `r` entries mean.

### A grammar that is fine, and a table that is not

Here is a grammar for assignments through pointers, as in C's `*p = q`: an assignment is a left value, `=`, and a right value, or just a right value on its own; a left value is an identifier or a `*` in front of a right value; and a right value is a left value used for its contents.

```text
S -> L = R | R
L -> * R | id
R -> L
```

It is not ambiguous. Every sentence has one tree. Save it and build its table:

<!-- code: id=slr-assign -->
```console
$ cat > assign.g <<'END'
S -> L = R | R
L -> * R | id
R -> L
END
$ ./slr < assign.g
  0  S' -> S
  1  S -> L = R
  2  S -> R
  3  L -> * R
  4  L -> id
  5  R -> L
      | =     | *     | id    | $     | S  | L  | R
0     |       | s4    | s5    |       | 1  | 2  | 3
1     |       |       |       | acc   |    |    |
2     | s6/r5 |       |       | r5    |    |    |
3     |       |       |       | r2    |    |    |
4     |       | s4    | s5    |       |    | 8  | 7
5     | r4    |       |       | r4    |    |    |
6     |       | s4    | s5    |       |    | 8  | 9
7     | r3    |       |       | r3    |    |    |
8     | r5    |       |       | r5    |    |    |
9     |       |       |       | r1    |    |    |
conflict: ACTION[2, =] = s6/r5
SLR(1): no, 1 entry with two or more actions
```

One conflict, in state 2, under `=`: shift 6, or reduce by production 5, R -> L. Look at the state to see why. `lr0` shows it:

<!-- code: id=lr0-assign -->
```console
$ ./lr0 < assign.g | sed -n '/^I2:/,/^$/p'
I2:
    S -> L . = R
    R -> L .
  goto(I2, =) = I6
```

I2 = { S -> L . = R, R -> L . }, reached from I0 on `L`. The first item shifts `=`. The second is complete, and rule 2 puts r5 under every token of FOLLOW(R). And FOLLOW(R) does contain `=`: `S -> L = R` puts `=` in FOLLOW(L), and `L -> * R` ends with R, so everything in FOLLOW(L) is in FOLLOW(R). Both actions land in ACTION[2, =].

But think about what reducing would mean *in this state*. I2 is reached by an `L` straight from the start. If the parser reduces that `L` to an `R` here, the `R` is at the start of the input, so it can only be the R of `S -> R`, and an S is followed only by `$`. In I2, an R can never be followed by `=`. The `=` in FOLLOW(R) comes from a different place altogether: an R inside `* R`, which is an L, which can stand before `=`. FOLLOW(R) is one set for the whole grammar; it mixes up every place an R can appear. So the SLR table puts a reduction under `=` that no correct input could ever need, and that reduction collides with the shift that `id = id` does need.

That is SLR's weakness: its lookahead is per nonterminal, not per state. The fix is to carry, inside each item, which tokens can really follow it *in this state*, and that is section 7.5's LR(1) item.

### When it goes wrong

**Reduce entries in every column.** Filling the whole row of a complete item with `r` is the LR(0) table. In the expression grammar it puts r2 under `*` in row 2, on top of s7: a conflict that the SLR table does not have. Use FOLLOW of the item's left side.

**FOLLOW of the wrong symbol.** The reduce for `T -> T * F .` goes under FOLLOW(T), the left side, not FOLLOW(F), the last symbol. Here they happen to be equal; in `E -> E + T .` they are not (FOLLOW(T) has `*`, FOLLOW(E) does not), and using FOLLOW(T) puts r1 under `*` in row 9, on top of s7.

**State numbers in place of production numbers.** `r5` means "reduce by production 5, F -> ( E )", not "go to state 5". Mixing them up gives a table that looks right and parses nothing. Write the numbered productions at the top of your answer, as `slr` does.

**The accept in the wrong place.** Accept goes in the row of the state holding `S' -> S .` (I1 here), under `$` only. It is not "reduce by production 0", and it does not go under any other token.

**Terminals in the GOTO part.** GOTO has columns only for nonterminals. A goto on a terminal is a shift, and it goes in ACTION as `sj`.

### In one line each

1. The table has ACTION (terminals and `$`: shift, reduce, accept, error) and GOTO (nonterminals: the next state after a reduction).
2. A terminal after the dot gives a shift; a complete item A -> α . gives a reduce under every token in FOLLOW(A); S' -> S . gives accept under `$`.
3. A grammar is SLR(1) when no ACTION entry gets two actions; the expression grammar is, and `slr` builds its table.
4. FOLLOW is one set per nonterminal, so SLR can put reductions where no input needs them; `assign.g` has such a conflict in state 2 though it is unambiguous.

**Check yourself**

<!-- q: idea=slr-table type=apply wrong="a left-recursive grammar must be transformed before an LR table can be built" -->
1. Build the SLR table for the list grammar `S -> ( L ) | a`, `L -> L , S | S`, from the nine sets of section 7.3's question 3. You need FOLLOW(S) and FOLLOW(L) first. Check with `slr`.

<details>
<summary>Answer</summary>

FOLLOW(S) = { $, ), , }: `$` as the start symbol; S ends `L -> L , S` and `L -> S`, so it takes FOLLOW(L). FOLLOW(L) = { ), , }, from `( L )` and `L , S`. Productions: 1 S -> ( L ), 2 S -> a, 3 L -> L , S, 4 L -> S.

<!-- code: id=slr-list -->
```console
$ printf 'S -> ( L ) | a\nL -> L , S | S\n' | ./slr
  0  S' -> S
  1  S -> ( L )
  2  S -> a
  3  L -> L , S
  4  L -> S
      | (   | )   | a   | ,   | $   | S  | L
0     | s2  |     | s3  |     |     | 1  |
1     |     |     |     |     | acc |    |
2     | s2  |     | s3  |     |     | 5  | 4
3     |     | r2  |     | r2  | r2  |    |
4     |     | s6  |     | s7  |     |    |
5     |     | r4  |     | r4  |     |    |
6     |     | r1  |     | r1  | r1  |    |
7     | s2  |     | s3  |     |     | 8  |
8     |     | r3  |     | r3  |     |    |
SLR(1): yes, no conflicts
```

Rows 3 (S -> a .) and 6 (S -> ( L ) .) reduce under `)`, `,` and `$`; rows 5 (L -> S .) and 8 (L -> L , S .) only under `)` and `,`, since `$` is not in FOLLOW(L). No conflicts: SLR(1), with the left recursion left in.

</details>

<!-- q: idea=slr-table type=fix wrong="filling a reduce row completely only adds harmless entries" -->
2. Find the fault. A student fills every complete item's row with its reduce in every column, `$` included. In the expression grammar's table, which entries now hold two actions? Which correct input would the student's parser get wrong if it always chose the reduce?

<details>
<summary>Answer</summary>

Rows 2 and 9: ACTION[2, *] = s7/r2 and ACTION[9, *] = s7/r1. (Rows 3, 5, 10, 11 just get extra reduces under `(` and `id`, where nothing else is, so they collide with nothing; but they now reduce where an error should be reported.) A parser choosing the reduce would turn `id * id` into `E` before shifting `*`, and then reject: `E * ...` cannot continue, the greedy failure of `sr -g` in section 7.1. Section 7.8's task 3 runs a table with one such entry.

</details>

<!-- q: idea=slr-table type=predict wrong="a blank entry means shift" -->
3. Predict. In the expression grammar's table, row 5 (F -> id .) has nothing under `(` and `id`. Give an input that reaches ACTION[5, id], and say at which token the parser stops.

<details>
<summary>Answer</summary>

`id id`: the parser shifts the first `id` into state 5 and then sees the second `id`. ACTION[5, id] is blank, so it reports an error at the second `id`, the first token that cannot belong to a correct expression. Compare `sr`, which shifted the second `i` and only gave up at `$` (section 7.1). Section 7.7 runs this.

</details>

## 7.5 CLR(1) and LALR(1) tables by hand

### Lookahead that knows where it is

`assign.g` broke the SLR table in state 2 because FOLLOW(R) is one set for the whole grammar. It holds `=`, because *somewhere* an R can be followed by `=`, and so the SLR rule put a reduction by R -> L under `=` in a state where no R is ever followed by `=`. What the table needed was the set of tokens that can follow the R *that this state is working on*.

That set can be computed. When the parser starts on a production, it knows what it was hoping to see after it, because the item that brought the production in says so. This section carries that knowledge in every item. The result is the *canonical LR(1)* table, called CLR(1) in Indian syllabi and exam papers, the most powerful table of the chapter. Then it shrinks that table to the *LALR(1)* table, which is what Yacc and Bison build, and which chapter 8 relies on.<!-- risk: Bison builds LALR(1) tables by default (other kinds only with %define lr.type); check the Bison manual -->

### An item with a note attached

<!-- teaches: lr1-item = LR(1) item, lookahead of an item, core of an item -->
An *LR(1) item* is an LR(0) item with one terminal attached, its *lookahead*:

```text
[A -> α . β, a]
```

It reads: "the parser is in the middle of A -> αβ, has seen α, and once the whole A has been reduced, the next token will be a". The LR(0) part, `A -> α . β`, is the item's *core*. Several items with the same core and different lookaheads are written as one, with the lookaheads separated by slashes: `[C -> . c C, c/d]` is two items, one with `c` and one with `d`.

The lookahead does nothing while the dot is in the middle. It only matters when the item is complete: `[A -> α ., a]` says "reduce by A -> α, but only if the next token is a". That replaces SLR's "under every token of FOLLOW(A)", and it is the whole improvement.

Think of the plan of section 7.3 with a note clipped to it: not only "I am in the middle of `C -> c C`", but also "and when this C is done, I expect a `c` or a `d`". The note was written by whoever started the plan, which is the subject of the next rule.

### Closure and goto, with lookaheads

The closure rule changes in one place. If `[A -> α . B β, a]` is in the set, then for every production `B -> γ` and every terminal b in FIRST(β a), add `[B -> . γ, b]`. Repeat until nothing changes.

Why FIRST(β a)? The new item starts a B. When that B is finished, what comes next is whatever comes after B in the item that asked for it: first β, and if β can vanish (or is empty), then the asking item's own lookahead, a. FIRST(β a) is exactly "the first token after B" (with chapter 5's FIRST, since `a` is a terminal FIRST(β a) never contains ε). This is the one place where an LR(1) construction needs FIRST sets.

goto changes not at all: move the dot over X in every item with X after it, keep each item's lookahead as it is, and take the closure. The start set is closure({ [S' -> . S, $] }): the whole input is an S followed by the end.

Because the closure can add lookaheads to items already in the set (a later item can reach the same core with a new lookahead), it must be repeated until a whole pass adds nothing, like FIRST in section 5.5, not done in one sweep like the LR(0) closure.

### The grammar the paper uses: S -> C C

Almost every paper's CLR question uses this grammar from the Dragon book, or one like it:

```text
0  S' -> S
1  S  -> C C
2  C  -> c C
3  C  -> d
```

Its sentences are two C's, each some `c`'s followed by one `d`: `d d`, `c d d`, `c d c c d`. Work the collection.

**I0** = closure({ [S' -> . S, $] }). The item has S after the dot, β empty, lookahead `$`: FIRST($) = { $ }, so add `[S -> . C C, $]`. That item has C after the dot, β = `C`, lookahead `$`: FIRST(C $) = FIRST(C) = { c, d }, so add `[C -> . c C, c/d]` and `[C -> . d, c/d]`. Both have terminals after the dot. Done:

```text
I0:  [S' -> . S, $]
     [S  -> . C C, $]
     [C  -> . c C, c/d]
     [C  -> . d, c/d]
```

Read the lookaheads: the first C of `C C` is followed by the second C, which starts with `c` or `d`, so the C items brought in for it carry `c/d`.

**goto(I0, S) = I1** = { [S' -> S ., $] }.

**goto(I0, C) = I2**: the dot moves in `[S -> . C C, $]`, giving `[S -> C . C, $]`. Now C is after the dot with β empty and lookahead `$`: FIRST($) = { $ }, so the closure adds `[C -> . c C, $]` and `[C -> . d, $]`. The second C is followed by the end, and its items say so.

**goto(I0, c) = I3**: `[C -> c . C, c/d]`, and the closure adds `[C -> . c C, c/d]`, `[C -> . d, c/d]` (β empty, so the lookahead c/d is passed on).

**goto(I0, d) = I4** = { [C -> d ., c/d] }.

**goto(I2, C) = I5** = { [S -> C C ., $] }. **goto(I2, c) = I6**: `[C -> c . C, $]` with `[C -> . c C, $]`, `[C -> . d, $]`. **goto(I2, d) = I7** = { [C -> d ., $] }.

**goto(I3, C) = I8** = { [C -> c C ., c/d] }; goto(I3, c) = I3 and goto(I3, d) = I4 (the same kernels again).

**goto(I6, C) = I9** = { [C -> c C ., $] }; goto(I6, c) = I6, goto(I6, d) = I7.

Ten sets. Compare I3 with I6, I4 with I7, I8 with I9: each pair has the same cores and different lookaheads. I4, `[C -> d ., c/d]`, is a `d` that ends the *first* C: what follows is the second C. I7, `[C -> d ., $]`, is a `d` that ends the *second* C: what follows is the end. In the LR(0) collection these were one state each; LR(1) splits them, because the parser is in a different situation in each.

### The CLR(1) table

<!-- teaches: clr-table = canonical LR(1) table, CLR(1) table, LR(1) grammar -->
The *canonical LR(1)* table is built like the SLR table, with rule 2 changed:

1. `[A -> α . a β, b]` in Ii, goto(Ii, a) = Ij: ACTION[i, a] = shift j (the lookahead plays no part).
2. `[A -> α ., a]` in Ii, A not S': ACTION[i, a] = reduce by A -> α, **under a only**.
3. `[S' -> S ., $]` in Ii: ACTION[i, $] = accept.
4. goto(Ii, A) = Ij: GOTO[i, A] = j.

A grammar whose CLR table has no entry with two actions is an *LR(1) grammar*. For `S -> C C`:

| State | c | d | $ | S | C |
|---|---|---|---|---|---|
| 0 | s3 | s4 | | 1 | 2 |
| 1 | | | acc | | |
| 2 | s6 | s7 | | | 5 |
| 3 | s3 | s4 | | | 8 |
| 4 | r3 | r3 | | | |
| 5 | | | r1 | | |
| 6 | s6 | s7 | | | 9 |
| 7 | | | r3 | | |
| 8 | r2 | r2 | | | |
| 9 | | | r2 | | |

Row 4 reduces `C -> d` under `c` and `d` only, and row 7 under `$` only. The SLR table would reduce under all three in both, since FOLLOW(C) = { c, d, $ }. Here that costs nothing (the extra entries collide with nothing), but it is exactly the precision `assign.g` needed.

### LALR(1): merge what looks the same

The CLR table is right, but it is big. Every LR(0) state can split into several LR(1) states, one per pattern of lookaheads, and for the grammar of a whole programming language the canonical LR table has several thousand states where the LR(0) collection has a few hundred.<!-- risk: state counts for a Pascal-like grammar (hundreds for SLR/LALR, thousands for CLR) are from the Dragon book's discussion; check the wording and figures --> The splitting was needed in a few states (state 2 of `assign.g`) and wasted in most (I4 and I7 above do the same thing with different notes).

<!-- teaches: lalr-table = LALR(1) table, merging states with the same core, LALR(1) grammar -->
The *LALR(1)* ("lookahead LR") table keeps the notes and drops the splitting: merge every group of LR(1) states that have the same cores into one state, whose items are the cores with the *union* of the lookaheads. Then build the table with the CLR rules from the merged states. A grammar whose LALR table has no entry with two actions is an *LALR(1) grammar*.

For `S -> C C`, three pairs have the same cores: I3 and I6, I4 and I7, I8 and I9. Merged:

```text
I36: [C -> c . C, c/d/$]      I47: [C -> d ., c/d/$]      I89: [C -> c C ., c/d/$]
     [C -> . c C, c/d/$]
     [C -> . d, c/d/$]
```

The merged states are named by joining the old numbers, as the Dragon book does: I36, I47, I89. Every shift or goto that led to I3 or I6 now leads to I36, and so on. The table:

| State | c | d | $ | S | C |
|---|---|---|---|---|---|
| 0 | s36 | s47 | | 1 | 2 |
| 1 | | | acc | | |
| 2 | s36 | s47 | | | 5 |
| 36 | s36 | s47 | | | 89 |
| 47 | r3 | r3 | r3 | | |
| 5 | | | r1 | | |
| 89 | r2 | r2 | r2 | | |

Seven states, the same number as the LR(0) collection of this grammar. That is always so: merging by core gives back one state per LR(0) state, because the cores *are* the LR(0) items. An LALR table has the size of an SLR table and lookaheads that are usually as precise as CLR's. Usually, not always: section 7.6 shows a grammar where the merge itself causes a conflict.

Real tools do not build the canonical collection and then merge it, which would cost the very size they are trying to save; they work out the LALR lookaheads directly on the LR(0) collection. By hand, in the exam, build CLR first and merge: it is the method the paper expects, and it gives the same table.

### What merging costs: a later error

Merging never changes what a correct input does. It can change *when* an error is found. Take the input `d` alone, which is wrong (a sentence needs two C's).

With the CLR table: state 0, `d`: s4. State 4, `$`: blank. Error, with the `$` in hand.

With the LALR table: state 0, `d`: s47. State 47, `$`: r3, reduce by C -> d, which pops back to state 0; GOTO[0, C] = 2. State 2, `$`: blank. Error, again with the `$` in hand.

The LALR parser made one reduction the CLR parser refused, because I47 carries `$` from I7. But it stopped at the same token: an LALR parser may reduce a few times more before it reports an error, but it never shifts a token that the CLR parser would have refused. Section 7.7 runs both tables on a similar input, `c d`.

### lr1.h: both tables in C

The LR(1) code goes in a third header, `lr1.h`, built on `slr.h`, whose table arrays and printing it reuses. It needs its own states, because an LR(1) state is a list of items *and* a lookahead set for each. Save this as `lr1.h`:

<!-- code: id=lr1-h mode=fragment file=lr1.h -->
```c
/* lr1.h: the canonical collection of LR(1) items, the canonical LR(1)
   table, and the LALR(1) table made by merging states with the same
   core. Needs slr.h (the table) and through it lr0.h. */
#include "slr.h"

typedef struct {
    int n, nk;                      /* items; the first nk are the kernel */
    int item[MAXI];                 /* the core: production and dot */
    set la[MAXI];                   /* its lookaheads, a set of terminals */
} state1;

state1 cs[MAXST];                   /* the LR(1) collection */
int ncs;
int cgo[MAXST][MAXSYM];             /* goto on the LR(1) states, or -1 */

int add1(state1 *s, int it, set la) /* 1 if s changed */
{
    int k;

    for (k = 0; k < s->n; k++)
        if (s->item[k] == it) {     /* the same core: join the lookaheads */
            if ((s->la[k] | la) == s->la[k])
                return 0;
            s->la[k] |= la;
            return 1;
        }
    if (s->n == MAXI)
        die("more than %d items in a state", MAXI);
    s->item[s->n] = it;
    s->la[s->n++] = la;
    return 1;
}

void closure1(state1 *s)            /* until a pass changes nothing */
{
    int k, b, p, q, d, changed;
    set f, la;

    do {
        changed = 0;
        for (k = 0; k < s->n; k++) {        /* [A -> alpha . B beta, a] */
            b = after(s->item[k]);
            if (b < 0 || !isnt[b])
                continue;
            p = PROD(s->item[k]);
            d = DOT(s->item[k]);
            f = firstof(rhs[p] + d + 1, len[p] - d - 1);    /* FIRST(beta) */
            la = f & ~bit(EPS);
            if (f & bit(EPS))
                la |= s->la[k];             /* beta can vanish: FIRST(a) */
            for (q = 0; q < np; q++)
                if (lhs[q] == b)
                    changed |= add1(s, ITEM(q, 0), la);
        }
    } while (changed);
}

void goto1(const state1 *from, int x, state1 *to)
{
    int k;

    to->n = 0;
    for (k = 0; k < from->n; k++)
        if (after(from->item[k]) == x)
            add1(to, from->item[k] + 1, from->la[k]);
    to->nk = to->n;
    closure1(to);
}

/* same kernel cores; with la, the same lookaheads as well */
int same1(const state1 *a, const state1 *b, int la)
{
    int i, j;

    if (a->nk != b->nk)
        return 0;
    for (i = 0; i < a->nk; i++) {
        for (j = 0; j < b->nk && b->item[j] != a->item[i]; j++)
            ;
        if (j == b->nk || (la && b->la[j] != a->la[i]))
            return 0;
    }
    return 1;
}

int addstate1(const state1 *s)
{
    int i;

    for (i = 0; i < ncs; i++)
        if (same1(&cs[i], s, 1))
            return i;
    if (ncs == MAXST)
        die("more than %d states", MAXST);
    cs[ncs] = *s;
    return ncs++;
}

void collection1(void)
{
    state1 t;
    int i, k, x;

    computefirst(0);                /* closure1 needs FIRST */
    memset(cgo, -1, sizeof cgo);
    t.n = 0;
    add1(&t, ITEM(0, 0), bit(END)); /* I0 = closure({ [S' -> . S, $] }) */
    t.nk = 1;
    closure1(&t);
    addstate1(&t);
    for (i = 0; i < ncs; i++)
        for (k = 0; k < cs[i].n; k++) {
            x = after(cs[i].item[k]);
            if (x < 0 || cgo[i][x] >= 0)
                continue;
            goto1(&cs[i], x, &t);
            cgo[i][x] = addstate1(&t);
        }
}

int row[MAXST];                     /* the table row of each LR(1) state */

/* the table: canonical LR(1) if merge is 0, LALR(1) if it is 1 */
void buildlr1(int merge)
{
    int i, j, k, it, x, p, t, r;
    const char *sep;

    memset(gt, -1, sizeof gt);
    nrow = 0;
    for (i = 0; i < ncs; i++) {     /* rows: one per state, or per core */
        for (j = 0; j < i && !(merge && same1(&cs[j], &cs[i], 0)); j++)
            ;
        row[i] = j < i ? row[j] : nrow++;
    }
    for (r = 0; r < nrow; r++) {    /* names: "36", or "4.11" if needed */
        rowname[r][0] = '\0';
        for (i = 0; i < ncs; i++)
            if (row[i] == r && i >= 10)
                break;
        sep = i < ncs ? "." : "";
        for (i = 0; i < ncs; i++)
            if (row[i] == r)
                sprintf(rowname[r] + strlen(rowname[r]), "%s%d",
                        rowname[r][0] != '\0' ? sep : "", i);
    }
    for (i = 0; i < ncs; i++) {
        r = row[i];
        for (x = 0; x < nsym; x++)
            if (isnt[x] && cgo[i][x] >= 0)
                gt[r][x] = row[cgo[i][x]];
        for (k = 0; k < cs[i].n; k++) {
            it = cs[i].item[k];
            x = after(it);
            p = PROD(it);
            if (x >= 0 && !isnt[x])         /* [A -> alpha . a beta, b] */
                addact(r, x, SHIFT + 4 * row[cgo[i][x]]);
            else if (x < 0 && p == 0)       /* [S' -> S ., $] */
                addact(r, END, ACCEPT);
            else if (x < 0)                 /* [A -> alpha ., a]: on a */
                for (t = 1; t < nsym; t++)
                    if (cs[i].la[k] & bit(t))
                        addact(r, t, REDUCE + 4 * p);
        }
    }
}

char *latext(set la)                /* "c/d/$" */
{
    static char buf[MAXSYM * (MAXLEN + 1)];
    int t;

    buf[0] = '\0';
    for (t = 2; t <= nsym; t++) {   /* terminals in grammar order, then $ */
        int s = t < nsym ? t : END;

        if (isnt[s] || !(la & bit(s)))
            continue;
        if (buf[0] != '\0')
            strcat(buf, "/");
        strcat(buf, name[s]);
    }
    return buf;
}
```

A state keeps a lookahead set beside each core, `la[k]`, as a bit set of the same `set` type FIRST used. `add1` adds a core with its lookaheads, or, if the core is there already, joins the new lookaheads to the old ones; it returns 1 if anything changed, which is how `closure1` knows when to stop. `closure1` is the new closure rule: for an item with nonterminal B after the dot, `firstof` gives FIRST(β) for the symbols after B, the lookahead is that set without ε, plus the item's own lookaheads if β can vanish, and every B production is added with it. The `do ... while (changed)` repeats the whole pass until nothing grows.

`goto1` copies each moved item's lookahead unchanged. `same1` compares two kernels: with `la` set it requires the same lookaheads too (the CLR test: two states are the same only if their notes are), and without it only the cores (the LALR test). `collection1` is `collection` of `lr0.h` with these functions.

`buildlr1` makes either table from the one collection. First it gives every LR(1) state a row: its own (CLR), or, if `merge` is on and an earlier state has the same cores, that state's row (LALR). Then it names the rows by joining the numbers of the states in them, "36", or "4.11" with a dot when a number has two digits, so that 4 and 11 cannot be read as 41 and 1. Then the four rules, with every state number passed through `row[]`, so that in the LALR table a shift to I3 or to I6 both become "s36". Since `addact` refuses duplicates, the two merged states' identical shifts become one entry, and their reductions on different lookaheads simply add up in the merged row; if they collide, `addact` counts a conflict.

`lr1.c` prints the items of the canonical collection with their lookaheads, then the CLR table, or the LALR table with `-l`. Save it as `lr1.c`:

<!-- code: id=lr1 mode=compile -->
```c
/* lr1.c: the canonical LR(1) items and table of a grammar, or with -l
   the LALR(1) table made by merging states with the same core.
   Reads a grammar on standard input (grammar.h's form). */
#include "lr1.h"

int main(int argc, char *argv[])
{
    int merge = argc > 1 && strcmp(argv[1], "-l") == 0;
    int i, k, x, p, shown[MAXSYM];

    prog = "lr1";
    readgrammar();
    augment();
    for (p = 0; p < np; p++)
        printf("%3d  %s\n", p, prodtext(p));
    collection1();
    for (i = 0; i < ncs; i++) {
        printf("\nI%d:\n", i);
        for (k = 0; k < cs[i].n; k++)
            printf("  %s [%s, %s]\n", k < cs[i].nk ? " " : "+",
                   itemtext(cs[i].item[k]), latext(cs[i].la[k]));
        memset(shown, 0, sizeof shown);
        for (k = 0; k < cs[i].n; k++) {
            x = after(cs[i].item[k]);
            if (x >= 0 && !shown[x]) {
                printf("  goto(I%d, %s) = I%d\n", i, name[x], cgo[i][x]);
                shown[x] = 1;
            }
        }
    }
    printf("\n");
    buildlr1(merge);
    printtable();
    printconflicts(merge ? "LALR(1)" : "LR(1)");
    return 0;
}
```

Build it and run it on `S -> C C`:

<!-- code: id=lr1-run -->
```console
$ gcc -Wall -Wextra lr1.c -o lr1
$ cat > cc.g <<'END'
S -> C C
C -> c C | d
END
$ ./lr1 < cc.g | sed '/^I5:/,$d'
  0  S' -> S
  1  S -> C C
  2  C -> c C
  3  C -> d

I0:
    [S' -> . S, $]
  + [S -> . C C, $]
  + [C -> . c C, c/d]
  + [C -> . d, c/d]
  goto(I0, S) = I1
  goto(I0, C) = I2
  goto(I0, c) = I3
  goto(I0, d) = I4

I1:
    [S' -> S ., $]

I2:
    [S -> C . C, $]
  + [C -> . c C, $]
  + [C -> . d, $]
  goto(I2, C) = I5
  goto(I2, c) = I6
  goto(I2, d) = I7

I3:
    [C -> c . C, c/d]
  + [C -> . c C, c/d]
  + [C -> . d, c/d]
  goto(I3, C) = I8
  goto(I3, c) = I3
  goto(I3, d) = I4

I4:
    [C -> d ., c/d]
```

The rest of the collection, and the table:

<!-- code: id=lr1-run2 -->
```console
$ ./lr1 < cc.g | sed -n '/^I5:/,$p'
I5:
    [S -> C C ., $]

I6:
    [C -> c . C, $]
  + [C -> . c C, $]
  + [C -> . d, $]
  goto(I6, C) = I9
  goto(I6, c) = I6
  goto(I6, d) = I7

I7:
    [C -> d ., $]

I8:
    [C -> c C ., c/d]

I9:
    [C -> c C ., $]

      | c   | d   | $   | S  | C
0     | s3  | s4  |     | 1  | 2
1     |     |     | acc |    |
2     | s6  | s7  |     |    | 5
3     | s3  | s4  |     |    | 8
4     | r3  | r3  |     |    |
5     |     |     | r1  |    |
6     | s6  | s7  |     |    | 9
7     |     |     | r3  |    |
8     | r2  | r2  |     |    |
9     |     |     | r2  |    |
LR(1): yes, no conflicts
```

The ten sets are the hand sets, with the same lookaheads and the same numbers, and the table is the hand CLR table. The merged table is the last part of the output with `-l`:

<!-- code: id=lr1-lalr -->
```console
$ ./lr1 -l < cc.g | tail -9
      | c   | d   | $   | S  | C
0     | s36 | s47 |     | 1  | 2
1     |     |     | acc |    |
2     | s36 | s47 |     |    | 5
36    | s36 | s47 |     |    | 89
47    | r3  | r3  | r3  |    |
5     |     |     | r1  |    |
89    | r2  | r2  | r2  |    |
LALR(1): yes, no conflicts
```

Rows 36, 47 and 89, and the same seven rows as the hand LALR table.

### assign.g again

Now the grammar SLR could not handle. Its LALR table:

<!-- code: id=lr1-assign -->
```console
$ ./lr1 -l < assign.g | tail -12
      | =     | *     | id    | $     | S    | L    | R
0     |       | s4.11 | s5.12 |       | 1    | 2    | 3
1     |       |       |       | acc   |      |      |
2     | s6    |       |       | r5    |      |      |
3     |       |       |       | r2    |      |      |
4.11  |       | s4.11 | s5.12 |       |      | 8.10 | 7.13
5.12  | r4    |       |       | r4    |      |      |
6     |       | s4.11 | s5.12 |       |      | 8.10 | 9
7.13  | r3    |       |       | r3    |      |      |
8.10  | r5    |       |       | r5    |      |      |
9     |       |       |       | r1    |      |      |
LALR(1): yes, no conflicts
```

No conflict. Row 2 shifts `=` to state 6 and reduces by production 5 only under `$`. In the canonical collection, which the full output lists before the table, state 2 is `[S -> L . = R, $]` and `[R -> L ., $]`. Follow the note on the second item back to where it was written: in I0, `[S -> . R, $]` brought in `[R -> . L, $]`, with lookahead FIRST(`$`) because nothing follows R in `S -> R`. Moving the dot over L kept the `$`. So in this state the reduction is entered under `$` alone, and the shift under `=` stands by itself, which is the argument made by hand in section 7.4, now done by the construction.

The canonical collection of `assign.g` has fourteen states (I0 to I13); the LALR table has ten rows, `4.11`, `5.12`, `7.13` and `8.10` being the merged ones, the same ten states as its LR(0) collection.

### When it goes wrong

**FOLLOW instead of FIRST(β a).** Giving `[C -> . c C]` in I0 the lookaheads FOLLOW(C) = { c, d, $ } makes every LR(1) item an SLR item in disguise, and the CLR table becomes the SLR table. The lookahead comes from the item that asked for C: FIRST of what follows C *in that item*, then that item's own lookahead.

**Forgetting the asking item's lookahead.** In `[S -> C . C, $]`, β after the second C is empty, so FIRST(β $) is { $ }. Writing nothing, or `c/d`, because "β is empty, so FIRST(β) is empty" loses the `$`, and row 7 gets no reduce: the parser rejects every correct input at its end.

**A goto that changes the lookahead.** Moving the dot never touches the lookahead: goto(I0, d) is `[C -> d ., c/d]`, the lookahead of `[C -> . d, c/d]`. New lookaheads are made only by closure.

**Merging states that differ in a core.** I2 = { [S -> C . C, $], [C -> . c C, $], [C -> . d, $] } and I6 = { [C -> c . C, $], ... } share two cores, but not all; they are not merged. Merge only when the cores are the same as a set.

**Merging without joining the lookaheads.** I47 reduces under `c`, `d` and `$`: all of I4's and all of I7's. Keeping only one of them gives a table that rejects either the first C or the second.

### In one line each

1. An LR(1) item [A -> α . β, a] adds a lookahead: reduce by A -> α only when a comes next.
2. Closure gives [B -> . γ, b] for every b in FIRST(β a); goto keeps lookaheads; start from [S' -> . S, $].
3. The CLR table reduces under the item's lookaheads only; `S -> C C` has ten CLR states and `lr1` prints them.
4. LALR merges states with the same cores, joining lookaheads: as many states as SLR, nearly CLR's precision, and the kind of table Yacc builds.
5. Merging can make an LALR parser reduce a little more before an error, never shift further.

**Check yourself**

<!-- q: idea=lr1-item type=apply wrong="the lookahead of a closure item is the FOLLOW set" -->
1. In `assign.g`, state 6 is goto(I2, =). Work out its items with lookaheads. Then say why the same core `[R -> L .]` gets `$` in one state and `=/$` in another. Check with `lr1`.

<details>
<summary>Answer</summary>

The kernel is `[S -> L = . R, $]`. R after the dot, β empty: add `[R -> . L, $]`. L after the dot, β empty: add `[L -> . * R, $]` and `[L -> . id, $]`. So every item in I6 has lookahead `$`: after the right side of an assignment comes the end. In I0, by contrast, `[S -> . L = R, $]` brings in L's items with lookahead FIRST(`= R $`) = { = }, and `[S -> . R, $]` brings in `[R -> . L, $]`, which brings in L's items with `$`, so L's items in I0 carry `=/$`. An `R -> L .` reached through those items (as in I8, goto(I4, L)) has `=/$`, because an L inside `* R` can stand on the left of `=`.

<!-- code: id=lr1-six -->
```console
$ ./lr1 < assign.g | sed -n '/^I6:/,/^$/p'
I6:
    [S -> L = . R, $]
  + [R -> . L, $]
  + [L -> . * R, $]
  + [L -> . id, $]
  goto(I6, R) = I9
  goto(I6, L) = I10
  goto(I6, *) = I11
  goto(I6, id) = I12
```

</details>

<!-- q: idea=lalr-table type=decide wrong="LALR merging can introduce shift-reduce conflicts" -->
2. Decide. A merged LALR state has a shift on `a` from one of its CLR states and a reduce on `a` from the other. Can that happen if both CLR states were conflict-free? Explain from what a shift depends on.

<details>
<summary>Answer</summary>

No. A shift on `a` comes from an item `[A -> α . a β, b]`, and whether it exists depends only on the core `A -> α . a β`, not on the lookahead. Merged states have the same cores, so if one has that shift, so does the other. And the reduce on `a` came from one of them, with its own lookahead `a`. So that CLR state already had both the shift and the reduce on `a`: a conflict before merging. Merging can only add *reduce-reduce* conflicts, where two reductions arrive from different states with the same lookahead; section 7.6 shows one.

</details>

<!-- q: idea=clr-table type=predict wrong="CLR and LALR detect every error at the same move" -->
3. Predict the moves of the CLR and the LALR parser for `S -> C C` on the input `c d` (wrong: only one C). How many moves does each make before the error, and which token is in hand when it stops?

<details>
<summary>Answer</summary>

CLR: s3 (`c`), s4 (`d`), then state 4 with `$`: blank. Two moves, then the error, at `$`. LALR: s36, s47, then state 47 with `$`: r3 (C -> d, goto 89 from 36), then state 89 with `$`: r2 (C -> c C, back to 0, goto 2), then state 2 with `$`: blank. Four moves, then the error, also at `$`. The LALR parser made two reductions the CLR parser refused, but neither shifted past the `$`. Section 7.7's `lrparse` shows both runs.

</details>

## 7.6 Shift-reduce and reduce-reduce conflicts

### Two orders in one square

Every table program of this chapter ends with a verdict line, and three of them have already said "no": `optable` on `amb.g`, `slr` on `assign.g`, and, if you tried them, `slr` on `dangle.g` and `lr1` on `amb.g`. A table entry with two actions is a square of the board game with two arrows for one card. The parser, which never guesses, cannot run on it.

The written paper asks about this in three forms: "show that the grammar is not SLR(1)" (find one entry with two actions and explain it from the items), "is the grammar LALR(1)? LR(1)?" (build the tables, or reason about them), and "resolve the conflicts using precedence and associativity". The lab meets it in chapter 8, where Bison prints `conflicts: 1 shift/reduce` and builds a parser anyway. This section teaches you to read a conflict, to say where it comes from, and to decide what to do about it.

### The two kinds

<!-- teaches: lr-conflict = shift-reduce conflict, reduce-reduce conflict, LR grammar classes -->
There are only two kinds of conflict, because an entry can only hold shifts and reduces (and two shifts on one token cannot happen: goto on a symbol is one state).

- A **shift-reduce conflict**: a state holds an item `A -> α . a β`, which shifts on a, and a complete item `B -> γ .` whose reduction is entered under a too. For SLR, "entered under a" means a is in FOLLOW(B); for CLR and LALR, a is in the item's lookahead.
- A **reduce-reduce conflict**: a state holds two complete items `A -> α .` and `B -> β .` whose reductions are entered under the same token.

Reading a conflict always takes the same four steps. Name the state and the token from the entry. List the state's items (`lr0` or `lr1` prints them). Say which item gives which action. Then find an input that reaches that state with that token next, and say what each action would do with it. The last step is what the paper's "explain" wants, and it also tells you whether the grammar is wrong or only the method is too weak.

The chapter has examples of every cause. There are four.

### Cause 1: the grammar is ambiguous

<!-- code: id=slr-amb -->
```console
$ ./slr < ../ch5/amb.g
  0  E' -> E
  1  E -> E + E
  2  E -> E * E
  3  E -> ( E )
  4  E -> id
      | +     | *     | (     | )     | id    | $     | E
0     |       |       | s2    |       | s3    |       | 1
1     | s4    | s5    |       |       |       | acc   |
2     |       |       | s2    |       | s3    |       | 6
3     | r4    | r4    |       | r4    |       | r4    |
4     |       |       | s2    |       | s3    |       | 7
5     |       |       | s2    |       | s3    |       | 8
6     | s4    | s5    |       | s9    |       |       |
7     | s4/r1 | s5/r1 |       | r1    |       | r1    |
8     | s4/r2 | s5/r2 |       | r2    |       | r2    |
9     | r3    | r3    |       | r3    |       | r3    |
conflict: ACTION[7, +] = s4/r1
conflict: ACTION[7, *] = s5/r1
conflict: ACTION[8, +] = s4/r2
conflict: ACTION[8, *] = s5/r2
SLR(1): no, 4 entries with two or more actions
```

Four conflicts, in states 7 and 8, under `+` and `*`. The items:

<!-- code: id=lr0-amb -->
```console
$ ./lr0 < ../ch5/amb.g | sed -n '/^I7:/,/^I9:/p'
I7:
    E -> E + E .
    E -> E . + E
    E -> E . * E
  goto(I7, +) = I4
  goto(I7, *) = I5

I8:
    E -> E * E .
    E -> E . + E
    E -> E . * E
  goto(I8, +) = I4
  goto(I8, *) = I5

I9:
```

I7 is `E -> E + E .` with `E -> E . + E` and `E -> E . * E`. Reach it with `id + id` on the stack (as `E + E`) and look at the next token.

- Next token `*`, input `id + id * id`: reduce by E -> E + E means "add first", shift means "multiply first". Both lead to a successful parse, with different trees.
- Next token `+`, input `id + id + id`: reduce means `(id + id) + id`, shift means `id + (id + id)`. Again both work.

I8, `E -> E * E .`, has the same two choices with `*` on the left. Each conflict is one of section 5.2's two trees, met at the moment the parser has to commit to one. That is the general rule: **an ambiguous grammar is never LR**. Two trees mean two rightmost derivations, so at some point two different handles are possible, and every LR table, CLR included, has a conflict there. `lr1` agrees:

<!-- code: id=lr1-amb -->
```console
$ ./lr1 -l < ../ch5/amb.g | tail -5
conflict: ACTION[9.15, +] = s4.12/r1
conflict: ACTION[9.15, *] = s5.13/r1
conflict: ACTION[10.16, +] = s4.12/r2
conflict: ACTION[10.16, *] = s5.13/r2
LALR(1): no, 4 entries with two or more actions
```

There are two cures. One is to rewrite the grammar so that it has one tree per sentence: `expr.g`, with its `T` and `F` layers, whose SLR table has no conflicts. The other is to keep the ambiguous grammar and choose one action in each conflicting entry, by the precedence and associativity you want:

| Entry | Choices | Choose | Because |
|---|---|---|---|
| ACTION[7, +] | s4 / r1 | r1 | `+` is left-associative: finish the `E + E` on the stack first |
| ACTION[7, *] | s5 / r1 | s5 | `*` binds tighter than `+`: let the `*` take the right operand |
| ACTION[8, +] | s4 / r2 | r2 | `*` binds tighter: finish the `E * E` before the `+` |
| ACTION[8, *] | s5 / r2 | r2 | `*` is left-associative |

The resolved table parses every expression with the usual meaning,<!-- risk: not run: no program here builds the hand-resolved amb.g table; claim from the choices above --> and it is smaller than `expr.g`'s: ten states instead of twelve, and no reductions by E -> T and T -> F, since those productions do not exist. That is why parser generators let you write the ambiguous grammar plus a precedence list: Yacc's `%left '+'` and `%left '*'` lines (section 8.3) make exactly these choices, entry by entry, and section 8.5 reads the report Bison writes when there is no such line.

### Cause 2: the dangling else

The `if`-statement grammar of section 5.2 is ambiguous in one place, the `else` that could belong to either of two `if`s:

<!-- code: id=slr-dangle -->
```console
$ ./slr < ../ch5/dangle.g | tail -2
conflict: ACTION[7, e] = s8/r1
SLR(1): no, 1 entry with two or more actions
$ ./lr0 < ../ch5/dangle.g | sed -n '/^I7:/,/^$/p'
I7:
    S -> i E t S .
    S -> i E t S . e S
  goto(I7, e) = I8
```

I7 holds `S -> i E t S .`, complete, and `S -> i E t S . e S`, which shifts `e`. FOLLOW(S) contains `e` (an S can stand before `e` in `i E t S e S`), so the reduction is entered under `e` too. Reach it with `i b t i b t a` on the stack, which is `i E t i E t S` after reductions, and `e` next. Reduce: the inner `if` is finished without an `else`, and the `e` goes to the outer one. Shift: the `e` goes to the inner one. Two trees, two actions. The CLR table has the same conflict, since the grammar is ambiguous.

The cure everyone uses is to choose shift. Shifting gives the `else` to the nearest `if` without an `else`, which is the rule of C, Pascal and almost every language with this construct, and it is also what Yacc and Bison do on their own when a shift-reduce conflict has no precedence to settle it, with a warning in the report.<!-- risk: Bison's default resolution: shift-reduce conflicts resolved as shift, reduce-reduce as the production that appears first in the grammar; check against the Bison manual section on conflicts --> Rewriting the grammar so that it has one tree is possible (separate "matched" and "unmatched" statements) but makes the grammar much longer, and nobody does it in practice.

### Cause 3: the method's lookahead is too rough

`assign.g` has one tree per sentence, and still `slr` reports a shift-reduce conflict in state 2 under `=`. Section 7.4 found the reason: FOLLOW(R) contains `=` because of an R somewhere else in the grammar, not because of the R in state 2. The LALR and CLR tables have no conflict:

<!-- code: id=assign-three -->
```console
$ ./slr < assign.g | tail -1
SLR(1): no, 1 entry with two or more actions
$ ./lr1 -l < assign.g | tail -1
LALR(1): yes, no conflicts
$ ./lr1 < assign.g | tail -1
LR(1): yes, no conflicts
```

This kind of conflict says nothing against the grammar. It says the table-building method is too weak for it, and the cure is a stronger method, LALR or CLR, with the grammar unchanged.

### Cause 4: merging joined two different notes

The last cause belongs to LALR alone. Here is a grammar from the Dragon book, small enough to work by hand:

```text
S -> a A d | b B d | a B e | b A e
A -> c
B -> c
```

After `a c`, the `c` can be an A (if `d` comes next, `a A d`) or a B (if `e` comes next, `a B e`). After `b c` it is the other way round: A before `e`, B before `d`. In each case one token of lookahead decides. Build both tables:

<!-- code: id=rr-tables -->
```console
$ cat > rr.g <<'END'
S -> a A d | b B d | a B e | b A e
A -> c
B -> c
END
$ ./lr1 < rr.g | tail -1
LR(1): yes, no conflicts
$ ./lr1 -l < rr.g | tail -3
conflict: ACTION[69, d] = r5/r6
conflict: ACTION[69, e] = r5/r6
LALR(1): no, 2 entries with two or more actions
$ ./lr1 < rr.g | sed -n '/^I6:/,/^$/p;/^I9:/,/^$/p'
I6:
    [A -> c ., d]
    [B -> c ., e]

I9:
    [B -> c ., d]
    [A -> c ., e]
```

The canonical table has no conflict: the grammar is LR(1). The LALR table has two reduce-reduce conflicts in the merged state 69. The canonical collection shows why. I6, reached by `a c`, holds `[A -> c ., d]` and `[B -> c ., e]`; I9, reached by `b c`, holds `[B -> c ., d]` and `[A -> c ., e]`. The cores are the same, `A -> c .` and `B -> c .`, so LALR merges the two states and joins the notes: both items get `d/e`. Now under `d` the merged state reduces by A -> c (from I6) and by B -> c (from I9), and under `e` the same. Each CLR state knew which way round it was; the merged one has forgotten.

Notice that the merge produced a reduce-reduce conflict, not a shift-reduce one. That is always so: section 7.5's question 2 showed that a shift depends only on cores, so if a merged state shifts on a token, every state in the merge shifted on it, and any reduce on that token was already a conflict before merging. Merging can only bring reductions together.

The cure: use the canonical LR(1) table, or rewrite the grammar slightly (for example, give A and B different tokens if the language allows it). In practice LALR's merges almost never cause trouble with a real programming language's grammar, which is why tools use LALR.

### Which method is enough?

Each cause above was a grammar that one method could handle and a weaker one could not. Together they show the four classes of grammar nested inside each other:

| Grammar | LR(0) | SLR(1) | LALR(1) | LR(1) |
|---|---|---|---|---|
| list grammar (`S -> ( L ) \| a`, `L -> L , S \| S`) | yes | yes | yes | yes |
| `expr.g` | no: I1, I2, I9 mix reduce and shift | yes | yes | yes |
| `assign.g` | no | no: state 2 under `=` | yes | yes |
| `rr.g` | no | no | no: state 69 under `d`, `e` | yes |
| `amb.g`, `dangle.g` | no | no | no | no: ambiguous |

Every LR(0) grammar is SLR(1), every SLR(1) grammar is LALR(1), and every LALR(1) grammar is LR(1); each row of the table is a grammar that sits in one class and not in the one before it. To answer "which is the weakest method that works?", build the tables in that order and stop at the first with no conflicts. To answer "is it LR(1)?" for a grammar you suspect is ambiguous, one sentence with two parse trees settles it without any table.

Top-down parsing fits into the same picture. Every LL(1) grammar is LR(1), so a bottom-up parser can do anything chapter 6's parser can, and more: `expr.g`, left recursion and all, is SLR(1) but not LL(1). The reverse is not true. And an LL(1) grammar need not be SLR(1): section 7.8 has one.

### When it goes wrong

**Calling every conflict a sign of ambiguity.** `assign.g` and `rr.g` are unambiguous and have conflicts in some tables. A conflict in an SLR or LALR table only shows that the method is too weak; only a conflict in the canonical LR(1) table, or two parse trees for one sentence, says something about the grammar.

**Showing "not SLR" with a full table and no explanation.** The paper wants one entry, the state's items, the FOLLOW set or lookahead that put the reduce there, and the shift item. Four lines. A table with `s6/r5` in it and no reason gets half the marks.

**Resolving a conflict by deleting a production.** Removing `R -> L` from `assign.g` makes the conflict disappear, along with the language: `*p = q` is no longer a sentence. A resolution chooses one *action* in one *entry*; the grammar stays the same.

**Resolving in the wrong direction.** In ACTION[7, +] of `amb.g`, shift makes `+` right-associative. A table resolved that way parses everything and gives `id - id - id` the wrong value once `-` is added. Check each choice against one input, as the table of choices above does.

**Expecting LALR to report the same states as CLR.** A conflict reported in state "69" by `lr1 -l` is in a merged state. To explain it, look up the CLR states 6 and 9 and their lookaheads, as above.

### In one line each

1. A shift-reduce conflict: a shift item and a complete item whose reduce is entered under the same token; a reduce-reduce conflict: two complete items under the same token.
2. Read a conflict from the state's items and an input that reaches it with that token next.
3. Ambiguity gives conflicts in every table; resolve them by precedence and associativity, or shift for the dangling else, as Yacc does.
4. A conflict only in SLR is FOLLOW being too rough; a conflict only in LALR comes from merging and is always reduce-reduce.
5. LR(0) ⊂ SLR(1) ⊂ LALR(1) ⊂ LR(1), and every LL(1) grammar is LR(1).

**Check yourself**

<!-- q: idea=lr-conflict type=apply wrong="every state of an ambiguous grammar has a conflict" -->
1. The grammar `E -> E + E | id` is ambiguous. Build its SLR table with `slr`. How many conflicts are there, of which kind, and which choice makes `+` left-associative?

<details>
<summary>Answer</summary>

One shift-reduce conflict, in the state holding `E -> E + E .` and `E -> E . + E`, under `+`. Reduce makes `+` left-associative: with `E + E` on the stack and another `+` next, the first addition is finished first. Other states, such as the one with `E -> id .`, have no conflict: ambiguity puts conflicts only where two trees part ways.

<!-- code: id=slr-plus -->
```console
$ printf 'E -> E + E | id\n' | ./slr
  0  E' -> E
  1  E -> E + E
  2  E -> id
      | +     | id    | $     | E
0     |       | s2    |       | 1
1     | s3    |       | acc   |
2     | r2    |       | r2    |
3     |       | s2    |       | 4
4     | s3/r1 |       | r1    |
conflict: ACTION[4, +] = s3/r1
SLR(1): no, 1 entry with two or more actions
```

</details>

<!-- q: idea=lr-conflict type=decide wrong="a conflict in the SLR table means the grammar is ambiguous" -->
2. Decide, for each grammar, the weakest method among SLR(1), LALR(1) and LR(1) whose table has no conflicts, and run the tools to check: (a) `S -> C C`, `C -> c C | d`; (b) `assign.g`; (c) `rr.g`.

<details>
<summary>Answer</summary>

(a) SLR(1): its SLR table, with FOLLOW(C) = { c, d, $ }, already has one action per entry. (b) LALR(1): SLR fails in state 2. (c) LR(1): LALR fails in the merged state 69.

<!-- code: id=classify-three -->
```console
$ for g in cc.g assign.g rr.g; do echo "== $g"; ./slr < $g | tail -1; ./lr1 -l < $g | tail -1; ./lr1 < $g | tail -1; done
== cc.g
SLR(1): yes, no conflicts
LALR(1): yes, no conflicts
LR(1): yes, no conflicts
== assign.g
SLR(1): no, 1 entry with two or more actions
LALR(1): yes, no conflicts
LR(1): yes, no conflicts
== rr.g
SLR(1): no, 2 entries with two or more actions
LALR(1): no, 2 entries with two or more actions
LR(1): yes, no conflicts
```

</details>

<!-- q: idea=lr-conflict type=fix wrong="a reduce-reduce conflict can be settled by preferring the shorter production" -->
3. Find the fault. A student explains `rr.g`'s LALR conflict as "the grammar is ambiguous, because A -> c and B -> c both derive c". What is wrong with the explanation, and what is the right one?

<details>
<summary>Answer</summary>

Two nonterminals deriving the same string is not ambiguity: ambiguity needs one *sentence* with two trees, and every sentence of `rr.g` (`a c d`, `b c d`, `a c e`, `b c e`) has exactly one, since the token after `c` decides between A and B. The canonical LR(1) table has no conflict, which proves the grammar is unambiguous. The conflict comes from merging I6 and I9, which have the same cores but opposite lookaheads, so the merged state reduces by both A -> c and B -> c under both `d` and `e`.

</details>

## 7.7 The LR driver in C: one table, one stack

### A table nobody reads yet

Sections 7.4 and 7.5 built three kinds of table, and every one of them is still only printed. The lab question is "implement an SLR parser for the expression grammar", and the paper's is "using the table, show the moves of the parser on `id * id + id`": the stack, the input and the action at every move, as `pp` printed for the predictive parser in section 6.4. Both need the program that reads a table and parses.

That program is the same for all three kinds of table. SLR, CLR and LALR differ only in how the entries were worked out; the parser that follows them is one loop, the *LR driver*. This section writes it twice, as section 6.4 did for the top-down driver: first `lr.c`, the lab program, with the expression grammar's SLR table typed in; then `lrparse.c`, which builds any of the three tables for any grammar and parses with it, so that you can check every trace you work by hand.

### Plates with a square written on them

The stack of plates is back for the last time in this chapter, and this is where the picture breaks for good. In `sr.c` each plate carried a grammar symbol, and the parser had to read several plates, compare them with every right side and check FOLLOW, to decide each move. An LR parser writes on every plate the square of the board game it was on after that plate went down: a state number from the collection. The state on the top plate sums up everything below it, because it holds exactly the items that are valid for the whole stack, the viable prefix of section 7.3. So the parser decides every move from two things only: the state on the top plate and the next token. One look in the table, no searching.

The symbols are still useful to people, so the trace keeps them, but the parser never reads them: the states alone are enough to parse.

### The driver: one loop, four cases

<!-- teaches: lr-driver = LR parsing algorithm, LR driver, configuration of an LR parser -->
The *LR parsing algorithm* keeps a stack that starts as state 0, and the input with `$` at its end. At every move, let s be the state on top and a the next token, and look up ACTION[s, a]:

1. **shift t**: push a and then t; a is used up, and the next token comes into view.
2. **reduce by A -> β**: pop |β| symbols with their states (one pair for each symbol of β, none for A -> ε). The state now on top, call it u, is the state the parser was in before it began reading β. Push A, and then GOTO[u, A]. No input is used.
3. **accept**: stop; the input is a sentence.
4. **error** (a blank): stop and report it.

The stack and the rest of the input together are called the parser's *configuration*, and a trace is the list of configurations, one per move, with the action taken in each.

Case 2 is the step people get wrong, and it is where the board game goes back in time. After popping β, the parser is on the square it occupied *before* β was read, the square u. From u it now plays the card A, as if the A had come from the input, and GOTO[u, A] is where that card leads. In the expression grammar, an `E` made from `id + id` lands in state 1 if it started at state 0 (the whole input) and in state 8 if it started at state 4 (inside brackets). The uncovered state is how the parser knows which.

### lr.c: the SLR parser, table typed in

The table is section 7.4's, in lab-sheet style: one character per token, `i` for `id`. ACTION entries are strings, `"s5"`, `"r2"`, `"acc"`, or `""` for a blank, because that is how the table looks on paper and the program can read the number with `atoi`. GOTO entries are numbers, -1 for a blank. Save this as `lr.c`:

<!-- code: id=lr mode=compile -->
```c
/* lr.c: an SLR parser for the expression grammar, table typed in
       1 E -> E + T    2 E -> T     3 T -> T * F
       4 T -> F        5 F -> ( E ) 6 F -> i
   i stands for id. Reads one line; every character but a blank is one
   token, and $ is added at the end. Prints the stack (states and
   symbols), the rest of the input and the action at every move. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

const char *ts = "i+*()$";              /* ACTION columns */
const char *nts = "ETF";                /* GOTO columns */
const char *ACTION[12][6] = {
    /*        i     +     *     (     )      $   */
    /*  0 */ {"s5", "",   "",   "s4", "",    ""   },
    /*  1 */ {"",   "s6", "",   "",   "",    "acc"},
    /*  2 */ {"",   "r2", "s7", "",   "r2",  "r2" },
    /*  3 */ {"",   "r4", "r4", "",   "r4",  "r4" },
    /*  4 */ {"s5", "",   "",   "s4", "",    ""   },
    /*  5 */ {"",   "r6", "r6", "",   "r6",  "r6" },
    /*  6 */ {"s5", "",   "",   "s4", "",    ""   },
    /*  7 */ {"s5", "",   "",   "s4", "",    ""   },
    /*  8 */ {"",   "s6", "",   "",   "s11", ""   },
    /*  9 */ {"",   "r1", "s7", "",   "r1",  "r1" },
    /* 10 */ {"",   "r3", "r3", "",   "r3",  "r3" },
    /* 11 */ {"",   "r5", "r5", "",   "r5",  "r5" },
};
const int GOTO[12][3] = {               /* -1: blank */
    /*         E   T   F  */
    /*  0 */ { 1,  2,  3 },
    /*  1 */ {-1, -1, -1 },
    /*  2 */ {-1, -1, -1 },
    /*  3 */ {-1, -1, -1 },
    /*  4 */ { 8,  2,  3 },
    /*  5 */ {-1, -1, -1 },
    /*  6 */ {-1,  9,  3 },
    /*  7 */ {-1, -1, 10 },
    /*  8 */ {-1, -1, -1 },
    /*  9 */ {-1, -1, -1 },
    /* 10 */ {-1, -1, -1 },
    /* 11 */ {-1, -1, -1 },
};
const char lhs[7] = { 0, 'E', 'E', 'T', 'T', 'F', 'F' };
const char *rhs[7] = { "", "E+T", "T", "T*F", "F", "(E)", "i" };

int state[100];                         /* the stack: state[0] .. state[top] */
char sym[100];                          /* sym[k] is the symbol under state[k] */
int top;
char input[100];
int moves;

void show(int ip, const char *action)
{
    char s[400];
    int k, n;

    n = sprintf(s, "%d", state[0]);
    for (k = 1; k <= top; k++)
        n += sprintf(s + n, " %c %d", sym[k], state[k]);
    printf("%3d  %-26s%14s   %s\n", ++moves, s, input + ip, action);
}

int main(void)
{
    char line[100], act[40];
    const char *e, *col;
    int ip = 0, n = 0, i, p, g;

    if (fgets(line, sizeof line, stdin) == NULL) {
        fprintf(stderr, "lr: no input\n");
        return 1;
    }
    for (i = 0; line[i] != '\0' && line[i] != '\n'; i++)
        if (line[i] != ' ' && line[i] != '\t' && n < 98)
            input[n++] = line[i];
    input[n] = '$';
    printf("%3s  %-26s%14s   %s\n", "", "STACK", "INPUT", "ACTION");
    state[0] = 0;                       /* the stack starts as state 0 */
    for (;;) {
        col = strchr(ts, input[ip]);
        e = (col == NULL) ? "" : ACTION[state[top]][col - ts];
        if (e[0] == 's' && top < 98) {  /* shift: push a and the state */
            sprintf(act, "shift %d", atoi(e + 1));
            show(ip, act);
            top++;
            sym[top] = input[ip++];
            state[top] = atoi(e + 1);
        } else if (e[0] == 'r') {       /* reduce by A -> beta */
            p = atoi(e + 1);
            sprintf(act, "reduce by %c -> %s", lhs[p], rhs[p]);
            show(ip, act);
            top -= strlen(rhs[p]);      /* pop |beta| states */
            g = GOTO[state[top]][strchr(nts, lhs[p]) - nts];
            top++;
            sym[top] = lhs[p];          /* push A and GOTO[s, A] */
            state[top] = g;
        } else if (e[0] == 'a') {
            show(ip, "accept");
            return 0;
        } else {
            sprintf(act, "error: ACTION[%d, %c] is blank", state[top],
                    input[ip]);
            show(ip, act);
            printf("rejected\n");
            return 1;
        }
    }
}
```

The two tables are section 7.4's table split in two, with the rows in state order and the columns named by `ts` and `nts`; looking up a column is the `strchr` and pointer difference of `pp.c`. The productions are two arrays indexed by production number, with index 0 left empty so that `rhs[2]` is production 2. The length of a right side, |β|, is `strlen(rhs[p])`, since every symbol is one character.

The stack is two arrays side by side: `state[k]` is the state on plate k and `sym[k]` the symbol that was pushed with it; plate 0 has state 0 and no symbol. That is why a reduce pops `strlen(rhs[p])` plates and not twice that: one plate holds a symbol and its state together. `show` prints the plates bottom first, state 0, then each symbol followed by its state, so a stack reads `0 E 1 + 6`.

The loop is the four cases. A token that is not in `ts` gives an empty entry, an error. In the reduce branch, the order of the lines is the algorithm's: pop first (`top -= ...`), then read the uncovered state `state[top]`, then look up GOTO for that state and the left side, then push.

### The running expression, move by move

<!-- code: id=lr-run -->
```console
$ gcc -Wall -Wextra lr.c -o lr
$ echo 'i + i * i' | ./lr
     STACK                              INPUT   ACTION
  1  0                                 i+i*i$   shift 5
  2  0 i 5                              +i*i$   reduce by F -> i
  3  0 F 3                              +i*i$   reduce by T -> F
  4  0 T 2                              +i*i$   reduce by E -> T
  5  0 E 1                              +i*i$   shift 6
  6  0 E 1 + 6                           i*i$   shift 5
  7  0 E 1 + 6 i 5                        *i$   reduce by F -> i
  8  0 E 1 + 6 F 3                        *i$   reduce by T -> F
  9  0 E 1 + 6 T 9                        *i$   shift 7
 10  0 E 1 + 6 T 9 * 7                     i$   shift 5
 11  0 E 1 + 6 T 9 * 7 i 5                  $   reduce by F -> i
 12  0 E 1 + 6 T 9 * 7 F 10                 $   reduce by T -> T*F
 13  0 E 1 + 6 T 9                          $   reduce by E -> E+T
 14  0 E 1                                  $   accept
```

Fourteen moves. Put this trace beside `sr`'s from section 7.1: the actions are the same, in the same order, and the symbols on the stack are the same at every move. The LR parser is `sr` with a state on every plate, and the states made every decision that `sr` needed FOLLOW and a search of the stack for.

Walk the moves where the states earn their keep.

- **Move 4**, `0 T 2` with `+` next: ACTION[2, +] = r2, reduce by E -> T. Pop one plate (`T 2`); state 0 is uncovered; GOTO[0, E] = 1; push `E 1`. The next line shows `0 E 1`.
- **Move 9**, `0 E 1 + 6 T 9` with `*` next: ACTION[9, *] = s7. This is the decision at move 9 of section 7.1, where `sr -g` went wrong. State 9 holds `E -> E + T .` and `T -> T . * F`, and the table entry for `*` is the shift. No search for right sides, no FOLLOW test at run time; the work was done when the table was built.
- **Move 12**, `... + 6 T 9 * 7 F 10` with `$` next: ACTION[10, $] = r3, reduce by T -> T * F. Pop three plates (`F 10`, `* 7`, `T 9`); state 6 is uncovered; GOTO[6, T] = 9. The stack is `0 E 1 + 6 T 9` again, with a bigger T.
- **Move 13**: ACTION[9, $] = r1, reduce by E -> E + T. Pop three plates; state 0 is uncovered; GOTO[0, E] = 1.
- **Move 14**: ACTION[1, $] = acc.

The paper's layout differs from this one in a small way. The Dragon book's second edition keeps the states in a STACK column and the symbols in a separate SYMBOLS column; many answer keys interleave them as here, often without blanks (`0E1+6T9`).<!-- risk: Dragon 2nd ed. LR trace layout (separate STACK and SYMBOLS columns); check the figure for the moves of an LR parser on id * id + id --> Both are right; say which you are using, and keep the states, since they are what the parser reads.

### When the input is wrong

The same four inputs `sr` rejected:

<!-- code: id=lr-errors exit=1 -->
```console
$ for s in 'i+*i' 'i i' '(i' ''; do echo "== $s"; echo "$s" | ./lr; done
== i+*i
     STACK                              INPUT   ACTION
  1  0                                  i+*i$   shift 5
  2  0 i 5                               +*i$   reduce by F -> i
  3  0 F 3                               +*i$   reduce by T -> F
  4  0 T 2                               +*i$   reduce by E -> T
  5  0 E 1                               +*i$   shift 6
  6  0 E 1 + 6                            *i$   error: ACTION[6, *] is blank
rejected
== i i
     STACK                              INPUT   ACTION
  1  0                                    ii$   shift 5
  2  0 i 5                                 i$   error: ACTION[5, i] is blank
rejected
== (i
     STACK                              INPUT   ACTION
  1  0                                    (i$   shift 4
  2  0 ( 4                                 i$   shift 5
  3  0 ( 4 i 5                              $   reduce by F -> i
  4  0 ( 4 F 3                              $   reduce by T -> F
  5  0 ( 4 T 2                              $   reduce by E -> T
  6  0 ( 4 E 8                              $   error: ACTION[8, $] is blank
rejected
==
     STACK                              INPUT   ACTION
  1  0                                      $   error: ACTION[0, $] is blank
rejected
```

`i+*i` is rejected at move 6, with the `*` in hand: in state 6 (after `E +`) the table has entries only under `id` and `(`, the tokens that can begin a `T`. `sr` shifted that `*` and found out at the end of the input. `i i` is rejected at the second `i`, in state 5. This is the *valid prefix property* of LR parsers: the parser never shifts a token unless the input read so far, that token included, is the beginning of some correct sentence. So the error is reported at the first token where the input stops being the start of anything correct, and the message can name the state and the token.

The SLR and LALR tables may do a few reductions before the blank entry is reached, as in `(i`, where three reductions happen with `$` in hand before ACTION[8, $] turns out blank; the canonical LR table finds the error with no reductions at all. Even so, no LR parser ever shifts the wrong token.

### lrparse.c: any grammar, any of the three tables

To check a hand trace for a grammar of your own, you do not want to type in a table. `lrparse` builds the table with `slr.h` or `lr1.h`, by default SLR, with `-l` LALR and with `-c` canonical LR, refuses to run if the table has a conflict, and then drives it. The grammar is a file in `grammar.h`'s form, and the input is a second argument in quotes, with blanks between tokens, so that a token can be a word like `id`. Save it as `lrparse.c`:

<!-- code: id=lrparse mode=compile -->
```c
/* lrparse.c: the LR driver for any grammar. Builds the SLR(1) table,
   or with -l the LALR(1) table, or with -c the canonical LR(1) table,
   then parses one input, tokens separated by blanks, and prints every
   move.    usage: lrparse [-l | -c] grammar-file 'id + id * id' */
#include "lr1.h"

int stk[200], ssym[200], top;       /* states, and the symbol under each */
int in[100], nin;                   /* the input tokens, then $ */
int moves;

void show(int ip, const char *action)
{
    char s[2000], rest[1000];
    int k, n = 0;

    n = sprintf(s, "%s", rowname[stk[0]]);
    for (k = 1; k <= top && n < 1900; k++)
        n += sprintf(s + n, " %s %s", name[ssym[k]], rowname[stk[k]]);
    for (n = 0, k = ip; k < nin && n < 900; k++)
        n += sprintf(rest + n, "%s%s", k > ip ? " " : "", name[in[k]]);
    printf("%3d  %-30s%22s   %s\n", ++moves, s, rest, action);
}

int main(int argc, char *argv[])
{
    char line[1000], what[200], *w;
    int kind = 's', i, a, t, p, s;

    prog = "lrparse";
    if (argc == 4 && (strcmp(argv[1], "-l") == 0 || strcmp(argv[1], "-c") == 0)) {
        kind = argv[1][1];
        argv++;
        argc--;
    }
    if (argc != 3)
        die("usage: lrparse [-l | -c] grammar-file 'tokens'");
    if (freopen(argv[1], "r", stdin) == NULL)
        die("cannot open %s", argv[1]);
    readgrammar();
    augment();
    if (kind == 's') {
        collection();
        buildslr();
    } else {
        collection1();
        buildlr1(kind == 'l');
    }
    if (conflicts > 0)
        die("the %s table has %d conflict%s: not that kind of grammar",
            kind == 's' ? "SLR(1)" : kind == 'l' ? "LALR(1)" : "LR(1)",
            conflicts, conflicts == 1 ? "" : "s");
    strncpy(line, argv[2], sizeof line - 1);
    for (w = strtok(line, " \t\n"); w != NULL; w = strtok(NULL, " \t\n")) {
        for (t = 2; t < nsym && (isnt[t] || strcmp(name[t], w) != 0); t++)
            ;
        if (t == nsym)
            die("%s is not a terminal of the grammar", w);
        if (nin == 99)
            die("input too long");
        in[nin++] = t;
    }
    in[nin++] = END;
    printf("%3s  %-30s%22s   %s\n", "", "STACK", "INPUT", "ACTION");
    for (i = 0;;) {                 /* i: the next input token */
        s = stk[top];
        a = nact[s][in[i]] ? act[s][in[i]][0] : 0;
        if (KIND(a) == SHIFT && top < 198) {
            sprintf(what, "shift %s", rowname[NUM(a)]);
            show(i, what);
            top++;
            ssym[top] = in[i++];
            stk[top] = NUM(a);
        } else if (KIND(a) == REDUCE) {
            p = NUM(a);
            sprintf(what, "reduce by %s", prodtext(p));
            show(i, what);
            top -= len[p];
            s = gt[stk[top]][lhs[p]];
            top++;
            ssym[top] = lhs[p];
            stk[top] = s;
        } else if (KIND(a) == ACCEPT) {
            show(i, "accept");
            return 0;
        } else {
            sprintf(what, "error: ACTION[%s, %s] is blank", rowname[s],
                    name[in[i]]);
            show(i, what);
            printf("rejected\n");
            return 1;
        }
    }
}
```

The first half builds the table. `freopen` connects the grammar file to standard input, because `readgrammar` reads from there; then `augment`, the collection and the builder of the kind asked for. Any conflict stops the program: a table with two actions in one entry is not a table this driver can follow, and section 7.6 says what to do instead. Then the input string is cut into words with `strtok`, and each word is looked up among the grammar's terminals, so a typing slip like `ID` is caught before parsing starts.

The second half is `lr.c`'s loop on the table arrays of `slr.h`: `act[s][a][0]` is the one action in the entry (`nact` is 0 for a blank), `KIND` and `NUM` take it apart, a reduce pops `len[p]` plates, which is 0 for an ε-production, and GOTO is `gt`. The stack holds row numbers; `show` prints their names, so the LALR trace shows `36` and `47` as in the hand table.

<!-- code: id=lrparse-run -->
```console
$ gcc -Wall -Wextra lrparse.c -o lrparse
$ ./lrparse ../ch5/expr.g 'id + id * id'
     STACK                                          INPUT   ACTION
  1  0                                     id + id * id $   shift 5
  2  0 id 5                                   + id * id $   reduce by F -> id
  3  0 F 3                                    + id * id $   reduce by T -> F
  4  0 T 2                                    + id * id $   reduce by E -> T
  5  0 E 1                                    + id * id $   shift 6
  6  0 E 1 + 6                                  id * id $   shift 5
  7  0 E 1 + 6 id 5                                * id $   reduce by F -> id
  8  0 E 1 + 6 F 3                                 * id $   reduce by T -> F
  9  0 E 1 + 6 T 9                                 * id $   shift 7
 10  0 E 1 + 6 T 9 * 7                               id $   shift 5
 11  0 E 1 + 6 T 9 * 7 id 5                             $   reduce by F -> id
 12  0 E 1 + 6 T 9 * 7 F 10                             $   reduce by T -> T * F
 13  0 E 1 + 6 T 9                                      $   reduce by E -> E + T
 14  0 E 1                                              $   accept
```

The same fourteen moves as `lr`, with `id` for `i` and productions written in full. Now the grammars of sections 7.4 to 7.6. `assign.g` cannot be parsed with its SLR table, but can with LALR:

<!-- code: id=lrparse-assign -->
```console
$ ./lrparse -l assign.g 'id = * id'
     STACK                                          INPUT   ACTION
  1  0                                        id = * id $   shift 5.12
  2  0 id 5.12                                   = * id $   reduce by L -> id
  3  0 L 2                                       = * id $   shift 6
  4  0 L 2 = 6                                     * id $   shift 4.11
  5  0 L 2 = 6 * 4.11                                id $   shift 5.12
  6  0 L 2 = 6 * 4.11 id 5.12                           $   reduce by L -> id
  7  0 L 2 = 6 * 4.11 L 8.10                            $   reduce by R -> L
  8  0 L 2 = 6 * 4.11 R 7.13                            $   reduce by L -> * R
  9  0 L 2 = 6 L 8.10                                   $   reduce by R -> L
 10  0 L 2 = 6 R 9                                      $   reduce by S -> L = R
 11  0 S 1                                              $   accept
$ ./lrparse assign.g 'id = * id'; echo "exit status $?"
lrparse: the SLR(1) table has 1 conflict: not that kind of grammar
exit status 1
```

At move 3, state 2 with `=` next, the LALR table shifts: the entry that was `s6/r5` in the SLR table is `s6` alone. Moves 6 to 9 are the right side of the assignment being built up from the inside: `id` becomes L, then R, then `* R` becomes L, then R.

And the error that section 7.5 traced by hand, `S -> C C` on the input `c d`, with the canonical and the LALR table:

<!-- code: id=lrparse-cc exit=1 -->
```console
$ ./lrparse -c cc.g 'c d'; ./lrparse -l cc.g 'c d'
     STACK                                          INPUT   ACTION
  1  0                                              c d $   shift 3
  2  0 c 3                                            d $   shift 4
  3  0 c 3 d 4                                          $   error: ACTION[4, $] is blank
rejected
     STACK                                          INPUT   ACTION
  1  0                                              c d $   shift 36
  2  0 c 36                                           d $   shift 47
  3  0 c 36 d 47                                        $   reduce by C -> d
  4  0 c 36 C 89                                        $   reduce by C -> c C
  5  0 C 2                                              $   error: ACTION[2, $] is blank
rejected
```

The canonical parser stops after two shifts, in state 4, because `[C -> d ., c/d]` has no `$`. The LALR parser makes two more moves, reductions by C -> d and C -> c C, both allowed because the merged states carry `$`, and then stops in state 2, also at `$`. Neither shifted anything wrong.

### When it goes wrong

**GOTO from the state before the pop.** After reducing by T -> T * F in state 10, the GOTO must be looked up in the uncovered state, 6, not in 10. GOTO[10, T] is blank, and a program that uses it pushes -1 or crashes. Pop first, then look.

**Popping 2|β| in a two-array stack, or |β| in a one-array stack.** If the stack is one array with symbols and states alternating (`0 E 1 + 6`), a reduce pops 2|β| entries. If states and symbols are side by side, as here, it pops |β|. Mixing the two leaves half a plate behind.

**Reading `r5` as "go to state 5".** `r5` reduces by production 5; the state after the reduction comes from GOTO, never from the `r` entry. In a hand trace this shows up as a stack whose top state does not match the symbol under it: `F` with state 5 instead of 3.

**Forgetting to push the left side.** A reduce that pops β and pushes only the GOTO state leaves the symbols and states out of step. The parse may still succeed, since the parser only reads states, but the printed stack no longer shows the sentential form, and the examiner reads that column.

**An ε-production that pops something.** For A -> ε, |β| is 0: pop nothing, then push A and GOTO[s, A] for the state s on top. A driver that assumes every right side has at least one symbol pops a plate it should not.

### In one line each

1. The LR driver looks only at the state on top and the next token: shift pushes the token and a state, reduce pops |β| plates and pushes A with GOTO of the uncovered state.
2. The same driver runs SLR, LALR and CLR tables; only the entries differ.
3. `lr.c` is the lab program, with the SLR table typed in; its trace is `sr`'s with a state on every plate.
4. LR parsers detect an error at the first token that cannot continue a correct sentence; SLR and LALR may reduce first but never shift it.
5. `lrparse` builds any of the three tables for any grammar, refuses tables with conflicts, and prints the trace to check hand work.

**Check yourself**

<!-- q: idea=lr-driver type=predict wrong="every token costs exactly one move" -->
1. The Dragon book's example input is `id * id + id`. Before running anything, say how many moves the SLR parser makes (shifts, reductions and the accept), and what the stack is just before the `+` is shifted. Then run `lr`.

<details>
<summary>Answer</summary>

Shifts: one per token, 5. Reductions: F -> id for each `id` (3); T -> F for the first and the last operand (2; the middle F goes straight into T -> T * F); T -> T * F once; E -> T once, for `id * id`; E -> E + T once: 8. With the accept, 5 + 8 + 1 = 14 moves. Just before `+` is shifted the stack is `0 E 1`: the whole of `id * id` has become one E.

<!-- code: id=lr-dragon -->
```console
$ echo 'i * i + i' | ./lr
     STACK                              INPUT   ACTION
  1  0                                 i*i+i$   shift 5
  2  0 i 5                              *i+i$   reduce by F -> i
  3  0 F 3                              *i+i$   reduce by T -> F
  4  0 T 2                              *i+i$   shift 7
  5  0 T 2 * 7                           i+i$   shift 5
  6  0 T 2 * 7 i 5                        +i$   reduce by F -> i
  7  0 T 2 * 7 F 10                       +i$   reduce by T -> T*F
  8  0 T 2                                +i$   reduce by E -> T
  9  0 E 1                                +i$   shift 6
 10  0 E 1 + 6                             i$   shift 5
 11  0 E 1 + 6 i 5                          $   reduce by F -> i
 12  0 E 1 + 6 F 3                          $   reduce by T -> F
 13  0 E 1 + 6 T 9                          $   reduce by E -> E+T
 14  0 E 1                                  $   accept
```

</details>

<!-- q: idea=lr-driver type=fix wrong="the GOTO can be looked up before popping" -->
2. Find the fault. A student's reduce step is

```c
g = GOTO[state[top]][strchr(nts, lhs[p]) - nts];
top -= strlen(rhs[p]);
top++;
sym[top] = lhs[p];
state[top] = g;
```

What happens on `i + i` at the first reduction? Predict, then make the faulty copy and run it.

<details>
<summary>Answer</summary>

The GOTO is read from the state on top *before* the pop: state 5, after the shift of `i`. GOTO[5, F] is -1 (row 5 has no gotos), so the parser pushes `F` with state -1. The next move reads `ACTION[-1][...]`, outside the table: undefined behaviour, a crash or garbage. The GOTO must come from the uncovered state, after the pop.

<!-- code: id=lr-badgoto exit=139 varies -->
```console
$ sed '/pop |beta| states/{h;d;};/strchr(nts, lhs\[p\])/G' lr.c > lrbad.c
$ gcc -Wall -Wextra lrbad.c -o lrbad
$ echo 'i + i' > plus.txt
$ ./lrbad < plus.txt
     STACK                              INPUT   ACTION
  1  0                                   i+i$   shift 5
  2  0 i 5                                +i$   reduce by F -> i
Segmentation fault
```

The `sed` moves the pop line below the GOTO line, the same hold-and-append trick as in section 7.1. What you see after the first reduction depends on what lies outside the array; on this machine it was a crash. Never trust a parser whose stack ever holds a state that is not a row of the table.

</details>

<!-- q: idea=lr-driver type=apply wrong="an LR parser needs the grammar's symbols on its stack" -->
3. Use `lrparse -l` on the list grammar (`S -> ( L ) | a`, `L -> L , S | S`) with the input `( a , ( a ) )`. How many reductions are there, and why is there no reduction by a production you would expect from removing left recursion?

<details>
<summary>Answer</summary>

Seven reductions: S -> a twice (one per `a`), L -> S twice (the first element of each list), S -> ( L ) twice (one per pair of brackets) and L -> L , S once (the second element of the outer list). There is no `M -> , S M` or `M -> ε` because the left recursion was never removed: `L -> L , S` is parsed as it stands, which is the advantage of bottom-up parsing this chapter began with.

<!-- code: id=lrparse-list -->
```console
$ printf 'S -> ( L ) | a\nL -> L , S | S\n' > list.g
$ ./lrparse -l list.g '( a , ( a ) )'
     STACK                                          INPUT   ACTION
  1  0                                    ( a , ( a ) ) $   shift 26
  2  0 ( 26                                 a , ( a ) ) $   shift 37
  3  0 ( 26 a 37                              , ( a ) ) $   reduce by S -> a
  4  0 ( 26 S 5                               , ( a ) ) $   reduce by L -> S
  5  0 ( 26 L 4.10                            , ( a ) ) $   shift 9
  6  0 ( 26 L 4.10 , 9                          ( a ) ) $   shift 26
  7  0 ( 26 L 4.10 , 9 ( 26                       a ) ) $   shift 37
  8  0 ( 26 L 4.10 , 9 ( 26 a 37                    ) ) $   reduce by S -> a
  9  0 ( 26 L 4.10 , 9 ( 26 S 5                     ) ) $   reduce by L -> S
 10  0 ( 26 L 4.10 , 9 ( 26 L 4.10                  ) ) $   shift 8.12
 11  0 ( 26 L 4.10 , 9 ( 26 L 4.10 ) 8.12                   ) $   reduce by S -> ( L )
 12  0 ( 26 L 4.10 , 9 S 11                           ) $   reduce by L -> L , S
 13  0 ( 26 L 4.10                                    ) $   shift 8.12
 14  0 ( 26 L 4.10 ) 8.12                               $   reduce by S -> ( L )
 15  0 S 1                                              $   accept
```

Look at move 12: the inner list has just become an S, in state 11, and `L , S` on top is reduced to one L, which lands in the same state as the first L did. However long the list, the stack never holds more than one `L , S` at a time: left recursion keeps an LR parser's stack short, where right recursion would make it grow with the list.

</details>

## 7.8 Practice: the full bottom-up question set

You can now run a shift-reduce parse and name its handles (7.1), build an operator-precedence table from precedence rules or from LEADING and TRAILING, parse with it and find precedence functions (7.2), build the canonical LR(0) collection (7.3), the SLR table (7.4), the CLR and LALR tables (7.5), read and resolve conflicts (7.6), and run any of the tables by hand and in C (7.7). The written paper's bottom-up question is usually all of it at once: "construct the LR(0) items, build the SLR (or CLR, or LALR) table, say whether the grammar is SLR(1), and show the moves of the parser on the given input". The lab asks for a parser with its table typed in. These tasks climb from reading a table you are given, through a broken table and two grammars that test where each method stops, to a lab program from a blank file. Work in `~/cdlab/ch7`. Do each hand answer on paper first, then check it: `./lr0` for the items, `./slr` and `./lr1` for the tables, `./lrparse` for the moves, `./optable` for operator precedence.

### Guided

<!-- q: idea=lr-driver type=apply wrong="a reduction uses up an input token" -->
**1. Moves from a table you are given.** Section 5.7's boolean grammar is in `../ch5/bool.g`: `B -> B or T | T`, `T -> T and F | F`, `F -> not F | ( B ) | true | false`. Let `slr` build its table and check that it is SLR(1). Then, using only that printed table, write the SLR parser's moves on `not true and ( false or true )`: stack, input and action on every line. Before you start, say how many moves there will be. Check with `lrparse`.

<details>
<summary>Answer</summary>

Count first. Eight tokens, so 8 shifts. Reductions: five F's (F -> true twice, F -> false, F -> not F, F -> ( B )); three T -> F, for `not true`, for `false` and for the last `true` (the F made by `( B )` is the right operand of `and`, so it goes straight into T -> T and F); T -> T and F once; B -> T twice (the `false` inside the brackets, and the whole line); B -> B or T once. That is 12 reductions, and with the accept, 8 + 12 + 1 = 21 moves.

<!-- code: id=practice-bool -->
```console
$ ./slr < ../ch5/bool.g | tail -1
SLR(1): yes, no conflicts
$ ./lrparse ../ch5/bool.g 'not true and ( false or true )'
     STACK                                          INPUT   ACTION
  1  0                             not true and ( false or true ) $   shift 4
  2  0 not 4                       true and ( false or true ) $   shift 6
  3  0 not 4 true 6                and ( false or true ) $   reduce by F -> true
  4  0 not 4 F 10                  and ( false or true ) $   reduce by F -> not F
  5  0 F 3                         and ( false or true ) $   reduce by T -> F
  6  0 T 2                         and ( false or true ) $   shift 9
  7  0 T 2 and 9                      ( false or true ) $   shift 5
  8  0 T 2 and 9 ( 5                    false or true ) $   shift 7
  9  0 T 2 and 9 ( 5 false 7                  or true ) $   reduce by F -> false
 10  0 T 2 and 9 ( 5 F 3                      or true ) $   reduce by T -> F
 11  0 T 2 and 9 ( 5 T 2                      or true ) $   reduce by B -> T
 12  0 T 2 and 9 ( 5 B 11                     or true ) $   shift 8
 13  0 T 2 and 9 ( 5 B 11 or 8                   true ) $   shift 6
 14  0 T 2 and 9 ( 5 B 11 or 8 true 6                   ) $   reduce by F -> true
 15  0 T 2 and 9 ( 5 B 11 or 8 F 3                    ) $   reduce by T -> F
 16  0 T 2 and 9 ( 5 B 11 or 8 T 12                   ) $   reduce by B -> B or T
 17  0 T 2 and 9 ( 5 B 11                             ) $   shift 14
 18  0 T 2 and 9 ( 5 B 11 ) 14                          $   reduce by F -> ( B )
 19  0 T 2 and 9 F 13                                   $   reduce by T -> T and F
 20  0 T 2                                              $   reduce by B -> T
 21  0 B 1                                              $   accept
```

Two places to check your hand trace. Move 4: in state 10, `F -> not F .`, the reduce pops two plates (`not 4` and `F 10`) and uncovers state 0, so `F` goes to GOTO[0, F] = 3, not to state 10 again. And the last reductions, with `$` in hand: `F -> ( B )` uncovers state 9 (after `and`), whose GOTO on F is 13, and then `T -> T and F` pops three plates back to state 0. Every reduction leaves the input column unchanged; only shifts move it.

</details>

<!-- q: idea=operator-precedence type=apply wrong="a right-associative operator yields .> to itself" -->
**2. Operator precedence with a right-associative operator.** For `E -> E - T | T`, `T -> F ^ T | F`, `F -> ( E ) | id` (subtraction, left-associative; exponentiation, higher and right-associative): compute LEADING and TRAILING, build the operator-precedence table with the four rules, and find the precedence functions. Then write the moves of the operator-precedence parser on `id - id ^ id ^ id`, with the relation at each move, and say how the expression is grouped. Check the sets, the table and the functions with `optable`.

<details>
<summary>Answer</summary>

LEADING(F) = { (, id }, LEADING(T) = { ^, (, id }, LEADING(E) = { -, ^, (, id }; TRAILING(F) = { ), id }, TRAILING(T) = { ^, ), id }, TRAILING(E) = { -, ^, ), id }. `T -> F ^ T` gives TRAILING(F) ·> ^ and ^ <· LEADING(T), which includes `^` itself: `^ <· ^`, the right associativity. `E -> E - T` gives TRAILING(E) ·> -, which includes `-`: `- ·> -`, the left associativity.

<!-- code: id=practice-pow -->
```console
$ cat > pow.g <<'END'
E -> E - T | T
T -> F ^ T | F
F -> ( E ) | id
END
$ ./optable < pow.g
LEADING(E) = { -, ^, (, id }
TRAILING(E) = { -, ^, ), id }
LEADING(T) = { ^, (, id }
TRAILING(T) = { ^, ), id }
LEADING(F) = { (, id }
TRAILING(F) = { ), id }
       -   ^   (   )  id   $
  -   .>  <.  <.  .>  <.  .>
  ^   .>  <.  <.  .>  <.  .>
  (   <.  <.  <.   =  <.
  )   .>  .>      .>      .>
 id   .>  .>      .>      .>
  $   <.  <.  <.      <.
operator precedence: yes, one relation per entry
        -    ^    (    )   id    $
f       2    2    0    4    4    0
g       1    3    3    0    3    0
```

The moves, using the table:

| Move | Stack | Relation | Input | Action |
|---|---|---|---|---|
| 1 | `$` | <· | `id - id ^ id ^ id $` | shift |
| 2 | `$ id` | ·> | `- id ^ id ^ id $` | reduce E -> id |
| 3 | `$ E` | <· | `- id ^ id ^ id $` | shift |
| 4 | `$ E -` | <· | `id ^ id ^ id $` | shift |
| 5 | `$ E - id` | ·> | `^ id ^ id $` | reduce E -> id |
| 6 | `$ E - E` | <· | `^ id ^ id $` | shift |
| 7 | `$ E - E ^` | <· | `id ^ id $` | shift |
| 8 | `$ E - E ^ id` | ·> | `^ id $` | reduce E -> id |
| 9 | `$ E - E ^ E` | <· | `^ id $` | shift (`^ <· ^`) |
| 10 | `$ E - E ^ E ^` | <· | `id $` | shift |
| 11 | `$ E - E ^ E ^ id` | ·> | `$` | reduce E -> id |
| 12 | `$ E - E ^ E ^ E` | ·> | `$` | reduce E -> E ^ E (the right one) |
| 13 | `$ E - E ^ E` | ·> | `$` | reduce E -> E ^ E |
| 14 | `$ E - E` | ·> | `$` | reduce E -> E - E |
| 15 | `$ E` | | `$` | accept |

Grouping: `id - (id ^ (id ^ id))`. Move 9 is the right associativity at work: with `E ^ E` on the stack and another `^` next, the parser shifts instead of reducing. At move 12 the walk down from the top `^` meets the other `^`, and `^ <· ^` stops it, so the handle is only the top `E ^ E`.

</details>

<!-- q: idea=slr-table type=fix wrong="a wrong table entry makes every input fail" -->
**3. A table with one wrong entry.** A student's `lr.c` has row 9 typed as `{"", "r1", "r1", "", "r1", "r1"}`. Which of `i+i*i`, `i*i+i` and `i+i` are still accepted? For the one that fails, at which move, and which rule of section 7.4 did the student break? Predict, then make the faulty copy and run it.

<details>
<summary>Answer</summary>

The `*` column of row 9 should be s7, from the item `T -> T . * F` in I9 (rule 1). The student put the reduce by E -> E + T there as well, as if `*` were in FOLLOW(E). Only an input that reaches state 9 with `*` next is affected: a `*` straight after `E + T`. So `i*i+i` and `i+i` are accepted, and `i+i*i` fails: at move 9 the parser reduces `E + T` to E, and in state 1 the `*` has no entry.

<!-- code: id=practice-row9 -->
```console
$ sed 's|/\*  9 \*/ {"",   "r1", "s7",|/*  9 */ {"",   "r1", "r1",|' lr.c > lr9.c
$ gcc -Wall -Wextra lr9.c -o lr9
$ for s in 'i+i*i' 'i*i+i' 'i+i'; do echo "== $s"; echo "$s" | ./lr9 | tail -2; done
== i+i*i
 10  0 E 1                                *i$   error: ACTION[1, *] is blank
rejected
== i*i+i
 13  0 E 1 + 6 T 9                          $   reduce by E -> E+T
 14  0 E 1                                  $   accept
== i+i
  9  0 E 1 + 6 T 9                          $   reduce by E -> E+T
 10  0 E 1                                  $   accept
```

It is the greedy mistake of `sr -g` in section 7.1, written into a table. Habit 7 catches it: compare a typed table with `slr`'s before trusting it.

</details>

### On your own

<!-- q: idea=lr-conflict type=decide unseen="lalr-not-slr" wrong="if SLR fails, the grammar needs CLR" -->
**4. Where does SLR stop?** Classify the grammar

```text
S -> A a | b A c | d c | b d a
A -> d
```

Is it LR(0)? SLR(1)? LALR(1)? LR(1)? For each "no", name the state, the items and the token of one conflict, and explain it the way the paper wants. Check with the tools.

<details>
<summary>Answer</summary>

Not LR(0): I4 = goto(I0, d) = { S -> d . c, A -> d . } mixes a shift with a complete item. Not SLR(1): FOLLOW(A) = { a, c } (from `A a` and `b A c`), so the reduce by A -> d goes under `c` in I4, where the item `S -> d . c` shifts `c`: ACTION[4, c] = s8/r5. The same happens in I7 = goto(I3, d) = { S -> b d . a, A -> d . } under `a`: s10/r5. But in I4, reached by `d` from the start, an A could only be the A of `A a`, followed by `a`; and in I7, after `b d`, only the A of `b A c`, followed by `c`. LR(1) items carry exactly that: `[A -> d ., a]` in I4 and `[A -> d ., c]` in I7, so neither reduce meets its shift. LALR(1): yes, since no two states with the same cores have conflicting lookaheads to merge. LR(1): yes, as every LALR(1) grammar is.

<!-- code: id=practice-lalr -->
```console
$ cat > lalr.g <<'END'
S -> A a | b A c | d c | b d a
A -> d
END
$ ./slr < lalr.g | tail -3
conflict: ACTION[4, c] = s8/r5
conflict: ACTION[7, a] = s10/r5
SLR(1): no, 2 entries with two or more actions
$ ./lr1 -l < lalr.g | tail -1
LALR(1): yes, no conflicts
$ ./lr1 < lalr.g | sed -n '/^I4:/,/^$/p'
I4:
    [S -> d . c, $]
    [A -> d ., a]
  goto(I4, c) = I8
```

The usual wrong answer is "SLR fails, so the grammar needs CLR": try LALR first; here it is enough, and Yacc would build it without a complaint.

</details>

<!-- q: idea=lr1-item type=apply wrong="every LL(1) grammar is SLR(1)" -->
**5. An LL(1) grammar that SLR cannot parse.** For

```text
S -> A a A b | B b B a
A -> eps
B -> eps
```

show that it is LL(1) (use `../ch6/ll1`). Build its SLR table and explain the conflict. Then build I0 of the canonical LR(1) collection by hand and say why the LALR table has no conflict. Finally, write the LALR parser's moves on `b a`, where four of the six moves read no input.

<details>
<summary>Answer</summary>

LL(1): the two S productions start with `a` and `b`, so M[S, a] and M[S, b] each get one production, and A and B vanish under their FOLLOW sets. SLR: I0 holds the complete items `A -> .` and `B -> .` (closure of `S -> . A a A b` and `S -> . B b B a`), and FOLLOW(A) = FOLLOW(B) = { a, b }, so both reductions go under both tokens: two reduce-reduce conflicts in state 0. LR(1): I0 = { [S' -> . S, $], [S -> . A a A b, $], [S -> . B b B a, $], [A -> ., a], [B -> ., b] }. The lookahead of A's item is FIRST(`a A b $`) = { a }, and of B's FIRST(`b B a $`) = { b }: reduce by A -> ε only under `a`, by B -> ε only under `b`. No conflict, and none appears later.

<!-- code: id=practice-ab -->
```console
$ cat > ab.g <<'END'
S -> A a A b | B b B a
A -> eps
B -> eps
END
$ ../ch6/ll1 < ab.g | tail -1
LL(1): yes, no entry has two productions
$ ./slr < ab.g | tail -3
conflict: ACTION[0, a] = r3/r4
conflict: ACTION[0, b] = r3/r4
SLR(1): no, 2 entries with two or more actions
$ ./lrparse -l ab.g 'b a'
     STACK                                          INPUT   ACTION
  1  0                                              b a $   reduce by B -> eps
  2  0 B 3                                          b a $   shift 5
  3  0 B 3 b 5                                        a $   reduce by B -> eps
  4  0 B 3 b 5 B 7                                    a $   shift 9
  5  0 B 3 b 5 B 7 a 9                                  $   reduce by S -> B b B a
  6  0 S 1                                              $   accept
```

In the trace, moves 1 and 3 reduce by B -> ε: they pop nothing and push `B` with its GOTO state, with the input unchanged. Moves 5 and 6 are the reduce by S -> B b B a and the accept, also with no input read. Only moves 2 and 4 shift. So a bottom-up parser is not always the stronger choice for a given method: this grammar suits LL(1) and LALR(1), but not SLR(1).

</details>

### A lab program from a blank file

<!-- q: idea=lr-driver type=build wrong="the GOTO is taken from the state that was on top before the reduction" -->
**6. An SLR parser for S -> C C.** For the grammar `S -> C C`, `C -> c C | d` (productions 1 S -> C C, 2 C -> c C, 3 C -> d), build the canonical LR(0) collection and the SLR table by hand, and check the table with `slr`. Then write `lrcc.c` from a blank file: an SLR parser in the style of `lr.c` with this table typed in. It reads one line, every character but a blank is one token, and `$` is added at the end. It prints the header line and one numbered line per move with exactly `lr.c`'s `printf` formats: the move number, the stack (state 0, then each symbol followed by its state, separated by blanks), the rest of the input, and the action, which is `shift N`, `reduce by A -> rhs` (the right side written without blanks, as in `lr.c`), `accept`, or `error: ACTION[N, c] is blank`. After an error it prints `rejected` and exits with status 1. With no input line at all it prints `lrcc: no input` to standard error and exits with status 1.

**Done when** `./lrcc` on the input `c c d d` prints exactly the model's output below. You can test that without anyone marking it: in the understudy folder (the one that holds `book/`), in the terminal, run `python3 scripts/us_practice.py check compiler-design 7.8 --in ~/cdlab/ch7`. It builds your `lrcc.c` and compares what it prints for this input with the model. Then test it, as habit 3 asks, on an empty line and on `d d d`, which must be rejected, and on `d d`, which must be accepted.

<details>
<summary>Answer</summary>

The hand work first. The LR(0) collection has seven states: I0 = { S' -> . S, S -> . C C, C -> . c C, C -> . d }; I1 = { S' -> S . }; I2 = goto(I0, C) = { S -> C . C, C -> . c C, C -> . d }; I3 = goto(I0, c) = { C -> c . C, C -> . c C, C -> . d }; I4 = goto(I0, d) = { C -> d . }; I5 = goto(I2, C) = { S -> C C . }; I6 = goto(I3, C) = { C -> c C . }. From I2 and I3, `c` leads to I3 and `d` to I4. FOLLOW(S) = { $ }, FOLLOW(C) = { c, d, $ }.

| State | c | d | $ | S | C |
|---|---|---|---|---|---|
| 0 | s3 | s4 | | 1 | 2 |
| 1 | | | acc | | |
| 2 | s3 | s4 | | | 5 |
| 3 | s3 | s4 | | | 6 |
| 4 | r3 | r3 | r3 | | |
| 5 | | | r1 | | |
| 6 | r2 | r2 | r2 | | |

<!-- code: id=practice-cc -->
```console
$ ./slr < cc.g
  0  S' -> S
  1  S -> C C
  2  C -> c C
  3  C -> d
      | c   | d   | $   | S  | C
0     | s3  | s4  |     | 1  | 2
1     |     |     | acc |    |
2     | s3  | s4  |     |    | 5
3     | s3  | s4  |     |    | 6
4     | r3  | r3  | r3  |    |
5     |     |     | r1  |    |
6     | r2  | r2  | r2  |    |
SLR(1): yes, no conflicts
```

The program is `lr.c` with a new table, new `ts`, `nts`, `lhs` and `rhs`, and a new name in the "no input" message; the driver does not change, which is the point of section 7.7.

<!-- code: id=lrcc check="gcc -Wall -Wextra lrcc.c -o lrcc && ./lrcc" stdin="c c d d\n" -->
```c
/* lrcc.c: an SLR parser for the grammar
       1 S -> C C      2 C -> c C      3 C -> d
   Reads one line; every character but a blank is one token, and $ is
   added at the end. Prints the stack (states and symbols), the rest of
   the input and the action at every move. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

const char *ts = "cd$";                 /* ACTION columns */
const char *nts = "SC";                 /* GOTO columns */
const char *ACTION[7][3] = {
    /*        c     d     $   */
    /* 0 */ {"s3", "s4", ""   },
    /* 1 */ {"",   "",   "acc"},
    /* 2 */ {"s3", "s4", ""   },
    /* 3 */ {"s3", "s4", ""   },
    /* 4 */ {"r3", "r3", "r3" },
    /* 5 */ {"",   "",   "r1" },
    /* 6 */ {"r2", "r2", "r2" },
};
const int GOTO[7][2] = {                /* -1: blank */
    /*        S   C  */
    /* 0 */ { 1,  2 },
    /* 1 */ {-1, -1 },
    /* 2 */ {-1,  5 },
    /* 3 */ {-1,  6 },
    /* 4 */ {-1, -1 },
    /* 5 */ {-1, -1 },
    /* 6 */ {-1, -1 },
};
const char lhs[4] = { 0, 'S', 'C', 'C' };
const char *rhs[4] = { "", "CC", "cC", "d" };

int state[100];                         /* the stack: state[0] .. state[top] */
char sym[100];                          /* sym[k] is the symbol under state[k] */
int top;
char input[100];
int moves;

void show(int ip, const char *action)
{
    char s[400];
    int k, n;

    n = sprintf(s, "%d", state[0]);
    for (k = 1; k <= top; k++)
        n += sprintf(s + n, " %c %d", sym[k], state[k]);
    printf("%3d  %-26s%14s   %s\n", ++moves, s, input + ip, action);
}

int main(void)
{
    char line[100], act[40];
    const char *e, *col;
    int ip = 0, n = 0, i, p, g;

    if (fgets(line, sizeof line, stdin) == NULL) {
        fprintf(stderr, "lrcc: no input\n");
        return 1;
    }
    for (i = 0; line[i] != '\0' && line[i] != '\n'; i++)
        if (line[i] != ' ' && line[i] != '\t' && n < 98)
            input[n++] = line[i];
    input[n] = '$';
    printf("%3s  %-26s%14s   %s\n", "", "STACK", "INPUT", "ACTION");
    state[0] = 0;                       /* the stack starts as state 0 */
    for (;;) {
        col = strchr(ts, input[ip]);
        e = (col == NULL) ? "" : ACTION[state[top]][col - ts];
        if (e[0] == 's' && top < 98) {  /* shift: push a and the state */
            sprintf(act, "shift %d", atoi(e + 1));
            show(ip, act);
            top++;
            sym[top] = input[ip++];
            state[top] = atoi(e + 1);
        } else if (e[0] == 'r') {       /* reduce by A -> beta */
            p = atoi(e + 1);
            sprintf(act, "reduce by %c -> %s", lhs[p], rhs[p]);
            show(ip, act);
            top -= strlen(rhs[p]);      /* pop |beta| states */
            g = GOTO[state[top]][strchr(nts, lhs[p]) - nts];
            top++;
            sym[top] = lhs[p];          /* push A and GOTO[s, A] */
            state[top] = g;
        } else if (e[0] == 'a') {
            show(ip, "accept");
            return 0;
        } else {
            sprintf(act, "error: ACTION[%d, %c] is blank", state[top],
                    input[ip]);
            show(ip, act);
            printf("rejected\n");
            return 1;
        }
    }
}
```
```text
     STACK                              INPUT   ACTION
  1  0                                  ccdd$   shift 3
  2  0 c 3                               cdd$   shift 3
  3  0 c 3 c 3                            dd$   shift 4
  4  0 c 3 c 3 d 4                         d$   reduce by C -> d
  5  0 c 3 c 3 C 6                         d$   reduce by C -> cC
  6  0 c 3 C 6                             d$   reduce by C -> cC
  7  0 C 2                                 d$   shift 4
  8  0 C 2 d 4                              $   reduce by C -> d
  9  0 C 2 C 5                              $   reduce by S -> CC
 10  0 S 1                                  $   accept
```

Moves 5 and 6 are the ones to check in your own run: each reduces by C -> c C, and each uncovers state 3 or 0. After move 5 the uncovered state is 3 (the first `c`), so the new C goes to GOTO[3, C] = 6, and move 6 reduces again; after move 6 the uncovered state is 0, so the C goes to GOTO[0, C] = 2. A program that takes the GOTO from the state before the pop fails exactly here.

The tests that must be rejected, the shortest sentence, and no input:

<!-- code: id=lrcc-tests -->
```console
$ gcc -Wall -Wextra lrcc.c -o lrcc
$ echo 'd d' | ./lrcc
     STACK                              INPUT   ACTION
  1  0                                    dd$   shift 4
  2  0 d 4                                 d$   reduce by C -> d
  3  0 C 2                                 d$   shift 4
  4  0 C 2 d 4                              $   reduce by C -> d
  5  0 C 2 C 5                              $   reduce by S -> CC
  6  0 S 1                                  $   accept
$ echo 'd d d' | ./lrcc | tail -2
  5  0 C 2 C 5                             d$   error: ACTION[5, d] is blank
rejected
$ echo '' | ./lrcc; echo "exit status $?"
     STACK                              INPUT   ACTION
  1  0                                      $   error: ACTION[0, $] is blank
rejected
exit status 1
$ ./lrcc < /dev/null; echo "exit status $?"
lrcc: no input
exit status 1
```

`d d d` is caught at the third `d`, in state 5: after `S -> C C .` only `$` may come. The empty line is caught at once, since ACTION[0, $] is blank: every sentence has at least two `d`'s. The likeliest mistake in this task is the table, not the code: putting r3 only under `c` and `d` (as the CLR row 4 would have it) and forgetting `$`, which makes the second C's `d` unreducible, so that every correct input is rejected at its end.

</details>
