# Chapter 10: Optimisation, code generation and the run-time picture

<!-- contents -->
- [10.1 Basic blocks, leaders and flow graphs, in C](#101-basic-blocks-leaders-and-flow-graphs-in-c)
- [10.2 Local optimisation: constant folding, propagation, common subexpressions, dead code](#102-local-optimisation-constant-folding-propagation-common-subexpressions-dead-code)
- [10.3 The DAG of a basic block](#103-the-dag-of-a-basic-block)
- [10.4 Loop optimisation: code motion, induction variables, strength reduction](#104-loop-optimisation-code-motion-induction-variables-strength-reduction)
- [10.5 Peephole optimisation](#105-peephole-optimisation)
- [10.6 A simple code generator: register and address descriptors, getreg](#106-a-simple-code-generator-register-and-address-descriptors-getreg)
- [10.7 Activation records and storage allocation](#107-activation-records-and-storage-allocation)
- [10.8 Practice: the mini compiler, end to end](#108-practice-the-mini-compiler-end-to-end)
<!-- /contents -->

## 10.1 Basic blocks, leaders and flow graphs, in C

### Code that is correct and slow

Chapter 9 ended with a translator that works: `tiny-tac` turns any Tiny program into three-address code, and `tacrun` proves the code computes what the program says. Look at what it wrote for the running statement, though: `t1 = rate * 60`, `t2 = init + t1`, `pos = t2`. With `init` 10 and `rate` 2.5 written just above, all three lines compute a number that was known before the program ran, 160. Chapter 1's optimiser phase removed one copy and folded `inttofloat(60)` to `60.0`; this chapter builds that phase for Tiny and then the last station of the assembly line, the code generator.

Before an optimiser may change anything, it must know which instructions *always run together*. `pos = t2` can only be rewritten using what `t2 = init + t1` stored if nothing can run between them, and a jump into the middle, or out of it, would break that. So the first job is to cut the code into pieces that are entered only at the top and left only at the bottom. Those pieces are called basic blocks, and the map of how control passes between them is the flow graph. Every optimisation in this chapter works on one or the other, and the paper asks for both by hand: "Partition the three-address code into basic blocks and draw the flow graph."

### Straight runs and the junctions between them

Think of the code as a railway line with points. A basic block is a stretch of track with no points in it: a train that enters at one end must run to the other. Points are where track joins (a label, which a jump can arrive at) or splits (a jump, which may leave). Cut the line at every set of points, and every piece is a straight run.

The picture breaks at one place. A railway map is drawn once; the flow graph is drawn from the code, and the code is what an optimiser changes. Removing a jump can join two blocks into one, and section 10.5's peephole optimiser does exactly that. The blocks belong to the code as it is at that moment.

### Leaders and blocks

<!-- teaches: basic-block = basic block -->
A *basic block* is a sequence of consecutive instructions with two properties: control can enter only at the first instruction (no jump lands in the middle), and leaves only after the last (no jump or halt before it). <!-- teaches: leader = leader -->
To find the blocks, find their first instructions, the *leaders*, with three rules:

1. The first instruction of the program is a leader.
2. Any instruction that is the target of a jump (conditional or not) is a leader.
3. Any instruction that immediately follows a jump is a leader.

Each block then runs from one leader up to, but not including, the next leader (or to the end of the program). Rule 2 cuts where track joins, rule 3 where it splits; together they guarantee the two properties.

In code with labels, as `tiny-tac` writes it, a target is the label line `L1:`, which belongs to the instruction after it; in code with numbered instructions, the target is the instruction with that number. The rules are the same.

Here is `gcd.tiny` from section 9.7, as `tiny-tac` translated it, with the lines numbered:

```text
 1  a = 48
 2  b = 18
 3  L1:
 4  ifFalse a != b goto L2
 5  ifFalse a > b goto L3
 6  t1 = a - b
 7  a = t1
 8  goto L4
 9  L3:
10  t2 = b - a
11  b = t2
12  L4:
13  goto L1
14  L2:
```

Apply the rules. Rule 1: line 1. Rule 2: the targets `L1`, `L2`, `L3` and `L4`, lines 3, 14, 9 and 12. Rule 3: the lines after the four jumps (lines 4, 5, 8 and 13): lines 5, 6, 9 and 14. In order, the leaders are 1, 3, 5, 6, 9, 12 and 14, and the blocks:

| Block | Lines | Instructions |
|---|---|---|
| B1 | 1–2 | `a = 48`, `b = 18` |
| B2 | 3–4 | `L1:`, `ifFalse a != b goto L2` |
| B3 | 5 | `ifFalse a > b goto L3` |
| B4 | 6–8 | `t1 = a - b`, `a = t1`, `goto L4` |
| B5 | 9–11 | `L3:`, `t2 = b - a`, `b = t2` |
| B6 | 12–13 | `L4:`, `goto L1` |
| B7 | 14 | `L2:` (the exit) |

Line 4 is a jump, but it is not a leader: it is the last instruction of B2. Jumps *end* blocks; their targets and successors *start* them.

### The flow graph

<!-- teaches: flow-graph = flow graph, successor, predecessor -->
A *flow graph* has one node per basic block and an edge from block B to block C when C can run right after B. There are two ways that happens:

- B ends in a jump (conditional or not) to the first instruction of C;
- B does not end in an unconditional `goto`, and C comes right after B in the code: B *falls through* to C.

C is then a *successor* of B, and B a *predecessor* of C. A conditional jump gives its block two successors: its target, and the block below it. Here is the flow graph of `gcd.tiny`:

<!-- fig: id=gcd-flow
title: Flow graph of gcd.tiny
kind: flow
dir: down
box b1: B1\na = 48, b = 18
box b2: B2\nL1: ifFalse a != b goto L2
box b3: B3\nifFalse a > b goto L3
box b4: B4\nt1 = a - b, a = t1, goto L4
box b5: B5\nL3: t2 = b - a, b = t2
box b6: B6\nL4: goto L1
box b7: B7\nL2: exit
b1 -> b2
b2 -> b3: a != b
b2 -> b7: a == b
b3 -> b4: a > b
b3 -> b5: a <= b
b4 -> b6
b5 -> b6
b6 -> b2
hot: b2 b3 b4 b5 b6
-->
![Flow chart, top to bottom. B1 a = 48, b = 18 leads to B2 L1: ifFalse a != b goto L2. B2 L1: ifFalse a != b goto L2 leads to B3 ifFalse a > b goto L3 (a != b). B2 L1: ifFalse a != b goto L2 leads to B7 L2: exit (a == b). B3 ifFalse a > b goto L3 leads to B4 t1 = a - b, a = t1, goto L4 (a > b). B3 ifFalse a > b goto L3 leads to B5 L3: t2 = b - a, b = t2 (a <= b). B4 t1 = a - b, a = t1, goto L4 leads to B6 L4: goto L1. B5 L3: t2 = b - a, b = t2 leads to B6 L4: goto L1. B6 L4: goto L1 leads back to B2 L1: ifFalse a != b goto L2. B2 L1: ifFalse a != b goto L2, B3 ifFalse a > b goto L3, B4 t1 = a - b, a = t1, goto L4, B5 L3: t2 = b - a, b = t2 and B6 L4: goto L1 are highlighted.](fig/10-optimisation-codegen/gcd-flow.svg)

The loop is visible as a cycle: B2 → B3 → B4 or B5 → B6 → B2. The highlighted blocks are the loop; section 10.4 finds such cycles and moves work out of them. B1 runs once, B7 once.

### blocks.c: leaders by program

Finding leaders is a lab program in its own right ("write a C program to find the leaders and the basic blocks of three-address code"), and habit 7 wants a checker for the hand work anyway. `blocks.c` reads code in `tiny-tac`'s form and prints the blocks and the flow graph. Make this chapter's folder, and copy in the two translators you will use all chapter:

<!-- code: id=ch10-setup -->
```console
$ mkdir cdlab/ch10
$ cd cdlab/ch10
$ cp ../ch9/tiny-tac ../ch9/tacrun ../ch9/gcd.tiny ../ch2/prog.tiny .
```

<!-- code: id=blocks mode=compile -->
```c
/* blocks.c: split three-address code into basic blocks and print the
   flow graph. Reads the code on standard input, one instruction per line,
   labels on lines of their own (L1:), as tiny-tac writes it. */
#include <stdio.h>
#include <string.h>

#define MAX 500

char code[MAX][80];
int n;                      /* lines read */
int leader[MAX];            /* 1 if line i starts a block */
int block[MAX];             /* the block that line i belongs to, from 1 */
int nblocks;

int islabel(int i)
{
    return code[i][strlen(code[i]) - 1] == ':';
}

int isjump(int i)           /* goto, if ... goto, ifFalse ... goto */
{
    return strncmp(code[i], "goto ", 5) == 0 || strncmp(code[i], "if", 2) == 0;
}

const char *target(int i)   /* the label a jump names: its last word */
{
    return strrchr(code[i], ' ') + 1;
}

int line_of(const char *l)  /* the line that holds "l:" */
{
    for (int i = 0; i < n; i++)
        if (islabel(i) && strncmp(code[i], l, strlen(l)) == 0
            && code[i][strlen(l)] == ':')
            return i;
    return -1;
}

int main(void)
{
    while (n < MAX && fgets(code[n], sizeof code[n], stdin)) {
        code[n][strcspn(code[n], "\n")] = '\0';
        if (code[n][0] != '\0')
            n++;
    }
    if (n == 0)
        return 0;

    /* the three rules for leaders */
    leader[0] = 1;                              /* 1: the first instruction */
    for (int i = 0; i < n; i++)
        if (isjump(i)) {
            int t = line_of(target(i));
            if (t >= 0)
                leader[t] = 1;                  /* 2: the target of a jump */
            if (i + 1 < n)
                leader[i + 1] = 1;              /* 3: the line after a jump */
        }

    for (int i = 0; i < n; i++) {               /* number the blocks */
        if (leader[i])
            nblocks++;
        block[i] = nblocks;
    }

    for (int i = 0; i < n; i++) {
        if (leader[i])
            printf("B%d:\n", block[i]);
        printf("    %s\n", code[i]);
    }

    printf("\nFlow graph\n");
    for (int b = 1; b <= nblocks; b++) {
        int last = -1;
        for (int i = 0; i < n; i++)
            if (block[i] == b)
                last = i;
        printf("B%d ->", b);
        if (isjump(last)) {                     /* the jump's target */
            int t = line_of(target(last));
            if (t >= 0)
                printf(" B%d", block[t]);
        }
        if (strncmp(code[last], "goto ", 5) != 0) {
            if (last + 1 < n)
                printf(" B%d", block[last + 1]);   /* falls through */
            else
                printf(" exit");
        }
        printf("\n");
    }
    return 0;
}
```

**Reading.** All lines are read first, blank ones skipped, because rule 2 needs to look forwards: a jump on line 13 makes line 3 a leader.

**The rules.** `leader[0] = 1` is rule 1. Then one pass over the lines: for each jump, `line_of` finds the line of its target label (rule 2), and the line after it is marked too (rule 3). `isjump` knows a jump by its first word, `goto`, `if` or `ifFalse`, and `target` takes the last word of the line, which in every jump is the label.

**Blocks.** A counter goes up at every leader, so `block[i]` is the number of the block that line `i` is in. Printing the blocks is then printing the lines, with a heading at each leader.

**The flow graph.** For each block, its last line decides the edges: a jump adds an edge to the block of its target; anything but an unconditional `goto` also falls through to the next block, or to `exit` if it is the last block.

<!-- code: id=blocks-run -->
```console
$ gcc -Wall -Wextra blocks.c -o blocks
$ ./tiny-tac < gcd.tiny | ./blocks
B1:
    a = 48
    b = 18
B2:
    L1:
    ifFalse a != b goto L2
B3:
    ifFalse a > b goto L3
B4:
    t1 = a - b
    a = t1
    goto L4
B5:
    L3:
    t2 = b - a
    b = t2
B6:
    L4:
    goto L1
B7:
    L2:

Flow graph
B1 -> B2
B2 -> B7 B3
B3 -> B5 B4
B4 -> B6
B5 -> B6
B6 -> B2
B7 -> exit
```

The seven blocks of the table, and the edges of the figure. The program prints a conditional jump's target first and its fall-through second; the figure has the same edges.

Now `prog.tiny`, the loop with the running statement before it:

<!-- code: id=blocks-prog -->
```console
$ ./tiny-tac < prog.tiny | ./blocks | sed -n '/Flow graph/,$p'
Flow graph
B1 -> B2
B2 -> B4 B3
B3 -> B2
B4 -> exit
```

Four blocks: the five straight-line instructions before the loop (B1), the loop test (B2), the loop body (B3), and the exit (B4). The body ends with `goto L1`, so B3's only successor is B2: a two-block loop.

### When it goes wrong

**Making every label a leader but forgetting rule 3.** The instruction after a conditional jump can be reached by falling through when the condition fails; it starts a new block even if no label is on it, like line 5 above.

**Putting the jump at the start of the next block.** A jump *ends* its block. The block after it starts with the instruction after the jump.

**Missing the fall-through edge.** A block ending in `ifFalse` has two successors. Only a block ending in an unconditional `goto` has one, and only a block ending the program has none but the exit.

**A block that starts with a label nobody jumps to.** Rule 2 is about targets of jumps, not labels: a label that no jump names does not start a block. `tiny-tac` never writes one, but code edited by hand can have one.

### In one line each

1. A basic block is entered only at its first instruction and left only after its last.
2. Leaders: the first instruction, every jump target, every instruction after a jump; each block runs from one leader to the next.
3. A jump ends a block; its target and the instruction after it start blocks.
4. The flow graph has an edge for each jump to its target's block, and for each fall-through to the next block.
5. A loop is a cycle in the flow graph.

**Check yourself**

<!-- q: idea=leader type=apply wrong="only labels are leaders" -->
1. Find the leaders and basic blocks of this numbered code (the paper's form), and draw its flow graph:

```text
1  i = 1
2  j = 1
3  t1 = 10 * i
4  t2 = t1 + j
5  a = t2
6  j = j + 1
7  if j <= 10 goto 3
8  i = i + 1
9  if i <= 10 goto 2
```

<details>
<summary>Answer</summary>

Leaders: 1 (rule 1), 3 and 2 (targets of the jumps at 7 and 9), and 8 (after the jump at 7). Line 10, after the jump at 9, does not exist. Blocks: B1 = {1}, B2 = {2}, B3 = {3–7}, B4 = {8–9}. Edges: B1 → B2 (fall-through); B2 → B3 (fall-through); B3 → B3 (the jump at 7 back to its own first line) and B3 → B4 (fall-through); B4 → B2 (the jump at 9) and B4 → exit. Two loops: B3 alone, the inner loop on `j`, and B2–B3–B4, the outer loop on `i`. The usual slip is to make line 3's block start at line 2, missing that line 2 is a target too.

</details>

<!-- q: idea=flow-graph type=predict wrong="the else part has no edge to the end of the if" -->
2. Predict how many basic blocks `tiny-tac` and `blocks` produce for `if (a < b) x = 1; else x = 2; y = x;`, and the edges. Then check.

<details>
<summary>Answer</summary>

<!-- code: id=blocks-if -->
```console
$ printf 'if (a < b) x = 1; else x = 2; y = x;\n' | ./tiny-tac | ./blocks
B1:
    ifFalse a < b goto L1
B2:
    x = 1
    goto L2
B3:
    L1:
    x = 2
B4:
    L2:
    y = x

Flow graph
B1 -> B3 B2
B2 -> B4
B3 -> B4
B4 -> exit
```

Four. B1 is the `ifFalse` alone; B2 is `x = 1` and `goto L2`; B3 starts at `L1:` with `x = 2`; B4 starts at `L2:` with `y = x`. Edges: B1 → B3 (the jump) and B1 → B2 (fall-through); B2 → B4 (the `goto`); B3 → B4 (fall-through); B4 → exit. The `goto L2` after the `then` part makes the block after it a new leader, which is why `x = 2` does not join B2.

</details>

## 10.2 Local optimisation: constant folding, propagation, common subexpressions, dead code

### Work the compiler can do once

Here is block B1 of `prog.tiny` again, as `tiny-tac` wrote it:

```text
init = 10
rate = 2.5
t1 = rate * 60
t2 = init + t1
pos = t2
```

Every time this program runs, the machine multiplies 2.5 by 60 and adds 10. The answer is 160 every time, and the compiler, which has the whole block in front of it, can work it out once, while compiling, and write `pos = 160`. That is the whole idea of optimisation: do at compile time what need not be done at run time, and do at run time only what must be done, once. This section builds the optimiser for Tiny, `tiny-opt.c`, the sixth file of the mini compiler, with the four transformations that work inside one basic block, the *local* optimisations. They are the first four entries of the paper's list of "machine-independent optimisations", and every one is checked here against `tacrun`, because an optimisation that changes what a program computes is a bug, however fast.

### A notebook per block

The optimiser reads a block from top to bottom, keeping a notebook of what it has learnt so far: "`rate` holds 2.5", "`t5` holds whatever `t3` holds", "`c - d` has been computed into `t3`". Each instruction is first rewritten using the notebook, and then the notebook is updated with what the instruction did. When an instruction assigns a name, every page that mentions that name is torn out, since it may no longer be true.

The notebook is thrown away at the start of every block. Inside a block, the instructions run in order, every time; at a leader, control may arrive from somewhere else, where `rate` was never 2.5. That is why the blocks of section 10.1 come first, and what *local* means: nothing learnt is carried across a block boundary. Carrying it across safely needs data-flow analysis over the flow graph, *global* optimisation, which the paper treats in theory and this book leaves out.

### The four transformations

<!-- teaches: constant-folding = constant folding, constant propagation, copy propagation -->
**Constant folding.** An operator whose operands are both constants is computed by the compiler: `t1 = 2.5 * 60` becomes `t1 = 150`.

**Constant propagation.** After `rate = 2.5`, a later use of `rate` in the same block, before `rate` is assigned again, can be replaced by `2.5`. This is what feeds folding: `t1 = rate * 60` is not foldable until propagation has turned it into `t1 = 2.5 * 60`.

**Copy propagation.** After a copy `x = y`, a later use of `x` can be replaced by `y`, as long as neither has been assigned since. Its purpose is to make the copy itself unnecessary, so that dead-code elimination can remove it.

<!-- teaches: cse = common subexpression elimination, available expression -->
**Common subexpression elimination.** If `c - d` was computed into `t3` and neither `c`, `d` nor `t3` has been assigned since, a second `t5 = c - d` becomes the copy `t5 = t3`: the expression is *available* in `t3`.

<!-- teaches: dead-code = dead code, dead-code elimination -->
**Dead-code elimination.** An instruction whose result is never used is *dead* and can be deleted. In Tiny's code the safe case is a temporary: a `t` is made by the translator for one use, so if no instruction reads it, its assignment does nothing. A source variable such as `pos` is never deleted here, because a variable's value may be used after the block, by the next block or by whoever reads the program's results.

Follow B1 through them. The notebook starts empty.

| Instruction | Rewritten as | Notebook afterwards |
|---|---|---|
| `init = 10` | `init = 10` | init holds 10 |
| `rate = 2.5` | `rate = 2.5` | init 10, rate 2.5 |
| `t1 = rate * 60` | propagate: `t1 = 2.5 * 60`; fold: `t1 = 150` | ..., t1 holds 150 |
| `t2 = init + t1` | propagate both: `t2 = 10 + 150`; fold: `t2 = 160` | ..., t2 holds 160 |
| `pos = t2` | propagate: `pos = 160` | ..., pos holds 160 |

Now `t1` and `t2` are assigned and never read: dead. Delete them, and B1 is

```text
init = 10
rate = 2.5
pos = 160
```

Three instructions, none of them arithmetic. And a common subexpression: section 9.4's `a = b * (c - d) / (c - d)` gave `t3 = c - d`, `t4 = b * t3`, `t5 = c - d`, `t6 = t4 / t5`, `a = t6`. At `t5 = c - d`, the notebook has "`c - d` is in `t3`", so it becomes `t5 = t3`, a copy; copy propagation then turns `t6 = t4 / t5` into `t6 = t4 / t3`; `t5` is dead. What is left is one subtraction instead of two.

One more step finishes chapter 1's job. `t6 = t4 / t3` followed by `a = t6`, where `t6` is read nowhere else, can be written as one instruction, `a = t4 / t3`: compute straight into the target. This is the copy that chapter 1's optimiser removed from `id1 = t3`.

### tiny-opt.c

<!-- code: id=tiny-opt mode=compile -->
```c
/* tiny-opt.c: local optimisation of three-address code, block by block:
   constant folding, constant and copy propagation, common subexpressions,
   then dead code. Reads tiny-tac's code on standard input and writes the
   optimised code in the same form. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX 500

char code[MAX][80];             /* the program, one instruction a line */
int n;
int dead[MAX];                  /* 1: line removed */

struct { char name[16], val[16]; } known[MAX];            /* name holds val */
int nknown;
struct { char op[8], a[16], b[16], in[16]; } avail[MAX];  /* in = a op b */
int navail;

int islabel(const char *s) { return s[strlen(s) - 1] == ':'; }
int isjump(const char *s)  { return strncmp(s, "goto", 4) == 0 || strncmp(s, "if", 2) == 0; }
int istemp(const char *s)  { return s[0] == 't' && isdigit((unsigned char)s[1]); }
int isconst(const char *s) { return isdigit((unsigned char)s[0]) || s[0] == '.' || s[0] == '-'; }

const char *subst(const char *x)        /* what x is known to hold */
{
    for (int i = 0; i < nknown; i++)
        if (strcmp(known[i].name, x) == 0)
            return known[i].val;
    return x;
}

void kill(const char *x)                /* x is assigned: tear out its pages */
{
    int j = 0;

    for (int i = 0; i < nknown; i++)
        if (strcmp(known[i].name, x) != 0 && strcmp(known[i].val, x) != 0)
            known[j++] = known[i];
    nknown = j;
    j = 0;
    for (int i = 0; i < navail; i++)
        if (strcmp(avail[i].a, x) != 0 && strcmp(avail[i].b, x) != 0
            && strcmp(avail[i].in, x) != 0)
            avail[j++] = avail[i];
    navail = j;
}

int fold(const char *op, double a, double b, double *v)  /* 1 if folded */
{
    if (strcmp(op, "+") == 0) *v = a + b;
    else if (strcmp(op, "-") == 0) *v = a - b;
    else if (strcmp(op, "*") == 0) *v = a * b;
    else if (strcmp(op, "/") == 0 && b != 0) *v = a / b;
    else if (strcmp(op, "<") == 0) *v = a < b;
    else if (strcmp(op, "<=") == 0) *v = a <= b;
    else if (strcmp(op, ">") == 0) *v = a > b;
    else if (strcmp(op, ">=") == 0) *v = a >= b;
    else if (strcmp(op, "==") == 0) *v = a == b;
    else if (strcmp(op, "!=") == 0) *v = a != b;
    else return 0;
    return 1;
}

void copy(int i, const char *x, const char *y)   /* line i becomes x = y */
{
    char yy[16];

    strcpy(yy, y);                      /* y may point into a table kill changes */
    kill(x);
    if (strcmp(x, yy) != 0) {
        strcpy(known[nknown].name, x);
        strcpy(known[nknown++].val, yy);
    }
    sprintf(code[i], "%s = %s", x, yy);
}

void local(int i)                       /* rewrite line i from the notebook */
{
    char w[6][16], a[16], b[16], c[16];
    double v;
    int k = sscanf(code[i], "%15s %15s %15s %15s %15s %15s",
                   w[0], w[1], w[2], w[3], w[4], w[5]);

    if (k <= 0 || islabel(code[i]) || strcmp(w[0], "goto") == 0)
        return;
    if (strcmp(w[0], "if") == 0 || strcmp(w[0], "ifFalse") == 0) {
        if (k == 6)                     /* if a relop b goto L */
            sprintf(code[i], "%s %s %s %s goto %s", w[0], subst(w[1]), w[2],
                    subst(w[3]), w[5]);
        else                            /* if a goto L */
            sprintf(code[i], "%s %s goto %s", w[0], subst(w[1]), w[3]);
        return;
    }
    if (k == 3) {                       /* x = y: propagate into y */
        copy(i, w[0], subst(w[2]));
        return;
    }
    if (k == 4) {                       /* x = minus y: as y minus "" */
        strcpy(w[4], w[3]);
        strcpy(w[3], w[2]);
        strcpy(w[2], "");
    }
    strcpy(a, w[2][0] ? subst(w[2]) : "");   /* x = y op z */
    strcpy(b, subst(w[4]));
    if (k == 4 && isconst(b)) {
        sprintf(c, "%g", -strtod(b, NULL));
        copy(i, w[0], c);               /* folding a minus */
        return;
    }
    if (k == 5 && isconst(a) && isconst(b)
        && fold(w[3], strtod(a, NULL), strtod(b, NULL), &v)) {
        sprintf(c, "%g", v);            /* constant folding */
        copy(i, w[0], c);
        return;
    }
    for (int j = 0; j < navail; j++)    /* a common subexpression? */
        if (strcmp(avail[j].op, w[3]) == 0 && strcmp(avail[j].a, a) == 0
            && strcmp(avail[j].b, b) == 0) {
            copy(i, w[0], avail[j].in);
            return;
        }
    kill(w[0]);
    if (k == 4)
        sprintf(code[i], "%s = minus %s", w[0], b);
    else
        sprintf(code[i], "%s = %s %s %s", w[0], a, w[3], b);
    if (strcmp(a, w[0]) != 0 && strcmp(b, w[0]) != 0) {
        strcpy(avail[navail].op, w[3]); /* now available in x */
        strcpy(avail[navail].a, a);
        strcpy(avail[navail].b, b);
        strcpy(avail[navail++].in, w[0]);
    }
}

int used(const char *t, int skip)       /* does any live line but skip read t? */
{
    char w[6][16];

    for (int i = 0; i < n; i++) {
        if (dead[i] || i == skip)
            continue;
        int k = sscanf(code[i], "%15s %15s %15s %15s %15s %15s",
                       w[0], w[1], w[2], w[3], w[4], w[5]);
        for (int j = 1; j < k; j++)     /* word 0 is written, not read ... */
            if (strcmp(w[j], t) == 0)
                return 1;
        if (k > 0 && isjump(code[i]) && strcmp(w[0], t) == 0)
            return 1;                   /* ... except in a jump */
    }
    return 0;
}

void into_target(void)                  /* t = y op z; v = t  ->  v = y op z */
{
    char t[16], eq[4], v[16], src[16];

    for (int i = 0; i + 1 < n; i++)
        if (sscanf(code[i], "%15s %3s", t, eq) == 2 && strcmp(eq, "=") == 0
            && istemp(t)
            && sscanf(code[i + 1], "%15s %3s %15s", v, eq, src) == 3
            && strcmp(eq, "=") == 0 && strcmp(src, t) == 0
            && strchr(code[i + 1] + strlen(v) + 3, ' ') == NULL
            && !used(t, i + 1)) {
            sprintf(code[i + 1], "%s = %s", v, strchr(code[i], '=') + 2);
            dead[i] = 1;
        }
}

void dead_code(void)                    /* delete unread temporaries */
{
    char x[16], eq[4];
    int changed;

    do {
        changed = 0;
        for (int i = 0; i < n; i++)
            if (!dead[i] && sscanf(code[i], "%15s %3s", x, eq) == 2
                && strcmp(eq, "=") == 0 && istemp(x) && !used(x, i)) {
                dead[i] = 1;
                changed = 1;
            }
    } while (changed);
}

int main(void)
{
    while (n < MAX && fgets(code[n], sizeof code[n], stdin)) {
        code[n][strcspn(code[n], "\n")] = '\0';
        if (code[n][0] != '\0')
            n++;
    }
    for (int i = 0; i < n; i++) {       /* a new block: a new notebook */
        if (i == 0 || islabel(code[i]) || isjump(code[i - 1]))
            nknown = navail = 0;
        local(i);
    }
    into_target();
    dead_code();
    for (int i = 0; i < n; i++)
        if (!dead[i])
            puts(code[i]);
    return 0;
}
```

**The notebook** is two tables. `known` holds pairs "name holds value", where the value is a constant or another name: constant and copy propagation in one table. `avail` holds the available expressions: operator, operands, and the name that holds the result. `subst` looks a name up in `known`; `kill` tears out every page that mentions a name, whether as the name, the value, an operand or the holder.

**`local`** rewrites one instruction. Jumps get their operands propagated and nothing else. A copy `x = y` propagates into `y` and records "x holds y". A unary `x = minus y` is moved into the shape of a binary one, with an empty first operand and the operator `minus`, so that the same code folds it and finds it again. An operation propagates into both operands, then tries, in order: folding, if both are now constants (but never a division by zero, which must happen at run time, where it belongs); a common subexpression, if the same operator and operands are available, turning the line into a copy; and otherwise it keeps the operation, kills `x`, and records the expression as available in `x`, unless `x` is one of its own operands (`i = i + 1`, whose result is not `i + 1` of the *new* `i`).

**`main`** clears the notebook at every leader, using section 10.1's rules (a label, or the line after a jump), and runs `local` on every line. Then two clean-up passes over the whole code: `into_target` merges `t = ...` and `v = t` when `t` is read nowhere else, and `dead_code` deletes assignments to temporaries that no line reads, repeating until nothing changes, because deleting one may make another dead.

### Optimise, then check

<!-- code: id=tiny-opt-run -->
```console
$ gcc -Wall -Wextra tiny-opt.c -o tiny-opt
$ ./tiny-tac < prog.tiny | ./tiny-opt
init = 10
rate = 2.5
pos = 160
L1:
ifFalse pos >= 100 goto L2
t3 = rate != 0
t4 = 7.25 / t3
pos = pos - t4
goto L1
L2:
```

B1 is the three lines worked out by hand, with `pos = 160`. The loop is changed too: `t5 = pos - t4` and `pos = t5` became `pos = pos - t4`. Nothing else in the loop could be improved locally: `rate` is 2.5 in B1, but the notebook was cleared at `L1:`, because the loop test can be reached from the bottom of the loop as well, where the optimiser, working locally, does not know what `rate` holds. Section 10.4 deals with such work inside loops.

Then the test that matters: the optimised code must leave every variable as the original did.

<!-- code: id=tiny-opt-check -->
```console
$ ./tiny-tac < prog.tiny | ./tacrun
init = 10
rate = 2.5
pos = 94.75
$ ./tiny-tac < prog.tiny | ./tiny-opt | ./tacrun
init = 10
rate = 2.5
pos = 94.75
$ ./tiny-tac < gcd.tiny | ./tiny-opt | ./tacrun
a = 6
b = 6
```

The same values, `pos = 94.75` both times, and `gcd` still finds 6. And the common subexpression of section 9.4, with values given to `b`, `c` and `d` so that `tacrun` can check it:

<!-- code: id=tiny-opt-cse -->
```console
$ printf 'b = 3; c = 7; d = 2;\na = b * (c - d) / (c - d);\ny = b * (c - d) + 1;\n' > cse.tiny
$ ./tiny-tac < cse.tiny | ./tiny-opt
b = 3
c = 7
d = 2
a = 3
y = 16
$ ./tiny-tac < cse.tiny | ./tacrun
b = 3
c = 7
d = 2
a = 3
y = 16
$ ./tiny-tac < cse.tiny | ./tiny-opt | ./tacrun
b = 3
c = 7
d = 2
a = 3
y = 16
```

Every operand is a constant here, so everything folds, `a` to 3 and `y` to 16, the same values `tacrun` computes from the unoptimised code. Take the constants away, and the common subexpressions show:

<!-- code: id=tiny-opt-cse2 -->
```console
$ printf 'a = b * (c - d) / (c - d);\ny = b * (c - d) + 1;\n' | ./tiny-tac | ./tiny-opt
t1 = c - d
t2 = b * t1
a = t2 / t1
y = t2 + 1
```

`c - d` is computed once, into `t1`, and `b * t1` once, into `t2`, and both are used again across the two statements, which are in one block. The second statement's whole left part, `b * (c - d)`, was found available.

### When it goes wrong

**Propagating across a block boundary.** Clearing the notebook only at labels, and not after jumps, or not at all, carries "rate holds 2.5" into the loop, where it would be wrong if the loop changed `rate`. The notebook is local; the boundary is every leader.

**Forgetting to kill.** After `c = 5`, an available `c - d` in `t3` is no longer `c - d`. Every assignment must tear out every page that mentions the name assigned, as operand, value or holder.

**Deleting a variable as dead.** `pos = 160` is read by no later line, but it is the program's result. Only temporaries, which the translator made for one use inside a block, are safe to delete with a local view.

**Folding a division by zero.** `t1 = 1 / 0` must not become anything at compile time; the program's behaviour at run time (an error, or infinity) is the program's, not the compiler's to change.

**`i = i + 1` made available as `i + 1` in `i`.** The next `i + 1` would become `i`, which now holds the *new* value: off by one. An expression is never recorded as available in one of its own operands.

### In one line each

1. Local optimisation works inside one basic block, with a notebook that is cleared at every leader.
2. Constant and copy propagation replace a name by what it is known to hold; constant folding computes operators on constants.
3. A common subexpression, computed again with nothing changed in between, becomes a copy of the first result.
4. Dead-code elimination deletes assignments whose results are never read; locally, only temporaries.
5. Check every optimisation by running the code before and after with `tacrun`.

**Check yourself**

<!-- q: idea=cse type=apply wrong="c - d is still available after c is assigned" -->
1. Optimise this block by hand, naming the transformation at each step, then check with `tiny-opt`:

```text
t1 = c - d
x = t1
c = 5
t2 = c - d
y = t2 + x
```

<details>
<summary>Answer</summary>

`x = t1` records "x holds t1". `c = 5` records "c holds 5" and kills every page mentioning `c`: the available `c - d` is gone. So `t2 = c - d` is *not* a common subexpression; constant propagation turns it into `t2 = 5 - d`. `y = t2 + x` becomes `y = t2 + t1` by copy propagation; `into_target` then cannot merge `t2` (it is read by `y`'s line, not copied). Result: `t1 = c - d`, `x = t1`, `c = 5`, `t2 = 5 - d`, `y = t2 + t1`.

<!-- code: id=opt-check1 -->
```console
$ printf 't1 = c - d\nx = t1\nc = 5\nt2 = c - d\ny = t2 + x\n' | ./tiny-opt
t1 = c - d
x = t1
c = 5
t2 = 5 - d
y = t2 + t1
```

The trap is to replace `t2 = c - d` by `t2 = t1`: the text is the same, but `c` changed in between.

</details>

<!-- q: idea=dead-code type=decide wrong="every assignment no later line reads is dead" -->
2. In `pos = 160` at the end of B1 of `prog.tiny`, no later line in B1 reads `pos`. Why does `tiny-opt` keep it, and what would a whole-program optimiser need to know to decide?

<details>
<summary>Answer</summary>

`pos` is a source variable: its value may be read in another block (here B2's test reads it) or be the program's result. A local optimiser sees only its block, so it must assume every variable is *live* on leaving the block. A global optimiser would compute liveness over the flow graph: `pos` is read at the start of B2, so it is live at the end of B1, and the assignment stays. `tiny-opt` deletes only temporaries, which `tiny-tac` uses only inside one block.

</details>

<!-- q: idea=constant-folding type=predict wrong="the optimiser can fold rate * 60 inside the loop" -->
3. Predict the output of `tiny-opt` for `r = 2; while (i < 10) { x = r * 3; i = i + 1; }` and say why `r * 3` is not folded. Then check.

<details>
<summary>Answer</summary>

<!-- code: id=opt-check3 -->
```console
$ printf 'r = 2; while (i < 10) { x = r * 3; i = i + 1; }\n' | ./tiny-tac | ./tiny-opt
r = 2
L1:
ifFalse i < 10 goto L2
x = r * 3
i = i + 1
goto L1
L2:
```

`r * 3` is in the loop body, a different block from `r = 2`, and the notebook is cleared at the leader `L1:` and again after the `ifFalse`; the optimiser does not know, locally, that `r` is still 2 there. The body's `t1 = r * 3` and `x = t1` merge into `x = r * 3`, and `t2 = i + 1`, `i = t2` into `i = i + 1`. Moving `r * 3` out of the loop, or folding it, needs the loop view of section 10.4.

</details>

## 10.3 The DAG of a basic block

### The question the notebook answers, drawn

`tiny-opt` found common subexpressions by keeping a list of available expressions. The paper asks for the same knowledge as a picture: "Construct the DAG for the basic block below, and use it to write optimised code." The picture is a *directed acyclic graph*, a DAG: the expression tree of section 5.1 with every repeated subtree drawn once and shared. Where the tree for `a = b * -c + b * -c` has two copies of `b * -c`, the DAG has one node with two arrows into it. Building a DAG is how a compiler finds common subexpressions inside a block, and reading it back out gives the optimised code.

### Trees that share their branches

<!-- teaches: dag = DAG of a basic block, leaf, interior node, node label -->
The DAG of a basic block has:

- a **leaf** for the *initial* value of each name used in the block, labelled with the name and a subscript 0: `b0` is "whatever `b` held when the block started";
- an **interior node** for each operation, labelled with its operator, with arrows to the nodes of its operands;
- next to each node, a list of the **names that hold its value** at the end of the block (or, while building, at that moment).

It is built instruction by instruction. For `x = y op z`:

1. Find the nodes that currently hold `y` and `z`. A name holds the node it was last attached to; a name not yet attached holds its leaf, made if needed.
2. Look for an existing node with operator `op` and exactly those two children. If there is one, it is a common subexpression: use it. If not, make one.
3. Attach `x` to that node, and detach `x` from any node it was attached to before.

A copy `x = y` makes no node: it attaches `x` to the node that holds `y`.

Here is the Dragon book's example block, worked step by step:

```text
a = b + c
b = a - d
c = b + c
d = a - d
```

| Instruction | Operands' nodes | Node used | Names attached afterwards |
|---|---|---|---|
| `a = b + c` | leaves b0, c0 | new node 1: `+` (b0, c0) | 1: a |
| `b = a - d` | node 1 (holds a), leaf d0 | new node 2: `−` (1, d0) | 1: a; 2: b |
| `c = b + c` | node 2 (holds b), leaf c0 | new node 3: `+` (2, c0) | 1: a; 2: b; 3: c |
| `d = a - d` | node 1 (holds a), leaf d0 | node 2 already is `−` (1, d0): reuse it | 1: a; 2: b, d; 3: c |

Two things in the table are the whole lesson. In the third line, `b + c` is *not* the same as the first line's `b + c`, although the text is identical: `b` now holds node 2, not the leaf b0. The DAG compares *nodes*, values, not names, and that is what makes it safe where a text search would be wrong. In the last line, `a - d` *is* the same as the second line's: `a` still holds node 1 and `d` still holds d0. So node 2 gets a second name, `d`.

<!-- fig: id=dag-abcd
title: DAG of the block a = b + c; b = a - d; c = b + c; d = a - d
kind: flow
dir: down
box n3: + (node 3)\nnames: c
box n2: − (node 2)\nnames: b, d
box n1: + (node 1)\nnames: a
box b0: b0
box c0: c0
box d0: d0
n3 -> n2
n3 -> c0
n2 -> n1
n2 -> d0
n1 -> b0
n1 -> c0
hot: n2
-->
![Flow chart, top to bottom. + (node 3) names: c leads to − (node 2) names: b, d. + (node 3) names: c leads to c0. − (node 2) names: b, d leads to + (node 1) names: a. − (node 2) names: b, d leads to d0. + (node 1) names: a leads to b0. + (node 1) names: a leads to c0. − (node 2) names: b, d is highlighted.](fig/10-optimisation-codegen/dag-abcd.svg)

Three interior nodes for four instructions: one subtraction is gone. Node 2 is highlighted: two names, one computation.

### Reading the code back out

To write the block from its DAG, visit the interior nodes in an order where every node comes after its children (here 1, 2, 3), and for each, compute it into *one* of its names; then copy into any other name that is still needed. Node 2 holds both `b` and `d`; compute it once, into `d`, and copy to `b` only if `b` is live after the block. With all four names live at the exit:

```text
a = b + c
d = a - d
c = d + c
b = d
```

Node 3's operand is now written `d`, the name chosen for node 2, not `b`. If `b` were dead at the exit (if no later block read it), the last line would go too, and the block would be three instructions.

`tiny-opt` reaches the same code from the other side. Its available-expressions list is the DAG's interior nodes, and its "name holds value" pages are the DAG's lists of names:

<!-- code: id=dag-opt -->
```console
$ printf 'a = b + c\nb = a - d\nc = b + c\nd = a - d\n' | ./tiny-opt
a = b + c
b = a - d
c = b + c
d = b
```

It kept `b = a - d` and turned `d = a - d` into the copy `d = b`: node 2 computed once, into `b`, and copied to `d`. That is the same DAG read out with the other name chosen; both are correct, and `tacrun` agrees, with some starting values put in front:

<!-- code: id=dag-check -->
```console
$ printf 'b = 1\nc = 2\nd = 3\na = b + c\nb = a - d\nc = b + c\nd = a - d\n' | ./tacrun
b = 0
c = 2
d = 0
a = 3
$ printf 'b = 1\nc = 2\nd = 3\na = b + c\nd = a - d\nc = d + c\nb = d\n' | ./tacrun
b = 0
c = 2
d = 0
a = 3
```

The order of the names in the output is the order they were first met, and the values match: the rewritten block is the original block.

### What else a DAG shows

**Dead code.** An interior node with no live names attached and no live node above it computes something nobody reads; it can be left out when the code is read back. Section 10.2's dead temporaries are exactly such nodes.

**Which names are inputs.** The leaves are the names whose values the block *reads* from before it. Here `b`, `c` and `d` are read from before; `a` is not, so the block does not care what `a` held when it started.

**Algebraic identities, if you allow them.** A DAG builder can also treat `b + c` and `c + b` as one node (`+` is commutative), or replace `x * 1` by `x`'s node. That is a choice; the paper's DAGs usually do not, unless the question says so.

### When it goes wrong

**Matching by name instead of by node.** Making node 3 the same as node 1 because both are written `b + c`. Always look up which node each operand name holds *now*.

**Forgetting to detach.** When `b` is reassigned at line 2, it must leave the leaf b0. A DAG with `b` still listed on b0 reads back with a wrong value for `b`.

**Losing a live name.** Node 2 holds `b` and `d`; writing only `d = a - d` and dropping `b` changes the program if `b` is read after the block. Every live name on a node needs its value at the end, by a copy if necessary.

**An order that computes a node before its children.** Reading back in any order where node 3 comes before node 2 uses `d` before it has its new value. Children first.

### In one line each

1. A DAG has a leaf per initial value (`b0`), an interior node per distinct operation, and on each node the names that hold its value.
2. For `x = y op z`: find the nodes `y` and `z` hold now, reuse a node with the same operator and children or make one, then move `x` to it.
3. A reused node is a common subexpression; the DAG compares values, not text.
4. Read the code back children first, computing each node once and copying to its other live names.
5. Nodes with no live names are dead code.

**Check yourself**

<!-- q: idea=dag type=apply wrong="each b * -c gets its own node" -->
1. Construct the DAG for the block `t1 = minus c`, `t2 = b * t1`, `t3 = minus c`, `t4 = b * t3`, `t5 = t2 + t4`, `a = t5`, and write the optimised block, assuming only `a` is live at the exit. Check the code with `tiny-opt`.

<details>
<summary>Answer</summary>

Leaves `c0` and `b0`. `t1 = minus c` makes node 1, `minus (c0)`, names t1. `t2 = b * t1` makes node 2, `* (b0, 1)`, names t2. `t3 = minus c` finds node 1 again: names t1, t3. `t4 = b * t3`: `t3` holds node 1, so `* (b0, 1)` is node 2 again: names t2, t4. `t5 = t2 + t4` makes node 3, `+ (2, 2)`, with both arrows to node 2; names t5. `a = t5` attaches `a` to node 3. Three interior nodes. With only `a` live: `t1 = minus c`, `t2 = b * t1`, `a = t2 + t2`.

<!-- code: id=dag-q1 -->
```console
$ printf 't1 = minus c\nt2 = b * t1\nt3 = minus c\nt4 = b * t3\nt5 = t2 + t4\na = t5\n' | ./tiny-opt
t1 = minus c
t2 = b * t1
a = t2 + t2
```

This is section 9.3's expression, and the result is the one that section promised: (2) and (3) gone.

</details>

<!-- q: idea=dag type=predict wrong="the second x + y always reuses the first node" -->
2. Draw the DAG for `p = x + y`, `x = 1`, `q = x + y`. How many `+` nodes are there? Then say what `tiny-opt` will do with `q`'s line.

<details>
<summary>Answer</summary>

Two `+` nodes. `p = x + y` makes `+ (x0, y0)`. `x = 1` attaches `x` to a constant leaf `1`, detaching it from `x0`. `q = x + y` looks for `+ (1, y0)`, which does not exist: a new node. Same text, different values. `tiny-opt` propagates the constant: `q = 1 + y`, and keeps `p = x + y` as it was.

</details>

## 10.4 Loop optimisation: code motion, induction variables, strength reduction

### Where the time goes

A program spends almost all its time in loops. B1 of `prog.tiny` runs once; the loop body runs nine times, and in a real program a loop body can run a billion times. An instruction saved before the loop saves one instruction; an instruction saved inside it saves one per pass. So the paper's second list of optimisations is about loops, and it is the one the examiner asks you to apply by hand: "Apply code motion and strength reduction to the following loop."

Here is a loop that has something for each of them. It adds up `y * z * i` for `i` from 0 to 9:

<!-- code: id=loop-tiny -->
```console
$ cat > loop.tiny <<'END'
n = 10; y = 3; z = 4;
i = 0; s = 0;
while (i < n) {
    s = s + y * z * i;
    i = i + 1;
}
END
$ ./tiny-tac < loop.tiny | ./tiny-opt > loop.tac
$ cat loop.tac
n = 10
y = 3
z = 4
i = 0
s = 0
L1:
ifFalse i < n goto L2
t1 = y * z
t2 = t1 * i
s = s + t2
i = i + 1
goto L1
L2:
$ ./tacrun < loop.tac
n = 10
y = 3
z = 4
i = 10
s = 540
```

The loop is B2 (the test) and B3 (the body), and the local optimiser of section 10.2 has done what it could inside them. Each pass of the loop runs six instructions, two of them multiplications, and the loop makes ten passes. This section removes both multiplications from the loop by hand, without changing `s`.

### Finding the loop

<!-- teaches: code-motion = loop-invariant computation, code motion, preheader -->
In the flow graph, a loop is a cycle with a single entry: a block, the *header* (here B2, the test), through which every path into the loop passes, and a *back edge* from inside the loop to the header (B3 → B2, the `goto L1`). Whatever is placed just before the header, in a new block called the *preheader*, runs once, before the first pass, and on every path into the loop. That is where work taken out of the loop goes.

### Code motion

A computation is *loop-invariant* if it gives the same value on every pass: its operands are constants, or names that nothing inside the loop assigns, or other invariant results. `t1 = y * z` is one: `y` and `z` are assigned in B1 and never in the loop. Every pass computes 12 again.

*Code motion* moves a loop-invariant computation into the preheader, so that it runs once. Moving it is only safe under conditions, and the paper wants you to state them:

1. its operands are invariant, as above;
2. the name it assigns is assigned nowhere else in the loop, so that every use in the loop sees this value;
3. moving it does not change what the program leaves behind when the loop runs zero times, or exits early. Moving `x = y * z` out of a loop that never runs would give `x` a value the original program never gave it. A temporary such as `t1`, which nothing reads after the loop, is always safe here; a source variable is safe only if its block runs on every pass that can leave the loop (the Dragon book says: its block *dominates* every exit), or if the variable is not used after the loop.
<!-- risk: the code-motion conditions as stated in Dragon 2nd ed. 9.5.? (invariant operands; the only definition in the loop; dominates all exits or dead after); check the section and the wording -->

`t1` meets all three. After the move:

```text
t1 = y * z          (preheader: once)
L1:
ifFalse i < n goto L2
t2 = t1 * i
s = s + t2
i = i + 1
goto L1
L2:
```

One multiplication gone from every pass. The classic case is a loop test with an invariant expression in it, such as `while (i < limit * 2)`: the `limit * 2` is computed on every test unless it is moved out.

### Induction variables and strength reduction

<!-- teaches: strength-reduction = induction variable, strength reduction, induction-variable elimination -->
`i` goes up by 1 on every pass: `i = i + 1` is its only assignment in the loop. A name assigned in the loop only by adding or subtracting a constant is a *basic induction variable*. `t2 = t1 * i` is then also predictable: every time `i` goes up by 1, `t2` goes up by `t1`, which is 12. A name that is a linear function of an induction variable, `t2 = c × i` (or `c × i + d`) with `c` invariant, is an induction variable too, in `i`'s family.

*Strength reduction* replaces the multiplication by the cheaper addition that tracks it: give `t2` its starting value in the preheader, `t2 = t1 * i` (with `i` 0 there, that is 0), and in the loop, right after `i = i + 1`, add `t2 = t2 + t1`. The multiplication in the body goes:

<!-- code: id=loop-reduced -->
```console
$ cat > loop2.tac <<'END'
n = 10
y = 3
z = 4
i = 0
s = 0
t1 = y * z
t2 = t1 * i
L1:
ifFalse i < n goto L2
s = s + t2
i = i + 1
t2 = t2 + t1
goto L1
L2:
END
$ ./tacrun < loop2.tac
n = 10
y = 3
z = 4
i = 10
s = 540
```

The same `s`, 540, from code whose loop has no multiplication at all: the pass is `ifFalse`, two additions, one more addition and the `goto`, five instructions instead of six, and the two multiplications run once each, before the loop. On a machine where a multiplication takes several times as long as an addition, the name *strength reduction* explains itself. The most common place for it is the one you met in section 9.2: an array index, `t1 = i * 4`, `t2 = a[t1]`, inside a loop on `i`, becomes `t1 = t1 + 4`, a pointer moving through the array four bytes at a time.

Where must the new `t2 = t2 + t1` go? It must keep `t2` equal to `t1 * i` at every point where `t2` is read. It is read at the top of the body, by `s = s + t2`, before `i` changes; so it must be updated *after* that read and before the next pass, and the natural place is next to the update of `i` that it tracks. Question 2 below asks what happens when it is moved.

### Induction-variable elimination

After strength reduction, `i` is used only to count: in the test `i < n` and in its own increment. The test can be written with `t2` instead: `i < n` is the same as `t1 * i < t1 * n`, that is `t2 < t3` with `t3 = t1 * n` computed in the preheader (this needs `t1` positive, here 12, so that multiplying both sides keeps the order). Then nothing reads `i` inside the loop except `i = i + 1`, and both can go. This is *induction-variable elimination*, and it is the third step the paper often asks for. Here it has a catch, which is the point of doing it carefully: `i` is a source variable, and `tacrun` prints its final value, 10. If anything after the loop reads `i`, it must stay. A compiler removes it only when liveness analysis proves it dead after the loop; by hand, say which assumption you made.
<!-- risk: claim that induction-variable elimination needs t1 > 0 for the rewritten test and that i must be dead after the loop; check against Dragon 9.? -->

### When it goes wrong

**Moving something that is not invariant.** `t2 = t1 * i` reads `i`, which the loop changes; moving it out computes it once, with `i` at 0, and `s` comes out 0. Check every operand against every assignment in the loop.

**Moving an assignment to a variable used after the loop.** If the loop can run zero times, the variable now gets a value it never had in the original. Temporaries are safe; for source variables, check condition 3.

**The strength-reduced update in the wrong place.** The update must keep the induction variable equal to its formula wherever it is read; one place too early or too late is off by one step on every pass. Run both versions with `tacrun`.

**Forgetting the starting value.** `t2 = t2 + t1` with no `t2 = t1 * i` in the preheader starts from whatever `t2` held; in `tacrun` that is 0, which happens to be right here because `i` starts at 0. Start `i` at 1 and the sum is off by 12 on every pass. Always initialise in the preheader from the formula.

### In one line each

1. Loops are where the time goes; a loop has a header that every entry passes through, and a back edge to it.
2. Code motion moves a loop-invariant computation into the preheader, when its operands are invariant, it is the only assignment of its name in the loop, and the move cannot change what a zero-pass loop leaves behind.
3. A basic induction variable changes by a constant on every pass; `c × i` is an induction variable in its family.
4. Strength reduction replaces `t = c × i` in the loop by `t = t + c` after `i`'s update, with `t` initialised in the preheader.
5. Induction-variable elimination rewrites the test in terms of the new variable and deletes the old one, if it is dead after the loop.

**Check yourself**

<!-- q: idea=strength-reduction type=apply wrong="the array offset must be multiplied on every pass" -->
1. Apply code motion and strength reduction to this loop from section 9.2's sum over an array, and say how many instructions each pass runs before and after:

```text
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

<details>
<summary>Answer</summary>

Nothing is loop-invariant here except the constants, so code motion has nothing to move. `i` is a basic induction variable (`i = i + 1`, after copy propagation), and `t1 = i * 4` is in its family. Initialise `t1 = i * 4` in the preheader, delete it from the body, and add `t1 = t1 + 4` after `i`'s update. With section 10.2's merges as well (`s = s + t2`, `i = i + 1`):

```text
    t1 = i * 4
L1: ifFalse i < n goto L2
    t2 = a[t1]
    s = s + t2
    i = i + 1
    t1 = t1 + 4
    goto L1
L2:
```

Eight instructions per pass before (counting the test and the `goto`), six after, and none of them a multiplication. If `i` is not used after the loop, induction-variable elimination can go further: test `t1 < t5` with `t5 = n * 4` in the preheader, and delete `i`.

</details>

<!-- q: idea=strength-reduction type=fix wrong="the order of the two additions does not matter" -->
2. A student puts `t2 = t2 + t1` *before* `i = i + 1` in `loop2.tac`. Predict `s`, then check with `tacrun`.

<details>
<summary>Answer</summary>

`t2` is read by `s = s + t2` at the top of each pass, so the order of the two additions *after* it does not change `s`: both put `t2` at `t1 × (i + 1)` before the next pass reads it. `s` is still 540.

<!-- code: id=loop-order -->
```console
$ sed -n '1,10p' loop2.tac > loop3.tac
$ printf 't2 = t2 + t1\ni = i + 1\ngoto L1\nL2:\n' >> loop3.tac
$ ./tacrun < loop3.tac
n = 10
y = 3
z = 4
i = 10
s = 540
```

The place that matters is relative to the *read*: put the update before `s = s + t2`, and every pass adds `t1` too much, giving 660. The rule is "keep `t2 = t1 × i` true wherever `t2` is read", not "right after `i` changes" for its own sake.

</details>

## 10.5 Peephole optimisation

### Small mistakes in finished code

Look at the end of the loop body in `gcd`'s code, after `tiny-opt`:

```text
a = a - b
goto L4
L3:
b = b - a
L4:
goto L1
```

The `then` branch jumps to `L4`, and all `L4` does is jump to `L1`. Every pass that takes the `then` branch makes two jumps where one would do. No local optimisation in section 10.2 can see it, because the two jumps are in different blocks. A *peephole* optimiser looks at the code through a small window, a few instructions at a time, including across block boundaries, and replaces any pattern it recognises with a shorter or faster one. It is usually run last, on the target code, and repeated until nothing changes, since one improvement can open the way for another.

<!-- teaches: peephole = peephole optimisation, jump to a jump, unreachable code, algebraic simplification -->
The paper lists these patterns:

| Pattern | Before | After |
|---|---|---|
| jump to a jump | `goto L4` ... `L4: goto L1` | `goto L1` |
| unreachable code | `goto L1`, then instructions with no label | delete them up to the next label |
| redundant load after store | `ST a, R0` then `LD R0, a` | delete the load: R0 already holds `a` |
| algebraic simplification | `x = x + 0`, `x = x * 1` | delete; `x = y * 0` becomes `x = 0` |
| strength reduction | `x = y * 2`, `x = y ^ 2` | `x = y + y`, `x = y * y` |
| machine idioms | `x = x + 1` | `INC x`, where the machine has it |

Apply the first to `gcd`'s code. `goto L4` becomes `goto L1`; then no jump names `L4` any more, so in the flow graph B4 now goes straight to B2, and the block `L4: goto L1` is reached only by falling through from B5. The `else` branch still passes through it. Check that nothing changed in the result:

<!-- code: id=peep-gcd -->
```console
$ ./tiny-tac < gcd.tiny | ./tiny-opt | sed 's/goto L4/goto L1/' > gcdp.tac
$ cat gcdp.tac
a = 48
b = 18
L1:
ifFalse a != b goto L2
ifFalse a > b goto L3
a = a - b
goto L1
L3:
b = b - a
L4:
goto L1
L2:
$ ./tacrun < gcdp.tac
a = 6
b = 6
```

Same `a` and `b`, 6 and 6, and the `then` path through the loop is one jump shorter.

The redundant-load pattern appears in code from a code generator that handles each instruction alone: `a = b + c` ends with `ST a, R0`, and the next instruction, `d = a * 2`, starts with `LD R0, a`. Inside one block, the simple code generator of section 10.6 avoids it by remembering what each register holds; a peephole pass catches what is left, but only if no label stands between the two instructions, since a jump to that label would arrive with R0 holding something else.
<!-- risk: claim that peephole optimisers typically run on target code and are repeated until no change (Dragon 8.7); check -->

### When it goes wrong

**Deleting a load across a label.** `ST a, R0`, `L5:`, `LD R0, a`: another path may jump to `L5` with something else in R0. The window must not cross a label for this pattern.

**Check yourself**

<!-- q: idea=peephole type=apply wrong="code after a goto is always dead" -->
1. Apply peephole optimisation to this code and name each pattern you use:

```text
    goto L2
    x = 1
L2: goto L3
    y = 2
L3: z = z * 1
    w = v * 2
```

<details>
<summary>Answer</summary>

`x = 1` follows an unconditional `goto` and has no label: unreachable, delete it. `goto L2` jumps to `L2: goto L3`: jump to a jump, so it becomes `goto L3`; now no jump names `L2`, and the line before it is an unconditional `goto`, so nothing can arrive at `L2: goto L3` or at the `y = 2` after it: unreachable, and both go. `z = z * 1` does nothing: algebraic simplification deletes it. `w = v * 2` becomes `w = v + v`: strength reduction. What is left is `goto L3`, `L3: w = v + v`, and then `goto L3` jumps to the very next instruction, so it goes as well: `w = v + v`. Six instructions became one, which is why the passes are repeated.

</details>

## 10.6 A simple code generator: register and address descriptors, getreg

### The last station

The assembly line of section 1.2 ends with the code generator: three-address code in, machine instructions out. Chapter 1 showed GCC's real output for the running statement, `mulsd` and `addsd` on x86-64 registers. Between `t2 = init + t1` and such instructions lie two decisions the intermediate code never made: which *register* holds each value, and when a value must be *loaded* from memory or *stored* back to it. Registers are few and fast, memory is plentiful and slow, and the whole art of a simple code generator is to load each value as rarely as possible and store only what must be stored.

The paper asks for this in one fixed form: "Generate code for the following basic block using the simple code generation algorithm, showing the register and address descriptors after each instruction." This section gives you the algorithm of the Dragon book, a C program that runs it and prints the descriptors, and the reasons behind each choice, so that you can fill in the table by hand and check it.

### The target machine

The book's machine is the Dragon book's simple register machine. It has a few general registers, R1, R2, R3, and these instructions:

| Instruction | Means |
|---|---|
| `LD R1, x` | load the value of `x` from memory into R1 |
| `LD R1, #60` | load the constant 60 into R1 |
| `ST x, R1` | store R1 into the memory of `x` |
| `ADD R1, R2, R3` | R1 = R2 + R3 (and `SUB`, `MUL`, `DIV`) |

An arithmetic instruction works only on registers, so both operands must be in registers first. Every name has its home in memory; a temporary has a home too, used only if it has to be stored.
<!-- risk: the target machine (LD, ST, ADD dst, src1, src2, #constant) follows Dragon 2nd ed. 8.2; check the notation -->

### Two descriptors: the compiler's map of where values are

<!-- teaches: code-generator = simple code generator, register descriptor, address descriptor -->
The generator keeps two tables, updated after every instruction it writes.

- The **register descriptor** says, for each register, which names' current values it holds: "R3 holds `t2` and `pos`". At the start of the block, every register is empty.
- The **address descriptor** says, for each name, where its current value can be found: in its memory home, in one or more registers, or both. At the start, every source variable is in memory; a temporary is nowhere yet.

These are what let the generator avoid work. If `rate` is already in R1, `rate * 60` needs no load. If `pos` is in R3 and also in memory, R3 can be reused without a store. If `t1`'s only copy is in R1 and `t1` is still needed, R1 cannot be reused until `t1` is stored.

For each instruction `x = y op z`, the algorithm is:

1. Choose registers with `getreg`: Ry for `y`, Rz for `z`, Rx for `x`.
2. If `y` is not in Ry, write `LD Ry, y`. The same for `z` and Rz.
3. Write `OP Rx, Ry, Rz`.
4. Update the descriptors: Rx now holds *only* `x`; `x`'s address is *only* Rx (its memory copy is now out of date); `x` is removed from every other register.

For a copy `x = y`: make sure `y` is in some register Ry (load it if not), then *add* `x` to Ry's names and set `x`'s address to only Ry. No instruction is written for the copy itself: `x` and `y` now share a register.

At the end of the block, for every source variable whose current value is not in memory, write `ST x, R`. Temporaries are dead at the end of the block and are never stored there.

### getreg

<!-- teaches: getreg = getreg, spill, next use -->
`getreg` makes the choices, using one more piece of knowledge: whether a name is *used again* later in the block, worked out by looking ahead in the block. For an operand `y`:

- if `y` is already in a register, use it;
- else, if a register is empty, use it;
- else, choose the register that is cheapest to empty: one whose names all have an up-to-date copy elsewhere, or are not needed again. Store (*spill*) any name it holds that is needed and has no other copy.

For the result `x`, in order:

1. a register that holds only `x`;
2. Ry, if `y` is not used again in the block and its value is safe elsewhere (in memory, or `y` is a temporary that is now dead, or a constant): `x` may overwrite it;
3. Rz, under the same condition for `z`;
4. an empty register;
5. the cheapest register, spilling as above.

Rules 2 and 3 are the ones that save the most: when an operand dies at this instruction, its register is free for the result, and the instruction overwrites one of its own operands.
<!-- risk: getreg rules for operands and result follow Dragon 2nd ed. 8.6.3; this program's order (Ry, Rz, empty, cheapest) is one consistent reading; check -->

### codegen.c

`codegen.c` runs the algorithm on one basic block of copies and binary operations, and prints after each instruction the code it wrote and the two descriptors. A name of the form `t` and digits is a temporary; every other name is a source variable, in memory at the start and live at the exit. The number of registers is an argument, 3 by default.

<!-- code: id=codegen mode=compile -->
```c
/* codegen.c: the simple code generator for one basic block.
   Reads three-address code (x = y op z, x = y), one instruction per line,
   and writes register-machine code, showing after each instruction the
   register descriptor and the address descriptor.
   usage: ./codegen [registers]      (3 registers by default) */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX 100
#define MAXREG 8

struct { char x[16], y[16], op[4], z[16]; } ins[MAX];  /* x = y op z */
int n, nreg = 3;

char reg[MAXREG][MAX][16];       /* register descriptor: names in each */
int nin[MAXREG];
char var[MAX][16];               /* address descriptor: for each name ... */
int inmem[MAX];                  /* ... is its memory copy up to date? */
int nvars;

int istemp(const char *s) { return s[0] == 't' && isdigit((unsigned char)s[1]); }
int isconst(const char *s) { return isdigit((unsigned char)s[0]) || s[0] == '.' || s[0] == '-'; }

int v(const char *s)             /* a name's slot; a new name is in memory */
{
    for (int i = 0; i < nvars; i++)
        if (strcmp(var[i], s) == 0)
            return i;
    strcpy(var[nvars], s);
    inmem[nvars] = !istemp(s);   /* temporaries start nowhere */
    return nvars++;
}

int holds(int r, const char *s)
{
    for (int i = 0; i < nin[r]; i++)
        if (strcmp(reg[r][i], s) == 0)
            return 1;
    return 0;
}

int where(const char *s)         /* a register holding s, or -1 */
{
    for (int r = 0; r < nreg; r++)
        if (holds(r, s))
            return r;
    return -1;
}

void drop(int r, const char *s)  /* r no longer holds s */
{
    int j = 0;
    for (int i = 0; i < nin[r]; i++)
        if (strcmp(reg[r][i], s) != 0) {
            if (j != i)
                strcpy(reg[r][j], reg[r][i]);
            j++;
        }
    nin[r] = j;
}

int live(const char *s, int from)   /* is s read at or after instruction from,
                                       before it is written? (or a name at exit) */
{
    for (int i = from; i < n; i++) {
        if (strcmp(ins[i].y, s) == 0 || strcmp(ins[i].z, s) == 0)
            return 1;
        if (strcmp(ins[i].x, s) == 0)
            return 0;
    }
    return !istemp(s);
}

int usedlater(const char *s, int from)   /* read again in this block? */
{
    for (int i = from; i < n; i++) {
        if (strcmp(ins[i].y, s) == 0 || strcmp(ins[i].z, s) == 0)
            return 1;
        if (strcmp(ins[i].x, s) == 0)
            return 0;
    }
    return 0;
}

int done(int r, const char *s, int from) /* r holds only s, and r may now be reused */
{
    return r >= 0 && nin[r] == 1 && strcmp(reg[r][0], s) == 0
        && !usedlater(s, from) && (isconst(s) || inmem[v(s)] || istemp(s));
}

int cost(int r, int from)        /* stores needed to empty r */
{
    int c = 0;
    for (int i = 0; i < nin[r]; i++)
        if (!inmem[v(reg[r][i])] && live(reg[r][i], from))
            c++;
    return c;
}

void spill(int r, int from)      /* store what only r holds, then empty r */
{
    for (int i = 0; i < nin[r]; i++)
        if (!inmem[v(reg[r][i])] && live(reg[r][i], from)) {
            printf("    ST %s, R%d\n", reg[r][i], r + 1);
            inmem[v(reg[r][i])] = 1;
        }
    nin[r] = 0;
}

int cheapest(int from, int avoid)   /* an empty register, else the cheapest */
{
    int best = -1;
    for (int r = 0; r < nreg; r++)
        if (r != avoid && (best < 0 || cost(r, from) < cost(best, from)
                           || (nin[r] == 0 && nin[best] > 0)))
            best = r;
    return best;
}

int getreg_operand(const char *y, int i, int avoid)   /* load y if needed */
{
    int r = where(y);
    if (r >= 0)
        return r;
    r = cheapest(i, avoid);
    spill(r, i);
    if (isconst(y))
        printf("    LD R%d, #%s\n", r + 1, y);
    else
        printf("    LD R%d, %s\n", r + 1, y);
    strcpy(reg[r][nin[r]++], y);
    return r;
}

int only(int r, const char *s)   /* r holds s and nothing else */
{
    return nin[r] == 1 && strcmp(reg[r][0], s) == 0;
}

int getreg_result(int i, int ry, int rz)
{
    const char *x = ins[i].x, *y = ins[i].y, *z = ins[i].z;

    for (int r = 0; r < nreg; r++)                      /* 1: holds only x */
        if (only(r, x))
            return r;
    if (done(ry, y, i + 1))
        return ry;                                      /* 2: y's, if y is done */
    if (done(rz, z, i + 1))
        return rz;                                      /* 3: z's, likewise */
    for (int r = 0; r < nreg; r++)                      /* 4: an empty one */
        if (nin[r] == 0)
            return r;
    int r = cheapest(i + 1, -1);                        /* 5: spill the cheapest */
    spill(r, i + 1);
    return r;
}

void show(void)
{
    printf("    registers:");
    for (int r = 0; r < nreg; r++) {
        printf(" R%d=", r + 1);
        for (int i = 0; i < nin[r]; i++)
            printf("%s%s", i ? "," : "", reg[r][i]);
    }
    printf("\n    addresses:");
    for (int k = 0; k < nvars; k++) {
        if (isconst(var[k]))
            continue;
        int first = 1;
        printf(" %s=", var[k]);
        if (inmem[k]) {
            printf("%s", var[k]);
            first = 0;
        }
        for (int r = 0; r < nreg; r++)
            if (holds(r, var[k])) {
                printf("%sR%d", first ? "" : ",", r + 1);
                first = 0;
            }
        if (first)
            printf("-");
    }
    printf("\n");
}

const char *opname(const char *op)
{
    return strcmp(op, "+") == 0 ? "ADD" : strcmp(op, "-") == 0 ? "SUB"
         : strcmp(op, "*") == 0 ? "MUL" : "DIV";
}

int main(int argc, char *argv[])
{
    char line[100];

    if (argc > 1)
        nreg = atoi(argv[1]);
    if (nreg < 2 || nreg > MAXREG) {
        fprintf(stderr, "codegen: registers must be 2 to %d\n", MAXREG);
        return 1;
    }
    while (n < MAX && fgets(line, sizeof line, stdin)) {
        int k = sscanf(line, "%15s = %15s %3s %15s",
                       ins[n].x, ins[n].y, ins[n].op, ins[n].z);
        if (k == 2 || k == 4)
            n++;
    }
    for (int i = 0; i < n; i++) {
        const char *x = ins[i].x;
        if (ins[i].op[0] == '\0')
            printf("%s = %s\n", x, ins[i].y);
        else
            printf("%s = %s %s %s\n", x, ins[i].y, ins[i].op, ins[i].z);
        v(ins[i].y);
        if (ins[i].op[0] != '\0')
            v(ins[i].z);
        v(x);
        if (ins[i].op[0] == '\0') {                     /* x = y */
            int ry = getreg_operand(ins[i].y, i, -1);
            for (int r = 0; r < nreg; r++)
                drop(r, x);
            strcpy(reg[ry][nin[ry]++], x);
            inmem[v(x)] = 0;
        } else {                                        /* x = y op z */
            int ry = getreg_operand(ins[i].y, i, -1);
            int rz = getreg_operand(ins[i].z, i, ry);
            int rx = getreg_result(i, ry, rz);
            printf("    %s R%d, R%d, R%d\n", opname(ins[i].op), rx + 1, ry + 1, rz + 1);
            for (int r = 0; r < nreg; r++)
                drop(r, x);
            nin[rx] = 0;
            strcpy(reg[rx][nin[rx]++], x);
            inmem[v(x)] = 0;
        }
        show();
    }
    printf("exit\n");                                   /* names live at exit */
    for (int k = 0; k < nvars; k++)
        if (!istemp(var[k]) && !isconst(var[k]) && !inmem[k] && where(var[k]) >= 0) {
            printf("    ST %s, R%d\n", var[k], where(var[k]) + 1);
            inmem[k] = 1;
        }
    return 0;
}
```

**The tables.** `reg[r]` is the register descriptor: up to `MAX` names for register `r`, `nin[r]` of them. The address descriptor is split in two: `inmem[k]` says whether name `k`'s memory copy is up to date, and the registers that hold it are found by searching `reg` (`holds`, `where`). `v` gives a name its slot the first time it is seen, in memory if it is a source variable and nowhere if it is a temporary.

**Looking ahead.** `usedlater` answers "is `s` read again in this block before it is written?". `live` is the same, except that a source variable still counts as live at the end, since the next block may read it. `cost` counts the stores needed to empty a register: its names that are live and have no up-to-date memory copy. `spill` writes those stores and empties the register.

**getreg** is two functions. `getreg_operand` is the operand rule, with `avoid` so that loading `z` never throws out the `y` just loaded. `getreg_result` is the five result rules in order; `done` is the test of rules 2 and 3.

**main** reads the block, then for each instruction prints it, gets the registers, writes the loads and the operation, updates the descriptors as in step 4 (`drop` takes `x` out of every register, then Rx gets `x` alone), and prints the descriptors with `show`. At the exit it stores every source variable that is in a register and not in memory.

### The paper's block, worked

The Dragon book's example block, with `t1`, `t2`, `t3` as temporaries and `a`, `b`, `c`, `d` live at the exit:

<!-- code: id=codegen-run -->
```console
$ gcc -Wall -Wextra codegen.c -o codegen
$ printf 't1 = a - b\nt2 = a - c\nt3 = t1 + t2\na = d\nd = t3 + t2\n' > block.tac
$ ./codegen < block.tac
t1 = a - b
    LD R1, a
    LD R2, b
    SUB R2, R1, R2
    registers: R1=a R2=t1 R3=
    addresses: a=a,R1 b=b t1=R2
t2 = a - c
    LD R3, c
    SUB R1, R1, R3
    registers: R1=t2 R2=t1 R3=c
    addresses: a=a b=b t1=R2 c=c,R3 t2=R1
t3 = t1 + t2
    ADD R2, R2, R1
    registers: R1=t2 R2=t3 R3=c
    addresses: a=a b=b t1=- c=c,R3 t2=R1 t3=R2
a = d
    LD R3, d
    registers: R1=t2 R2=t3 R3=d,a
    addresses: a=R3 b=b t1=- c=c t2=R1 t3=R2 d=d,R3
d = t3 + t2
    ADD R2, R2, R1
    registers: R1=t2 R2=d R3=a
    addresses: a=R3 b=b t1=- c=c t2=R1 t3=- d=R2
exit
    ST a, R3
    ST d, R2
```

Walk the decisions.

- **`t1 = a - b`**. Nothing is in a register: `a` into R1, `b` into R2 (both empty). For `t1`, rule 2 fails (`a` is used again by `t2 = a - c`), rule 3 holds (`b` is not used again, and is in memory), so `t1` overwrites R2: `SUB R2, R1, R2`. `a` is now in memory *and* R1; `b` only in memory.
- **`t2 = a - c`**. `a` is already in R1: no load. `c` goes into R3, the empty one. For `t2`, rule 2 holds now: `a` is not read again (the next mention of `a` *writes* it), and its memory copy is up to date. `t2` overwrites R1.
- **`t3 = t1 + t2`**. Both operands in registers: no loads. `t1` is dead after this, a temporary: rule 2, `t3` overwrites R2. `t1` is now nowhere, and nobody will miss it.
- **`a = d`**. A copy: `d` is loaded into R3, whose `c` is safe in memory and not needed again; then `a` joins `d` in R3. `a`'s memory copy is now out of date: `a` is only in R3.
- **`d = t3 + t2`**. `t3` dies here: `d` overwrites R2. `d` leaves R3, which now holds only `a`.
- **exit**. `a` (in R3) and `d` (in R2) are live and not in memory: store them. `b` and `c` were never changed. `t2` and `t3` are dead.

Ten instructions for five lines of three-address code: four loads, four operations, and only the two stores that the block's results require. The Dragon book's own table for this block makes some register choices differently. Both are correct: the algorithm fixes the rules, not every tie, and the examiner checks that the code computes the block and that the descriptors are right after each step.
<!-- risk: claim that the Dragon book's table for this block (2nd ed. Fig. 8.16, with t, u, v) differs in some register choices; check -->

Now with only two registers, where values must be spilled:

<!-- code: id=codegen-two -->
```console
$ ./codegen 2 < block.tac | grep -v 'registers:\|addresses:'
t1 = a - b
    LD R1, a
    LD R2, b
    SUB R2, R1, R2
t2 = a - c
    ST t1, R2
    LD R2, c
    SUB R1, R1, R2
t3 = t1 + t2
    LD R2, t1
    ADD R2, R2, R1
a = d
    ST t2, R1
    LD R1, d
d = t3 + t2
    ST a, R1
    LD R1, t2
    ADD R2, R2, R1
exit
    ST d, R2
```

At `t2 = a - c`, both registers are taken: R1 holds `a`, still needed as the operand, and R2 holds `t1`, needed later and nowhere else. Loading `c` must avoid R1, so R2 is emptied, which costs `ST t1, R2`, and `t1` is later loaded back for `t3`. Every spill is a store and a later load: the price of too few registers, and the reason real compilers spend so much effort on register allocation.

### When it goes wrong

**An operation on a value in memory.** `ADD R1, R2, a` is not an instruction of this machine; every operand must be loaded first.

**Forgetting that `x`'s memory copy is stale.** After `t3 = t1 + t2` writes R2, or after `a = d`, the old value of `a` in memory is wrong. An address descriptor that still lists `a`'s memory gives no store at the exit, and the program loses its result.

**Overwriting a register whose value is still needed.** Choosing Ry for `x` when `y` is read later, or when its only copy is in Ry, destroys a value. That is what rules 2 and 3 check, and what a spill repairs.

**Storing temporaries at the exit.** They are dead; the stores are wasted. Only source variables, live at the exit, are stored.

### In one line each

1. The code generator chooses registers and decides loads and stores; arithmetic works only on registers.
2. The register descriptor lists the names each register holds; the address descriptor lists where each name's current value is.
3. For `x = y op z`: getreg, load `y` and `z` if needed, `OP Rx, Ry, Rz`, then Rx holds only `x` and `x` is only in Rx.
4. getreg reuses an operand's register for the result when the operand dies there and is safe; otherwise an empty one, or the cheapest after a spill.
5. At the end of the block, store every live variable whose current value is only in a register.

**Check yourself**

<!-- q: idea=getreg type=apply wrong="every operand is loaded every time" -->
1. Generate code by hand, with the descriptors after each instruction, for the unoptimised running statement `t1 = rate * 60`, `t2 = init + t1`, `pos = t2`, with three registers. How many loads, and why is there exactly one store? Check with `codegen`.

<details>
<summary>Answer</summary>

<!-- code: id=codegen-pos -->
```console
$ printf 't1 = rate * 60\nt2 = init + t1\npos = t2\n' | ./codegen
t1 = rate * 60
    LD R1, rate
    LD R2, #60
    MUL R1, R1, R2
    registers: R1=t1 R2=60 R3=
    addresses: rate=rate t1=R1
t2 = init + t1
    LD R3, init
    ADD R3, R3, R1
    registers: R1=t1 R2=60 R3=t2
    addresses: rate=rate t1=R1 init=init t2=R3
pos = t2
    registers: R1=t1 R2=60 R3=t2,pos
    addresses: rate=rate t1=R1 init=init t2=R3 pos=R3
exit
    ST pos, R3
```

Three loads: `rate`, the constant 60, and `init`, each into an empty register. `t1` overwrites R1, since `rate` is not used again and is safe in memory (rule 2); `t2` overwrites R3, `init`'s register, for the same reason. `pos = t2` writes nothing: `pos` joins `t2` in R3. At the exit, `pos` is the only source variable not in memory, so it is the one store; `t1` and `t2` are temporaries and dead. Compare GCC's real code in section 1.1: one load of `rate`, the constant folded in, an add with `init`, and one store into `pos`. The shape is the same.

</details>

<!-- q: idea=code-generator type=fix wrong="a copy needs a load and a store" -->
2. A student's table for `a = d` in the worked block shows the code `LD R3, d` and `ST a, R3`, and leaves `a`'s address as "a, R3". What is wrong, and what does it cost?

<details>
<summary>Answer</summary>

The store is premature: `a`'s value only has to reach memory by the end of the block, and here `a` is not changed again, so `ST a, R3` belongs at the exit, where the algorithm writes it once. Storing at the copy is not wrong in its result, but it is an instruction the algorithm avoids, and if `a` were assigned again later in the block, it would be wasted. After the copy, `a`'s address should be only R3 (and `d`'s "d, R3"); with the store, "a, R3" would be right, but then no store is needed at the exit.

</details>

## 10.7 Activation records and storage allocation

### Where a call keeps its things

Everything so far assumed that a name such as `pos` has one home in memory. A recursive function breaks that: in `fact(4)`, which calls `fact(3)`, which calls `fact(2)`, there are three `n`s alive at once, with three different values, and the code of `fact` is the same for all of them. Each *call* needs its own copy of the function's parameters and locals, made when the call starts and thrown away when it returns. The block of memory that holds one call's data is its *activation record*, or *frame*, and the paper asks for its fields and for the three ways a compiler allocates storage.

<!-- teaches: activation-record = activation record, control stack, static allocation, stack allocation, heap allocation -->
**Three kinds of storage.** *Static* allocation fixes a name's address when the program is compiled: global variables and `static` locals, one copy for the whole run. *Stack* allocation puts each call's activation record on the *control stack*, pushed at the call and popped at the return; calls nest, so the last record made is always the first freed, which is exactly what a stack does. *Heap* allocation is for data that outlives the call that made it, such as what `malloc` returns and `strdup` in chapter 8; it is freed in no fixed order, explicitly or by a garbage collector.

**The fields of an activation record**, in the Dragon book's order:

| Field | Holds |
|---|---|
| actual parameters | the values the caller passes |
| returned value | the result, for the caller to collect |
| control link | a pointer to the caller's activation record |
| access link | a pointer to the record of the enclosing function, for languages with nested functions (C has none) |
| saved machine status | the return address and the registers the call must restore |
| local data | the function's local variables |
| temporaries | values such as the `t1`, `t2` of chapter 9 that did not fit in registers |
<!-- risk: field order and names of the activation record follow Dragon 2nd ed. Fig. 7.5; check; real x86-64 frames pass the first parameters in registers -->

The records of the calls in progress form a chain through their control links, the *call stack* a debugger shows. Here are three of them, seen from C:

<!-- code: id=frames -->
```c
/* frames.c: each call of fact has its own n, in its own activation
   record; the addresses show the stack growing as the calls go deeper. */
#include <stdio.h>

int fact(int n, int *callers_n)     /* callers_n: the caller's own n */
{
    if (callers_n != NULL)
        printf("fact(%d): its n is %ld bytes from its caller's n\n",
               n, (long)((char *)&n - (char *)callers_n));
    else
        printf("fact(%d): the first call\n", n);
    if (n <= 1)
        return 1;
    return n * fact(n - 1, &n);
}

int main(void)
{
    printf("4! = %d\n", fact(4, NULL));
    return 0;
}
```
```text
fact(4): the first call
fact(3): its n is -32 bytes from its caller's n
fact(2): its n is -32 bytes from its caller's n
fact(1): its n is -32 bytes from its caller's n
4! = 24
```

Every call's `n` is 32 bytes *below* its caller's: each activation record of `fact` takes 32 bytes here, and on x86-64 the stack grows towards lower addresses. The size and the direction belong to this machine and this compiler; the four separate `n`s, one per record, belong to every machine with recursion.
<!-- checked: how=run date=2026-11-21 ref="frames.c, GCC 13 -O0 x86-64: each n 32 bytes below its caller's" -->

**Check yourself**

<!-- q: idea=activation-record type=decide wrong="all locals live on the stack" -->
1. For each, say whether its storage is static, on the stack, or on the heap: (a) `errors` in `tiny.l`; (b) `n` in `fact`; (c) the string `strdup` returns in `post.l`; (d) `t1` in `expr-tac.y`'s `newtemp`, the `char name[16]`.

<details>
<summary>Answer</summary>

(a) Static: a global, one copy for the run. (b) Stack: one per call, in that call's activation record. (c) Heap: it must outlive the scanner's action that made it, until the parser's action frees it. (d) Stack: a local array of `newtemp`, gone when `newtemp` returns, which is exactly why `newtemp` returns `strdup(name)`, a heap copy, and never `name` itself.

</details>

## 10.8 Practice: the mini compiler, end to end

The mini compiler is complete. A Tiny program goes through `tiny-tac` (the Flex scanner of chapter 4, the Yacc parser of chapter 8 and the translation of chapter 9) to three-address code; `tiny-opt` optimises it block by block (10.2); `blocks` shows its basic blocks and flow graph (10.1); `codegen` turns one block into register code with its descriptors (10.6); and `tacrun` checks at every step that nothing changed what the program computes. The paper's last questions take a fragment of code through the same stations by hand: blocks and flow graph, DAG, loop optimisations, code generation. The lab's last program is usually a back end: "generate target code from three-address code". These tasks do the hand questions on code you have not seen, each checked by a program, and end with that back end, which joins the stations into a compiler from Tiny source to assembly. Work in `~/cdlab/ch10`.

### Guided

<!-- q: idea=flow-graph type=apply wrong="the if inside the loop adds no block" -->
**1. A new program through the chain.** Save this as `squares.tiny`. Before running anything, predict how many basic blocks its optimised code has, draw the flow graph, and work out the final value of `s`. Then check all three with `tiny-tac`, `tiny-opt`, `blocks` and `tacrun`.

```c
n = 5; i = 1; s = 0;
while (i <= n) {
    if (i != 3) s = s + i * i;
    i = i + 1;
}
```

<details>
<summary>Answer</summary>

<!-- code: id=practice-squares -->
```console
$ cat > squares.tiny <<'END'
n = 5; i = 1; s = 0;
while (i <= n) {
    if (i != 3) s = s + i * i;
    i = i + 1;
}
END
$ ./tiny-tac < squares.tiny | ./tiny-opt > squares.tac
$ ./blocks < squares.tac | sed -n '/Flow graph/,$p'
Flow graph
B1 -> B2
B2 -> B6 B3
B3 -> B5 B4
B4 -> B5
B5 -> B2
B6 -> exit
$ ./tacrun < squares.tac
n = 5
i = 6
s = 46
```

Six blocks: B1 the three assignments; B2 the loop test at `L1`; B3 the `if` test; B4 the body of the `if`, `t1 = i * i` and `s = s + t1`; B5 at `L3`, the `i = i + 1` and the jump back, where both paths of the `if` meet; B6 the exit. The flow graph has the loop B2 → B3 → (B4 →) B5 → B2, with B3 → B5 skipping B4 when `i` is 3. `s` is 1 + 4 + 16 + 25 = 46, without the 9. The `if` adds two blocks, its body and the meeting point after it.

</details>

<!-- q: idea=dag type=decide wrong="b * c and c * b are always one node" -->
**2. Same value, different text.** Draw the DAG for this block, first as the paper usually wants it, then treating `*` as commutative. In each case, what is `x`?

```text
t1 = a + b
t2 = c * t1
t3 = a + b
t4 = t3 * c
x = t2 - t4
```

<details>
<summary>Answer</summary>

Without commutativity: `t3 = a + b` reuses node 1 (`+ (a0, b0)`), so `t3` joins `t1`. `t2 = c * t1` is `* (c0, 1)`; `t4 = t3 * c` is `* (1, c0)`, the same children in the *other order*, so a new node. Two `*` nodes, and `x = t2 - t4` computes the difference of two equal values: `t1 = a + b`, `t2 = c * t1`, `t4 = t1 * c`, `x = t2 - t4`. With commutativity, `* (1, c0)` is recognised as `* (c0, 1)`: one `*` node, and `x` is that node minus itself, which a builder that also knows `y - y = 0` reduces to `x = 0`. `tiny-opt` compares operands in order, as the paper does by default, so it finds the `a + b` and not the product. Say which convention you use; both are correct with their assumption.

</details>

### On your own

<!-- q: idea=code-motion type=apply wrong="(k + 1) * i is loop-invariant" -->
**3. A loop to improve.** Save this as `kloop.tiny`, translate and optimise it with the tools, and then, by hand, apply code motion and strength reduction to the loop. Check with `tacrun` that `j` does not change.

```c
n = 8; k = 5; i = 0; j = 0;
while (i < n) {
    j = j + (k + 1) * i;
    i = i + 1;
}
```

<details>
<summary>Answer</summary>

<!-- code: id=practice-kloop -->
```console
$ cat > kloop.tiny <<'END'
n = 8; k = 5; i = 0; j = 0;
while (i < n) {
    j = j + (k + 1) * i;
    i = i + 1;
}
END
$ ./tiny-tac < kloop.tiny | ./tiny-opt
n = 8
k = 5
i = 0
j = 0
L1:
ifFalse i < n goto L2
t1 = k + 1
t2 = t1 * i
j = j + t2
i = i + 1
goto L1
L2:
```

`t1 = k + 1` is invariant (`k` is not assigned in the loop) and is a temporary, so it moves to the preheader. `t2 = t1 * i` is not invariant, since it reads `i`, but it is an induction variable in `i`'s family: initialise it in the preheader and add `t1` to it after `i = i + 1`.

<!-- code: id=practice-kloop2 -->
```console
$ cat > kloop2.tac <<'END'
n = 8
k = 5
i = 0
j = 0
t1 = k + 1
t2 = t1 * i
L1:
ifFalse i < n goto L2
j = j + t2
i = i + 1
t2 = t2 + t1
goto L1
L2:
END
$ ./tiny-tac < kloop.tiny | ./tiny-opt | ./tacrun
n = 8
k = 5
i = 8
j = 168
$ ./tacrun < kloop2.tac
n = 8
k = 5
i = 8
j = 168
```

`j` is 168 both times, 6 × (0 + 1 + ... + 7). The loop body lost its addition `k + 1` and its multiplication. The usual wrong answer moves `(k + 1) * i` out whole: it reads `i`, and `j` would then be 0.

</details>

<!-- q: idea=getreg type=apply wrong="the repeated b * c is computed twice" -->
**4. Optimise, then generate.** For the block below, first remove the common subexpression (by DAG or with `tiny-opt`), then generate code with two registers, showing the descriptors after each instruction. How many instructions does the optimised block need, and how many the original? Check with `codegen 2`.

```text
t1 = b * c
t2 = a + t1
t3 = b * c
x = t2 - t3
```

<details>
<summary>Answer</summary>

<!-- code: id=practice-codegen -->
```console
$ printf 't1 = b * c\nt2 = a + t1\nt3 = b * c\nx = t2 - t3\n' > cse.tac
$ ./tiny-opt < cse.tac > cse2.tac
$ cat cse2.tac
t1 = b * c
t2 = a + t1
x = t2 - t1
$ ./codegen 2 < cse2.tac
t1 = b * c
    LD R1, b
    LD R2, c
    MUL R1, R1, R2
    registers: R1=t1 R2=c
    addresses: b=b c=c,R2 t1=R1
t2 = a + t1
    LD R2, a
    ADD R2, R2, R1
    registers: R1=t1 R2=t2
    addresses: b=b c=c t1=R1 a=a t2=R2
x = t2 - t1
    SUB R2, R2, R1
    registers: R1=t1 R2=x
    addresses: b=b c=c t1=R1 a=a t2=- x=R2
exit
    ST x, R2
$ ./codegen 2 < cse.tac | grep -c '^    [A-Z]'
12
```

`t3` is the same node as `t1`, so the block becomes `t1 = b * c`, `t2 = a + t1`, `x = t2 - t1`. With two registers: `b` into R1, `c` into R2, `MUL R1, R1, R2` (`b` is dead and safe, so `t1` takes R1); `a` into R2, which `c` held but no longer needs, then `ADD R2, R2, R1`; `SUB R2, R2, R1`, overwriting `t2`, which dies there; and at the exit `ST x, R2`. Seven instructions. The original block, with the multiplication done twice and only two registers, needs twelve: the last command counts the instructions `codegen` writes for it, spills included. Optimising first is cheaper than generating well.

</details>

### A lab program from a blank file

<!-- q: idea=code-generator type=build unseen="asm-backend" wrong="ifFalse a < b jumps on JLT" -->
**5. asm.c: a back end.** Write `asm.c`, which turns the three-address code that `tiny-tac` and `tiny-opt` write into assembly for a machine with one working register, R0. It reads the code on standard input and writes one line per instruction, indented by four blanks, with labels unindented, as follows. A constant operand is written with `#` in front (`#48`); a number is a word that starts with a digit, `.` or `-`.

| Three-address code | Assembly |
|---|---|
| `L1:` | `L1:` |
| `goto L` | `JMP L` |
| `x = y` | `MOV R0, y`, `MOV x, R0` |
| `x = minus y` | `MOV R0, y`, `NEG R0`, `MOV x, R0` |
| `x = y op z`, op one of `+ - * /` | `MOV R0, y`, `ADD R0, z` (or `SUB`, `MUL`, `DIV`), `MOV x, R0` |
| `x = y relop z` | `MOV R0, y`, `CMP R0, z`, `SETcc R0`, `MOV x, R0` |
| `if y relop z goto L` | `MOV R0, y`, `CMP R0, z`, `Jcc L` |
| `ifFalse y relop z goto L` | the same, with the *opposite* condition |
| `if a goto L` / `ifFalse a goto L` | `MOV R0, a`, `CMP R0, #0`, `JNE L` / `JEQ L` |

The condition codes `cc` are `LT LE GT GE EQ NE` for `< <= > >= == !=`; the opposite of `<` is `GE`, of `<=` is `GT`, and so on. For anything else, it prints `asm: line N: cannot translate: ` and the line to standard error and exits with status 1.

**Done when** `./asm` with `gcd.tiny`'s optimised code (from `./tiny-tac < gcd.tiny | ./tiny-opt`) as its input prints exactly the model's output below. You can test that without anyone marking it: in the understudy folder (the one that holds `book/`), in the terminal, run `python3 scripts/us_practice.py check compiler-design 10.8 --in ~/cdlab/ch10`. It builds your `asm.c` and compares what it prints for this input with the model. Then test it, as habit 3 asks, on an empty input, on `prog.tiny`'s code, which has a comparison stored as a value, and on `x = y % 2`, which must be refused with status 1.

<details>
<summary>Answer</summary>

<!-- code: id=asm check="gcc -Wall -Wextra asm.c -o asm && ./asm" stdin="a = 48\nb = 18\nL1:\nifFalse a != b goto L2\nifFalse a > b goto L3\na = a - b\ngoto L4\nL3:\nb = b - a\nL4:\ngoto L1\nL2:\n" -->
```c
/* asm.c: three-address code to assembly for a one-register machine.
   Reads tiny-tac's (or tiny-opt's) code on standard input and writes
   MOV/ADD/SUB/MUL/DIV/NEG, CMP, SETcc and jumps, using R0. */
#include <stdio.h>
#include <string.h>
#include <ctype.h>

const char *rel[] = { "<", "<=", ">", ">=", "==", "!=" };
const char *cc[]  = { "LT", "LE", "GT", "GE", "EQ", "NE" };   /* true when */
const char *ncc[] = { "GE", "GT", "LE", "LT", "NE", "EQ" };   /* false when */

int relop(const char *op)           /* index in rel[], or -1 */
{
    for (int i = 0; i < 6; i++)
        if (strcmp(op, rel[i]) == 0)
            return i;
    return -1;
}

const char *arith(const char *op)
{
    return strcmp(op, "+") == 0 ? "ADD" : strcmp(op, "-") == 0 ? "SUB"
         : strcmp(op, "*") == 0 ? "MUL" : strcmp(op, "/") == 0 ? "DIV" : NULL;
}

const char *addr(const char *s)     /* a constant is written #c */
{
    static char buf[2][20];
    static int k;
    k = !k;
    if (isdigit((unsigned char)s[0]) || s[0] == '.' || s[0] == '-')
        snprintf(buf[k], sizeof buf[k], "#%s", s);
    else
        snprintf(buf[k], sizeof buf[k], "%s", s);
    return buf[k];
}

int main(void)
{
    char line[100], w[6][16];
    int k, r, lineno = 0;

    while (fgets(line, sizeof line, stdin)) {
        lineno++;
        k = sscanf(line, "%15s %15s %15s %15s %15s %15s",
                   w[0], w[1], w[2], w[3], w[4], w[5]);
        if (k <= 0)
            continue;
        if (w[0][strlen(w[0]) - 1] == ':') {               /* L1: */
            printf("%s\n", w[0]);
        } else if (strcmp(w[0], "goto") == 0) {            /* goto L */
            printf("    JMP %s\n", w[1]);
        } else if (strcmp(w[0], "if") == 0 || strcmp(w[0], "ifFalse") == 0) {
            int t = strcmp(w[0], "if") == 0;
            printf("    MOV R0, %s\n", addr(w[1]));
            if (k == 6 && (r = relop(w[2])) >= 0) {        /* if a relop b goto L */
                printf("    CMP R0, %s\n", addr(w[3]));
                printf("    J%s %s\n", t ? cc[r] : ncc[r], w[5]);
            } else {                                       /* if a goto L */
                printf("    CMP R0, #0\n");
                printf("    J%s %s\n", t ? "NE" : "EQ", w[3]);
            }
        } else if (k == 3) {                               /* x = y */
            printf("    MOV R0, %s\n", addr(w[2]));
            printf("    MOV %s, R0\n", w[0]);
        } else if (k == 4 && strcmp(w[2], "minus") == 0) { /* x = minus y */
            printf("    MOV R0, %s\n", addr(w[3]));
            printf("    NEG R0\n");
            printf("    MOV %s, R0\n", w[0]);
        } else if (k == 5 && arith(w[3])) {                /* x = y op z */
            printf("    MOV R0, %s\n", addr(w[2]));
            printf("    %s R0, %s\n", arith(w[3]), addr(w[4]));
            printf("    MOV %s, R0\n", w[0]);
        } else if (k == 5 && (r = relop(w[3])) >= 0) {     /* x = y relop z */
            printf("    MOV R0, %s\n", addr(w[2]));
            printf("    CMP R0, %s\n", addr(w[4]));
            printf("    SET%s R0\n", cc[r]);
            printf("    MOV %s, R0\n", w[0]);
        } else {
            fprintf(stderr, "asm: line %d: cannot translate: %s", lineno, line);
            return 1;
        }
    }
    return 0;
}
```
```text
    MOV R0, #48
    MOV a, R0
    MOV R0, #18
    MOV b, R0
L1:
    MOV R0, a
    CMP R0, b
    JEQ L2
    MOV R0, a
    CMP R0, b
    JLE L3
    MOV R0, a
    SUB R0, b
    MOV a, R0
    JMP L4
L3:
    MOV R0, b
    SUB R0, a
    MOV b, R0
L4:
    JMP L1
L2:
```

**How it works.** Each line is split into words, and the first word and the number of words decide the kind, as in `tacrun`. Two small tables carry the condition codes: `cc[r]` is the code that is true when comparison `r` holds, `ncc[r]` the one that is true when it fails, which is what `ifFalse` needs: `ifFalse a != b goto L2` jumps when `a == b`, so it becomes `JEQ L2`. `addr` writes a constant with `#`; it alternates between two buffers, because one `printf` sometimes needs two of its results at once. Every value passes through R0: the simplest possible register allocation, one register and every result stored at once. Set the output beside `codegen`'s: there, values stayed in registers across instructions, and only the block's results were stored. That difference is what section 10.6's descriptors buy.

The whole compiler, from source to assembly, is now one command line:

<!-- code: id=asm-chain -->
```console
$ gcc -Wall -Wextra asm.c -o asm
$ ./tiny-tac < gcd.tiny | ./tiny-opt | ./asm | head -8
    MOV R0, #48
    MOV a, R0
    MOV R0, #18
    MOV b, R0
L1:
    MOV R0, a
    CMP R0, b
    JEQ L2
$ ./tiny-tac < prog.tiny | ./tiny-opt | ./asm | grep -c .
22
```

And the tests of habit 3:

<!-- code: id=asm-tests exit=1 -->
```console
$ ./asm < /dev/null
$ ./tiny-tac < prog.tiny | ./tiny-opt | ./asm | grep -A3 'CMP R0, #0'
    CMP R0, #0
    SETNE R0
    MOV t3, R0
    MOV R0, #7.25
$ printf 'x = y %% 2\n' | ./asm
asm: line 1: cannot translate: x = y % 2
```

`grep -A3` prints each matching line and the three after it. `prog.tiny`'s stored comparison, `t3 = rate != 0`, became `CMP` and `SETNE`. And `%`, which Tiny does not have, is refused. From `pos = init + rate * 60;` in chapter 1 to `MOV R0, #160` and `MOV pos, R0` here, every station of the assembly line is a program you wrote.

</details>
