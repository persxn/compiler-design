# Chapter 3: Regular expressions and finite automata

<!-- contents -->
- [3.1 Regular expressions as patterns: union, concatenation, star and the shorthands](#31-regular-expressions-as-patterns-union-concatenation-star-and-the-shorthands)
- [3.2 From a regular expression to an NFA: Thompson's construction](#32-from-a-regular-expression-to-an-nfa-thompsons-construction)
- [3.3 Epsilon-closure and the subset construction, by hand and in C](#33-epsilon-closure-and-the-subset-construction-by-hand-and-in-c)
- [3.4 A DFA straight from a regular expression: nullable, firstpos, lastpos, followpos](#34-a-dfa-straight-from-a-regular-expression-nullable-firstpos-lastpos-followpos)
- [3.5 Minimising a DFA by splitting groups](#35-minimising-a-dfa-by-splitting-groups)
- [3.6 A table-driven DFA recogniser in C](#36-a-table-driven-dfa-recogniser-in-c)
- [3.7 Practice: one regular expression carried to a minimal DFA and a C recogniser](#37-practice-one-regular-expression-carried-to-a-minimal-dfa-and-a-c-recogniser)
<!-- /contents -->

## 3.1 Regular expressions as patterns: union, concatenation, star and the shorthands

### Writing a pattern down so a machine can read it

Section 1.3 called the rule behind a token name its *pattern*, and gave the patterns in English: "an identifier is a letter followed by letters and digits". In chapter 2 you turned each such sentence into C by thinking hard about it, and section 2.5 drew each one as a board before coding it. That worked for Tiny, but English has two faults that matter from now on.

The first is that English is loose. Does "letters and digits" allow an underscore? May a number be `5.`? You answered these questions one at a time inside `gettoken`, and the answers are now scattered through the code. The second fault is worse: no program can read English. Chapter 4 hands the patterns to Flex, which writes the scanner for you, and Flex needs the pattern for each token in a notation precise enough to build a machine from. The written paper asks for the same notation: "write a regular expression for identifiers", "for strings over {a, b} ending in abb", "construct the automaton for (a|b)*abb".

That notation is the *regular expression*. This section teaches the three operations it is built from, the shorthands that make it pleasant to write, and the way to give names to pieces of it. By the end you will have a small C program that tests any pattern against any strings, so that every expression you write by hand in this chapter can be checked before you trust it.

### Strings, languages and the empty string

Three words first, because the operations are defined with them.

An *alphabet* is a finite set of symbols. For a scanner it is the character set; for the paper's exercises it is often just `{a, b}` or `{0, 1}`. A *string* is a finite sequence of symbols from the alphabet: `abb`, `rate`, `60`. The string with no symbols at all is written ε (the Greek letter epsilon). It is a real string with length zero, in the same way that `""` is a real C string with `strlen` zero.

A *language* is a set of strings. It may be finite, like `{a, b}`, or infinite, like "every string of a's and b's that ends in `abb`". The language of identifiers is the infinite set `{a, b, ..., pos, init, rate, rate2, ...}`. A pattern, in the sense of section 1.3, is a description of a language: a lexeme matches the pattern when it is one of the strings in the set.

A regular expression `r` is a formula, and `L(r)` is the language it describes. The formula is finite; the language usually is not. That is the whole point: a finite formula you can type, standing for an infinite set you cannot.

### Three operations and two kinds of brick

<!-- teaches: regex = regular expression, union, concatenation, Kleene star, precedence -->
A *regular expression* over an alphabet is built from two kinds of brick and three ways of joining them:

| Expression | Its language | In words |
|---|---|---|
| `ε` | `{ε}` | only the empty string |
| `a` (one symbol) | `{a}` | only the one-symbol string `a` |
| `r|s` | `L(r)` together with `L(s)` | *union*: a string from either |
| `rs` | every `xy` with `x` from `L(r)`, `y` from `L(s)` | *concatenation*: a string from `r` followed by one from `s` |
| `r*` | `{ε}`, `L(r)`, `L(rr)`, `L(rrr)`, ... together | *Kleene star*: zero or more strings from `r`, one after another |
| `(r)` | `L(r)` | parentheses only group |

Nothing else is a regular expression. Every pattern in this chapter, and every Flex pattern in the next, is these bricks and these joins, sometimes written with a shorthand.

Without parentheses the operators bind in a fixed order, the *precedence*: star binds tightest, then concatenation, then union. All three group from the left. So `a|bc*` means `a|(b(c*))`, never `(a|b)c*` and never `a|(bc)*`. This is the same idea as `*` binding tighter than `+` in `init + rate * 60`, and it is the source of most wrong answers.

Some exam papers and automata textbooks write union as `+`, so that `(a+b)*abb` means exactly what this book writes as `(a|b)*abb`. When you see `+` between two expressions inside a question, read it as `|`. The shorthand `r+` below is a different thing, written after a single expression, and you can always tell them apart by what comes after the `+`.
<!-- risk: claim that some exam papers and textbooks use + for union -->

### Working out a language by hand

Take the alphabet `{a, b}` and build up, one operation at a time, the expression this chapter carries all the way to a C program.

1. `a|b`: one string from `{a}` or one from `{b}`. The language is `{a, b}`: every string of length one.
2. `(a|b)(a|b)`: a string from `{a, b}` followed by another. That gives `aa`, `ab`, `ba`, `bb`: every string of length two.
3. `(a|b)*`: zero or more strings from `{a, b}` in a row. Zero gives ε, one gives `a` and `b`, two gives the four above, and so on. The language is every string over the alphabet, including the empty one.
4. `(a|b)*abb`: any string over the alphabet, followed by exactly `abb`. The language is every string of a's and b's that ends in `abb`: `abb`, `aabb`, `babb`, `abbabb`, ...

To test whether a given string is in `L((a|b)*abb)`, you split it into the parts the expression asks for. For `babb`: let the star take `b` (one trip round), then `a`, `b`, `b` match the last three symbols. It fits. For `abba`: whatever the star takes, the string has to end with `abb`, and `abba` ends with `bba`. It does not fit. For `abb` itself: the star takes nothing at all, which it is allowed to do, and `abb` matches the rest.

That last case is the one people miss. A star may take zero trips, so `L(a*)` contains ε, and `(a|b)*abb` matches `abb` with nothing in front.

Now the precedence trap in action. Compare `ab*` and `(ab)*`:

- `ab*` is `a(b*)`: one `a`, then zero or more b's. Its language: `a`, `ab`, `abb`, `abbb`, ... It never contains ε, and every string starts with `a`.
- `(ab)*` is the pair `ab`, zero or more times: ε, `ab`, `abab`, ... It contains ε, and `abb` is not in it.

They look almost the same on paper, and they describe different sets. When you write an expression in the paper, put the parentheses in whenever you are not certain the precedence does what you mean; extra parentheses never change a language.

### The shorthands

Three operations are enough to describe every pattern a scanner needs, but writing `0|1|2|3|4|5|6|7|8|9` every time you mean "a digit" is miserable. So every practical notation adds shorthands. Each one is an abbreviation for something you could already write; none of them adds power.

| Shorthand | Means | Example |
|---|---|---|
| `r+` | `rr*`: one or more | `digit+` is at least one digit |
| `r?` | `r|ε`: zero or one | `(.digits)?` is an optional fraction |
| `[abc]` | `a|b|c` | `[ab]*abb` is `(a|b)*abb` |
| `[a-z]` | `a|b|...|z`, the range in character order | `[0-9]` is a digit |
| `[^abc]` | any one character except those listed | `[^\n]` is any character but newline |

<!-- teaches: regular-definition = regular definition, character class -->
A bracketed set is called a *character class*. Inside it, `-` between two characters means a range, and a `^` right after the `[` means "not". The class `[A-Za-z_]` is "a letter or an underscore", exactly what `isalpha(c) || c == '_'` said in section 2.2.

Long patterns are still hard to read in one line, so the Dragon book (and Flex, in chapter 4) lets you give names to pieces and use the names in later pieces. A *regular definition* is a list of lines `name -> expression`, where each expression may use the names defined above it, and never its own name or a later one. Here are Tiny's identifiers and numbers as a regular definition, written to match exactly what `tiny-scan` accepts:

```text
letter_  ->  [A-Za-z_]
digit    ->  [0-9]
id       ->  letter_ ( letter_ | digit )*
digits   ->  digit+
number   ->  digits ( . digits )?
```

Read `number` slowly. `digits` is one or more digits; then, optionally, a point followed by one or more digits. So `60` and `2.5` match, and `5.` does not, because after the point the `?` part needs at least one digit and there is none. That is Tiny's rule "a digit must follow the point" from section 2.3, now written in one line instead of found by reading the code. The point here is a literal character; in the notations that give `.` a special meaning (POSIX and Flex do, as you will see in a moment), it must be written `\.`.

Because no name may use itself or a later name, you can always expand a regular definition back into one plain regular expression by substituting from the top down. That rule is what keeps a regular definition regular: it can never describe a recursive structure. Hold on to that; it is the limit of this whole chapter, and it comes back at the end of the section.

Now apply the definitions to the running statement, `pos = init + rate * 60;`. The lexemes `pos`, `init` and `rate` each match `id`: a `letter_`, then `letter_` or `digit` symbols. The lexeme `60` matches `number` with the optional part empty. The operators each need their own tiny expression, one symbol long: `=`, `+`, `*`, `;`. A whole scanner's worth of patterns is a list like this, one expression per token name, and chapter 4 hands such a list to Flex.

### Testing patterns with a program

You will write many expressions in this chapter, and your eye is not a reliable judge of which strings they match. The C library on Linux has a regular-expression matcher, declared in `regex.h`, that understands the POSIX *extended* notation: `|`, `*`, parentheses, and the shorthands `+`, `?` and bracket classes, written exactly as in the tables above. A dozen lines of C turn it into a testing tool. You give it a pattern as an argument and some strings on standard input, one per line, and it says for each line whether the whole line is in the pattern's language.

Make this chapter's folder first. Open a terminal in your home folder:

<!-- code: id=ch3-setup -->
```console
$ mkdir cdlab/ch3
$ cd cdlab/ch3
```

Save this as `match.c`:

<!-- code: id=match mode=compile -->
```c
/* match.c: is each input line, whole, in the language of a pattern?
   The pattern is a POSIX extended regular expression, given as argv[1]. */
#include <stdio.h>
#include <string.h>
#include <regex.h>

int main(int argc, char *argv[])
{
    char whole[512], line[256], msg[128];
    regex_t re;
    int err;

    if (argc != 2) {
        fprintf(stderr, "usage: %s 'pattern' < lines\n", argv[0]);
        return 2;
    }
    /* ^( ... )$ ties the pattern to the start and end of the line,
       so that it must match the whole line, not just a piece of it */
    snprintf(whole, sizeof whole, "^(%s)$", argv[1]);
    err = regcomp(&re, whole, REG_EXTENDED | REG_NOSUB);
    if (err != 0) {
        regerror(err, &re, msg, sizeof msg);
        fprintf(stderr, "bad pattern: %s\n", msg);
        return 2;
    }
    while (fgets(line, sizeof line, stdin) != NULL) {
        line[strcspn(line, "\n")] = '\0';           /* drop the newline */
        printf("%-4s \"%s\"\n",
               regexec(&re, line, 0, NULL, 0) == 0 ? "yes" : "no", line);
    }
    regfree(&re);
    return 0;
}
```

`regcomp` translates the pattern into the library's own internal form, once; `regexec` then tests one string against it and returns 0 when it matches. `REG_EXTENDED` selects the extended notation, and `REG_NOSUB` says you only want yes or no, not where the match is. The `^` and `$` added around your pattern mean "start of the line" and "end of the line". Without them `regexec` would say yes whenever *some piece* of the line matched, and `xabbx` would pass the test for `(a|b)*abb`, because it contains `abb`. A pattern for a token must describe the whole lexeme, so the program asks the whole-line question.

Build it and try the expression from above on six strings, the last one empty:

<!-- code: id=match-abb -->
```console
$ gcc -Wall -Wextra match.c -o match
$ printf 'abb\nbabb\naabb\nab\nabba\n\n' | ./match '(a|b)*abb'
yes  "abb"
yes  "babb"
yes  "aabb"
no   "ab"
no   "abba"
no   ""
```

The single quotes around the pattern matter. Without them the shell itself would try to read `(`, `|` and `*` as its own syntax (a subshell, a pipe and a file-name wildcard) before your program ever saw them.

Every line of the output is a prediction you can make first. `abb` is yes because the star may take nothing. `babb` and `aabb` are yes. `ab` is too short to end in `abb`. `abba` does not end in `abb`. The empty string does not end in `abb` either.

Now the precedence pair, on the same five strings:

<!-- code: id=match-prec -->
```console
$ printf '\na\nab\nabb\nabab\n' | ./match 'ab*'
no   ""
yes  "a"
yes  "ab"
yes  "abb"
no   "abab"
$ printf '\na\nab\nabb\nabab\n' | ./match '(ab)*'
yes  ""
no   "a"
yes  "ab"
no   "abb"
yes  "abab"
```

Look at the first line of each answer: the empty string is in `(ab)*` and not in `ab*`. Then `abb` swaps sides. The two expressions differ by two parentheses and agree on only one of these five strings, `ab`.

Finally, the regular definitions for Tiny, expanded into POSIX notation (the library has no names, so you substitute them by hand):

<!-- code: id=match-tiny -->
```console
$ printf 'pos\ninit\nrate\n_v1\n9lives\nrate2\n' | ./match '[A-Za-z_][A-Za-z0-9_]*'
yes  "pos"
yes  "init"
yes  "rate"
yes  "_v1"
no   "9lives"
yes  "rate2"
$ printf '60\n2.5\n5.\n.5\n007\n2.5.1\n' | ./match '[0-9]+(\.[0-9]+)?'
yes  "60"
yes  "2.5"
no   "5."
no   ".5"
yes  "007"
no   "2.5.1"
```

The identifier pattern accepts every name in the running statement and refuses `9lives`, which starts with a digit. The number pattern accepts `60` and `2.5`, refuses `5.` (Tiny's rule) and `.5` (a number must start with a digit), accepts `007` (nothing forbids leading zeros), and refuses `2.5.1`, because the optional fraction can appear only once. Each of those answers is a decision about Tiny that is now written down in one line.

### What a regular expression cannot say

Regular expressions are not the right notation for everything a compiler reads. Try to write one for "strings of balanced parentheses": `()`, `(())`, `()(())`, and so on. You can write `(\()*(\))*` for "some opens followed by some closes", but nothing in the notation makes the number of closes equal the number of opens. The star repeats without counting, and no fixed expression can count to an arbitrary depth. Chapter 5 brings in a stronger notation for the structure of statements and expressions, where nesting is the whole point. The division of labour between the stations of the assembly line follows this limit: the scanner handles everything regular (names, numbers, operators, comments), and the parser handles everything that nests.

### Equal expressions

Different expressions can describe the same language, and the paper sometimes asks you to show it. A few equalities are worth knowing by heart, because they also let you simplify an expression before building a machine for it:

| Equality | Why it holds |
|---|---|
| `r|s = s|r` | union does not care about order |
| `εr = rε = r` | the empty string adds nothing to a concatenation |
| `r** = r*` | repeating "zero or more" zero or more times changes nothing |
| `(r|s)* = (r*s*)*` | both are "any mixture of r-strings and s-strings" |
| `r* = ε|rr*` | zero trips, or one trip followed by zero or more |
| `r(sr)* = (rs)*r` | both are `r`, `rsr`, `rsrsr`, ... |

Concatenation, by contrast, does care about order: `ab` and `ba` are different expressions with different languages. To show that two expressions are *not* equal, you need one string that is in one language and not in the other, as `abb` was for `ab*` and `(ab)*`. To check a claimed equality before an exam, put the same test strings through `match` with both patterns and compare the two columns.

### When it goes wrong

**The empty string sneaks in.** `[a-z]*` for "identifiers" accepts the empty string, because a star may take zero trips. A token pattern that matches ε is a bug: a scanner using it could produce a token without reading any input. Use `+` or a first symbol outside the star: `[a-z][a-z0-9]*`.

**Precedence surprises.** You meant "an `a` or a `b`, then a `c`" and wrote `a|bc`. That is `a|(bc)`; `ac` is not in it. Run the strings through `match`; the column of yes and no answers shows the grouping you really wrote.

**The shell eats the pattern.** Typed without quotes, `./match (a|b)*abb` never reaches your program; the shell reports a syntax error near `(` because parentheses and `|` are shell syntax. Always wrap the pattern in single quotes.
<!-- risk: what bash prints for ./match (a|b)*abb without quotes; not run -->

**A broken pattern.** An unmatched parenthesis makes `regcomp` fail, and the program prints the library's own explanation:

<!-- code: id=match-bad exit=2 -->
```console
$ printf 'abb\n' | ./match '(a|b*abb'
bad pattern: Unmatched ( or \(
```

The fix is in the pattern, not in the program: count the parentheses.

### In one line each

1. A regular expression describes a language, a set of strings, with a finite formula.
2. The bricks are ε and single symbols; the joins are union `|`, concatenation, and star `*`, which binds tightest, then concatenation, then union.
3. `+`, `?` and character classes are shorthands; they add convenience, not power.
4. A regular definition names pieces of an expression; each line may use only names defined above it, so it can always be expanded into one expression.
5. A pattern tested against whole lexemes must be anchored; `match` adds `^( )$` for you.
6. Regular expressions cannot count, so nesting is left to the parser.

**Check yourself**

<!-- q: idea=regex type=predict wrong="a*b* contains ba" -->
1. Without running anything, which of these strings are in `L(a*b*)`: ε, `aab`, `ba`, `bbb`, `abab`? Then check with `match`.

<details>
<summary>Answer</summary>

ε, `aab` and `bbb` are in it; `ba` and `abab` are not. `a*b*` is "some a's, then some b's", with either part allowed to be empty. Once a `b` has been read, no `a` can follow, so `ba` and `abab` fail. The tempting wrong answer is to read `a*b*` as "a's and b's in any order", which is `(a|b)*`. The check: `printf '\naab\nba\nbbb\nabab\n' | ./match 'a*b*'` prints yes, yes, no, yes, no.

</details>

<!-- q: idea=regular-definition type=apply wrong="digits? makes the fraction optional" -->
2. Extend the regular definition for Tiny's `number` so that an optional exponent is allowed, as in `6.02E23` or `1E-9`: the letter `E`, an optional sign, and one or more digits. Then give the expanded POSIX pattern and two strings it must refuse.

<details>
<summary>Answer</summary>

Add two lines and change `number`:

```text
exponent ->  E ( \+ | - )? digits
number   ->  digits ( \. digits )? exponent?
```

Expanded: `[0-9]+(\.[0-9]+)?(E(\+|-)?[0-9]+)?`, quoted for the shell. It must refuse `1E` (the exponent needs at least one digit) and `E5` (a number starts with a digit). The `+` sign has to be escaped, `\+`, because unescaped it would be the "one or more" shorthand applied to the `(` before it. A common slip is to put the `?` on the exponent's digits instead of on the whole exponent; that makes the digits optional and accepts `1E`.

</details>

<!-- q: idea=regex type=decide wrong="the two are equal because both contain ab" -->
3. A classmate claims that `(a|b)*` and `a*|b*` are the same. Decide, and prove your answer with one string.

<details>
<summary>Answer</summary>

They are different. `ab` is in `(a|b)*` (two trips of the star, one picking `a`, one picking `b`) but not in `a*|b*`, which is "all a's, or all b's". One string in one language and not the other is a complete proof of inequality. Both languages do contain ε, `a`, `aa`, `b` and `bb`, which is why a few quick tests can fool you: test strings that mix the symbols.

</details>

## 3.2 From a regular expression to an NFA: Thompson's construction

### A board for `(a|b)*abb`

You can now write `(a|b)*abb` and test it with `match`. But `match` hides the machine inside the C library, and a scanner is a machine: the boards of section 2.5, where each card sends you to the next square. So the question for this section is: given a regular expression, how do you get its board?

Try drawing one by thinking, the way you drew Tiny's operators. Start on square 0 and turn up the cards of `babb`. The first card is `b`. Is this `b` part of the "anything" at the front, or is it the start of... no, the ending starts with `a`. Fine, stay. The next card is `a`. Is this the `a` of the final `abb`, or just another symbol of the "anything"? You cannot know until you have seen what follows, and if the string is `abababb` you will guess wrong twice before you guess right. For this expression the right board exists, and section 3.3 finds it, but nobody finds it reliably by staring. You need a recipe that turns any expression into a board without any cleverness, so that a program can follow it, and so can you in an exam with the clock running.

The recipe is Thompson's construction. Its price is that the board it builds is of a looser kind than chapter 2's, and this section explains that kind first.

### Standing on several squares at once

The book's picture for a finite automaton is a board game: each square says, for each input card, where that card sends you. On the boards of chapter 2 every square has exactly one arrow for each card (or none, which means "no token"), so your position is always one square. Section 2.5 warned that the picture would break here, and it breaks in two ways.

First, a square may have *two* arrows for the same card. Turn up an `a` on such a square and you could go either way. Second, a square may have arrows labelled ε, which you may follow without turning up any card at all: a free move. With such a board, after any number of cards you are not on one square but on a whole set of squares, every square some sequence of choices could have brought you to. The way to keep the picture honest is to imagine a counter on every one of those squares at once. The string is accepted if, when the cards run out, at least one counter stands on a winning square.

No real game is played like that, and no real machine guesses. Section 3.3 shows how a program follows all the counters at once, and how it turns the loose board back into a strict one.

### Two kinds of automaton

<!-- teaches: nfa = NFA, nondeterministic finite automaton, DFA, epsilon edge -->
A *nondeterministic finite automaton*, an *NFA*, has:

- a finite set of states, numbered here 0, 1, 2, ...;
- an input alphabet;
- edges, each from one state to another and labelled with either one symbol or ε; a state may have any number of edges with the same label, including none;
- one start state;
- a set of accepting states.

An NFA *accepts* a string if there is at least one path from the start state to an accepting state whose labels, read in order with the ε's left out, spell the string. Other paths may get stuck or end on non-accepting squares; that does not matter. One good path is enough.

A *deterministic finite automaton*, a *DFA*, is the special case with no ε edges and at most one edge for each symbol out of each state. Every transition diagram in chapter 2 was a DFA, which is why `relop` could keep its position in one `int`. A DFA has exactly one path for any string, so running it is a loop; an NFA needs more care, but it is far easier to *build*.

### Thompson's construction: one small rule per operator

<!-- teaches: thompson-construction = Thompson's construction -->
*Thompson's construction* builds an NFA for an expression by following the expression's own structure. There is one rule for each kind of brick and one rule for each operator, and each rule takes the NFAs already built for the pieces and joins them with a few ε edges. The construction is named after Ken Thompson, who used it in the 1960s to search text with regular expressions.
<!-- risk: history: Thompson's construction attributed to Ken Thompson, 1968 paper -->

Write `N(r)` for the NFA of expression `r`. Every `N(r)` the rules build has exactly one start state and exactly one accepting state, which here is called its *final* state. In the drawings below, `i` is a new start state and `f` a new final state.

For ε, and for one symbol `a`: two new states and one edge.

```text
        ε                  a
  i ---------> f     i ---------> f
```

For `s|t`: a new start with ε edges into both pieces, and a new final with ε edges out of both. A counter placed on `i` spreads to both starts for free, so the machine is in both pieces at once; either one may reach `f`.

```text
          ε  +-------+  ε
       +---> | N(s)  | ----+
       |     +-------+     v
       i                   f
       |     +-------+     ^
       +---> | N(t)  | ----+
          ε  +-------+  ε
```

For `st`: no new states at all. The final state of `N(s)` and the start state of `N(t)` become one state. Whatever path spells a string of `s` now carries straight on into `N(t)`.

```text
  +-------+-------+
  | N(s)  * N(t)  |       * : N(s)'s final, now also N(t)'s start
  +-------+-------+
```

For `s*`: a new start and a new final, and four ε edges. From `i` you may enter `N(s)` or skip it (zero trips). From the final state of `N(s)` you may go round again to its start, or leave to `f`.

```text
                 ε (round again)
               +-------------+
               v             |
     i ---> [start  N(s)  final] ---> f
     |  ε                          ε  ^
     +--------------------------------+
                  ε (zero trips)
```

Parentheses add nothing; `N((s))` is `N(s)`.

Look at what these rules guarantee, because each guarantee is used later. No edge ever enters a start state and no edge ever leaves a final state, which is what makes the merge in `st` safe: the merged state cannot accidentally connect anything else. Each rule adds at most two states, so an expression with *n* symbols and operators gets at most *2n* states; the NFA grows in proportion to the expression, never faster. And every state has either one edge on a symbol, or at most two ε edges, or nothing (the final state), so the machine is small and regular enough to store in a few arrays.

### Building `N((a|b)*abb)` step by step

The rules are applied from the inside out, in the order the precedence dictates. First break the expression into its pieces, smallest first:

| Piece | Built from | Rule |
|---|---|---|
| r1 = `a` | | symbol |
| r2 = `b` | | symbol |
| r3 = `a|b` | r1, r2 | union |
| r4 = `(a|b)*` | r3 | star |
| r5 = `(a|b)*a` | r4, `a` | concatenation |
| r6 = `(a|b)*ab` | r5, `b` | concatenation |
| r7 = `(a|b)*abb` | r6, `b` | concatenation |

Now number the states the way the Dragon book does, so that your answer looks like the one in the paper's key: the outermost new start first, then left to right through the expression, each new final state after everything inside it.

1. The star of r4 is the outermost operator on the left, so its new start is state **0**.
2. Inside it, the union r3 gets its new start, state **1**.
3. Inside the union, `a` gets states **2** and **3** (edge 2 `a` 3), and `b` gets **4** and **5** (edge 4 `b` 5). The union's ε edges are 1 to 2 and 1 to 4.
4. The union's new final is state **6**, with ε edges 3 to 6 and 5 to 6. That finishes `N(a|b)`: start 1, final 6.
5. The star's new final is state **7**. Its four ε edges: 0 to 1 (enter), 0 to 7 (zero trips), 6 to 1 (round again), 6 to 7 (leave). That finishes `N((a|b)*)`: start 0, final 7.
6. Concatenation with `a`: the `a` would get two states, but its start merges with 7, so only one new state appears, **8**, with edge 7 `a` 8.
7. The same for `b`: new state **9**, edge 8 `b` 9. And again for the last `b`: new state **10**, edge 9 `b` 10.

State 0 is the start and state 10 the only accepting state: eleven states and thirteen edges.

<!-- fig: id=thompson-nfa
title: Thompson's NFA for (a|b)*abb
kind: flow
dir: right
round n0: 0\nstart
box n1: 1
box n2: 2
box n3: 3
box n4: 4
box n5: 5
box n6: 6
box n7: 7
box n8: 8
box n9: 9
box n10: 10\naccept
n0 -> n1
n0 -> n7
n1 -> n2
n1 -> n4
n2 -> n3: a
n4 -> n5: b
n3 -> n6
n5 -> n6
n6 ..> n1
n6 -> n7
n7 -> n8: a
n8 -> n9: b
n9 -> n10: b
hot: n10
-->
![Flow chart, top to bottom. 0 start leads to 1. 0 start leads to 7. 1 leads to 2. 1 leads to 4. 2 leads to 3 (a). 4 leads to 5 (b). 3 leads to 6. 5 leads to 6. 6 leads back to 1 by a dashed arrow. 6 leads to 7. 7 leads to 8 (a). 8 leads to 9 (b). 9 leads to 10 accept (b). 10 accept is highlighted.](fig/03-regex-automata/thompson-nfa.svg)

*Figure 3.2a: The NFA Thompson's construction builds for `(a|b)*abb`. Labelled arrows read a symbol; every unlabelled arrow is an ε edge. The highlighted state accepts; the dashed ε edge from 6 back to 1 is the star's "round again".*

Find the pieces in the figure. States 1 to 6 are the union: two parallel lanes, one for `a`, one for `b`. Wrapped around them, 0 and 7 with the edge 0 to 7 and the dashed loop 6 to 1 are the star. States 8, 9, 10 are `abb`, hanging off 7 like carriages off an engine.

### Walking it: one good path

Does this NFA accept `babb`? Look for a path that spells it. The star has to take the first `b`, so from 0 go ε to 1, ε to 4, read `b` to 5, ε to 6. Now the star is finished: go ε to 7, read `a` to 8, `b` to 9, `b` to 10. Ten is accepting, the cards are used up, so `babb` is accepted.

| Step | Move | Card | State after |
|---|---|---|---|
| 1 | 0 to 1 | ε | 1 |
| 2 | 1 to 4 | ε | 4 |
| 3 | 4 to 5 | `b` | 5 |
| 4 | 5 to 6, 6 to 7 | ε | 7 |
| 5 | 7 to 8 | `a` | 8 |
| 6 | 8 to 9 | `b` | 9 |
| 7 | 9 to 10 | `b` | 10, accepting |

The guess was at state 6 after the `b`: round again (6 to 1) or leave (6 to 7)? Leaving was right here. For `abababb` you would have to go round again five times before leaving. The path is easy to *check* once someone shows it to you; *finding* it means trying choices. For `abba` every choice fails: any path that reaches 10 must read `abb` as its last three cards, and the last three cards of `abba` are `bba`. That trying of choices is the price of the easy construction, and section 3.3 removes it by following every choice at once.

### Thompson's construction in C

The construction is mechanical, so a program can do it, and having the program means you can check every hand construction in this chapter. The program below reads one expression on standard input, over letters and digits, with `|`, `*`, parentheses and concatenation. It prints the NFA as plain lines, one edge per line, in a form section 3.3's program will read: `states`, `start` and `final` lines, then `from symbol to` for each edge, with `eps` for ε.

The structure of the program is the structure of the precedence rules. `expr` handles the loosest operator, `|`; it calls `term` for each operand. `term` handles concatenation; it calls `factor` for each piece. `factor` handles the tightest operator, `*`; it calls `atom`, which is a single symbol or a whole parenthesised expression, for which it calls `expr` again. A function that calls itself through others like this is a small *recursive-descent parser*; chapter 6 makes it the main tool for parsing Tiny, and you do not need to know more about it now than that each function handles one level of precedence.

Every function takes the state its piece starts in, which already exists, and returns the piece's final state. That one convention gives you concatenation for free: `term` starts each factor in the state where the previous one ended, which is exactly the merge of the `st` rule.

Save it as `thompson.c`:

<!-- code: id=thompson stdin="(a|b)*abb\n" -->
```c
/* thompson.c: Thompson's construction.
   Reads one regular expression: letters and digits as symbols, | * ( )
   and concatenation. Prints its NFA: "states N", "start S", "final F",
   then one line "from symbol to" per edge, with eps for an epsilon edge.
   States are numbered in the order the Dragon book numbers them. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAXE 400

char re[200];                   /* the expression */
int pos;                        /* the parser's place in re */
int nstates;                    /* states made so far: 0 .. nstates-1 */
int nedges;
int efrom[MAXE], eto[MAXE];
char esym[MAXE];                /* the symbol, or 0 for epsilon */

int expr(int s);

void fail(const char *why)
{
    fprintf(stderr, "error at character %d: %s\n", pos + 1, why);
    exit(1);
}

int newstate(void)
{
    return nstates++;
}

void edge(int from, char sym, int to)
{
    if (nedges == MAXE)
        fail("expression too long");
    efrom[nedges] = from;
    esym[nedges] = sym;
    eto[nedges] = to;
    nedges++;
}

/* one past the end of the atom that starts at p: a symbol or ( ... ) */
int atomend(int p)
{
    int depth = 0;

    if (re[p] == '\0')
        return p;
    if (re[p] != '(')
        return p + 1;
    do {
        if (re[p] == '\0')
            return p;
        if (re[p] == '(')
            depth++;
        else if (re[p] == ')')
            depth--;
        p++;
    } while (depth > 0);
    return p;
}

/* how many | stand at the top level of the expression starting at p */
int bars(int p)
{
    int depth = 0, n = 0;

    for (; re[p] != '\0'; p++) {
        if (re[p] == '(')
            depth++;
        else if (re[p] == ')') {
            if (depth == 0)
                break;              /* the end of this expression */
            depth--;
        } else if (re[p] == '|' && depth == 0)
            n++;
    }
    return n;
}

/* Each function below builds the NFA for its piece of the expression,
   starting in state s, which already exists, and returns the piece's
   final state. */

int atom(int s)                 /* a symbol, or ( expr ) */
{
    int f;

    if (re[pos] == '(') {
        pos++;
        f = expr(s);
        if (re[pos] != ')')
            fail("missing )");
        pos++;
        return f;
    }
    if (!isalnum((unsigned char)re[pos]))
        fail("symbol or ( expected");
    f = newstate();
    edge(s, re[pos], f);            /* rule for a symbol: s --a--> f */
    pos++;
    return f;
}

int factor(int s)               /* an atom, perhaps followed by * */
{
    int i, fi, f;

    if (re[atomend(pos)] != '*')
        return atom(s);
    i = newstate();                 /* rule for r*: a new start for r, */
    fi = atom(i);
    pos++;                          /* (step over the *) */
    f = newstate();                 /* and a new final state */
    edge(s, 0, i);                  /* enter r */
    edge(s, 0, f);                  /* or skip it: zero trips */
    edge(fi, 0, i);                 /* round again */
    edge(fi, 0, f);                 /* or leave */
    return f;
}

int term(int s)                 /* factors one after another */
{
    int f = s;

    if (re[pos] == '\0' || re[pos] == '|' || re[pos] == ')')
        fail("an operand is missing");
    while (re[pos] != '\0' && re[pos] != '|' && re[pos] != ')')
        f = factor(f);              /* rule for rs: the next piece starts
                                       where the last one ended */
    return f;
}

int expr(int s)                 /* terms separated by | */
{
    int k = bars(pos), start[50], j, si, f, ft, join;

    if (k == 0)
        return term(s);
    if (k > 50)
        fail("too many |");
    /* a|b|c is (a|b)|c: k unions, nested, the innermost on the left.
       Their new start states are made first, outermost first. */
    start[0] = s;
    for (j = 1; j < k; j++) {
        start[j] = newstate();
        edge(start[j - 1], 0, start[j]);
    }
    si = newstate();
    edge(start[k - 1], 0, si);
    f = term(si);
    for (j = k - 1; j >= 0; j--) {
        pos++;                      /* step over the | */
        si = newstate();
        edge(start[j], 0, si);
        ft = term(si);
        join = newstate();          /* rule for r|s: a new final state */
        edge(f, 0, join);
        edge(ft, 0, join);
        f = join;
    }
    return f;
}

int main(void)
{
    int s, f, i, j;

    if (fgets(re, sizeof re, stdin) == NULL) {
        fprintf(stderr, "no expression on input\n");
        return 1;
    }
    re[strcspn(re, "\n")] = '\0';
    s = newstate();
    f = expr(s);
    if (re[pos] != '\0')
        fail("unexpected )");
    printf("states %d\nstart %d\nfinal %d\n", nstates, s, f);
    for (i = 0; i < nstates; i++)       /* edges grouped by from state */
        for (j = 0; j < nedges; j++)
            if (efrom[j] == i) {
                if (esym[j] == 0)
                    printf("%d eps %d\n", i, eto[j]);
                else
                    printf("%d %c %d\n", i, esym[j], eto[j]);
            }
    return 0;
}
```
```text
states 11
start 0
final 10
0 eps 1
0 eps 7
1 eps 2
1 eps 4
2 a 3
3 eps 6
4 b 5
5 eps 6
6 eps 1
6 eps 7
7 a 8
8 b 9
9 b 10
```

Compare the output with Figure 3.2a, edge by edge: the same eleven states, the same thirteen edges, the same numbers. The numbers agree because the program makes its new states in the order of the hand numbering.

### Reading the program for the viva

An examiner who sees this program will ask how each rule of the construction appears in it. Here is where to point.

- **Symbol rule**: the last four lines of `atom`. One new state `f`, one edge from `s` to `f` on the symbol.
- **Concatenation rule**: the `while` loop in `term`, `f = factor(f)`. There is no code for the merge, because there is nothing to do: the next piece is simply started in the state where the last ended. That is why the NFA for `abb` adds only three states, 8, 9 and 10, not six.
- **Star rule**: `factor`. The new inner start `i` is made *before* the starred atom is built, so that it gets the lower number, as in the hand numbering (state 1 before states 2 to 6). That is why `factor` must know that a star is coming before it builds the atom; `atomend` looks ahead past the atom, skipping a whole parenthesised group if need be, to see whether a `*` follows.
- **Union rule**: `expr`. It needs to know how many `|` are coming for the same reason: the new start states come first. `bars` counts the `|` at this level, ignoring any inside parentheses. With more than one `|`, the expression `a|b|c` is treated as `(a|b)|c`, since union groups from the left, and each union gets its own new start and final, exactly as you would build it by hand.

A simpler program could make the states in any order and still produce a correct NFA. The look-ahead exists only so that its numbers match a hand construction in the Dragon style, which is what you will compare it with.

Now try it on the precedence pair from section 3.1 and on the three kinds of input every program should be tested with:

<!-- code: id=thompson-runs exit=1 -->
```console
$ gcc -Wall -Wextra thompson.c -o thompson
$ echo 'ab*' | ./thompson
states 5
start 0
final 4
0 a 1
1 eps 2
1 eps 4
2 b 3
3 eps 2
3 eps 4
$ echo '(ab)*' | ./thompson
states 5
start 0
final 4
0 eps 1
0 eps 4
1 a 2
2 b 3
3 eps 1
3 eps 4
$ ./thompson < /dev/null; echo "exit status $?"
no expression on input
exit status 1
$ echo '(a|b' | ./thompson
error at character 5: missing )
```

For `ab*`: state 0 is the start, `a` takes you to 1, and the star around `b` is built from state 1, with new states 2 (start of `b`), 3 (end of `b`) and 4 (the star's final). The skip edge `1 eps 4` is what lets `a` alone be accepted. For `(ab)*` the star is built from state 0 around the whole `ab`, and the skip edge `0 eps 4` is what makes ε accepted. Compare these two outputs with the two columns `match` printed in section 3.1: the edge that skips the star is the difference in both cases. The empty input and the broken expression are refused with a message on `stderr` and exit status 1.

### When it goes wrong

**The shortcut star.** Drawing a star as two ε edges added to `N(s)` itself, one from its start to its final ("zero trips") and one back from its final to its start ("again"), with no new states, looks like a harmless saving. It is not. Build `(a*b)*` that way. For `a*`: states 1 and 2, edge 1 `a` 2, plus 1 ε 2 and 2 ε 1. Concatenate `b`: edge 2 `b` 3. Then the outer star, by the same shortcut, adds 1 ε 3 and 3 ε 1, with start 1 and final 3. Now try the string `a`: 1, read `a` to 2, ε to 1, ε to 3. Accepted. But every string in `(a*b)*` is either empty or ends in `b`; `a` is not in it. Check that with the library:

<!-- code: id=match-astarb -->
```console
$ printf '\na\nb\naab\naba\n' | ./match '(a*b)*'
yes  ""
no   "a"
yes  "b"
yes  "aab"
no   "aba"
```

The shortcut went wrong because the "again" edge of the inner star and the "zero trips" edge of the outer one met at the same state, and together they made a path that the expression never allowed. Thompson's new `i` and `f` keep every operator's ε edges on states of its own, which is what the guarantee "no edge enters a start, none leaves a final" protects. In the exam, build stars with the two new states every time.

**Building the wrong grouping.** For `ab*` you build a star around `ab`. The NFA then accepts `abab` and refuses `abb`. Before you start, write the table of pieces (r1, r2, ...) as above; the table forces the precedence onto paper before you draw a single state.

**Different numbers from the answer key.** State numbers are names; two NFAs with the same edges between differently numbered states are the same machine. If your structure matches and only the numbers differ, renumber once in the Dragon order (outermost new start first, then left to right) so that the rest of the question, which usually continues by turning the NFA into a DFA (§3.3), can be compared line by line with the key.

### In one line each

1. An NFA may have several edges for one symbol and free ε edges; it accepts if any path spells the string and ends in an accepting state.
2. A DFA is the special case with no ε and at most one edge per symbol per state.
3. Thompson's construction has one rule per brick and per operator; each adds at most two states, so the NFA grows only in proportion to the expression.
4. Union and star add a new start and a new final; concatenation merges two states and adds none.
5. `N((a|b)*abb)` has 11 states, 0 to 10, and 13 edges; `thompson` prints it in the same numbering.

**Check yourself**

<!-- q: idea=thompson-construction type=predict wrong="concatenation adds two states per symbol" -->
1. Without running the program, how many states does Thompson's construction give for `(a|b)*` alone, and for `abc`? Then check with `thompson`.

<details>
<summary>Answer</summary>

`(a|b)*` has 8 states, 0 to 7: states 0 to 7 of Figure 3.2a, without the `abb` tail. `abc` has 4: the first symbol makes a start and a final, and each following symbol adds just one state because its start merges with the previous final. The tempting wrong answer for `abc` is 6, from building three separate two-state NFAs and forgetting that concatenation merges. `echo 'abc' | ./thompson` prints `states 4` and the edges `0 a 1`, `1 b 2`, `2 c 3`.

</details>

<!-- q: idea=nfa type=apply wrong="the path must use the 0 to 7 edge first" -->
2. In Figure 3.2a, write down a path that accepts `aabb`, as a list of states. Which edge does the path use exactly once that a path for `abb` does not use at all?

<details>
<summary>Answer</summary>

0, 1, 2, 3 (read `a`), 6, 7, 8 (read `a`), 9 (read `b`), 10 (read `b`). The star takes the first `a` and the tail takes `abb`. The path for `abb` goes straight 0, 7, 8, 9, 10 using the "zero trips" edge 0 to 7; the path for `aabb` instead uses the union lane for `a` (2 to 3) and leaves the star by 6 to 7. The "round again" edge 6 to 1 is not needed for `aabb`; it would be for `aaabb`.

</details>

<!-- q: idea=thompson-construction type=fix wrong="the union rule is the fault" -->
3. A student builds `N(a|b*)` and draws: new start 0 with ε to 1 and to 3; edge 1 `a` 2; for `b*`, edge 3 `b` 4 with ε 3 to 4 and ε 4 to 3; then ε from 2 and from 4 to a new final 5. Is the NFA wrong for this expression? What rule did the student skip, and does it matter here?

<details>
<summary>Answer</summary>

The student used the shortcut star for `b*` (no new start and final for the star). Here the NFA still happens to accept exactly `a`, ε, `b`, `bb`, ..., because the star's states are not shared with any other loop, so it gives the right language this time. But it is not Thompson's construction, and the habit breaks the moment the starred piece sits inside another star or next to one, as `(a*b)*` showed. In the paper, a "construct using Thompson's construction" question expects the two new states, and with them `N(a|b*)` has 8 states, not 6.

</details>

## 3.3 Epsilon-closure and the subset construction, by hand and in C

### Following every counter at once

Section 3.2 ended with a machine that has to guess. In Figure 3.2a, after reading a `b`, are you about to go round the star again or leave it for the `abb` tail? A person can try one choice, back up when it fails and try another, but that is slow, and in the worst case the number of paths to try grows with every card.

There is a better way, and the board-game picture already contains it. Put a counter on every square you *could* be on. When a card is turned up, move every counter at once along that card's arrows, and remove the counters that have nowhere to go. Then let every counter take all the free ε moves it can. Now the set of squares with counters on them is exactly the set of places some path could have reached. You never guess, because you follow every guess at the same time. At the end, if any counter stands on the accepting square, the string is accepted.

This section turns that idea into two operations you can do by hand, then into an algorithm that does all the moving once, in advance, and produces a DFA: a board where one counter is always enough. That algorithm is the *subset construction*, and the lab exam's "convert this NFA to a DFA" question is exactly it.

### Two operations on sets of states

<!-- teaches: epsilon-closure = epsilon-closure, move -->
The counters need two operations, and both work on a *set* of NFA states.

- **ε-closure(T)** is the set of states reachable from some state in `T` by following ε edges only, zero or more of them. Every state of `T` is in its own closure (zero ε moves). This is "let every counter take all the free moves it can".
- **move(T, a)** is the set of states reachable from some state in `T` by exactly one edge labelled `a`. This is "turn up card `a` and move every counter along an `a` arrow". Counters with no `a` arrow vanish.

Reading one symbol `a` from the set `T` takes you to ε-closure(move(T, a)): first the card, then the free moves. Starting takes you to ε-closure({start}): before any card is turned up, counters spread along the free moves from the start square.

The closure is computed with a stack, because one ε move can lead to a state that has ε moves of its own. Put every state of `T` on the stack and into the result. Then, while the stack is not empty, pop a state `s` and look at each ε edge out of it, to `u`; if `u` is not yet in the result, add it and push it. The stack remembers which states still have edges to look at; the "not yet in the result" test makes sure no state is pushed twice, so the loop ends even when ε edges form a cycle, as 6 to 1 and back through the star does.

Compute ε-closure({0}) in Figure 3.2a, step by step. The ε edges you need are: 0 to 1 and 7; 1 to 2 and 4; 3 to 6; 5 to 6; 6 to 1 and 7.

| Step | Pop | ε edges out | Added and pushed | Result after | Stack after |
|---|---|---|---|---|---|
| start | | | 0 | {0} | 0 |
| 1 | 0 | to 1, to 7 | 1, 7 | {0, 1, 7} | 1 7 |
| 2 | 7 | none | | {0, 1, 7} | 1 |
| 3 | 1 | to 2, to 4 | 2, 4 | {0, 1, 2, 4, 7} | 2 4 |
| 4 | 4 | none | | {0, 1, 2, 4, 7} | 2 |
| 5 | 2 | none | | {0, 1, 2, 4, 7} | (empty) |

So ε-closure({0}) = {0, 1, 2, 4, 7}. Read that as "before any input, the machine could be at the start, inside the star at either lane's entrance, or already past the star waiting for the `a` of `abb`." Notice that 3, 5 and 6 are missing: those are reached only after reading something.

### Running the NFA on `babb` with sets

Now run the whole string, keeping the set of squares with counters at each step. For each card: move, then close.

| After | move(T, card) | ε-closure of that: the new T | Accepting state 10 in T? |
|---|---|---|---|
| start | | {0, 1, 2, 4, 7} | no |
| `b` | from 4 to 5: {5} | {1, 2, 4, 5, 6, 7} | no |
| `a` | from 2 to 3, from 7 to 8: {3, 8} | {1, 2, 3, 4, 6, 7, 8} | no |
| `b` | from 4 to 5, from 8 to 9: {5, 9} | {1, 2, 4, 5, 6, 7, 9} | no |
| `b` | from 4 to 5, from 9 to 10: {5, 10} | {1, 2, 4, 5, 6, 7, 10} | yes |

The string is accepted, with no guessing. Look at the row after the first `b`: the closure of {5} goes 5 to 6, then 6 to both 1 and 7, then 1 to 2 and 4. The counter that read the `b` has spread back to both lanes of the star (round again) and on to state 7 (leave). Both choices that you had to guess between in section 3.2 are now held at once.

Look at the third row: `move` found two counters that could read the `a`, one in the star's `a` lane (2 to 3) and one at the start of the tail (7 to 8). The set keeps both possibilities open: "this `a` was just more of the anything" and "this `a` starts `abb`". The fourth and fifth rows show the tail's counter moving on through 9 and 10 while the star's counters keep cycling.

This is a complete way to run an NFA, and it is how `match` and many regular-expression libraries work. But it repeats the same set arithmetic every time a set comes back. And the sets do come back: there are only finitely many sets of 11 states, and if you ran a long string you would see the same four or five sets over and over.

### Doing the work once: the subset construction

<!-- teaches: subset-construction = subset construction, Dstates, Dtran, unmarked state -->
If the same sets keep coming back, compute each one once and give it a name. Each such set becomes one state of a new machine; reading a symbol from that set always gives the same next set, so the new machine has exactly one edge per symbol from each state. It is a DFA. That is the *subset construction*: every DFA state stands for a subset of the NFA's states.

The Dragon book's algorithm keeps two tables. `Dstates` is the list of DFA states found so far, each a set of NFA states, and each either *marked* (its edges have been worked out) or *unmarked*. `Dtran` is the DFA's table of edges: one row per DFA state, one column per symbol.

1. The first DFA state is ε-closure({start}). Put it in `Dstates`, unmarked.
2. While there is an unmarked state `T` in `Dstates`: mark it; for each input symbol `a`, compute `U` = ε-closure(move(T, a)). If `U` is not already in `Dstates`, add it, unmarked. Set `Dtran[T, a] = U`.
3. A DFA state is accepting if its set contains an accepting NFA state.

If `U` comes out empty, no counter survives that symbol: the DFA has no edge there, and reaching that point means the string is rejected. (Some books add an explicit "dead" state for the empty set; section 3.5 shows when you need it.)

Work it for Figure 3.2a, with the symbols in the order `a`, `b`. Each row marks one state and may find new ones.

| Mark | Its set | On `a`: move, then closure | On `b`: move, then closure |
|---|---|---|---|
| A | {0, 1, 2, 4, 7} | {3, 8} gives {1, 2, 3, 4, 6, 7, 8}: new, call it **B** | {5} gives {1, 2, 4, 5, 6, 7}: new, **C** |
| B | {1, 2, 3, 4, 6, 7, 8} | {3, 8}: B again | {5, 9} gives {1, 2, 4, 5, 6, 7, 9}: new, **D** |
| C | {1, 2, 4, 5, 6, 7} | {3, 8}: B | {5}: C |
| D | {1, 2, 4, 5, 6, 7, 9} | {3, 8}: B | {5, 10} gives {1, 2, 4, 5, 6, 7, 10}: new, **E** |
| E | {1, 2, 4, 5, 6, 7, 10} | {3, 8}: B | {5}: C |

After E is marked there are no unmarked states left, so the construction stops. Only E contains state 10, so only E accepts. The DFA:

| State | `a` | `b` |
|---|---|---|
| → A | B | C |
| B | B | D |
| C | B | C |
| D | B | E |
| E * | B | C |

(→ marks the start state, `*` an accepting state, as in the Dragon book's tables.)

Every row has exactly one entry per symbol, so this is a board for a single counter. Run `babb`: A, `b` to C, `a` to B, `b` to D, `b` to E. Accepting. These are the same sets as in the table for `babb` above: C is {1, 2, 4, 5, 6, 7}, the set after the first `b`; B, D and E are the next three. The subset construction did the set arithmetic of every possible run, once.

Read what the states mean, because it makes the DFA easy to check. B means "the input so far ends in `a`", D "ends in `ab`", E "ends in `abb`" (accept). A and C both mean "no useful ending yet". Every `a` sends you to B, from anywhere, because an `a` is always a possible start of `abb`. A `b` moves you one step along `a`, `ab`, `abb` if you are on the way, and otherwise back to C. A and C behave identically, a hint that this DFA has one state too many; section 3.5 removes it.

Why must the construction stop? Each DFA state is a different subset of the NFA's states, and an NFA with *n* states has only 2^*n* subsets. In practice far fewer appear: here 5 out of 2048. But the bound is real; there are expressions whose DFA does need exponentially many states, and you will see one in the experiment below.

### The subset construction in C

The program reads an NFA in exactly the form `thompson` prints, so the two can be joined with a pipe. The one design decision is how to store a set of NFA states. The program uses the bits of an `unsigned long long`: bit `s` is 1 when state `s` is in the set. With that choice, adding a state is `t |= 1ULL << s`, testing is `t & (1ULL << s)`, and the question "is this set already in `Dstates`?" is one `==` comparison. The price is a limit of 64 NFA states, which is plenty for any expression in an exam paper.

`epsto[s]` holds the set of states one ε edge away from `s`, and `symto[s][c]` the set one `c` edge away, so `move` is a loop that ORs together the rows of the states in the set. `closure` is the stack algorithm, line for line. `main` is the Dragon algorithm: `dstate[]` is `Dstates`, and since new states are always added at the end, "the next unmarked state" is just the next index `i`.

Save it as `subset.c`:

<!-- code: id=subset mode=compile -->
```c
/* subset.c: the subset construction.
   Reads an NFA in the form thompson prints. Prints the DFA: the set of
   NFA states in each DFA state, then "start", "final" and one line
   "from symbol to" for each edge. DFA states are named A, B, C, ... */
#include <stdio.h>
#include <string.h>

#define MAXN 64                 /* NFA states: one bit each in a set */
#define MAXD 26                 /* DFA states: named A to Z */

typedef unsigned long long set; /* bit s is 1 when state s is in */

int nn = -1, nstart = -1, nfinal = -1;
set epsto[MAXN];                /* epsto[s]: states one eps edge from s */
set symto[MAXN][256];           /* symto[s][c]: states one c edge from s */
int used[256];                  /* the symbols that appear */
set dstate[MAXD];               /* Dstates */
int dtran[MAXD][256];           /* Dtran: a DFA state, or -1 for none */
int nd;                         /* DFA states found so far */

set bit(int s)
{
    return 1ULL << s;
}

/* every state reachable from t by eps edges alone, t included */
set closure(set t)
{
    int stack[MAXN], top = 0, s, u;

    for (s = 0; s < nn; s++)
        if (t & bit(s))
            stack[top++] = s;
    while (top > 0) {
        s = stack[--top];
        for (u = 0; u < nn; u++)
            if ((epsto[s] & bit(u)) && !(t & bit(u))) {
                t |= bit(u);
                stack[top++] = u;
            }
    }
    return t;
}

/* every state one c edge away from a state in t */
set move(set t, int c)
{
    set r = 0;
    int s;

    for (s = 0; s < nn; s++)
        if (t & bit(s))
            r |= symto[s][c];
    return r;
}

int readnfa(void)
{
    char line[100], sym[16];
    int a, b;

    while (fgets(line, sizeof line, stdin) != NULL) {
        if (sscanf(line, "states %d", &nn) == 1) {
            if (nn < 1 || nn > MAXN) {
                fprintf(stderr, "subset: 1 to %d NFA states only\n", MAXN);
                return 0;
            }
        } else if (sscanf(line, "start %d", &nstart) == 1
                   || sscanf(line, "final %d", &nfinal) == 1) {
            ;
        } else if (sscanf(line, "%d %15s %d", &a, sym, &b) == 3
                   && a >= 0 && a < nn && b >= 0 && b < nn) {
            if (strcmp(sym, "eps") == 0)
                epsto[a] |= bit(b);
            else {
                symto[a][(unsigned char)sym[0]] |= bit(b);
                used[(unsigned char)sym[0]] = 1;
            }
        } else {
            fprintf(stderr, "subset: bad line: %s", line);
            return 0;
        }
    }
    if (nn < 0 || nstart < 0 || nstart >= nn || nfinal < 0 || nfinal >= nn) {
        fprintf(stderr, "subset: no complete NFA on input\n");
        return 0;
    }
    return 1;
}

int main(void)
{
    int i, j, c, s;
    set u;
    const char *sep;

    if (!readnfa())
        return 1;
    dstate[0] = closure(bit(nstart));       /* A = eps-closure({start}) */
    nd = 1;
    for (i = 0; i < nd; i++)                /* i: the next unmarked state */
        for (c = 0; c < 256; c++) {
            dtran[i][c] = -1;
            if (!used[c])
                continue;
            u = closure(move(dstate[i], c));
            if (u == 0)
                continue;                   /* no counter survives c */
            for (j = 0; j < nd && dstate[j] != u; j++)
                ;
            if (j == nd) {                  /* a new DFA state, unmarked */
                if (nd == MAXD) {
                    fprintf(stderr, "subset: more than %d DFA states\n", MAXD);
                    return 1;
                }
                dstate[nd++] = u;
            }
            dtran[i][c] = j;
        }

    for (i = 0; i < nd; i++) {
        printf("%c = {", 'A' + i);
        sep = "";
        for (s = 0; s < nn; s++)
            if (dstate[i] & bit(s)) {
                printf("%s%d", sep, s);
                sep = " ";
            }
        printf("}\n");
    }
    printf("start A\nfinal");
    for (i = 0; i < nd; i++)
        if (dstate[i] & bit(nfinal))
            printf(" %c", 'A' + i);
    printf("\n");
    for (i = 0; i < nd; i++)
        for (c = 0; c < 256; c++)
            if (dtran[i][c] >= 0)
                printf("%c %c %c\n", 'A' + i, c, 'A' + dtran[i][c]);
    return 0;
}
```

Build it and feed it the NFA for `(a|b)*abb` straight from `thompson`, then test the empty input and an input that is not an NFA:

<!-- code: id=subset-runs exit=1 -->
```console
$ gcc -Wall -Wextra subset.c -o subset
$ echo '(a|b)*abb' | ./thompson | ./subset
A = {0 1 2 4 7}
B = {1 2 3 4 6 7 8}
C = {1 2 4 5 6 7}
D = {1 2 4 5 6 7 9}
E = {1 2 4 5 6 7 10}
start A
final E
A a B
A b C
B a B
B b D
C a B
C b C
D a B
D b E
E a B
E b C
$ ./subset < /dev/null; echo "exit status $?"
subset: no complete NFA on input
exit status 1
$ echo 'hello' | ./subset
subset: bad line: hello
```

The five sets are the five rows of the hand table, with the same letters, because the program marks states in the same order you did (first found, first marked) and tries the symbols in character order, `a` before `b`. The transition lines are the DFA table read row by row. When your hand answer and this output differ, one of the sets differs, and the first differing set is where your mistake is.

### An experiment: when the DFA explodes

The subset construction can need many more DFA states than the NFA has. The classic case is "the *n*-th symbol from the end is `a`": `(a|b)*a(a|b)` for the second from the end, `(a|b)*a(a|b)(a|b)` for the third, and so on. To know whether the third symbol from the end was an `a`, a machine reading left to right has to remember the last three symbols at every moment, and there are 2 × 2 × 2 = 8 possible lasts-three. Count the DFA states with `grep -c`, which counts the lines that contain ` = `, one per DFA state:

<!-- code: id=subset-explode -->
```console
$ echo '(a|b)*a(a|b)' | ./thompson | grep states
states 14
$ echo '(a|b)*a(a|b)' | ./thompson | ./subset | grep -c ' = '
5
$ echo '(a|b)*a(a|b)(a|b)' | ./thompson | grep states
states 19
$ echo '(a|b)*a(a|b)(a|b)' | ./thompson | ./subset | grep -c ' = '
9
$ echo '(a|b)*a(a|b)(a|b)(a|b)' | ./thompson | grep states
states 24
$ echo '(a|b)*a(a|b)(a|b)(a|b)' | ./thompson | ./subset | grep -c ' = '
17
```

Each extra `(a|b)` adds five NFA states, a fixed amount (14, 19, 24), while the number of DFA states roughly doubles (5, 9, 17). That is the trade-off between the two machines in one table: an NFA is small to build and slow to run (a set of states per character); a DFA is fast to run (one table lookup per character) and can be large to build. Scanner generators such as Flex build the DFA, because a scanner reads every character of every program, and the patterns of real tokens almost never explode like this.

### When it goes wrong

**A closure that stops after one step.** If you write down only the states one ε edge away, A comes out as {0, 1, 7}, missing 2 and 4, which are two ε edges away. Then move(A, `b`) is empty, and your DFA rejects every string that starts with `b`, including `babb`. The cure is the stack: a state you add must itself be examined, so keep going until nothing new appears.

**Forgetting the closure after the move.** If you take move(A, `a`) = {3, 8} as the next state without closing it, the set has no counters at the lanes' entrances 2 and 4, so a second `a` has nowhere to go and `aabb` is rejected. Every new DFA state is a closure: "move, *then* close", every time.

**Forgetting the closure of the start.** Starting from {0} instead of ε-closure({0}) gives a start state with no symbol edges at all, since state 0 has only ε edges. Every non-empty string is rejected. Always begin with the closure.

**Losing track of which states are marked.** By hand, it is easy to work out the edges of a state twice or to forget one. Work strictly in the order the states were found, one row per state, as in the table above, and stop only when every state you have named has its row.

**`subset: more than 26 DFA states`.** The program names states with single letters. If an exam-size expression hits this limit, the expression is unusual (see the experiment) or it is not what you meant to type; print the NFA with `thompson` alone and check it first.

### In one line each

1. ε-closure(T) is every state reachable from T by ε edges alone; it is computed with a stack and includes T.
2. move(T, a) is every state one `a` edge from T; reading `a` takes you to ε-closure(move(T, a)).
3. Running an NFA with sets of states follows every path at once; no guessing.
4. The subset construction computes every set that can appear, once; each set is a DFA state, and a set containing the NFA's accepting state accepts.
5. For `(a|b)*abb` it gives five states, A to E, with E accepting; `thompson | subset` prints the same table.
6. A DFA can need up to 2^n states for an n-state NFA; real token patterns rarely come close.

**Check yourself**

<!-- q: idea=epsilon-closure type=apply wrong="closure of {3,8} is {3,6,8}" -->
1. In Figure 3.2a, compute ε-closure({3, 8}) by hand, with the stack. Which state is added last?

<details>
<summary>Answer</summary>

Start with result {3, 8}, stack 3 8. Pop 8: no ε edges. Pop 3: ε to 6, add and push. Pop 6: ε to 1 and 7, add both, push both. Pop 7: none. Pop 1: ε to 2 and 4, add and push. Pop 4: none. Pop 2: none. Result {1, 2, 3, 4, 6, 7, 8}, which is B. With this pop order, 2 and 4 are added last (in the same step). The tempting wrong answer {3, 6, 8} stops after one ε step; the closure continues through 6 to 1 and 7, and through 1 to 2 and 4.

</details>

<!-- q: idea=subset-construction type=predict wrong="the DFA for (a|b)*a has three states" -->
2. Predict how many DFA states the subset construction gives for `(a|b)*a`, and which of them accept. Then check with `echo '(a|b)*a' | ./thompson | ./subset`.

<details>
<summary>Answer</summary>

Three. The NFA is states 0 to 8 (8 being the end of the final `a`). A = ε-closure({0}) = {0, 1, 2, 4, 7}. On `a`: move gives {3, 8}, closed to {1, 2, 3, 4, 6, 7, 8}, call it B; it contains 8, so it accepts. On `b`: {5} closed to {1, 2, 4, 5, 6, 7}, C. From B and from C, `a` gives B again and `b` gives C again. So A, B and C, with only B accepting. As with `(a|b)*abb`, A and C behave the same; a two-state DFA (last symbol `a` or not) would do, and section 3.5 finds it.

</details>

<!-- q: idea=epsilon-closure type=predict wrong="the visited test is only an optimisation" -->
3. A student's `closure` function has the test `if (epsto[s] & bit(u))` instead of `if ((epsto[s] & bit(u)) && !(t & bit(u)))`. Predict what `subset` does on the NFA for `(a|b)*abb`, and on the NFA for `(a*)*`. Print the second NFA with `thompson` first.

<details>
<summary>Answer</summary>

On `(a|b)*abb` it still gives the right DFA. Its ε edges form no cycle: the way back from 6 to 1 leads only to 2 and 4, which have no ε edges, and going round the star again needs a symbol edge. So without the test a state can at most be pushed twice (6 is reached from both 3 and 5), which is wasted work but finishes. On `(a*)*` it never finishes. `thompson` prints, among others, `1 eps 4` and `4 eps 1`: the inner star's "zero trips" edge and the outer star's "round again" edge make an ε cycle. Without the test, popping 1 pushes 4, popping 4 pushes 1, and so on for ever; the 64-entry stack overflows, and the sanitizer or a crash reports it. The "not already in the result" half of the test is what guarantees each state is pushed at most once, which both ends the loop and bounds the stack. The tempting wrong belief is that the test is only an optimisation; on an ε cycle it is what makes the loop end.

</details>

## 3.4 A DFA straight from a regular expression: nullable, firstpos, lastpos, followpos

### Skipping the NFA

Thompson's construction followed by the subset construction always works, but look at what it cost for `(a|b)*abb`: an 11-state NFA, thirteen edges, five closures computed by hand, and at the end a DFA with one state too many. In the paper, that is a lot of writing to get wrong, and many papers ask for something else: "construct the DFA directly from the regular expression `(a|b)*abb` without building an NFA", sometimes adding "using the followpos function". This section is that method. It works on the expression's syntax tree, computes four small functions, and builds the DFA states as sets of *positions* in the expression, not sets of NFA states. For `(a|b)*abb` it gives four states straight away.

### Positions: the symbols you might read next

<!-- teaches: direct-dfa = position, augmented regular expression, end marker # -->
Give every symbol occurrence in the expression its own number, left to right. These numbers are called *positions*. In `(a|b)*abb` there are five symbol occurrences, so five positions: `a` is 1, `b` is 2, the next `a` is 3, then `b` 4 and `b` 5. The two `a`'s are different positions even though they are the same symbol; that difference is the whole trick.

The method needs one more thing: a way to see that the input may stop. Add a special end marker `#` to the end of the expression, with its own position. The *augmented regular expression* is `(a|b)*abb#`, and `#` is position 6. Reading "the next symbol is `#`" means "the string could end here and be in the language".

Now the idea in board-game terms. A square of the new DFA is a set of positions: the positions whose symbol could be the *next* card. At the start, which symbols of `(a|b)*abb#` could be the first one read? Position 1 (an `a` from the star), position 2 (a `b` from the star), or position 3 (the `a` of `abb`, if the star takes nothing). So the start square is {1, 2, 3}. If the card turned up is `a`, it matched position 1 or position 3, and the new square is every position that can come right after 1 or right after 3. The function that says what can come right after a position is called `followpos`, and once you have it, building the DFA is a loop.

The other three functions exist only to compute `followpos`, and they are computed on the syntax tree.

### The syntax tree and three functions on it

The syntax tree of an expression has a leaf for each position (and for ε, if the expression has one) and an inner node for each operator: `|` and `•` (concatenation) with two children, `*` with one. Precedence decides the shape. For `(a|b)*abb#` the concatenations group from the left, so the root is the last `•`, with `(a|b)*abb` on its left and `#` on its right; and so on down to the star of the union of positions 1 and 2.

<!-- teaches: followpos = nullable, firstpos, lastpos, followpos -->
For every node *n* of the tree, three facts about the strings that the subexpression at *n* can produce:

- **nullable(n)** is true if the subexpression can produce the empty string.
- **firstpos(n)** is the set of positions that can be the first symbol of a string it produces.
- **lastpos(n)** is the set of positions that can be the last symbol of a string it produces.

They are computed bottom-up, from the leaves to the root, by these rules:

| Node *n* | nullable(n) | firstpos(n) | lastpos(n) |
|---|---|---|---|
| leaf ε | true | ∅ | ∅ |
| leaf with position *i* | false | {*i*} | {*i*} |
| `c1 | c2` | nullable(c1) or nullable(c2) | firstpos(c1) ∪ firstpos(c2) | lastpos(c1) ∪ lastpos(c2) |
| `c1 • c2` | nullable(c1) and nullable(c2) | if nullable(c1): firstpos(c1) ∪ firstpos(c2), else firstpos(c1) | if nullable(c2): lastpos(c1) ∪ lastpos(c2), else lastpos(c2) |
| `c1*` | true | firstpos(c1) | lastpos(c1) |

The concatenation row is the only one that needs thought, so say it in words. A string of `c1 • c2` starts with a string of `c1`, so its first symbol comes from `firstpos(c1)`; but if `c1` can be empty, the first symbol might instead come from `c2`. Symmetrically for the last symbol, with the roles swapped. The star row says a starred expression can always be empty (zero trips) and otherwise starts and ends the way its body does.

Compute them for `(a|b)*abb#`, bottom-up. The leaves are easy: position *i* has firstpos and lastpos {*i*} and is not nullable.

| Node | nullable | firstpos | lastpos | Why |
|---|---|---|---|---|
| `1 | 2` | false | {1, 2} | {1, 2} | union: both unions |
| `(1|2)*` | **true** | {1, 2} | {1, 2} | star: always nullable |
| `• (star, 3)` | false | {1, 2, 3} | {3} | left child nullable, so firstpos adds 3; right not nullable, so lastpos is just {3} |
| `• (…, 4)` | false | {1, 2, 3} | {4} | left not nullable: firstpos of left only |
| `• (…, 5)` | false | {1, 2, 3} | {5} | the same |
| `• (…, 6)` root | false | {1, 2, 3} | {6} | the same |

Row three is where the start square {1, 2, 3} comes from: because the star is nullable, the `a` at position 3 can be the first symbol.

<!-- fig: id=abb-tree
title: (a|b)*abb# with firstpos, lastpos
kind: flow
dir: down
box c4: •\n{1,2,3}\n{6}
box c3: •\n{1,2,3}\n{5}
box c2: •\n{1,2,3}\n{4}
box c1: •\n{1,2,3}\n{3}
box st: *\n{1,2}\n{1,2}
box un: |\n{1,2}\n{1,2}
box p1: a 1
box p2: b 2
box p3: a 3
box p4: b 4
box p5: b 5
box p6: `#` 6
c4 -> c3
c4 -> p6
c3 -> c2
c3 -> p5
c2 -> c1
c2 -> p4
c1 -> st
c1 -> p3
st -> un
un -> p1
un -> p2
hot: st
-->
![Flow chart, top to bottom. • {1,2,3} {6} leads to • {1,2,3} {5}. • {1,2,3} {6} leads to # 6. • {1,2,3} {5} leads to • {1,2,3} {4}. • {1,2,3} {5} leads to b 5. • {1,2,3} {4} leads to • {1,2,3} {3}. • {1,2,3} {4} leads to b 4. • {1,2,3} {3} leads to * {1,2} {1,2}. • {1,2,3} {3} leads to a 3. * {1,2} {1,2} leads to | {1,2} {1,2}. | {1,2} {1,2} leads to a 1. | {1,2} {1,2} leads to b 2. * {1,2} {1,2} is highlighted.](fig/03-regex-automata/abb-tree.svg)

*Figure 3.4a: The syntax tree of `(a|b)*abb#`. Each inner node shows firstpos as its upper set and lastpos as its lower set; a leaf with position i has firstpos and lastpos {i}. The highlighted star is the only nullable node.*

### followpos: what may come next

`followpos(i)` is the set of positions that can come immediately after position *i* in some string of the language. Only two kinds of node make one position follow another, and they give two rules:

1. **Concatenation** `c1 • c2`: every position that can end a string of `c1` can be followed by every position that can start a string of `c2`. For each *i* in lastpos(c1), add firstpos(c2) to followpos(*i*).
2. **Star** `c1*`: after one trip ends, the next trip can start. For each *i* in lastpos(n), add firstpos(n) to followpos(*i*).

A union makes nothing follow anything (you take one side or the other), and leaves make nothing follow anything. Go through the tree, any node order you like, and apply the two rules at every `•` and `*` node:

| Node | Rule | lastpos (from) | firstpos (added) | Effect |
|---|---|---|---|---|
| `(1|2)*` | star | {1, 2} | {1, 2} | followpos(1) and followpos(2) get {1, 2} |
| `• (star, 3)` | concatenation | {1, 2} of the star | {3} | followpos(1) and followpos(2) get 3 |
| `• (…, 4)` | concatenation | {3} | {4} | followpos(3) gets 4 |
| `• (…, 5)` | concatenation | {4} | {5} | followpos(4) gets 5 |
| `• (…, 6)` | concatenation | {5} | {6} | followpos(5) gets 6 |

Collect the result by position:

| Position | Symbol | followpos |
|---|---|---|
| 1 | `a` | {1, 2, 3} |
| 2 | `b` | {1, 2, 3} |
| 3 | `a` | {4} |
| 4 | `b` | {5} |
| 5 | `b` | {6} |
| 6 | `#` | ∅ |

Read it aloud: after an `a` or `b` of the star you may read another symbol of the star, or the `a` that starts `abb`; after that `a` comes the first `b`, then the second `b`, then the end. That is the expression, said in a different way.

### Building the DFA from followpos

Now the loop, which has the same shape as the subset construction, with positions in place of NFA states:

1. The start state is firstpos(root).
2. For each unmarked state *S* and each symbol `a`: let *U* be the union of followpos(*p*) for every position *p* in *S* whose symbol is `a`. If *U* is new, add it unmarked. The edge on `a` from *S* goes to *U*. (An empty *U* means no edge.)
3. A state is accepting if it contains the position of `#`.

For `(a|b)*abb#`, with symbol `a` at positions 1 and 3 and `b` at 2, 4 and 5:

| Mark | Set | On `a`: positions, then union of followpos | On `b`: positions, then union of followpos |
|---|---|---|---|
| A | {1, 2, 3} | 1, 3: {1, 2, 3} ∪ {4} = {1, 2, 3, 4}, **B** | 2: {1, 2, 3}, A |
| B | {1, 2, 3, 4} | 1, 3: {1, 2, 3, 4}, B | 2, 4: {1, 2, 3} ∪ {5} = {1, 2, 3, 5}, **C** |
| C | {1, 2, 3, 5} | 1, 3: B | 2, 5: {1, 2, 3} ∪ {6} = {1, 2, 3, 6}, **D** |
| D | {1, 2, 3, 6} | 1, 3: B | 2: A |

Only D contains position 6, so only D accepts.

| State | `a` | `b` |
|---|---|---|
| → A | B | A |
| B | B | C |
| C | B | D |
| D * | B | A |

Four states. Compare it with the subset construction's table in section 3.3: there, A and C both meant "no useful ending yet" and behaved identically; here they are one state, A. B, C and D are the subset table's B, D and E: "ends in `a`", "ends in `ab`", "ends in `abb`". The direct method merged A and C because both are the set {1, 2, 3} of positions: the same "next card could be any of these" situation, reached by different routes. It does not always produce the smallest DFA (section 3.5 shows how to make sure), but it often does, and it never needs the NFA.

Run `babb` through it: A, `b` back to A, `a` to B, `b` to C, `b` to D. Accept.

### Change one thing: `(a|b)*a`

Walk a second expression to see the method on its own. Augmented: `(a|b)*a#`, positions `a` 1, `b` 2, `a` 3, `#` 4.

- The star of `1 | 2` is nullable with firstpos and lastpos {1, 2}; the concatenation with 3 has firstpos {1, 2, 3} and lastpos {3}; the root, concatenated with 4, has firstpos {1, 2, 3}.
- followpos: the star gives 1 and 2 the set {1, 2}; the first concatenation gives them 3; the root gives followpos(3) = {4}. So followpos(1) = followpos(2) = {1, 2, 3}, followpos(3) = {4}.
- Start A = {1, 2, 3}. On `a` (positions 1, 3): {1, 2, 3, 4} = B, accepting. On `b` (position 2): {1, 2, 3} = A. From B, `a` gives B and `b` gives A.

Two states: "last symbol was `a`" (B, accept) and "was not" (A). Section 3.3's second check question got three states for this expression from the subset construction; the direct method gets the smallest answer at once.

### When it goes wrong

**Forgetting the `#`.** Without the end marker no state can be recognised as accepting: you would have to decide by hand which sets "end the expression", and it is easy to pick the wrong ones. Augment first, every time; it costs one leaf.

**Forgetting that a star is nullable.** If you treat `(1|2)*` as not nullable, the concatenation with 3 gets firstpos {1, 2} instead of {1, 2, 3}, the start state loses position 3, and the string `abb`, where the star takes nothing, can no longer be read: from A = {1, 2} the first `a` can only be an `a` of the star, which takes you to followpos(1) = {1, 2, 3}, one symbol behind. The DFA rejects `abb`. Every star node is nullable; mark it on the tree before anything else, as Figure 3.4a does.

**Using firstpos where lastpos belongs.** The concatenation rule for followpos goes from *lastpos of the left child* to *firstpos of the right child*. Swapped, it makes positions follow positions that come before them. The quick test: followpos(*i*) of the last real position must contain the `#` position and nothing else. If it does not, recheck the rule.

**Merging two positions because they hold the same symbol.** The two `a`'s of `(a|b)*abb` are positions 1 and 3, with different followpos. If you give them one number, the method cannot tell "an `a` from the star" from "the `a` that starts `abb`", and the result is wrong. Number every occurrence.

### In one line each

1. Augment the expression with `#` and number every symbol occurrence: these are the positions.
2. nullable, firstpos and lastpos are computed bottom-up on the syntax tree; the concatenation rule depends on whether a child is nullable, and every star is nullable.
3. followpos comes from two rules: at `c1 • c2`, lastpos(c1) is followed by firstpos(c2); at `n = c1*`, lastpos(n) is followed by firstpos(n).
4. The DFA's start is firstpos(root); from a set on symbol `a`, go to the union of followpos of its `a` positions; states containing `#`'s position accept.
5. For `(a|b)*abb` this gives four states directly, one fewer than Thompson plus subset.

**Check yourself**

<!-- q: idea=followpos type=apply wrong="followpos(2) includes 3" -->
1. For `(ab|b)*a`, augmented with `#`, number the positions and compute followpos for each.

<details>
<summary>Answer</summary>

Positions: `a` 1, `b` 2, `b` 3, `a` 4, `#` 5. The concatenation `ab` (positions 1 and 2) gives followpos(1) = {2}. The union `ab | b` has firstpos {1, 3} and lastpos {2, 3}. The star gives each of 2 and 3 the set {1, 3}. The concatenation with position 4 gives 2 and 3 also {4}. The root gives followpos(4) = {5}. So: followpos(1) = {2}, followpos(2) = {1, 3, 4}, followpos(3) = {1, 3, 4}, followpos(4) = {5}, followpos(5) = ∅. A common slip is followpos(1) = {2, 3}: position 3 is the other branch of the union, and inside one trip of the star you take one branch only, so nothing follows 1 except the `b` of `ab`.

</details>

<!-- q: idea=direct-dfa type=apply wrong="the start state is {1}" -->
2. Continue: build the DFA for `(ab|b)*a` from your followpos table. Which states accept? Does it accept `aa`?

<details>
<summary>Answer</summary>

Start A = firstpos(root) = {1, 3, 4} (the star is nullable, so position 4 can be first). On `a` (positions 1 and 4): {2} ∪ {5} = {2, 5}, call it B; it contains 5, so it accepts. On `b` (position 3): {1, 3, 4} = A. From B, on `a`: B has no `a` position (2 is `b`, 5 is `#`), so there is no edge. On `b` (position 2): {1, 3, 4} = A. So A on `a` to B, on `b` to A; B on `b` to A; B accepts. `aa` is rejected: A, `a` to B, then no `a` edge from B. Correct, because inside the star every `a` must be followed by `b`, and the final `a` ends the string.

</details>

<!-- q: idea=followpos type=fix wrong="the fault is in the star rule" -->
3. A student computes, for `(a|b)*abb#`, firstpos of the node `• (star, 3)` as {1, 2}. Predict which of `abb`, `babb` and `aabb` the student's DFA accepts, and name the rule they broke.

<details>
<summary>Answer</summary>

The student used "firstpos of the left child" without checking that the left child, the star, is nullable. Their start state is {1, 2}. On `a` from {1, 2}: position 1 only, so followpos(1) = {1, 2, 3}, which is the correct start state. From there on, the DFA runs one symbol "late": it can only accept strings that have at least one symbol before the final `abb`. So `abb` is rejected, while `babb` and `aabb` are accepted. The broken rule is the concatenation row: if nullable(c1), firstpos(c1 • c2) is firstpos(c1) ∪ firstpos(c2).

</details>

## 3.5 Minimising a DFA by splitting groups

### Five states or four?

You now have two DFAs for `(a|b)*abb`. The subset construction gave five states, A to E. The direct method gave four. Both accept exactly the strings ending in `abb`, and you can check that by running a few strings through each. So which is "the" DFA, and how would you know, in an exam, whether your answer has states to spare?

The paper asks this directly: "minimise the following DFA", or "construct the minimum-state DFA for (a|b)*abb". The lab asks it indirectly, because every state is a row of the table your C recogniser carries in section 3.6. And there is a precise answer. For every regular language there is one DFA with the fewest possible states, the same for everyone up to the names of the states, and there is an algorithm that finds it from any DFA for the language. This section is that algorithm, done by hand and by a program.
<!-- risk: uniqueness of the minimal DFA (up to state names) for a regular language; standard theorem -->

### When two squares are the same square

Back to the board game. Two squares are interchangeable if, whatever cards are still to come, starting from either square gives the same result: win from both or lose from both. If that holds, nobody watching the game could tell which of the two squares you were standing on, so the board can glue them into one.

<!-- teaches: dfa-minimisation = equivalent states, distinguishing string, partition, dead state -->
Put precisely: two states *s* and *t* of a DFA are *equivalent* if for every input string *w*, running *w* from *s* ends in an accepting state exactly when running it from *t* does. If some string *w* ends in an accepting state from one and not from the other, *w* is a *distinguishing string* for them, and they must stay apart.

Look at the subset DFA of section 3.3 with this in mind:

| State | `a` | `b` |
|---|---|---|
| → A | B | C |
| B | B | D |
| C | B | C |
| D | B | E |
| E * | B | C |

E and A are distinguished by the empty string: E accepts with no more input, A does not. D and A are distinguished by `b`: from D it leads to E (accept), from A to C (no). B and A are distinguished by `bb`: B, D, E against A, C, C. But A and C? On `a`, both go to B. On `b`, A goes to C and C goes to C: the same state. So from A and from C every string goes through exactly the same states after its first symbol, and no string can distinguish them. A and C are equivalent.

Checking every pair against every string is impossible, since there are infinitely many strings. The algorithm turns it around: instead of proving states equal, it finds the pairs that are *different* and keeps splitting until nothing more can be split. What is left together at the end is equivalent.

### The algorithm: start with two groups, split until stable

The Dragon book's version works on a *partition*: a division of the states into groups, every state in exactly one group.

1. **Start**: two groups, the accepting states and the non-accepting states. The empty string already distinguishes these.
2. **Split**: look at each group. Two states of the same group stay together only if, for every input symbol, their edges lead into the *same group* (not necessarily the same state). If some symbol sends them into different groups, they are distinguishable (by that symbol followed by whatever distinguishes those groups), so split the group by where its states go.
3. **Repeat** step 2 on the new partition until a whole round splits nothing.
4. **Build**: each group becomes one state of the minimal DFA. Its edges are those of any member, pointing to the groups. The start state is the group containing the old start; a group is accepting if its members are.

Two preparations come first. Remove any state that cannot be reached from the start (no string leads to it, so it does not matter what it does, but left in, it can make a group look different). And the DFA must be *complete*: every state needs an edge on every symbol. Where an edge is missing, it goes to a *dead state*: a non-accepting state whose every edge leads back to itself. Missing edges are where most hand errors in minimisation hide, so draw the dead state in if the table has blanks.

### Minimising the subset DFA by hand

Apply it to the five-state table. It has no blanks and every state is reachable.

**Round 0.** Accepting and non-accepting: {A, B, C, D} {E}.

**Round 1.** Group {E} has one state; it cannot split. Group {A, B, C, D}: write, for each state, the group each symbol leads to.

| State | `a` goes to | group | `b` goes to | group |
|---|---|---|---|---|
| A | B | {A B C D} | C | {A B C D} |
| B | B | {A B C D} | D | {A B C D} |
| C | B | {A B C D} | C | {A B C D} |
| D | B | {A B C D} | E | **{E}** |

D's `b` edge leads into a different group from the others', so D splits off. New partition: {A, B, C} {D} {E}.

**Round 2.** Group {A, B, C}, now judged against the new groups:

| State | `a` | group | `b` | group |
|---|---|---|---|---|
| A | B | {A B C} | C | {A B C} |
| B | B | {A B C} | D | **{D}** |
| C | B | {A B C} | C | {A B C} |

B splits off. New partition: {A, C} {B} {D} {E}.

**Round 3.** Group {A, C}: A goes to B on `a` and to C on `b`; C goes to B on `a` and to C on `b`. Same groups ({B} and {A, C}) on both symbols. Nothing splits. Stop.

Four groups, so four states. Name each group after its first member and copy the edges:

| State | `a` | `b` |
|---|---|---|
| → A (= A, C) | B | A |
| B | B | D |
| D | B | E |
| E * | B | A |

Rename B, D, E to B, C, D, and this is exactly the direct method's table from section 3.4. The two methods agree, as the uniqueness promise says they must, and now you *know* that four is the minimum, rather than hoping.

Notice what each round really discovered. Round 0 found the strings distinguished by the empty string. Round 1 found D, distinguished from A, B and C by the one-symbol string `b`. Round 2 found B, distinguished by the two-symbol string `bb`. Each round can find states that need a string one symbol longer to tell them apart, which is why the number of rounds is at most the number of states.

### The pair table, if your course uses it

Many courses teach the same idea as a table of pairs (sometimes called the table-filling or Myhill-Nerode method), and some papers ask for it by name. You list every pair of states, mark a pair as distinguishable when exactly one of them accepts, then go round marking any pair whose two states go, on some symbol, to a pair already marked, until a whole pass marks nothing. Unmarked pairs are equivalent.
<!-- risk: naming of the pair-table method (table-filling / Myhill-Nerode) as used in courses -->

For the five-state DFA: first mark (A,E), (B,E), (C,E), (D,E), since only E accepts. Then pass through the rest:

| Pair | On `a` | On `b` | Marked? |
|---|---|---|---|
| (A,D) | (B,B) | (C,E) marked | yes |
| (B,D) | (B,B) | (D,E) marked | yes |
| (C,D) | (B,B) | (C,E) marked | yes |
| (A,B) | (B,B) | (C,D) marked | yes |
| (B,C) | (B,B) | (D,C) marked | yes |
| (A,C) | (B,B) | (C,C) | no |

One more pass changes nothing. Only (A,C) is unmarked, so A and C merge: the same answer. The pair table and the splitting of groups are two ways of writing down the same reasoning; use the one your paper asks for.

### Minimising in C

The program reads a DFA in the form `subset` prints: lines `start X`, `final X Y ...` and `X a Y`, ignoring the `X = {...}` lines. It prints the partition after each round, then the minimal DFA in the same form, so its output could go into it again.

The partition is an array, `group[s]`, the group number of state *s*. A round computes a new array: two states end up in the same new group exactly when `same` says they were in the same group and every symbol leads both into the same group. Groups are numbered in order of their first state, so the printed partitions read from A onwards. Because rounds only ever split groups, "the number of groups did not grow" means "nothing split", and the loop stops. A missing edge leads to group −1, which stands for the dead state.

Save it as `minimise.c`:

<!-- code: id=minimise mode=compile -->
```c
/* minimise.c: minimises a DFA in the form subset prints, by splitting
   groups until no group splits. States are single letters A to Z.
   A missing edge goes to the dead state, which counts as group -1. */
#include <stdio.h>
#include <string.h>

#define MAXD 26

int nd;                         /* states A, B, ...: 0 .. nd-1 */
int start = -1;
int final[MAXD];
int tran[MAXD][256];            /* tran[s][c]: next state, -1 for none */
int used[256];                  /* the symbols that appear */
int group[MAXD];                /* the partition: group[s] */

/* the state named by a word such as "C", or -1 if it is not a name */
int state(const char *w)
{
    if (w[0] < 'A' || w[0] > 'Z' || w[1] != '\0')
        return -1;
    if (w[0] - 'A' >= nd)
        nd = w[0] - 'A' + 1;
    return w[0] - 'A';
}

/* the group that state s goes to on c; the dead state is group -1 */
int gto(int s, int c)
{
    return tran[s][c] < 0 ? -1 : group[tran[s][c]];
}

/* in the same group now, and every symbol leads both to the same group */
int same(int s, int t)
{
    int c;

    if (group[s] != group[t])
        return 0;
    for (c = 0; c < 256; c++)
        if (used[c] && gto(s, c) != gto(t, c))
            return 0;
    return 1;
}

void show(int round, int ngroups)
{
    int g, s;
    const char *sep;

    printf("round %d:", round);
    for (g = 0; g < ngroups; g++) {
        printf(" {");
        sep = "";
        for (s = 0; s < nd; s++)
            if (group[s] == g) {
                printf("%s%c", sep, 'A' + s);
                sep = " ";
            }
        printf("}");
    }
    printf("\n");
}

int main(void)
{
    char line[200], a[16], sym[16], b[16], *p;
    int s, t, c, g, n, ngroups, round, newgroup[MAXD], rep[MAXD];

    memset(tran, -1, sizeof tran);          /* every byte 0xff: every int -1 */
    while (fgets(line, sizeof line, stdin) != NULL) {
        if (line[0] == '\n')
            continue;
        if (strncmp(line, "start ", 6) == 0) {
            sscanf(line + 6, "%15s", a);
            start = state(a);
        } else if (strncmp(line, "final", 5) == 0) {
            for (p = strtok(line + 5, " \n"); p != NULL; p = strtok(NULL, " \n"))
                if ((s = state(p)) >= 0)
                    final[s] = 1;
        } else if (sscanf(line, "%15s %15s %15s", a, sym, b) == 3
                   && strcmp(sym, "=") == 0) {
            ;                               /* "A = {0 1 2 4 7}": not needed */
        } else if (sscanf(line, "%15s %15s %15s", a, sym, b) == 3
                   && (s = state(a)) >= 0 && (t = state(b)) >= 0
                   && sym[1] == '\0') {
            tran[s][(unsigned char)sym[0]] = t;
            used[(unsigned char)sym[0]] = 1;
        } else {
            fprintf(stderr, "minimise: bad line: %s", line);
            return 1;
        }
    }
    if (nd == 0 || start < 0) {
        fprintf(stderr, "minimise: no DFA on input\n");
        return 1;
    }

    /* round 0: accepting against non-accepting, A's group first */
    ngroups = 1;
    for (s = 0; s < nd; s++) {
        group[s] = final[s] != final[0];
        if (group[s] == 1)
            ngroups = 2;
    }
    show(0, ngroups);
    for (round = 1; ; round++) {
        n = 0;
        for (s = 0; s < nd; s++) {
            for (t = 0; t < s && !same(s, t); t++)
                ;
            newgroup[s] = t < s ? newgroup[t] : n++;
        }
        if (n == ngroups)
            break;                          /* nothing split: finished */
        memcpy(group, newgroup, sizeof group);
        ngroups = n;
        show(round, ngroups);
    }

    printf("no group splits: %d states\n", ngroups);
    for (s = nd - 1; s >= 0; s--)           /* each group is named after */
        rep[group[s]] = s;                  /* its first state */
    printf("start %c\nfinal", 'A' + rep[group[start]]);
    for (g = 0; g < ngroups; g++)
        if (final[rep[g]])
            printf(" %c", 'A' + rep[g]);
    printf("\n");
    for (g = 0; g < ngroups; g++)
        for (c = 0; c < 256; c++)
            if (used[c] && tran[rep[g]][c] >= 0)
                printf("%c %c %c\n", 'A' + rep[g], c,
                       'A' + rep[group[tran[rep[g]][c]]]);
    return 0;
}
```

Join the whole chain so far, from expression to minimal DFA, and give the program the empty input as well:

<!-- code: id=minimise-runs exit=1 -->
```console
$ gcc -Wall -Wextra minimise.c -o minimise
$ echo '(a|b)*abb' | ./thompson | ./subset | ./minimise
round 0: {A B C D} {E}
round 1: {A B C} {D} {E}
round 2: {A C} {B} {D} {E}
no group splits: 4 states
start A
final E
A a B
A b A
B a B
B b D
D a B
D b E
E a B
E b A
$ echo '(a|b)*a' | ./thompson | ./subset | ./minimise
round 0: {A C} {B}
no group splits: 2 states
start A
final B
A a B
A b A
B a B
B b A
$ ./minimise < /dev/null
minimise: no DFA on input
```

The rounds for `(a|b)*abb` are the rounds you did by hand: {A B C D} {E}, then D splits off, then B, then nothing. The minimal DFA is named A, B, D, E after the first state of each group. For `(a|b)*a` the three subset states shrink to two: A and C merge, as both the direct method and section 3.3's second check question suggested. You now have a pipeline that turns any expression in the book's notation into its minimal DFA in three short programs, which is what you will use to check every hand answer in the practice.

### A dead state that matters

Every DFA so far had an edge for every symbol from every state. Here is one that does not, and the dead state changes the answer. Take the subset DFA for `ab*|b`, from the practice in section 3.7. Its table, with blanks where `subset` found an empty set:

| State | `a` | `b` |
|---|---|---|
| → A | B | C |
| B * | | D |
| C * | | |
| D * | | D |

Round 0: {A} {B, C, D}. Now fill the blanks with a dead state, `-`, which is a group on its own. In round 1, B goes to (`-`, {B C D}) on (`a`, `b`); C goes to (`-`, `-`); D goes to (`-`, {B C D}). C differs on `b`, so {B, C, D} splits into {B, D} and {C}. Round 2 splits nothing. Result: A, B (= B, D) and C. Read it: after `a` you are in B, which accepts and loops on `b`; after `b` you are in C, which accepts and has no way on. That is `ab*|b` exactly.

If you had ignored the blanks instead, treating "no edge" as "no difference", B, C and D would have stayed together, and the merged state would loop on `b`, accepting `bb`, which is not in the language. Blanks are real edges, to the dead state.

### When it goes wrong

**Splitting by state instead of by group.** In round 1 above, A goes to C on `b` and B goes to D. C and D are different states, but in round 1 they are in the same group, so A and B stay together in that round. Split only when the *groups* differ; otherwise you end with too many states, often the original DFA unchanged.

**Starting with one group.** If you begin with all states together, nothing ever splits (every edge leads into the one group), and you "prove" the whole DFA is one state. Round 0 must separate accepting from non-accepting.

**Leaving in an unreachable state.** An unreachable state does not change the language, but it can survive minimisation as a group of its own, and then your minimal DFA has one state more than the key's. Remove unreachable states first: start from the start state, follow every edge, and cross out every state you never reach. `minimise` does not do this for you, because `subset` never produces an unreachable state; a DFA given in a question may well have one.

**Stopping one round early.** A round that splits something must be followed by another round, because the new groups can split older ones further. Stop only after a round in which nothing at all changed.

### In one line each

1. Two states are equivalent if no input string ends in acceptance from one and not the other.
2. Remove unreachable states; send every missing edge to a dead state.
3. Start with {accepting} and {non-accepting}; split any group whose members go, on some symbol, into different groups; repeat until nothing splits.
4. Each final group is a state of the minimal DFA, which is unique up to names.
5. For `(a|b)*abb`, A and C of the subset DFA merge, giving the same four states as the direct method; `thompson | subset | minimise` shows every round.

**Check yourself**

<!-- q: idea=dfa-minimisation type=apply wrong="A and B merge" -->
1. Minimise this DFA by splitting groups. Show every round.

| State | `0` | `1` |
|---|---|---|
| → A | B | C |
| B | A | C |
| C * | C | C |

<details>
<summary>Answer</summary>

Round 0: {A, B} {C}. Round 1: A goes to (B, C), that is ({A B}, {C}); B goes to (A, C), also ({A B}, {C}). Same groups, so no split. Stop. Two states: A (= A, B), which goes to A on `0` and to C on `1`, and C, accepting, looping on both. The language is "contains a `1`". You can check it: `printf 'start A\nfinal C\nA 0 B\nA 1 C\nB 0 A\nB 1 C\nC 0 C\nC 1 C\n' | ./minimise` prints round 0 and then `no group splits: 2 states`. Here the guess "A and B merge" is right; the point of the rounds is that you do not have to guess.

</details>

<!-- q: idea=dfa-minimisation type=predict wrong="a dead state never changes the result" -->
2. In a DFA, states P and Q are both accepting. On `x`, P goes to R and Q has no edge. R is non-accepting and has no edges at all. Can P and Q be equivalent? Answer with a distinguishing string or an argument.

<details>
<summary>Answer</summary>

Yes, they can be, as far as `x` is concerned. P's `x` leads to R, which is non-accepting with no way out: R is itself a dead state. Q's missing `x` edge leads to the dead state. So on `x` both reach "dead", and `x` followed by anything is rejected from both. R and the added dead state are in the same group throughout (both non-accepting, all edges to dead), so `x` does not split P from Q. Whether P and Q end up merged depends on their other edges. The lesson is that a missing edge and an edge to a dead state are the same thing; only a missing edge set against an edge to a *live* state distinguishes.

</details>

<!-- q: idea=dfa-minimisation type=fix wrong="the subset DFA was wrong" -->
3. A student minimises the five-state subset DFA for `(a|b)*abb` and gets five states: after round 0 they compared, in group {A, B, C, D}, A's `b` target C with B's `b` target D, found they were different states, and split every state off on its own. What exactly did they do wrong?

<details>
<summary>Answer</summary>

They split by target *state* instead of target *group*. In round 1, C and D are both in the group {A, B, C, D}, so A and B lead into the same group on `b` and must stay together for now. Only D, whose `b` leads into {E}, splits in round 1. Splitting by states makes every state look different from every other one whose edges go to different states, so no two states ever merge and the "minimal" DFA is the one you started with.

</details>

## 3.6 A table-driven DFA recogniser in C

### The lab question, and a better way to write the board

One of the commonest lab-exam questions in this part of the course reads like this: "Write a C program to simulate a DFA that accepts strings over {a, b} ending in `abb`", or "...that accepts identifiers", or "...given its transition table". You have the DFA: four states, from sections 3.4 and 3.5. What you need is the C.

Section 2.5 showed one way to write a board in C: `relop`, with one `case` per state, each case reading a character and choosing the next state with `if`s. That works, but every edge of the diagram becomes a line of code. Change the expression, and you rewrite the function. For the exam's DFA questions there is a shorter way that also matches how the paper presents DFAs: the *transition table* itself, typed into the program as a two-dimensional array, and a loop of four lines that follows it.

### The board as a grid

Picture the board game's rule book printed as a grid. One row per square, one column per kind of card; the cell where a row and a column meet says which square that card sends you to from that square. To play, you never look at the board at all. You keep your finger on a row number, read the card, look along the row to that card's column, and move your finger to the row named in the cell. When the cards run out, you look up whether your row is a winning one.

That is all a *table-driven* recogniser does, and it is exactly the table you wrote by hand in section 3.4. The picture holds well this time; it breaks only where the cards are not all on the grid. A real input can contain a character the table has no column for, and the program has to decide what that means.

### A recogniser for `(a|b)*abb`

<!-- teaches: dfa-simulation = transition table, table-driven recogniser -->
Here is the minimal DFA, with the direct method's names A to D, which the program numbers 0 to 3:

| State | `a` | `b` |
|---|---|---|
| → A (0) | B | A |
| B (1) | B | C |
| C (2) | B | D |
| D (3) * | B | A |

The program keeps three pieces of data: `trans`, the table, with one row per state and one column per symbol; `accepting`, one flag per state; and `column`, a function that turns an input character into a column number, or −1 for a character outside the alphabet. The whole machine is the loop in `main`: start in state 0, and for each character set `state = trans[state][column(c)]`. It reads one string per line and prints, for each, the states it passed through and the verdict, which is the trace the paper asks you to show by hand.

Save it as `dfa.c`:

<!-- code: id=dfa stdin="babb\nabb\nabba\n\naabbabb\nabcb\n" -->
```c
/* dfa.c: a table-driven DFA for (a|b)*abb, the minimal one of 3.4 and 3.5.
   Reads one string per line; prints the states it passes through and
   whether the string is accepted. */
#include <stdio.h>
#include <string.h>

#define NSTATES 4
#define NSYMS 2

/* trans[state][column]: where that symbol sends you from that state */
int trans[NSTATES][NSYMS] = {
    /*         a  b  */
    /* A */  { 1, 0 },
    /* B */  { 1, 2 },
    /* C */  { 1, 3 },
    /* D */  { 1, 0 },
};
int accepting[NSTATES] = { 0, 0, 0, 1 };
const char *name = "ABCD";

/* the column for an input character, or -1 if it is not in the alphabet */
int column(char c)
{
    if (c == 'a')
        return 0;
    if (c == 'b')
        return 1;
    return -1;
}

int main(void)
{
    char line[256];
    int state, col, i;

    while (fgets(line, sizeof line, stdin) != NULL) {
        line[strcspn(line, "\n")] = '\0';
        state = 0;                              /* every string starts at A */
        printf("%-8s %c", line[0] != '\0' ? line : "(empty)", name[state]);
        for (i = 0; line[i] != '\0'; i++) {
            col = column(line[i]);
            if (col < 0)
                break;
            state = trans[state][col];
            printf(" %c %c", line[i], name[state]);
        }
        if (line[i] != '\0')
            printf("  rejected: '%c' is not in the alphabet\n", line[i]);
        else if (accepting[state])
            printf("  accepted\n");
        else
            printf("  rejected\n");
    }
    return 0;
}
```
```text
babb     A b A a B b C b D  accepted
abb      A a B b C b D  accepted
abba     A a B b C b D a B  rejected
(empty)  A  rejected
aabbabb  A a B a B b C b D a B b C b D  accepted
abcb     A a B b C  rejected: 'c' is not in the alphabet
```

Read the first line of output as the trace you would write in the paper: start in A; `b` keeps you in A; `a` takes you to B; `b` to C; `b` to D, which accepts. The second line is `abb` with nothing in front: three steps, A to B to C to D. `abba` reaches D and then the final `a` drags it back to B, so it is rejected: the string must *end* in `abb`. The empty line never leaves A, which does not accept. `aabbabb` passes through D in the middle and is pulled back to B by the `a`, then climbs to D again. And `abcb` stops at the `c`.

### Walking the code

Take `abb` through `main`, keeping track of the variables:

| Step | `i` | `line[i]` | `column` | `state` before | `trans[state][col]` | `state` after |
|---|---|---|---|---|---|---|
| start | | | | | | 0 (A) |
| 1 | 0 | `a` | 0 | 0 | `trans[0][0]` = 1 | 1 (B) |
| 2 | 1 | `b` | 1 | 1 | `trans[1][1]` = 2 | 2 (C) |
| 3 | 2 | `b` | 1 | 2 | `trans[2][1]` = 3 | 3 (D) |
| end | 3 | `'\0'` | | | | 3: `accepting[3]` is 1 |

The loop does one array lookup per character and nothing else. It does not know or care what language it recognises; that knowledge is all in the three pieces of data at the top. To make the program recognise a different language, you change `trans`, `accepting`, `name` and `column`, and leave `main` alone. That is the reason to write a recogniser this way: the minimal DFA from `minimise`, or from your hand work, goes into the program by copying the table, row for row, and the loop is the same in every exam question.

The `break` on a character outside the alphabet is a decision, and it deserves one sentence in a viva: a string containing `c` is not a string over {a, b}, so it cannot be in the language; the program rejects it and says why, rather than reading `trans[state][-1]`, which would be outside the array.

Run it on the three kinds of input once more from the terminal, the last one being an input that is not even text over the alphabet:

<!-- code: id=dfa-runs -->
```console
$ gcc -Wall -Wextra dfa.c -o dfa
$ printf 'ababb\nbbbb\n' | ./dfa
ababb    A a B b C a B b C b D  accepted
bbbb     A b A b A b A b A  rejected
$ ./dfa < /dev/null
$ printf 'a b b\n' | ./dfa
a b b    A a B  rejected: ' ' is not in the alphabet
```

The empty input prints nothing, because there are no lines; that is different from an empty *line*, which is a string of length zero and gets a verdict. The spaced `a b b` is rejected at the first space, because a space is not in the alphabet either.

### Classes of characters: one table for Tiny's identifiers and numbers

For `{a, b}` there is one column per symbol. A scanner's alphabet is every character, and a table with 256 columns, nearly all identical, would be silly. The cure is the *character class* idea from section 3.1: characters that every state treats alike share one column. For Tiny's identifiers and numbers, four classes are enough: a letter or underscore, a digit, the point, and anything else.

Section 2.5 drew identifiers and numbers as two separate boards, Figure 2.5b, and promised that this chapter would join them into one and drive it from a table. Joining is the union rule of section 3.1: the pattern is `id | number`. Run the subset construction in your head on that union and the two start states (12 and 15 in Figure 2.5b) become one, because their closure is taken together, while everything after the first character stays apart, because a letter and a digit lead to different boards. The joined DFA, with Figure 2.5b's states beside it:

| State | Figure 2.5b | letter or `_` | digit | `.` | other | Accepts as |
|---|---|---|---|---|---|---|
| 0 → | 12 and 15 | 1 | 2 | | | |
| 1 | 13 | 1 | 1 | | | `id` |
| 2 | 16 | | 2 | 3 | | `num` |
| 3 | 17 | | 4 | | | |
| 4 | 19 | | 4 | | | `num` |

The starred states 14, 18 and 20 of Figure 2.5b have gone. In a scanner they were where the lookahead character was pushed back; here, recognising one whole lexeme per line, the end of the line is the end of the lexeme, so the program simply asks, at the end, whether the state it is in accepts, and as what. A blank cell is a missing edge, which the program writes as −1.

Save it as `tokdfa.c`:

<!-- code: id=tokdfa stdin="pos\ninit\nrate\n60\n2.5\n_v1\n5.\n9lives\n+\n" -->
```c
/* tokdfa.c: one DFA table for Tiny's identifiers and numbers
   (Figure 2.5b joined). Reads one lexeme per line and names its token. */
#include <stdio.h>
#include <string.h>
#include <ctype.h>

enum { LETTER, DIGIT, POINT, OTHER };   /* the columns: character classes */

/* trans[state][class]: the next state, or -1 for no edge */
int trans[5][4] = {
    /*            letter digit point other */
    /* 0 start */ {  1,    2,   -1,   -1 },
    /* 1 id    */ {  1,    1,   -1,   -1 },
    /* 2 int   */ { -1,    2,    3,   -1 },
    /* 3 point */ { -1,    4,   -1,   -1 },
    /* 4 frac  */ { -1,    4,   -1,   -1 },
};
/* the token each state accepts as; NULL for a state that does not accept */
const char *token[5] = { NULL, "id", "num", NULL, "num" };

int class(int c)
{
    if (isalpha(c) || c == '_')
        return LETTER;
    if (isdigit(c))
        return DIGIT;
    if (c == '.')
        return POINT;
    return OTHER;
}

int main(void)
{
    char line[256];
    int state, next, i;

    while (fgets(line, sizeof line, stdin) != NULL) {
        line[strcspn(line, "\n")] = '\0';
        state = 0;
        for (i = 0; line[i] != '\0'; i++) {
            next = trans[state][class((unsigned char)line[i])];
            if (next < 0)
                break;                  /* no edge: stuck */
            state = next;
        }
        if (line[i] != '\0')
            printf("%-8s no token: no edge from state %d on '%c'\n",
                   line, state, line[i]);
        else if (token[state] != NULL)
            printf("%-8s %s\n", line, token[state]);
        else
            printf("%-8s no token: the input ended in state %d\n", line, state);
    }
    return 0;
}
```
```text
pos      id
init     id
rate     id
60       num
2.5      num
_v1      id
5.       no token: the input ended in state 3
9lives   no token: no edge from state 2 on 'l'
+        no token: no edge from state 0 on '+'
```

Every lexeme of the running statement except the operators comes out right: `pos`, `init` and `rate` are `id`, `60` is `num`. `2.5` goes 0, 2, 3, 4 and ends in an accepting state. `5.` ends in state 3, which does not accept: the table says exactly what Tiny's rule says, that a point must be followed by a digit. `9lives` gets stuck in state 2 at the `l`, because a number cannot continue with a letter. And `+` has no edge from the start at all; it belongs to the operator board, Figure 2.5a, which could be joined into the same table with more columns and states.

Compare `9lives` with what `tiny-scan` did in chapter 2: it cut `9` as a number and then `lives` as an identifier, using the longest-match rule of section 2.3. A recogniser answers "is this whole string a token?"; a scanner answers "what is the longest token at the front of the input?", and it does that by running the same kind of table, remembering the last accepting state it passed, and backing up to it when it gets stuck. Chapter 4 hands that job to Flex, whose generated scanner is a loop of exactly this kind over tables it builds from your patterns, and section 4.2 shows the longest-match rule as Flex applies it.
<!-- risk: claim that flex's generated scanner is a table-driven DFA loop with backing up to the last accepting state -->

### Why tables

A switch per state, like `relop`, and a table, like `dfa`, describe the same machine. The table wins when the machine comes from an algorithm rather than from your head: the subset construction and `minimise` produce tables, and copying a table is far less error-prone than turning it into `if`s. It also wins when the machine changes: new token, new rows, same loop. That is the deal a scanner generator makes. You write regular expressions; it runs Thompson's construction or the direct method, the subset construction and minimisation, and emits a table and the loop. The switch wins when each state needs its own special action, such as counting lines or pushing back a character, and for hand-written scanners like `tiny-scan` it is often clearer.

The cost of a table is its size: states times columns. Classes keep the columns few. `tokdfa` has 5 × 4 = 20 cells; with one column per byte it would be 5 × 256 = 1,280, almost all of them −1.

### When it goes wrong

**Every string is rejected.** The usual cause is the newline. `fgets` keeps the `'\n'` at the end of the line, and if you forget the `strcspn` line, the newline is the last character of every string; `column('\n')` is −1, so every line ends with "rejected: ... is not in the alphabet". Both the printed string and the quoted character carry the line break, so each verdict comes out spread over three lines, with the trace on the second and a lone `'` starting the third. The cure is the one line that removes it.

**The rows are in a different order from the names.** If `trans` has its rows in the order A, B, C, D but `name` says `"ABDC"`, the program works and the trace lies. Write the state names as comments at the start of each row, as `dfa.c` does, and keep `name` in the same order.

**Rows and columns swapped.** Writing `trans[col][state]` instead of `trans[state][col]` compiles without a warning when the table happens to be square, and with a 4 × 2 table reads past the end of a row. The sanitizer catches the second case; the first just gives wrong answers. The rule: the row is where you are, the column is what you read.

**A negative index.** On most Linux machines `char` is signed, so a byte above 127 becomes a negative number. Passed to `isalpha`, or used to index a 256-column table, it reads outside the array. That is why `tokdfa` passes `(unsigned char)line[i]` to `class`.
<!-- risk: char is signed on x86 Linux GCC; ctype functions take unsigned char values or EOF -->

**The state is not reset.** If `state = 0` sits before the `while` instead of inside it, the second string starts where the first one ended. After `abb` ends in D, an empty line would be "accepted". Every string starts at the start state.

### In one line each

1. A table-driven recogniser keeps the DFA as data: `trans[state][column]`, an accepting flag per state, and a function from characters to columns.
2. The loop is the same for every DFA: start at 0, `state = trans[state][column(c)]` per character, then check `accepting[state]`.
3. A character outside the alphabet has no column; reject the string and say why, never index with −1.
4. Character classes keep a scanner's table small; Tiny's identifiers and numbers need five states and four columns.
5. A recogniser judges a whole string; a scanner finds the longest token at the front, which Flex does in chapter 4.

**Check yourself**

<!-- q: idea=dfa-simulation type=predict wrong="abbb is accepted because it contains abb" -->
1. Without running it, write the line `dfa` prints for `abbb` and for `babab`.

<details>
<summary>Answer</summary>

`abbb     A a B b C b D b A  rejected` and `babab    A b A a B b C a B b C  rejected`. In the first, the fourth symbol `b` takes D back to A: D's `b` edge goes to A, because after `abbb` the useful ending has been lost. In the second, the string ends in `ab`, which leaves you in C. The tempting wrong answer for `abbb` is "accepted, it contains `abb`"; the language is strings *ending* in `abb`. You can check with `printf 'abbb\nbabab\n' | ./dfa`.

</details>

<!-- q: idea=dfa-simulation type=apply wrong="only main must change" -->
2. Change `dfa.c` into a recogniser for `(a|b)*a`, using the two-state minimal DFA from section 3.4. Which lines change?

<details>
<summary>Answer</summary>

Only the data. `NSTATES` becomes 2; `trans` becomes `{ { 1, 0 }, { 1, 0 } }` (A: `a` to B, `b` to A; B: `a` to B, `b` to A); `accepting` becomes `{ 0, 1 }`; `name` becomes `"AB"`. `NSYMS`, `column` and `main` stay as they are. Check with `printf 'a\nba\nab\n\n' | ./dfa`: `a` and `ba` accepted, `ab` and the empty line rejected. The tempting wrong belief is that the loop must change for a new language; the point of the table is that it never does.

</details>

<!-- q: idea=dfa-simulation type=fix wrong="the table is wrong" -->
3. A student's `tokdfa` prints `id` for `9lives`. Their table is the one above. They changed only `class`, which now begins `if (isalnum(c) || c == '_') return LETTER;`. Explain the output.

<details>
<summary>Answer</summary>

`isalnum` is true for digits as well as letters, so every digit is now in the LETTER column. `9` takes state 0 to state 1 (the identifier state), and `lives` keeps it there, so the lexeme is accepted as `id`. The DIGIT column is never used, so no number can be recognised at all: `60` also prints `id`. The table is fine; the classes overlap. Each character must belong to exactly one class, and `class` must test them so that they cannot overlap: letters with `isalpha`, digits with `isdigit`.

</details>

## 3.7 Practice: one regular expression carried to a minimal DFA and a C recogniser

You can now take a pattern all the way from a line of notation to a running program: write it as a regular expression (3.1), build its NFA with Thompson's construction (3.2), turn the NFA into a DFA with ε-closures and the subset construction (3.3) or go straight to a DFA with followpos (3.4), minimise it by splitting groups (3.5), and type the minimal table into a C recogniser (3.6). The tasks climb from predicting what the chapter's programs print, through working new expressions alone, to the whole chain on an expression you have not seen, ending in a program checked without anyone marking it. Work in `~/cdlab/ch3`, where `match`, `thompson`, `subset`, `minimise`, `dfa` and `tokdfa` already are. Do each task by hand first, on paper, the way the written paper wants it; then use the programs to check.

### Guided

<!-- q: idea=thompson-construction type=predict wrong="ab*|b has seven states" -->
**1. Predict Thompson's NFA.** Build the Thompson NFA for `ab*|b` by hand, numbering in the Dragon order. How many states and edges does it have, and which state is final? Write the edges in the form `thompson` prints, then check:

<!-- code: id=practice-thompson answer -->
```console
$ echo 'ab*|b' | ./thompson
```

<details>
<summary>Answer</summary>

```console
$ echo 'ab*|b' | ./thompson
states 9
start 0
final 8
0 eps 1
0 eps 6
1 a 2
2 eps 3
2 eps 5
3 b 4
4 eps 3
4 eps 5
5 eps 8
6 b 7
7 eps 8
```

Nine states and eleven edges, final state 8. The union is the outermost operator, so its new start is 0. Its left operand `ab*` starts at the new state 1: `a` goes from 1 to 2, and the star around `b` is built from state 2, with the new inner start 3, `b` from 3 to 4, and the star's new final 5 (edges 2 to 3, 2 to 5, 4 to 3, 4 to 5). The right operand `b` starts at the new state 6 and ends at 7. The union's new final, 8, gets ε edges from 5 and from 7. The tempting wrong count is 7, from building `b*` with the shortcut star; Thompson's star always adds its own two states.

</details>

<!-- q: idea=subset-construction type=apply wrong="the empty move gives a state" -->
**2. Subset construction with missing edges.** Continue with the NFA of task 1. Compute the DFA by the subset construction, by hand, showing each set. Where does a move give the empty set, and what do you write in the table there? Then check:

<!-- code: id=practice-subset answer -->
```console
$ echo 'ab*|b' | ./thompson | ./subset
```

<details>
<summary>Answer</summary>

```console
$ echo 'ab*|b' | ./thompson | ./subset
A = {0 1 6}
B = {2 3 5 8}
C = {7 8}
D = {3 4 5 8}
start A
final B C D
A a B
A b C
B b D
D b D
```

A = ε-closure({0}) = {0, 1, 6}. On `a`: move gives {2}, closed through 2 to 3 and 5, and 5 to 8: B = {2, 3, 5, 8}. On `b`: {7}, closed to C = {7, 8}. From B, `a` moves nowhere (no `a` edge from 2, 3, 5 or 8): empty, so no edge. On `b`: {4}, closed through 4 to 3 and 5, and 5 to 8: D = {3, 4, 5, 8}. From C, both symbols give the empty set. From D, `a` gives nothing and `b` gives {4} again, which is D. B, C and D contain 8 and accept. Where a move is empty, the table gets a blank (no edge, the string is rejected), and `subset` prints no line for it; for minimisation the blank becomes an edge to a dead state, as section 3.5 showed with this same table.

</details>

### On your own

<!-- q: idea=direct-dfa type=apply unseen="a-anything-b" -->
**3. The direct method on a new expression.** Construct the DFA for `a(a|b)*b` (strings that start with `a` and end with `b`) directly, with nullable, firstpos, lastpos and followpos. Is your DFA already minimal? Check the number of states with `echo 'a(a|b)*b' | ./thompson | ./subset | ./minimise`, remembering that `minimise` counts only the live states, never the dead one.

<details>
<summary>Answer</summary>

Augmented: `a(a|b)*b#`, positions `a` 1, `a` 2, `b` 3, `b` 4, `#` 5. The star of `2 | 3` is nullable, with firstpos and lastpos {2, 3}. The concatenation of 1 with the star has firstpos {1} (1 is not nullable) and lastpos {1, 2, 3} (the star is nullable, so 1 can also be last). The next concatenation, with 4, has lastpos {4}; the root has firstpos {1}.

followpos: the first concatenation gives followpos(1) ⊇ {2, 3}; the star gives 2 and 3 the set {2, 3}; the concatenation with 4 gives 1, 2 and 3 the position 4; the root gives followpos(4) = {5}. So followpos(1) = followpos(2) = followpos(3) = {2, 3, 4}, followpos(4) = {5}.

DFA: A = {1}. On `a` (position 1): {2, 3, 4} = B. On `b`: no `b` position in A, no edge. From B: on `a` (position 2) {2, 3, 4} = B; on `b` (positions 3 and 4) {2, 3, 4} ∪ {5} = {2, 3, 4, 5} = C, accepting. From C: on `a`, B; on `b`, C.

| State | `a` | `b` |
|---|---|---|
| → A | B | |
| B | B | C |
| C * | B | C |

It is minimal. Round 0: {A, B} {C}. Round 1: A goes to ({A B}, dead) and B to ({A B}, {C}), so they split. Three live states, and `minimise` agrees with `no group splits: 3 states`. If you forgot that the star is nullable, lastpos of `a(a|b)*` would come out as {2, 3}, followpos(1) would miss 4, and your DFA would reject `ab`.

</details>

<!-- q: idea=dfa-minimisation type=decide unseen="exactly-one-1" -->
**4. Trust the program, or not?** Here is a DFA over {0, 1} given in a question. G is not reachable from A.

| State | `0` | `1` |
|---|---|---|
| → A | B | C |
| B | A | D |
| C * | E | F |
| D * | E | F |
| E * | E | F |
| F | F | F |
| G | A | G |

Minimise it by hand. Then feed the whole table, G included, to `minimise` and decide whether its answer is the minimal DFA. If not, say why, and what to do.

Augmented: `a(a|b)*b#`, positions `a` 1, `a` 2, `b` 3, `b` 4, `#` 5. The star of `2 | 3` is nullable, with firstpos and lastpos {2, 3}. The concatenation of 1 with the star has firstpos {1} (1 is not nullable) and lastpos {1, 2, 3} (the star is nullable, so 1 can also be last). The next concatenation, with 4, has lastpos {4}; the root has firstpos {1}.

followpos: the first concatenation gives followpos(1) ⊇ {2, 3}; the star gives 2 and 3 the set {2, 3}; the concatenation with 4 gives 1, 2 and 3 the position 4; the root gives followpos(4) = {5}. So followpos(1) = followpos(2) = followpos(3) = {2, 3, 4}, followpos(4) = {5}.

DFA: A = {1}. On `a` (position 1): {2, 3, 4} = B. On `b`: no `b` position in A, no edge. From B: on `a` (position 2) {2, 3, 4} = B; on `b` (positions 3 and 4) {2, 3, 4} ∪ {5} = {2, 3, 4, 5} = C, accepting. From C: on `a`, B; on `b`, C.

| State | `a` | `b` |
|---|---|---|
| → A | B | |
| B | B | C |
| C * | B | C |

It is minimal. Round 0: {A, B} {C}. Round 1: A goes to ({A B}, dead) and B to ({A B}, {C}), so they split. Three live states, and `minimise` agrees with `no group splits: 3 states`. If you forgot that the star is nullable, lastpos of `a(a|b)*` would come out as {2, 3}, followpos(1) would miss 4, and your DFA would reject `ab`.

</details>

<!-- q: idea=dfa-minimisation type=decide unseen="exactly-one-1" -->
**4. Trust the program, or not?** Here is a DFA over {0, 1} given in a question. G is not reachable from A.

| State | `0` | `1` |
|---|---|---|
| → A | B | C |
| B | A | D |
| C * | E | F |
| D * | E | F |
| E * | E | F |
| F | F | F |
| G | A | G |

Minimise it by hand. Then feed the whole table, G included, to `minimise` and decide whether its answer is the minimal DFA. If not, say why, and what to do.

<!-- code: id=practice-minimise answer -->
```console
$ printf 'start A\nfinal C D E\nA 0 B\nA 1 C\nB 0 A\nB 1 D\nC 0 E\nC 1 F\nD 0 E\nD 1 F\nE 0 E\nE 1 F\nF 0 F\nF 1 F\nG 0 A\nG 1 G\n' | ./minimise
```

<details>
<summary>Answer</summary>

```console
$ printf 'start A\nfinal C D E\nA 0 B\nA 1 C\nB 0 A\nB 1 D\nC 0 E\nC 1 F\nD 0 E\nD 1 F\nE 0 E\nE 1 F\nF 0 F\nF 1 F\nG 0 A\nG 1 G\n' | ./minimise
round 0: {A B F G} {C D E}
round 1: {A B} {C D E} {F G}
round 2: {A B} {C D E} {F} {G}
no group splits: 4 states
start A
final C
A 0 A
A 1 C
C 0 C
C 1 F
F 0 F
F 1 F
G 0 A
G 1 G
```

By hand, first cross out G: no edge from A, B, C, D, E or F leads to it. Then round 0: {A, B, F} {C, D, E}. Round 1: A goes to ({A B F}, {C D E}), B the same, F to ({A B F}, {A B F}); F splits off. C, D and E all go to ({C D E}, {A B F}) and stay. Round 2 splits nothing. Three states: A (= A, B), C (= C, D, E), accepting, and F, a dead state. The language is `0*10*`: exactly one `1`.

`minimise`, given G, finds four groups: in round 1 G looks like F (both its edges lead into the non-accepting group), and only in round 2 does G's `0` edge to A, now in a different group from F, split them. G survives as a state of its own. That answer is not minimal, and it is not wrong in what it accepts, because G can never be reached. The program assumes what `subset` guarantees, that every state is reachable; a DFA from a question carries no such promise. Remove unreachable states first, by hand or by deleting G's lines, and `minimise` then prints three states.

</details>

### Chapter project: `(a|b)*a(a|b)`, from expression to program

<!-- q: idea=dfa-simulation type=build unseen="second-last-a" -->
**5. The whole chain.** The language is "strings over {a, b} whose second-last symbol is `a`": `aa`, `ab`, `baa`, `bbab`, ... Its expression is `(a|b)*a(a|b)`. Carry it through the chapter, by hand at each step, checking each step with the programs before you go on:

1. Build its Thompson NFA. How many states?
2. Run the subset construction. How many DFA states, and which accept?
3. Minimise the result. Which states merge?
4. Write `second.c`, a table-driven recogniser for the minimal DFA, in the style of `dfa.c`: one string per line; print the string (or `(empty)`), the start state's letter, then for each symbol a space, the symbol, a space and the new state's letter; then two spaces and `accepted`, `rejected`, or `rejected: 'X' is not in the alphabet`. Name the minimal states A, B, C, D in the order `minimise` lists them.

Test it with these nine lines (the sixth is empty), then with an empty input:

```text
ab
ba
aa
bb
a

babab
abba
abc
```

**Done when** your program prints exactly the model's output for those nine lines. You can test that without anyone marking it: in the understudy folder (the one that holds `book/`), in the terminal, run `python3 scripts/us_practice.py check compiler-design 3.7 --in ~/cdlab/ch3`. It builds your `second.c` and compares what it prints with the model below.

<details>
<summary>Answer</summary>

**1. Thompson.** Fourteen states, 0 to 13. States 0 to 7 are `(a|b)*` exactly as in Figure 3.2a; the `a` goes from 7 to 8; then the union `(a|b)` starts at 8 (merged by the concatenation) with new lane starts 9 and 11, edges 9 `a` 10 and 11 `b` 12, and the new final 13.

**2. Subset.** A = {0, 1, 2, 4, 7}. B = ε-closure({3, 8}) = {1, 2, 3, 4, 6, 7, 8, 9, 11}. C = ε-closure({5}) = {1, 2, 4, 5, 6, 7}. From B, `a` moves to {3, 8, 10}: D = {1, 2, 3, 4, 6, 7, 8, 9, 10, 11, 13}; `b` moves to {5, 12}: E = {1, 2, 4, 5, 6, 7, 12, 13}. D and E contain 13 and accept. Edges: A (`a` B, `b` C), B (D, E), C (B, C), D (D, E), E (B, C).

**3. Minimise.** Round 0: {A B C} {D E}. Round 1: B's `a` leads into {D E}, A's and C's do not, so B splits off; D's `a` stays in {D E} while E's goes to {A B C}, so D and E split. Round 2: A and C go to (B, A-or-C) on (`a`, `b`): no split. Four states: A (= A, C), B, D, E. Renamed A, B, C, D, they mean "the last two symbols were (something)b or nothing yet", "...ba or just a", "...aa", "...ab". C and D accept: the second-last symbol was `a`.

<!-- code: id=second-chain -->
```console
$ echo '(a|b)*a(a|b)' | ./thompson | grep states
states 14
$ echo '(a|b)*a(a|b)' | ./thompson | ./subset | ./minimise
round 0: {A B C} {D E}
round 1: {A C} {B} {D} {E}
no group splits: 4 states
start A
final D E
A a B
A b A
B a D
B b E
D a D
D b E
E a B
E b A
```

**4. The program.** The table, in the order `minimise` printed (A, B, D, E renamed A, B, C, D):

| State | `a` | `b` |
|---|---|---|
| → A | B | A |
| B | C | D |
| C * | C | D |
| D * | B | A |

<!-- code: id=second check="gcc -Wall -Wextra second.c -o second && ./second" stdin="ab\nba\naa\nbb\na\n\nbabab\nabba\nabc\n" -->
```c
/* second.c: a table-driven DFA for (a|b)*a(a|b):
   strings over {a, b} whose second-last symbol is a. */
#include <stdio.h>
#include <string.h>

#define NSTATES 4
#define NSYMS 2

/* trans[state][column]; the states remember the last two symbols */
int trans[NSTATES][NSYMS] = {
    /*                     a  b  */
    /* A: ..b or nothing */ { 1, 0 },
    /* B: ..ba or a      */ { 2, 3 },
    /* C: ..aa           */ { 2, 3 },
    /* D: ..ab           */ { 1, 0 },
};
int accepting[NSTATES] = { 0, 0, 1, 1 };
const char *name = "ABCD";

int column(char c)
{
    if (c == 'a')
        return 0;
    if (c == 'b')
        return 1;
    return -1;
}

int main(void)
{
    char line[256];
    int state, col, i;

    while (fgets(line, sizeof line, stdin) != NULL) {
        line[strcspn(line, "\n")] = '\0';
        state = 0;
        printf("%-8s %c", line[0] != '\0' ? line : "(empty)", name[state]);
        for (i = 0; line[i] != '\0'; i++) {
            col = column(line[i]);
            if (col < 0)
                break;
            state = trans[state][col];
            printf(" %c %c", line[i], name[state]);
        }
        if (line[i] != '\0')
            printf("  rejected: '%c' is not in the alphabet\n", line[i]);
        else if (accepting[state])
            printf("  accepted\n");
        else
            printf("  rejected\n");
    }
    return 0;
}
```
```text
ab       A a B b D  accepted
ba       A b A a B  rejected
aa       A a B a C  accepted
bb       A b A b A  rejected
a        A a B  rejected
(empty)  A  rejected
babab    A b A a B b D a B b D  accepted
abba     A a B b D b A a B  rejected
abc      A a B b D  rejected: 'c' is not in the alphabet
```

`ab` and `aa` are accepted, `ba`, `bb` and `a` are not (`a` has no second-last symbol at all, nor has the empty line). `babab` ends in `ab`, accepted; `abba` ends in `ba`, rejected; `abc` stops at the `c`. The empty input, `./second < /dev/null`, prints nothing. The usual mistakes in this project: copying the subset DFA's five rows instead of the minimal four (it still works, so only the check against the model shows it); naming the states in a different order from the table's rows; and accepting only in the "..aa" state, forgetting that "..ab" also has `a` second from the end.

</details>
