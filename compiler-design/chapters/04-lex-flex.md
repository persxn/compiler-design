# Chapter 4: Lex and Flex programs

<!-- contents -->
- [4.1 The first Lex program: the three parts of a .l file, flex, gcc, run](#41-the-first-lex-program-the-three-parts-of-a-l-file-flex-gcc-run)
- [4.2 Patterns and actions: longest match first, then the earlier rule](#42-patterns-and-actions-longest-match-first-then-the-earlier-rule)
- [4.3 Counting programs: lines, words, characters, vowels and consonants](#43-counting-programs-lines-words-characters-vowels-and-consonants)
- [4.4 A Lex scanner for keywords, identifiers, numbers and operators](#44-a-lex-scanner-for-keywords-identifiers-numbers-and-operators)
- [4.5 Reading files and ending input: yyin, yyout, yywrap and noyywrap](#45-reading-files-and-ending-input-yyin-yyout-yywrap-and-noyywrap)
- [4.6 Start conditions: removing comments and scanning strings](#46-start-conditions-removing-comments-and-scanning-strings)
- [4.7 When flex or gcc complains: the usual errors and their cures](#47-when-flex-or-gcc-complains-the-usual-errors-and-their-cures)
- [4.8 Practice: the lab set of Lex programs, written from the question alone](#48-practice-the-lab-set-of-lex-programs-written-from-the-question-alone)
<!-- /contents -->

## 4.1 The first Lex program: the three parts of a .l file, flex, gcc, run

### A hundred and fifty lines you did not need to write

`tiny-scan.c` in chapter 2 took about a hundred and fifty lines of C to recognise eighteen kinds of token. Most of those lines did the same few jobs again and again: read a character, decide whether it can continue the lexeme, push back the one that cannot, compare the lexeme against a list. Then chapter 3 showed that every pattern in that scanner can be written as a regular expression in one line, and that a program can turn a regular expression into a DFA without any thinking on your part: `thompson`, `subset` and `minimise` did exactly that.

Put those two facts together and you get Lex. You write the patterns as regular expressions, and next to each pattern a few lines of C saying what to do when it matches. A program reads that list, builds one DFA for all the patterns together, and writes the whole scanner in C for you: the reading, the lookahead, the pushing back, the longest match. The original program, written at Bell Labs in the 1970s, was called `lex`; the one on every Linux machine today is its free rewrite, Flex. In the lab you will be asked for "a Lex program" and you will run it with `lex` or `flex`; on Linux both names run the same Flex 2.6.4.
<!-- risk: history: lex written at Bell Labs in the 1970s (Lesk and Schmidt, 1975); flex as its free rewrite -->
<!-- risk: on the book's Linux machines `lex` is flex 2.6.4 under another name; some lab machines have no `lex` link -->

This section writes the smallest useful Lex program, builds it and runs it, and then opens up what Flex made, so that when the examiner points at `yylex` or `yytext` you can say exactly what it is.

### A machine that builds the station

Chapter 1 pictured the compiler as an assembly line of stations, each handing its product to the next. The scanner is the first station. In chapter 2 you built that station by hand, part by part. Flex is a different kind of worker: you hand it a specification sheet, "when you see this shape, do that", and it builds the station for you. The station it builds is the board game of chapter 3: a DFA in which each square says where each input card sends you, plus the code that walks the board.

The picture is true in the way that matters: you never touch the station's insides, only the sheet. It stops being true in two places. First, the sheet is not just a list of shapes; each line carries C code of your own, and that code is pasted, character for character, into the station Flex builds. A mistake in your C is a mistake in the station, and GCC, not Flex, reports it. Second, Flex does not run anything. It writes a C file and stops. You still have to compile that file with GCC and run the program yourself, which is why every Lex program in this chapter takes three commands: `flex`, `gcc`, run.

### The three parts of a .l file

<!-- teaches: lex-structure = Lex specification, %% -->
A Lex program is a text file whose name ends in `.l`, called a *Lex specification*. It has three parts, separated by lines that hold only `%%`:

```text
definitions
%%
rules
%%
user code
```

The *definitions section* holds things Flex must know before it reads the rules: C code to copy to the top of the generated file (between the lines `%{` and `%}`), options that start with `%option`, and names for regular expressions, which section 4.2 uses. The *rules section* is the heart: one rule per line, a pattern starting in the very first column, then blanks, then an action in C. The *user code section* is plain C, copied unchanged to the end of the generated file; this is where your `main` goes.

Make this chapter's folder. Open a terminal in your home folder:

<!-- code: id=ch4-setup -->
```console
$ mkdir cdlab/ch4
$ cd cdlab/ch4
```

The first program finds the names and numbers in a line of Tiny and says what each piece is. Type it in your editor and save it as `first.l`, or paste the whole command below into the terminal: `cat` with a here-document writes everything up to the line `END` into the file, as in chapter 2.

<!-- code: id=first-write -->
```console
$ cat > first.l <<'END'
%{
/* first.l: names, numbers and everything else in a line of Tiny */
#include <stdio.h>
%}
%option noyywrap nounput noinput
%%
[A-Za-z_][A-Za-z0-9_]*   { printf("name   %s\n", yytext); }
[0-9]+                   { printf("number %s\n", yytext); }
[ \t\n]+                 { /* blanks: do nothing */ }
.                        { printf("other  %s\n", yytext); }
%%
int main(void)
{
    yylex();
    return 0;
}
END
```

Read it part by part before you build it.

**Definitions.** Lines 1 to 5. The `%{ ... %}` block is C that Flex copies to the top of its output, so `#include <stdio.h>` ends up where any C file wants it. The `%option` line switches off three things you do not need. `noyywrap` says "when the input ends, there is no further input to go on to"; without it the program would not link, for a reason section 4.5 explains. `nounput` and `noinput` leave out two helper functions, `yyunput` and `input`, that this program never calls; without them GCC warns that they are unused, as you will see in a moment.

**Rules.** Four of them. Each pattern is a regular expression in the notation of section 3.1: the identifier pattern of Tiny, then `[0-9]+` for a run of digits, then a run of blanks, tabs and newlines, then `.`, which in Lex means "any one character except a newline". Each action is a C block. The first two print what they found; the blank rule has an empty action, so blanks are read and thrown away; the last rule catches every other character.

**User code.** A `main` that calls `yylex` once and stops.

### Build it and run it

<!-- code: id=first-build -->
```console
$ flex first.l
$ ls
first.l  lex.yy.c
$ gcc -Wall -Wextra lex.yy.c -o first
$ echo 'pos = init + rate * 60;' | ./first
name   pos
other  =
name   init
other  +
name   rate
other  *
number 60
other  ;
```

`flex first.l` printed nothing, which means it found no fault in the specification. It wrote a C file called `lex.yy.c`; `ls` shows it beside your `.l` file. That name is fixed: whatever your specification is called, Flex writes `lex.yy.c` unless told otherwise. GCC compiled `lex.yy.c` with no warnings into the program `first`, named after its source as habit 2 asks, never `-o lex.yy.c`. Then the running statement went in on standard input and came out cut into pieces: three names, a number, and four other characters. The spaces vanished, because the blank rule swallowed them and printed nothing.

Run it again without `echo`, so that it reads from the keyboard. Type the statement, press Enter, then press Ctrl-D at the start of the next line to say "end of input":

<!-- code: id=first-keyboard stdin="x1 = 42;\n^D" -->
```console
$ ./first
x1 = 42;
name   x1
other  =
number 42
other  ;
```

Each line is answered as soon as you press Enter, and the program ends only when you press Ctrl-D. That tells you something about how it works: `yylex` does not read one line and stop. It keeps reading and matching until the input ends.

### What Flex wrote

`lex.yy.c` is an ordinary C file, and it is long:

<!-- code: id=first-inside -->
```console
$ wc -l lex.yy.c
1728 lex.yy.c
$ grep -n 'yy_accept\[\|yy_nxt\[\|define YY_DECL' lex.yy.c
366:static const flex_int16_t yy_accept[13] =
421:static const flex_int16_t yy_nxt[17] =
611:#define YY_DECL_IS_OURS 1
615:#define YY_DECL int yylex (void)
689:                    if ( yy_accept[yy_current_state] )
700:                    yy_current_state = yy_nxt[yy_base[yy_current_state] + yy_c];
706:            yy_act = yy_accept[yy_current_state];
711:                    yy_act = yy_accept[yy_current_state];
1042:           if ( yy_accept[yy_current_state] )
1053:           yy_current_state = yy_nxt[yy_base[yy_current_state] + yy_c];
1070:   if ( yy_accept[yy_current_state] )
1081:   yy_current_state = yy_nxt[yy_base[yy_current_state] + yy_c];
$ flex -v first.l 2>&1 | grep -E 'NFA states|DFA states'
  21/2000 NFA states
  11/1000 DFA states (37 words)
```

About seventeen hundred lines for your sixteen. Most of it is the same in every scanner Flex writes: buffer handling, the end-of-input logic, error messages. The parts that come from your rules are tables. `yy_nxt` is the transition table of chapter 3, squeezed to save space, and `yy_accept` says, for each DFA state, which rule (if any) accepts there. `YY_DECL` is the line that becomes the head of the scanner function: `int yylex (void)`.

The `-v` option makes Flex report what it built. From your four patterns it made a Thompson-style NFA (each pattern's NFA joined by ε-edges from one new start state, as in section 3.2), then ran the subset construction of section 3.3 to get a DFA, then minimised it. You did all three steps by hand for `(a|b)*abb`; Flex did them for four patterns at once, in the time it took to press Enter. The numbers before each slash are what it used, the numbers after are its limits.
<!-- risk: flex -v statistics format and the meaning of the counts (NFA/DFA states used against limits) are flex 2.6.4's -->

### What yylex does with your input

<!-- teaches: yylex = yylex -->
The function `yylex` is the scanner Flex wrote. Your `main` calls it once. Here is what happens inside, on the input `pos = init + rate * 60;` and a newline.

1. **The first call.** `yylex` notices that it has never run before. It sets its input to standard input (unless you chose a file, section 4.5) and reads a large block of input into a buffer: the buffer pair of section 2.6, managed for you.
2. **Run the board.** Starting at `p`, it walks the DFA: `p` is a letter, so the board moves to a square where the identifier rule accepts; `o`, still accepting; `s`, still accepting. The space has no arrow from this square. The walk stops.
3. **Back up to the last accepting square.** The walk remembers the last square where some rule accepted and where in the input that was. Here that is right after `s`, for rule 1. Had the walk gone past it into non-accepting squares before getting stuck, it would move the input back to that point: the pushing back that `ungetc` did in chapter 2, done for any number of characters.
4. **Set yytext and run the action.** The matched text, `pos`, becomes the lexeme, and rule 1's action runs: `printf("name   %s\n", yytext)`.
5. **Go round again.** The action did not say `return`, so `yylex` does not return. It starts a new walk at the space. The blank rule matches one space; its action does nothing. Then `=`: no pattern but `.` accepts, so rule 4 prints `other  =`. And so on to the `;` and the newline.
6. **The end.** At the end of the input `yylex` asks whether there is more input to go on to. `noyywrap` makes the answer no, so `yylex` returns 0 to `main`, and `main` returns.

Step 5 is the one people miss. `yylex` is a loop. It keeps matching until either an action executes a `return`, or the input ends. The first program never returns from an action, so one call reads everything. The Tiny scanner in section 4.4 does return, one token per call, which is exactly what a parser will want from chapter 6 on.

<!-- teaches: yytext = yytext -->
And what is `yytext`? It is a `char *` that points to the matched lexeme inside Flex's buffer. To make it usable as an ordinary C string, Flex saves the character just after the lexeme (here the space) and writes a `'\0'` over it while your action runs, then puts the saved character back before the next walk. Two consequences follow, and both appear in exams. First, `yytext` holds the lexeme only during the action; by the next match it points somewhere else, so if you want to keep a lexeme you copy it (`strdup(yytext)`, or `strcpy` into your own array). Second, Flex also sets `yyleng` to the lexeme's length, so you never need `strlen(yytext)`; section 4.3 counts characters with it.

One more thing hides in step 5. What if no rule at all matches a character? `first.l` cannot have that problem, because `.` and the blank rule together cover every character. Without them, Flex adds a rule of its own at the very end, the *default rule*: copy the unmatched character to the output unchanged. Section 4.2 shows it in action.

### When it goes wrong

**You changed `first.l`, rebuilt, and nothing changed.** You ran `gcc` without running `flex` first, so GCC compiled the old `lex.yy.c`. The `.l` file is never compiled itself; every change to it needs both commands again, in order. The same trap waits in the next sections: `lex.yy.c` always belongs to the specification you gave Flex last, so after `flex tiny.l` it is the Tiny scanner, not `first`, whatever you meant.

**`gcc first.l` gives pages of errors.** GCC is reading the specification as if it were C. Give GCC `lex.yy.c`.

**GCC warns about `yyunput` and `input`.** Here is what the same file prints without `nounput noinput`. `sed` copies `first.l` into a new file with those two words deleted, so the original stays as it is:

<!-- code: id=first-loud -->
```console
$ sed 's/ nounput noinput//' first.l > loud.l
$ flex loud.l
$ gcc -Wall -Wextra lex.yy.c -o loud
lex.yy.c:1132:16: warning: ‘input’ defined but not used [-Wunused-function]
 1132 | #else
      |                ^
lex.yy.c:1089:17: warning: ‘yyunput’ defined but not used [-Wunused-function]
 1089 |
      |                 ^
```

Two warnings, both about functions that Flex wrote and nobody calls. They are harmless, but habit 1 says a warning is a bug until you have dealt with it, and an examiner who sees a screen of warnings will ask about them. The cure is the option line you already have: `%option noyywrap nounput noinput` heads every program in this book. The line numbers point into `lex.yy.c`, not into your file, which is how you know the fault is not in your rules.

**The program sits there doing nothing.** It is waiting for standard input from the keyboard. Type something, or press Ctrl-D, or give it input with `echo ... |` or `< file`.

### In one line each

1. A `.l` file has three parts: definitions, rules, user code, separated by `%%`.
2. `flex file.l` writes the scanner as `lex.yy.c`; `gcc -Wall -Wextra lex.yy.c -o name` builds it; edit, then both again.
3. `%option noyywrap nounput noinput` lets the program link without a library and build without warnings.
4. `yylex` loops: walk the DFA as far as it goes, back up to the last accepting point, set `yytext` and `yyleng`, run the action; it returns only when an action returns or the input ends.
5. `yytext` points into Flex's buffer and is valid only during the action; copy it to keep it.

**Check yourself**

<!-- q: idea=yylex type=predict wrong="pos60 is a name then a number" -->
1. Predict what `first` prints for the input `rate2=pos60;` (no spaces), then for `2rate`.

<details>
<summary>Answer</summary>

For `rate2=pos60;`: `name   rate2`, `other  =`, `name   pos60`, `other  ;`. The identifier pattern allows digits after the first letter, and the walk goes on as far as it can, so `rate2` and `pos60` are each one name. The tempting answer splits `pos60` into a name and a number; that would need the walk to stop at the first digit, but the identifier square has an arrow on digits.

For `2rate`: `number 2`, then `name   rate`. At `2` only the number rule can start; it accepts `2`, and the `r` has no arrow from the number square, so the walk stops there. The next walk begins at `r`. Lex has no rule here that says "a number must not run into letters"; section 4.4 adds one.

</details>

<!-- q: idea=lex-structure type=fix wrong="flex is broken" -->
2. A student edits the number rule of `first.l` to print `num` instead of `number`, runs `gcc -Wall -Wextra lex.yy.c -o first` and then `./first`, and still sees `number`. What did they forget, and what are the two commands to type?

<details>
<summary>Answer</summary>

They forgot to run Flex again. The change is in `first.l`, but GCC compiled the `lex.yy.c` that Flex wrote from the old version. Type `flex first.l` and then `gcc -Wall -Wextra lex.yy.c -o first`.

</details>

<!-- q: idea=yytext type=explain wrong="yytext is a copy of the lexeme" -->
3. A program saves `char *last = yytext;` in the identifier action and prints `last` after `yylex` returns. Why might it print rubbish or the wrong text, and what should it do instead?

<details>
<summary>Answer</summary>

`yytext` is a pointer into Flex's input buffer, not a copy. After the action, Flex puts back the character it had replaced with `'\0'` and moves on, so `last` points at text that no longer ends where the lexeme ended, or at a part of the buffer that has since been refilled. Copy the lexeme during the action: `strcpy(saved, yytext)` into a big enough array, or `strdup(yytext)`.

</details>

<!-- q: idea=lex-structure type=decide wrong="put main in the rules section" -->
4. You need a counter that both an action and `main` use. In which part of the `.l` file do you declare `int count;`, and why not in the rules section?

<details>
<summary>Answer</summary>

In the definitions section, inside `%{ ... %}`. That C is copied to the top of `lex.yy.c`, before `yylex` and before your `main`, so both can see the variable. Lines in the rules section are read as patterns and actions; code there ends up inside `yylex` (if indented) or is taken as a pattern (if not), and `main` could not see it. The user code section would also work for `main`, but not for the actions, because it comes after `yylex` in the file.

</details>

## 4.2 Patterns and actions: longest match first, then the earlier rule

### Two patterns, one lexeme

In `first.l` every piece of input could match only one pattern, so there was never a question of which action to run. Tiny is not so kind. The lexeme `while` matches the identifier pattern and is also the keyword `while`. The characters `<=` could be one token, "less or equal", or `<` followed by `=`. Chapter 2 answered these questions with code: section 2.2 scanned a whole name and then looked it up in a keyword table, and section 2.3 wrote the helper `two` to peek at the character after `<`. In a Lex specification there is no such code. You simply write a rule for `while` and a rule for identifiers, a rule for `<` and a rule for `<=`, and Flex decides. You need to know exactly how it decides, because the examiner will ask, and because one wrong order of rules silently breaks a scanner.

<!-- teaches: lex-rule-order = rule order, earlier rule -->
Flex uses two rules, always in this order:

1. **Longest match.** Of all the patterns, the one that matches the longest piece of input, starting at the current position, wins.
2. **Earlier rule.** If two or more patterns match pieces of the same, longest, length, the one written first in the specification wins.

That is the whole of it. This section shows both rules at work on real input, shows how the DFA that Flex builds carries them out, and then shows what happens when you put your rules in the wrong order.

### The squares remember who they belong to

Go back to the board game of chapter 3. Section 3.3 built a DFA from an NFA by the subset construction: each square of the DFA is a set of NFA states. Flex joins the NFAs of all your patterns under one new start state, so a DFA square can contain the accepting states of several patterns at once. After reading `while`, the square you stand on contains the accepting state of the `"while"` pattern and the accepting state of the identifier pattern. Flex has to write one rule number on that square, and it writes the smaller one: the rule that comes first in the file. That is the "earlier rule" tie-break, and it is settled once, when Flex builds the tables, not while the scanner runs.

The longest-match rule lives in the scanner loop you met in section 4.1: walk the board as far as there are arrows, remembering the last accepting square passed, then back up to it. A walk that goes past `while` into `whiley` passes the square for `while` (keyword and identifier both accept there) and goes on to a square where only the identifier accepts. The last accepting square is the one after `y`, so the identifier wins, with the longer lexeme.

The picture of a player who backs up to the last good square is new to the board game. The DFA recogniser of section 3.6 read a whole line and answered at the end; it never had to choose where a lexeme stops. A scanner must cut a long input into many lexemes, so it keeps the extra piece of memory: where the last accepting square was.

### The notation of Lex patterns

<!-- teaches: lex-pattern = Lex pattern, default rule -->
Lex patterns are the regular expressions of section 3.1 with some extra notation. Here is all you need for the lab:

| Pattern | Matches |
|---|---|
| `a` | the character `a` |
| `"/*"`, `"+"` | exactly the characters between the quotes; special characters inside lose their meaning |
| `\.`, `\+`, `\n`, `\t` | a special character taken literally, or a newline, or a tab |
| `[abc]`, `[a-z0-9_]` | one character from the class |
| `[^abc]` | one character not in the class; a newline counts as "not in the class" |
| `.` | any one character except a newline |
| `r*`, `r+`, `r?` | zero or more, one or more, zero or one `r` |
| `r{2,4}` | two to four `r` in a row |
| `r|s`, `(r)` | `r` or `s`; grouping |
| `{name}` | the regular expression given that name in the definitions section |
| `^r` | `r`, but only at the start of a line |
| `r$` | `r`, but only at the end of a line |
| `r/s` | `r`, but only when `s` follows; `s` is not part of the lexeme |
<!-- risk: in flex a negated class [^...] matches newline unless \n is listed; . does not match newline -->

Blanks end a pattern. The pattern `end if` is the pattern `end` followed by an action that starts with `if`; to match a blank, quote it (`"end if"`) or escape it (`end\ if`). Section 4.7 shows the strange message GCC gives when you forget.

The `{name}` form is how you keep long patterns readable. In the definitions section, a line with a name in the first column and a regular expression after it defines the name; in the rules section, `{name}` stands for that expression. This is the regular definition of section 3.1, written in Lex:

<!-- code: id=defs-sketch mode=fragment -->
```lex
letter   [A-Za-z_]
digit    [0-9]
%%
{letter}({letter}|{digit})*    { /* an identifier */ }
{digit}+(\.{digit}+)?          { /* a number */ }
```

Flex puts parentheses around a name's expression when it substitutes it, so `{digit}+` means `([0-9])+`, as you would hope. Section 4.4 writes Tiny's scanner this way.
<!-- risk: flex wraps {name} expansions in parentheses; AT&T lex (and flex -l) does not -->

### Actions

An action is C. It is either one statement on the same line as the pattern (`printf("x\n");`) or a block in braces, which may run over several lines as long as it starts on the pattern's line. Three special forms are worth knowing:

- An action that is just `;` or `{ }` does nothing: the lexeme is read and dropped, as `first.l` did with blanks.
- An action that is just `|` means "same action as the next rule". `"if" |` on one line and `"while" { printf("keyword\n"); }` on the next give both keywords the same action.
- `ECHO;` copies the lexeme to the output unchanged. It is what the default rule does with an unmatched character.

### Watching the rules decide

Save this as `rules.l`. It has rules for two keywords, identifiers, numbers and four relational operators, and on purpose nothing for blanks, newlines or any other character, so that you can see the default rule.

<!-- code: id=rules-write -->
```console
$ cat > rules.l <<'END'
%{
/* rules.l: which rule wins? */
#include <stdio.h>
%}
%option noyywrap nounput noinput
%%
"if"                 { printf("[IF]"); }
"while"              { printf("[WHILE]"); }
[a-z][a-z0-9]*       { printf("[id %s]", yytext); }
[0-9]+(\.[0-9]+)?    { printf("[num %s]", yytext); }
"<"                  { printf("[LT]"); }
"<="                 { printf("[LE]"); }
"="                  { printf("[ASSIGN]"); }
"=="                 { printf("[EQ]"); }
%%
int main(void)
{
    yylex();
    return 0;
}
END
```

Before you run it, predict the output for these three lines. Then build and check:

<!-- code: id=rules-run -->
```console
$ flex rules.l
$ gcc -Wall -Wextra lex.yy.c -o rules
$ printf 'if iffy while whiley\nx<=y x<y x==5 x=5.x\npos = init + rate * 60;\n' | ./rules
[IF] [id iffy] [WHILE] [id whiley]
[id x][LE][id y] [id x][LT][id y] [id x][EQ][num 5] [id x][ASSIGN][num 5].[id x]
[id pos] [ASSIGN] [id init] + [id rate] * [num 60];
```

Every bracket is a rule's action; everything outside brackets was copied by the default rule. The spaces, the newlines, the `+`, the `*`, the `;` and one `.` matched none of your rules, so they came out unchanged, and the output keeps the layout of the input.

### Walking the hard cases

Take the lexemes one at a time. In each table, "matches" lists every rule that could match a piece of the input starting at that point, with the length of that piece.

**`if` followed by a space.**

| Rule | Matches | Length |
|---|---|---|
| `"if"` (rule 1) | `if` | 2 |
| identifier (rule 3) | `if` | 2 |

A tie on length, so the earlier rule wins: `[IF]`. The walk reached the square after `f`, which holds both rules' accepting states, and Flex had labelled it rule 1.

**`iffy`.** Rule 1 matches `if`, length 2; the identifier rule matches `iffy`, length 4. The longest match wins outright, and the order of the rules does not matter: `[id iffy]`. The same happens to `whiley`. This is why you never need code like chapter 2's keyword table in a Lex scanner: a keyword rule only ever wins when the whole name is the keyword.

**`<=`.** `"<"` matches 1 character, `"<="` matches 2. Longest match: `[LE]`. In `x<y`, `"<="` does not match at all (the next character is `y`), so `"<"` wins. Notice that `"<"` is written before `"<="` in the file, and it does not matter: order only breaks ties of equal length. The helper `two` from section 2.3 has become one extra line.

**`x==5`.** `"=="` (length 2) beats `"="` (length 1). In `x=5.x`, only `"="` matches.

**`5.x`**, the case that needs backing up. At `5` the number pattern accepts. The walk goes on: `.` has an arrow, because the pattern allows a fraction, so the walk moves to a square that is *not* accepting (it is waiting for a digit). Then `x`: no arrow. The walk is stuck on a non-accepting square. It backs up to the last accepting square, after `5`, and gives back the `.` it had read. The number rule runs with `yytext` = `5`. The next walk starts at `.`: no rule matches it, so the default rule copies it. Then `x` is an identifier. Hence `[num 5].[id x]`.

That last walk is exactly what `tiny-scan.c` had to code by hand for `5.` in section 2.3, and it is the reason Flex keeps the last accepting position. You can ask Flex to list the squares from which it may have to back up: the option `-b` (for "backing up") makes it write them to a file called `lex.backup`:

<!-- code: id=rules-backup -->
```console
$ flex -b rules.l
$ cat lex.backup
State #11 is non-accepting -
 associated rule line numbers:
        10
 out-transitions: [ 0-9 ]
 jam-transitions: EOF [ \000-/  :-\377 ]

Compressed tables always back up.
```

One square, non-accepting, belonging to the rule on line 10 of `rules.l` (the number rule), with arrows out only on digits: the square after `5.`. Backing up costs a little time, and large production scanners are sometimes tuned to have none, but for every scanner in this book it does not matter.

### Change one thing: the identifier rule first

Now the mistake that every class makes once. Put the general rule above the keyword. Save this as `order.l` and build it:

<!-- code: id=order-run -->
```console
$ cat > order.l <<'END'
%{
/* order.l: the general rule before the keyword */
#include <stdio.h>
%}
%option noyywrap nounput noinput
%%
[a-z][a-z0-9]*   { printf("[id %s]", yytext); }
"if"             { printf("[IF]"); }
%%
int main(void)
{
    yylex();
    return 0;
}
END
$ flex order.l
order.l:8: warning, rule cannot be matched
$ gcc -Wall -Wextra lex.yy.c -o order
$ echo 'if iffy' | ./order
[id if] [id iffy]
```

Flex warned, `rule cannot be matched`, with the line number of the `"if"` rule, and still wrote the scanner. Here is why. Any input that `"if"` matches is the two characters `if`, and the identifier rule matches those same two characters, so there is always a tie, and the tie always goes to the identifier rule, which is now first. No input exists on which `"if"` wins. Flex works this out when it builds the tables: no DFA square is labelled with rule 2. The program runs, and `if` comes out as an identifier.

Take this warning seriously; it is the only sign you get. In an exam, a scanner that calls every keyword an identifier looks as if it works until the examiner types a keyword.

### Why Flex decides this way

Longest match is what a programmer means. Nobody writing `iffy` means the keyword `if` followed by `fy`, and nobody writing `<=` means two operators. Chapter 2 applied the same rule by hand in every branch of `gettoken`; Lex makes it the law for every pattern.

The earlier-rule tie-break gives you control in the one case where longest match cannot decide: when one pattern is a special case of another, as a keyword is a special case of a name. Writing the special case first is short and clear. It has a cost: every keyword adds squares to the DFA. For Tiny's two keywords, or C's thirty-two, that does not matter. A compiler for a language with hundreds of keywords may instead match every name with one rule and look the lexeme up in a table inside the action, the way section 2.2 did. Both are correct answers in the lab; the keyword rules are the usual one.

### When it goes wrong

**Every keyword comes out as an identifier.** The identifier rule is above the keyword rules. Flex said `rule cannot be matched` when you built it. Move the keywords up.

**Blank lines or stray characters appear in the output.** The default rule is copying characters that no rule matched. Often it is the newline: `.` does not match `\n`, so a catch-all written as `.` lets every newline through. Use `.|\n`, or a separate `\n` rule. If you want to be told instead, `%option nodefault` makes an unmatched character an error; section 4.7 shows what that looks like.

**`unrecognized rule` on a line that holds `+` or `*`.** Those characters are operators in a pattern, so a pattern that is only `+` makes no sense. Quote it, `"+"`, or escape it, `\+`.

**A rule meant for `.` matches every character.** An unquoted `.` means "any character". For a literal full stop, write `"."` or `\.`.

### In one line each

1. The longest match wins; among equally long matches, the rule written first wins.
2. Longest match is done while scanning, by remembering the last accepting square and backing up to it.
3. The tie-break is done when Flex builds the DFA: a square that accepts for several rules is labelled with the first.
4. Put keywords and other special cases above the general pattern; `rule cannot be matched` means a rule is hidden behind an earlier one.
5. Characters no rule matches are copied to the output by the default rule.

**Check yourself**

<!-- q: idea=lex-rule-order type=predict wrong="whiles is the keyword while then s" -->
1. Predict what `rules` prints for `whiles<==5.25`.

<details>
<summary>Answer</summary>

`[id whiles][LE][ASSIGN][num 5.25]`. `whiles` is longer than `while`, so the identifier wins. At `<`, `"<="` matches 2 characters and wins over `"<"`. That leaves `=5.25`: `"=="` does not match (the next character is `5`), so `"="` wins. `5.25` is a complete number, and the walk accepts after the last `5`. The tempting answer `[LE][EQ]` takes `<==` as `<` followed by `==`, but the scanner cuts from the left, taking the longest match at each point, and never looks back to find a better split.

</details>

<!-- q: idea=lex-pattern type=fix wrong="the number rule is wrong" -->
2. A student's scanner prints a blank line after every line of output, although no action prints `\n`. The last rule is `.  { }`. What is happening, and what one change fixes it?

<details>
<summary>Answer</summary>

`.` does not match a newline, so each newline matches no rule and the default rule copies it to the output. Change the last rule to `.|\n  { }` (or add a rule `\n  { }`).

</details>

<!-- q: idea=lex-rule-order type=decide wrong="order does not matter for operators" -->
3. Does the order of `"="` and `"=="` in `rules.l` matter? Does the order of `"while"` and the identifier rule? Say why for each.

<details>
<summary>Answer</summary>

`"="` and `"=="` never tie, because one matches 1 character and the other 2; longest match decides, so their order does not matter. `"while"` and the identifier rule tie on the lexeme `while` (both length 5), so their order decides, and `"while"` must come first.

</details>

<!-- q: idea=lex-pattern type=apply wrong="[0-9]+.[0-9]+ matches only decimals" -->
4. Write a pattern for a decimal number with digits on both sides of the point, and say what is wrong with `[0-9]+.[0-9]+`.

<details>
<summary>Answer</summary>

`[0-9]+\.[0-9]+` (or `[0-9]+"."[0-9]+`). Unescaped, `.` means any character except a newline, so `[0-9]+.[0-9]+` also matches `12x34` and `5+3`, and in a scanner with a separate `"+"` rule it would swallow `5+3` as one "number", because that is the longer match.

</details>

## 4.3 Counting programs: lines, words, characters, vowels and consonants

### The first question on the lab sheet

Nearly every lab sheet opens its Lex part with a counting program: "Write a Lex program to count the number of lines, words and characters in the input", then "to count the vowels and consonants in a given string". They look like toys, and they are short, but the examiner uses them to see whether you understand three things that every larger Lex program depends on: where variables shared by actions and `main` must live, when a total is final, and what `yytext` and `yyleng` hold at the moment an action runs. You also get a free referee for the first one: the Linux command `wc` counts lines, words and characters, so you can check your program against it on any file.

### A tally counter for each kind of thing

Think of a person at a door with a mechanical tally counter in each hand, clicking one for every adult and the other for every child. They do not write down who came in; they only click. At closing time they read the counters off. A counting Lex program is that person: each rule is a kind of visitor, each action clicks a counter, and `main` reads the totals once, when the input has closed.

The picture is exact except in one place. A doorkeeper clicks once per visitor. An action can add any amount, and for characters it should: one rule may match a whole word, and the character counter must go up by the length of that word, not by one. That length is `yyleng`.

### Where yytext and yyleng come from

<!-- teaches: yyleng = yyleng, match length -->
Section 4.1 said that Flex sets `yytext` to the lexeme and puts a `'\0'` after it while the action runs. You can see the four lines that do it in the generated file; `grep -A5` prints the line that matches and the five lines after it. Flex runs them after every match, just before your action:

<!-- code: id=count-inside -->
```console
$ flex first.l
$ grep -A5 'define YY_DO_BEFORE_ACTION' lex.yy.c
#define YY_DO_BEFORE_ACTION \
        (yytext_ptr) = yy_bp; \
        yyleng = (int) (yy_cp - yy_bp); \
        (yy_hold_char) = *yy_cp; \
        *yy_cp = '\0'; \
        (yy_c_buf_p) = yy_cp;
```

`yy_bp` points at the first character of the match and `yy_cp` just past its last character. So `yytext` is the start of the match, and `yyleng` is the distance between the two pointers: the number of characters matched, worked out by one subtraction. Then the character at `yy_cp` is saved in `yy_hold_char` and replaced by `'\0'`, which is what turns the lexeme into a C string. After your action, Flex writes the saved character back.

So `yyleng` is always equal to `strlen(yytext)`, but it costs nothing, while `strlen` would walk the lexeme again. Use `yyleng` whenever you need the length. (The two would differ only on input that contains a `'\0'` byte, which text files do not.)

That `flex first.l` also replaced `lex.yy.c`: it is the `first` scanner again until the next `flex` command, which is the next program.

### Lines, words and characters

<!-- teaches: lex-counting = counting program, counters in the definitions section -->
What is a word? Take `wc`'s answer, so that the two programs can be compared: a word is a run of characters that are not blanks, tabs or newlines. So `rate*60;` is one word, and so is `10;`. A line is counted at each newline. Every character counts as a character, newlines and blanks included.

The program also remembers the longest word, which is a common second part of the question ("and print the longest word") and the reason `yyleng` matters here. Save it as `count.l`:

<!-- code: id=count-write -->
```console
$ cat > count.l <<'END'
%{
/* count.l: lines, words and characters, like wc, and the longest word */
#include <stdio.h>
int lines, words, chars;   /* the tally counters */
int longest;               /* length of the longest word so far */
char longword[100];        /* the longest word itself */
%}
%option noyywrap nounput noinput
%%
\n          { lines++; chars++; }
[ \t]+      { chars += yyleng; }
[^ \t\n]+   { words++;
              chars += yyleng;
              if (yyleng > longest) {
                  longest = yyleng;
                  snprintf(longword, sizeof longword, "%s", yytext);
              } }
%%
int main(void)
{
    yylex();
    printf("lines %d, words %d, characters %d\n", lines, words, chars);
    if (words > 0)
        printf("longest word: %s (%d characters)\n", longword, longest);
    return 0;
}
END
```

The counters are declared in the `%{ ... %}` block of the definitions section. That block is copied to the top of `lex.yy.c`, above both `yylex` and your `main`, so every action and `main` can use them. As global variables they start at zero without being set.

The three rules cover every character exactly once: the newline, runs of blanks and tabs, and runs of anything else. No character can reach the default rule, so nothing is copied to the output. `[^ \t\n]+` is a negated class: one or more characters that are not a blank, a tab or a newline.

The longest word is copied with `snprintf` into the program's own array, because `yytext` will point somewhere else after the action (section 4.1). The `sizeof longword` keeps a monstrous word from running past the array: `snprintf` stops writing at 99 characters and still ends the string.

`main` prints the totals after `yylex` returns, because that is the only moment they are final: `yylex` returns when the input ends.

### Walking a small input

Here is a two-line input, `x = 10;` and then a tab followed by `pos`. Follow the rules and the counters after each match:

| Input matched | Rule | `yyleng` | lines | words | chars | longest word |
|---|---|---|---|---|---|---|
| `x` | word | 1 | 0 | 1 | 1 | `x` (1) |
| blank | blanks | 1 | 0 | 1 | 2 | `x` |
| `=` | word | 1 | 0 | 2 | 3 | `x` |
| blank | blanks | 1 | 0 | 2 | 4 | `x` |
| `10;` | word | 3 | 0 | 3 | 7 | `10;` (3) |
| newline | newline | 1 | 1 | 3 | 8 | `10;` |
| tab | blanks | 1 | 1 | 3 | 9 | `10;` |
| `pos` | word | 3 | 1 | 4 | 12 | `10;`, since 3 is not more than 3 |
| newline | newline | 1 | 2 | 4 | 13 | `10;` |

Two things to notice. The word `10;` is one match of length 3, so `chars` jumps from 4 to 7 in a single action: that is the counter going up by more than one. And at `pos` the test `yyleng > longest` is false, because the lengths are equal, so the first of two equally long words is kept.

Now check the table, and then compare the program with `wc` on the Tiny program from chapter 2:

<!-- code: id=count-run -->
```console
$ flex count.l
$ gcc -Wall -Wextra lex.yy.c -o count
$ printf 'x = 10;\n\tpos\n' | ./count
lines 2, words 4, characters 13
longest word: 10; (3 characters)
$ ./count < ../ch2/prog.tiny
lines 7, words 37, characters 162
longest word: statement, (10 characters)
$ wc ../ch2/prog.tiny
  7  37 162 ../ch2/prog.tiny
$ ./count < /dev/null
lines 0, words 0, characters 0
```

The first run agrees with the table: 2 lines, 4 words, 13 characters, longest word `10;`. On `prog.tiny` the program and `wc` agree: 7 lines, 37 words, 162 characters (`wc` prints them in that order, then the file name). The longest word is `statement,` from the comment on line 1, comma included, because to `wc` and to this program a word is anything between blanks. The empty input gives zeros and no "longest word" line, which is what the `if (words > 0)` is for; without it the program would print an empty word of length 0.

`wc -c` counts bytes, not characters. For a file of plain ASCII, like every file in this book, the two are the same.
<!-- risk: wc counts bytes for -c; the equality with this program holds for ASCII input only -->

### Vowels and consonants

The second counting program needs one idea from section 4.2. A vowel is a letter, so the vowel pattern and the letter pattern both match a vowel, with the same length, 1. The tie goes to the earlier rule, so the vowel rule must come first. Save it as `vowels.l`:

<!-- code: id=vowels-write -->
```console
$ cat > vowels.l <<'END'
%{
/* vowels.l: count the vowels and consonants in the input */
#include <stdio.h>
int vowels, consonants;
%}
%option noyywrap nounput noinput
%%
[aeiouAEIOU]   { vowels++; }
[a-zA-Z]       { consonants++; }
.|\n           { /* digits, blanks, punctuation: not counted */ }
%%
int main(void)
{
    yylex();
    printf("vowels %d, consonants %d\n", vowels, consonants);
    return 0;
}
END
$ flex vowels.l
$ gcc -Wall -Wextra lex.yy.c -o vowels
$ echo 'pos = init + rate * 60;' | ./vowels
vowels 5, consonants 6
$ echo 'RHYTHM' | ./vowels
vowels 0, consonants 6
```

In the running statement the vowels are the `o` of `pos`, the two `i`s of `init`, and the `a` and `e` of `rate`: five. The consonants are `p`, `s`, `n`, `t`, `r`, `t`: six. The digits, blanks, operators and the newline all go to the third rule and are not counted. `RHYTHM` has no vowel in the program's sense, since `Y` is not in the class: six consonants. If the question says `y` is a vowel, it goes in the class; read the question.

Here the third rule matters more than it looks. Without it, the default rule would copy every digit, blank and operator to the output, and the answer line would arrive after a copy of the input with the letters cut out.

### Change one thing: count in the wrong place

A tempting shortcut is to print the counts from inside an action, say from the newline rule, so that you do not need to touch `main`. Then the program prints a line of running totals after each input line, and the "answer" is whichever line happens to be last. Worse, if the last line of the input has no newline, the last letters are never reported at all. The counts are final only when the input is: print them after `yylex` returns.

The other shortcut is to count characters with `chars++` in every rule. That is right for the newline rule, whose lexeme is always one character, and wrong for the word and blank rules, whose lexemes may be longer. On `prog.tiny` it would report one character per match, 76, instead of 162. Whenever a pattern can match more than one character, add `yyleng`.

### When it goes wrong

**The counts are all zero.** `main` never called `yylex`, or printed before calling it. The order in `main` is always: `yylex();` then the `printf`.

**The word count is far higher than `wc`'s.** The word pattern lost its `+`: `[^ \t\n]` matches one character at a time, so every character is counted as a word.

**The vowel count is zero and Flex said `rule cannot be matched`.** The letter rule is above the vowel rule, so every vowel ties and goes to the letter rule. Swap the two rules.

**`error: ‘count’ undeclared`.** The counter was declared in the user code section, below `yylex`, or inside `main`. Actions are inside `yylex`, which is near the top of `lex.yy.c`; declare shared variables in the `%{ ... %}` block.

### In one line each

1. Declare counters in the `%{ ... %}` block, so that actions and `main` can both use them.
2. Each action clicks its counter; for characters, add `yyleng`, the length of the match, worked out by Flex as the difference of two pointers.
3. Print totals after `yylex` returns; that is when the input has ended.
4. Make the rules cover every character, or the default rule will echo the rest.
5. `wc` checks a line, word and character counter for free.

**Check yourself**

<!-- q: idea=lex-counting type=predict wrong="the last line is counted" -->
1. Predict what `count` prints for `printf 'a b\nc'`, with no newline after `c`, and what `wc` prints for the same input. Are they still equal?

<details>
<summary>Answer</summary>

`lines 1, words 3, characters 5` and `longest word: a (1 characters)`. `wc` prints `1 3 5`. They agree: both count newlines, not lines, so a last line with no newline adds no line, and both still count its word and its character. The tempting answer is 2 lines, because you can see two lines of text.

</details>

<!-- q: idea=yyleng type=fix wrong="yyleng counts words" -->
2. A student's character count for `prog.tiny` is 76, not 162. Their blank rule is `[ \t]+ { chars++; }` and their word rule is `[^ \t\n]+ { words++; chars++; }`. Explain the number, then fix it.

<details>
<summary>Answer</summary>

Each match adds one character, whatever its length, so `chars` ends up as the number of matches: 37 words plus 7 newlines plus 32 runs of blanks, 76. Change both actions to add `yyleng` (`chars += yyleng;`). Only the newline rule may use `chars++`, because its lexeme is always exactly one character.
<!-- risk: the count 76 came from a scratch run of count.l with chars++ in every rule, not pasted in the chapter -->

</details>

<!-- q: idea=lex-rule-order type=predict wrong="the vowels are counted as consonants half the time" -->
3. In `vowels.l` the two rules are swapped: `[a-zA-Z]` first, then `[aeiouAEIOU]`. What does Flex print when building it, and what does the program print for the running statement?

<details>
<summary>Answer</summary>

Flex prints a `rule cannot be matched` warning for the vowel rule's line. The program prints `vowels 0, consonants 11`: every vowel matches both patterns with length 1, and the tie goes to the letter rule, now written first.
<!-- risk: swapped vowels.l run in a scratch folder on the build machine, not in the chapter -->

</details>

<!-- q: idea=lex-counting type=apply wrong="needs a second program" -->
4. Change `count.l` so that it also counts blank lines (lines with nothing but blanks or tabs on them). Which one pattern would you add, and where?

<details>
<summary>Answer</summary>

Add a rule `^[ \t]*\n { blank++; lines++; chars += yyleng; }` above the plain newline rule, and declare `int blank;` in the `%{ ... %}` block. At the start of a blank line it matches the blanks and the newline together, which is longer than either the blank rule or the newline rule alone, so longest match picks it. On an empty line it ties with the newline rule at length 1, and being written first it wins. Print `blank` in `main`.

</details>

## 4.4 A Lex scanner for keywords, identifiers, numbers and operators

### The same scanner, a fifth of the size

This is the program the lab asks for more than any other: "Write a Lex program to recognise the keywords, identifiers, numbers and operators of a small language." For this book it is also the second file of the mini compiler. In section 2.7 you finished `tiny-scan.c`, about a hundred and fifty lines of C, and tested it on `prog.tiny` and `bad.tiny`. Now you write the same scanner in Lex, as `tiny.l`, and you hold it to a hard standard: on both test files it must print exactly what `tiny-scan` prints, token for token, error message for error message. `diff` will be the judge.

The specification needs nothing you have not seen. Section 4.2 settles every choice between rules; regular definitions keep the patterns readable; and two features are new: a rule whose action *returns* a token, and Flex's own line counter.

### Station for station

On the assembly line of chapter 1, the scanner station hands one token at a time to the parser. `tiny-scan.c` did that through `gettoken`, and its `main` stood in for the parser, asking for tokens until `END`. `tiny.l` keeps that shape exactly: `yylex` plays `gettoken`, and `main` plays the parser. The station is built differently, but the product handed on is the same, which is why the two programs can be compared with `diff`.

Where the picture of identical stations breaks: `gettoken` returned a token and left the lexeme in the global array `lexeme`. `yylex` returns a token and leaves the lexeme in `yytext`, which, as you know from section 4.1, is only a pointer into Flex's buffer. The caller must use it, or copy it, before asking for the next token. `main` below prints it at once, so that is fine; a parser that keeps lexemes will need copies, which chapter 8 deals with.

### Returning one token per call

<!-- teaches: lex-token-scanner = token scanner in Lex, return in an action, yylineno -->
So far no action has contained `return`, so `yylex` ran to the end of the input in one call. An action may say `return IF;`. Then `yylex` stops at once and returns `IF` to its caller, with `yytext` still holding `if`. The next call of `yylex` starts a new walk exactly where the last one stopped. Nothing is lost between calls: the buffer and the position are kept in Flex's own static variables. That is how a Lex scanner hands over one token at a time.

At the end of the input `yylex` returns 0. So the token codes must keep 0 for "end", and every real token must have a different, non-zero code. In `tiny-scan.c`, `ID` was 0 and `END` was 19. In `tiny.l` the list starts with `END`, so `END` is 0 and `ID` is 1; that one change makes the loop `while ((t = yylex()) != END)` stop at exactly the right moment. Get this wrong and the scanner stops at the first identifier, because returning `ID` would look like the end of input.

`%option yylineno` asks Flex to keep the line number in the global `int yylineno`, starting at 1 and going up by one for every newline the scanner matches, whichever rule matched it. It replaces `next`, `back`, `line` and `tokline` from section 2.7. Because no Tiny token contains a newline, `yylineno` during a token's action is the line the token is on.
<!-- risk: flex's yylineno counts every matched newline, including those inside a multi-line match such as a block comment, and is not decremented on back-up (it only counts after a rule is chosen) -->

### tiny.l

Save it as `tiny.l`, in two pieces so that each fits on a screen. First the definitions and the rules:

<!-- code: id=tiny-write -->
```console
$ cat > tiny.l <<'END'
%{
/* tiny.l: the Flex scanner for Tiny; prints what tiny-scan prints */
#include <stdio.h>

enum { END, ID, NUM, IF, WHILE, ASSIGN, EQ, NE, LT, LE, GT, GE,
       PLUS, MINUS, TIMES, DIVIDE, LPAREN, RPAREN, SEMI };

const char *name[] = { "end", "id", "num", "if", "while", "=", "==",
                       "!=", "<", "<=", ">", ">=", "+", "-", "*", "/",
                       "(", ")", ";" };
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
"/*"                           { error("comment not closed"); return END; }

"if"          { return IF; }
"while"       { return WHILE; }
{id}          { return ID; }
{num}         { return NUM; }
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
END
```

That is the definitions and the rules. The user code comes next, with `>>`, which adds to the end of the file instead of replacing it:

<!-- code: id=tiny-write-user -->
```console
$ cat >> tiny.l <<'END'
%%
void error(const char *msg)
{
    fprintf(stderr, "line %d: error: %s: %s\n", yylineno, msg, yytext);
    errors++;
}

int main(void)
{
    int t;

    while ((t = yylex()) != END)
        printf("%3d  %-6s %s\n", yylineno, name[t], yytext);
    if (errors > 0) {
        fprintf(stderr, "%d lexical error(s)\n", errors);
        return 1;
    }
    return 0;
}
END
```

### Reading tiny.l for the viva

**The definitions section.** The `%{ ... %}` block holds the token codes, the names printed for them (in the same order, so `name[t]` is the name of token `t`), the error counter and a declaration of `error`, which is defined in the user code section but used in actions above it. Then the options, now with `yylineno`. Then four regular definitions: `letter` includes the underscore, as Tiny's identifiers do; `id` and `num` are built from the two before them, exactly as the regular definitions of section 3.1 were.

**Blanks and comments.** The first four rules return nothing, so `yylex` keeps going after them, which is how blanks and comments disappear from the token stream. `"//".*` is a line comment: two slashes, then anything up to (not including) the newline, which the blank rule then eats. The block comment pattern reads: `/*`, then any number of pieces that are either a non-star character, or one or more stars followed by a character that is neither a star nor a slash, then one or more stars, then `/`. It is the DFA of section 2.4's comment loop, written as one expression, and it handles `/* rate **/` and `/**/`. The fourth rule, `"/*"` alone, is the unclosed comment: if the comment is closed, the long pattern matches far more than two characters and wins; if it never closes, the long pattern cannot match at all, and only the two-character rule is left. It reports the error and returns `END`, so scanning stops, as `tiny-scan` did. Section 4.6 shows a clearer way to write comments.

**Keywords, names and numbers.** The keywords come before `{id}`, as section 4.2 requires. The two error patterns for numbers win by longest match. On `5.` followed by a blank, `{num}` matches only `5`, while `{digit}+"."` matches `5.`: two characters beat one, so the error action runs with `yytext` = `5.`. On `9lives`, `{num}{id}` matches all six characters and beats `{num}`'s one. On a good number such as `2.5`, `{num}` matches three characters and `{digit}+"."` only two, so the good rule wins. These two rules are section 2.3's error checks, done by longest match instead of by `if` statements.

**Operators.** Two-character operators win over their one-character prefixes by longest match, whatever the order; they are written first only so that a reader sees them first. `"!"` alone is an error, because Tiny has `!=` but no `!`.

**The catch-all.** `.` matches any other character and reports it. Because no action in an error rule returns, `yylex` simply goes on with the next character, which is the panic-mode recovery of section 2.7: drop the bad lexeme, carry on.

**`error` and `main`.** `error` prints the line, the message and the lexeme to `stderr`, word for word as in `tiny-scan.c`. `main` is `tiny-scan.c`'s `main` with `gettoken()` replaced by `yylex()`, `tokline` by `yylineno` and `lexeme` by `yytext`.

### Build it and hold it to tiny-scan

<!-- code: id=tiny-run -->
```console
$ flex tiny.l
$ gcc -Wall -Wextra lex.yy.c -o tiny
$ ./tiny < ../ch2/prog.tiny
  2  id     init
  2  =      =
  2  num    10
  2  ;      ;
  3  id     rate
  3  =      =
  3  num    2.5
  3  ;      ;
  4  id     pos
  4  =      =
  4  id     init
  5  +      +
  5  id     rate
  5  *      *
  5  num    60
  5  ;      ;
  6  while  while
  6  (      (
  6  id     pos
  6  >=     >=
  6  num    100
  6  )      )
  7  id     pos
  7  =      =
  7  id     pos
  7  -      -
  7  num    7.25
  7  /      /
  7  (      (
  7  id     rate
  7  !=     !=
  7  num    0
  7  )      )
  7  ;      ;
```

Compare that with section 2.7's run of `tiny-scan`: the comment on line 1 produced nothing, `+` is on line 5 although the newline after `init` ended a line with a name, `while` is a keyword, and `>=`, `!=` and `7.25` are each one token. Rather than compare by eye, let `diff` do it. `../ch2/tiny-scan` runs the chapter 2 program from its own folder (`..` is the folder above this one), on the same file. It prints nothing when two files are the same, and `&& echo same` prints `same` only if `diff` found no difference:

<!-- code: id=tiny-diff -->
```console
$ ../ch2/tiny-scan < ../ch2/prog.tiny > hand.out
$ ./tiny < ../ch2/prog.tiny > flex.out
$ diff hand.out flex.out && echo same
same
```

Now the empty input and the input that must be rejected, as habit 3 asks. `echo "exit $?"` shows the exit status of the command before it:

<!-- code: id=tiny-bad -->
```console
$ ./tiny < /dev/null; echo "exit $?"
exit 0
$ ./tiny < ../ch2/bad.tiny; echo "exit $?"
  1  id     x
  1  =      =
line 1: error: digit expected after '.': 5.
  1  ;      ;
  2  id     y
  2  =      =
line 2: error: letters after a number: 9lives
line 2: error: unexpected character: @
  2  num    2
  2  ;      ;
  3  id     z
  3  =      =
line 3: error: '!' without '=': !
  3  id     y
  3  ;      ;
line 3: error: comment not closed: /*
5 lexical error(s)
exit 1
```

The same five errors as `tiny-scan`, in the same places, with the same tokens scanned between them and the same exit status. Line 3 is worth following. `!` is reported, then `y` and `;` are scanned normally; then `/*` begins a comment that never ends. The long comment pattern reads to the end of the file looking for `*/`, finds none, and so does not match; the walk backs up to the last accepting point, which is the two-character rule, and line 4 is never scanned as code. Note that `yylineno` is still 3 at that moment: a newline the walk read and then gave back was never counted.

And the messages, compared separately from the tokens, because they go to a different stream:

<!-- code: id=tiny-bad-diff -->
```console
$ ../ch2/tiny-scan < ../ch2/bad.tiny > hand.out 2> hand.err; echo "exit $?"
exit 1
$ ./tiny < ../ch2/bad.tiny > flex.out 2> flex.err; echo "exit $?"
exit 1
$ diff hand.out flex.out && diff hand.err flex.err && echo same
same
```

Two programs, one written character by character and one generated from patterns, behave identically on all three tests. Keep `tiny.l`: in chapter 8 its actions return the same tokens to a parser that Yacc writes.

### Into the notebook: tokens with a symbol table

The other common wording of the question asks for the token stream itself, in the notation of section 1.3, with identifiers entered in a symbol table: `<id,1> <=> <id,2> <+> <id,3> <*> <num,60> <;>`. The `1`, `2`, `3` are entry numbers in the compiler's notebook (section 1.4). Section 2.8 built that notebook as a hash table; for a lab program of a few dozen names, an array searched from the top is enough, and it keeps the program short enough to write in an exam. Save this as `tokens.l`:

<!-- code: id=tokens-write -->
```console
$ cat > tokens.l <<'END'
%{
/* tokens.l: the token stream in chapter 1's notation, and the symbol table */
#include <stdio.h>
#include <string.h>

#define MAXNAMES 50
char names[MAXNAMES][32];   /* the notebook: one identifier per entry */
int nnames;                 /* entries used */

int install(const char *s)  /* the entry number of s; adds s if it is new */
{
    for (int i = 0; i < nnames; i++)
        if (strcmp(names[i], s) == 0)
            return i + 1;
    if (nnames == MAXNAMES) {
        fprintf(stderr, "symbol table full\n");
        return 0;
    }
    snprintf(names[nnames], sizeof names[nnames], "%s", s);
    return ++nnames;
}
%}
%option noyywrap nounput noinput

letter   [A-Za-z_]
digit    [0-9]

%%
"if"|"while"                      { printf("<%s> ", yytext); }
{letter}({letter}|{digit})*       { printf("<id,%d> ", install(yytext)); }
{digit}+(\.{digit}+)?             { printf("<num,%s> ", yytext); }
"=="|"!="|"<="|">="|[-+*/=<>();]  { printf("<%s> ", yytext); }
\n                                { putchar('\n'); }
[ \t]+                            { }
.                                 { printf("<error,%s> ", yytext); }
%%
int main(void)
{
    yylex();
    printf("symbol table:\n");
    for (int i = 0; i < nnames; i++)
        printf("%3d  %s\n", i + 1, names[i]);
    return 0;
}
END
$ flex tokens.l
$ gcc -Wall -Wextra lex.yy.c -o tokens
$ printf 'pos = init + rate * 60;\nwhile (pos >= 100) pos = pos - rate;\n' | ./tokens
<id,1> <=> <id,2> <+> <id,3> <*> <num,60> <;>
<while> <(> <id,1> <>=> <num,100> <)> <id,1> <=> <id,1> <-> <id,3> <;>
symbol table:
  1  pos
  2  init
  3  rate
```

The first output line is section 1.2's token stream, letter for letter. On the second line `pos` is `<id,1>` three times and `rate` is `<id,3>`: `install` found them in the notebook and gave back their old entry numbers, so the table at the end has only three entries. The program reaches the C function `install` straight from an action, because an action is ordinary C inside `yylex`, and `install` is defined above it in the `%{ ... %}` block.

Two patterns use new notation. `"if"|"while"` is one rule with a union, so both keywords share one action. `[-+*/=<>();]` is a class of nine single-character operators; the `-` is first in the class so that it is taken literally and not as a range.

To use section 2.8's hash table instead, replace the loop in `install` with that section's search and insert. The rest of the specification does not change: the scanner does not care how the notebook is kept.

### When it goes wrong

**The scanner stops after the first identifier.** `ID` is 0, so `main` takes the first `return ID;` for the end of input. Put `END` first in the `enum`.

**Every token is reported on line 1.** `%option yylineno` is missing; `yylineno` then stays at its starting value.

**`if` and `while` print as `id`.** The keyword rules are below `{id}`. Flex said `rule cannot be matched` twice when you built it.

**`5.` prints as `num 5` followed by an error for `.`.** The `{digit}+"."` rule is missing, so `{num}` matched `5` and the `.` went to the catch-all. This gives a different message from `tiny-scan`, and `diff` catches it.

**`undefined definition {ident}`.** A name in braces must be spelled exactly as in the definitions section.

### In one line each

1. An action that returns makes `yylex` return that token; the next call carries on from the same point.
2. `yylex` returns 0 at the end of input, so the end token must be 0 and no real token may be.
3. `%option yylineno` keeps the line number; it replaces the hand-written line counting of section 2.7.
4. Keywords above `{id}`; error patterns such as `{digit}+"."` and `{num}{id}` win by longest match.
5. An action can call any C function defined above it, such as `install` for the symbol table.

**Check yourself**

<!-- q: idea=lex-token-scanner type=predict wrong="rate2 is rate then 2" -->
1. Predict what `tiny` prints for the one line `while(rate2>=0)x=x-1.5e;`, including any messages.

<details>
<summary>Answer</summary>

```text
  1  while  while
  1  (      (
  1  id     rate2
  1  >=     >=
  1  num    0
  1  )      )
  1  id     x
  1  =      =
  1  id     x
  1  -      -
line 1: error: letters after a number: 1.5e
  1  ;      ;
1 lexical error(s)
```

`while(` has no blank, but `(` cannot continue a name, so `while` is complete and the keyword wins the tie with `{id}`. `rate2` is one identifier. `1.5e` is matched whole by `{num}{id}` (four characters, against three for `{num}`), so it is one error, not a number followed by a name. The exit status is 1.

</details>

<!-- q: idea=lex-token-scanner type=fix wrong="the identifier rule is wrong" -->
2. A classmate's version of `tiny.l` prints the right tokens for `prog.tiny`, and then, after the last `;`, an endless stream of lines `  8  id` until they press Ctrl-C (line 8, because the newline at the end of line 7 has been counted). Their `enum` reads `enum { ID, NUM, IF, ..., SEMI, END };`, their `name` array is in the same order (`"id"` first, `"end"` last), and their `main` is the one above. Explain the symptom and fix it.

<details>
<summary>Answer</summary>

In their `enum`, `ID` is 0 and `END` is 18. At the end of the input `yylex` returns 0, which their program takes for `ID`: it is not `END`, so the loop prints a line for a token called `id` and asks again, and `yylex` returns 0 again, for ever. Nothing ever returns 18, so the loop never ends. Put `END` first in the `enum`, so that 0 means the end and nothing else. The tempting answer blames the identifier rule, because `id` is what is printed; but the rule is never running at that point.
<!-- risk: the endless `  8  id` output came from a scratch run of the changed tiny.l, not pasted in the chapter -->

</details>

<!-- q: idea=lex-rule-order type=decide wrong="the unclosed-comment rule must come first" -->
3. In `tiny.l`, could the rule `"/*"` (the unclosed comment) be written above the long block-comment rule? Would anything change?

<details>
<summary>Answer</summary>

Nothing would change. The two rules only tie when they match the same length, and they never do: a closed comment is at least four characters (`/**/`), and the long pattern matches it whole, beating `"/*"`'s two by longest match. An unclosed comment is matched only by `"/*"`. Order breaks ties, and there is no tie here.

</details>

<!-- q: idea=symbol-table type=apply wrong="install the keywords too" -->
4. Change `tokens.l` so that numbers also go into a table of constants and print as `<num,1>`, `<num,2>`... in order of first appearance. What do you add, and what stays the same?

<details>
<summary>Answer</summary>

A second array and a second function like `install` (or one `install` that takes the table as a parameter), and the number rule's action becomes `printf("<num,%d> ", installnum(yytext));`, with the constants printed after the names in `main`. The patterns and their order stay exactly the same: what a token's attribute is, is decided in the action, not by the pattern.

</details>

## 4.5 Reading files and ending input: yyin, yyout, yywrap and noyywrap

### "Take the file name from the command line"

Every program so far has read standard input, so you ran it as `./tiny < ../ch2/prog.tiny`, and the shell opened the file. Lab questions are often worded differently: "write a Lex program that reads a C program from a file given as a command-line argument and writes the result to another file", or "count the lines and words in several files". The examiner will type `./a.out input.c output.c`, with no `<`. Your scanner must open the files itself.

This section shows the three things that decide where a Lex scanner reads, where it writes, and what it does when the input runs out: `yyin`, `yyout` and `yywrap`. The last one also answers a question left open in section 4.1: why every program in this book carries `noyywrap`.

### The belt that feeds the station

On the assembly line, the scanner station takes its raw material from an input belt and puts its product on an output belt. `yyin` is the input belt and `yyout` the output belt. Nothing in the station cares where the belts come from: standard input, a file, anything C can open as a `FILE *`. Swap the belt, and the same station works on different material.

When the input belt runs empty, the station does not just stop. It asks the foreman one question: "Is there another belt?" That question is a call to the function `yywrap`. If the foreman says no, the station shuts down and `yylex` returns 0. If the foreman says yes, the foreman has already swapped in a new belt, and the station carries on. The picture breaks in one way: the foreman is your code, and Flex trusts it completely. If `yywrap` says "yes, there is more" without swapping the belt, the station finds the same empty belt, asks again, and goes on asking for ever.
<!-- risk: not run: flex calling yywrap again and again when it returns 0 without a new yyin -->

### What yyin, yyout and yywrap are

<!-- teaches: yyin = yyin, yyout -->
`yyin` and `yyout` are two global variables of type `FILE *` defined in `lex.yy.c`. Both start as `NULL`. On its first call, `yylex` looks at them: if `yyin` is still `NULL` it sets it to `stdin`, and if `yyout` is `NULL` it sets it to `stdout`. So to read a file, you open it and store the pointer in `yyin` **before** the first call of `yylex`; to write to a file, you do the same with `yyout`. `yylex` reads from `yyin` in large blocks, and `ECHO` (including the default rule's copying) writes to `yyout`. Your own `printf` calls are not affected: `printf` always writes to standard output, so an action that should write to the output file must use `fprintf(yyout, ...)`.
<!-- risk: flex sets yyin to stdin and yyout to stdout on the first yylex call only if they are NULL (flex 2.6.4 manual, "The Generated Scanner") -->

That `NULL` check has a sharp edge. If you write `yyin = fopen(argv[1], "r");` and the file does not exist, `fopen` returns `NULL`, `yyin` is `NULL`, and `yylex` quietly reads the keyboard instead. The program seems to hang. Always test what `fopen` returned.

<!-- teaches: yywrap = yywrap, noyywrap, -lfl -->
`yywrap` is a function that `yylex` calls each time `yyin` reaches its end. It takes no arguments and returns an `int`:

- returns 1: there is no more input, so `yylex` returns 0 to its caller;
- returns 0: `yywrap` has pointed `yyin` at new input, so `yylex` goes on scanning from there.

Where does `yywrap` come from? Old Lex expected you to link a small library, `libl` (on Linux the Flex library `libfl`, linked with `-lfl`), which contains a `yywrap` that always returns 1, and also a `main` that calls `yylex` once. A program that writes its own `main` and its own `yywrap`, or tells Flex it needs none, does not need the library. `%option noyywrap` is that "none": it makes Flex write `yywrap` as a macro that is always 1, so no function called `yywrap` is needed. You can see it in the generated file:
<!-- risk: libfl provides both yywrap (returning 1) and main (calling yylex until it returns 0); exact behaviour of libfl's main not run here -->

<!-- code: id=wrap-inside -->
```console
$ flex first.l
$ grep -n 'define yywrap' lex.yy.c
326:#define yywrap() (/*CONSTCOND*/1)
```

Without the option, and without a `yywrap` of your own, `yylex` calls a function that exists nowhere, and the link fails with ``undefined reference to `yywrap'``. Section 4.7 shows the message. This book always writes its own `main`, so it always uses `noyywrap` (habit 4), except in a program like the one at the end of this section, which defines its own `yywrap` on purpose.

### A file in, a file out

Here is the usual lab exercise: copy a file, putting a line number in front of each line, reading the file named first on the command line and writing to the one named second. Either name may be left out, and then the scanner falls back on standard input or output. Save it as `lineno.l`:

<!-- code: id=lineno-write -->
```console
$ cat > lineno.l <<'END'
%{
/* lineno.l: copy a file, numbering its lines; argv[1] in, argv[2] out */
#include <stdio.h>
int n;                   /* lines numbered so far */
%}
%option noyywrap nounput noinput
%%
.*\n?    { fprintf(yyout, "%3d  %s", ++n, yytext); }
%%
int main(int argc, char *argv[])
{
    if (argc > 1 && (yyin = fopen(argv[1], "r")) == NULL) {
        perror(argv[1]);
        return 1;
    }
    if (argc > 2 && (yyout = fopen(argv[2], "w")) == NULL) {
        perror(argv[2]);
        return 1;
    }
    yylex();
    return 0;
}
END
```

There is one rule. `.*\n?` is "everything up to the end of the line, and the newline if there is one": the whole line, so the action runs once per line. Matching `.*` alone would leave the newline to the default rule, which would copy it to `yyout` after the numbered text; that also works, but this way one action handles the whole line. The `?` lets the last line match even if the file does not end with a newline. An empty line is just `\n`, and it gets a number too.

`main` opens the files before calling `yylex`, and tests each `fopen`. `perror` prints the file name and the system's reason, and the program returns 1 so that a script can tell it failed. Build it and try every case:

<!-- code: id=lineno-run exit=1 -->
```console
$ flex lineno.l
$ gcc -Wall -Wextra lex.yy.c -o lineno
$ ./lineno ../ch2/prog.tiny
  1  /* Tiny: the running statement, then a loop */
  2  init = 10;
  3  rate = 2.5;
  4  pos = init
  5      + rate * 60;   // 160
  6  while (pos >= 100)
  7      pos = pos - 7.25 / (rate != 0);
$ printf 'one\n\nthree' | ./lineno; echo
  1  one
  2
  3  three
$ ./lineno ../ch2/bad.tiny numbered.txt
$ cat numbered.txt
  1  x = 5. ;
  2  y = 9lives @ 2;
  3  z = !y; /* left open
  4  w = 1;
$ ./lineno nosuch.tiny
nosuch.tiny: No such file or directory
```

Four cases, four behaviours. With one name, the scanner read the file and wrote to the screen. With no name at all, `yyin` stayed `NULL`, `yylex` fell back on standard input, and the `printf` fed it three lines: the empty middle line got its number, and the last line, which had no newline, was still matched by `.*\n?` (the `echo` only ends the screen line that `lineno` left open). With two names, nothing appeared on the screen: every `fprintf(yyout, ...)` went into `numbered.txt`, which `cat` then showed. And with a name that does not exist, `perror` explained, and the exit status was 1.

Notice that `numbered.txt` was opened with `"w"`, which empties a file that already exists. In an exam, never give your own source file as the second argument.

### Many files, one scan: writing yywrap

The second classic is "count the lines and words in every file named on the command line, and the total", which is what `wc -lw` does. You could loop in `main`, opening each file and calling `yylex` again, but `yywrap` exists for exactly this: at the end of each file, `yylex` asks whether there is more, and your `yywrap` can say yes and open the next file. Save this as `multi.l`. It has no `noyywrap`, because it supplies its own `yywrap`:

<!-- code: id=multi-write -->
```console
$ cat > multi.l <<'END'
%{
/* multi.l: lines and words of each file named on the command line, like wc -lw */
#include <stdio.h>
int lines, words;        /* in the file being read */
int tlines, twords;      /* in all the files */
char **name;             /* the name of the file being read, then the rest */
int left;                /* how many names are left, counting that one */

int open_next(void)      /* yyin = the next file that opens; 0 if none is left */
{
    while (left > 0) {
        yyin = fopen(*name, "r");
        if (yyin != NULL)
            return 1;
        perror(*name);
        name++;
        left--;
    }
    return 0;
}
%}
%option nounput noinput
%%
\n          { lines++; }
[^ \t\n]+   { words++; }
[ \t]+      { }
%%
int yywrap(void)         /* called by yylex at the end of each file */
{
    printf("%5d %5d %s\n", lines, words, *name);
    tlines += lines;
    twords += words;
    lines = words = 0;
    fclose(yyin);
    name++;
    left--;
    return open_next() ? 0 : 1;   /* 0: go on with the new yyin; 1: stop */
}

int main(int argc, char *argv[])
{
    name = argv + 1;
    left = argc - 1;
    if (!open_next()) {
        fprintf(stderr, "usage: %s file...\n", argv[0]);
        return 1;
    }
    yylex();
    printf("%5d %5d total\n", tlines, twords);
    return 0;
}
END
```

Build it and give it the two Tiny files from chapter 2, then compare with `wc`, then add a name that does not exist between them:

<!-- code: id=multi-run -->
```console
$ flex multi.l
$ gcc -Wall -Wextra lex.yy.c -o multi
$ ./multi ../ch2/prog.tiny ../ch2/bad.tiny
    7    37 ../ch2/prog.tiny
    4    18 ../ch2/bad.tiny
   11    55 total
$ wc -lw ../ch2/prog.tiny ../ch2/bad.tiny
  7  37 ../ch2/prog.tiny
  4  18 ../ch2/bad.tiny
 11  55 total
$ ./multi ../ch2/prog.tiny nosuch.tiny ../ch2/bad.tiny
    7    37 ../ch2/prog.tiny
nosuch.tiny: No such file or directory
    4    18 ../ch2/bad.tiny
   11    55 total
```

The counts agree with `wc`, file by file and in total. The third run shows the program stepping over a file that will not open: `perror` reports it, and the scan goes on with the next name.

### Walking the end of each file

Follow `yyin` and the counters through the first run, `./multi ../ch2/prog.tiny ../ch2/bad.tiny`:

| Moment | `yyin` | `*name` | `left` | lines, words | What happens |
|---|---|---|---|---|---|
| `main` starts | `NULL` | `prog.tiny` | 2 | 0, 0 | `open_next` opens `prog.tiny` |
| first `yylex` call | `prog.tiny` | `prog.tiny` | 2 | 0, 0 | `yyin` is not `NULL`, so standard input is never used |
| end of `prog.tiny` | `prog.tiny` | `prog.tiny` | 2 | 7, 37 | `yylex` calls `yywrap`, which prints the line for `prog.tiny`, adds to the totals, zeroes the counters, closes the file and moves on |
| inside `yywrap` | `bad.tiny` | `bad.tiny` | 1 | 0, 0 | `open_next` opens `bad.tiny`; `yywrap` returns 0 |
| scanning goes on | `bad.tiny` | `bad.tiny` | 1 | 0, 0 | `yylex` throws away what was left of the old buffer and reads from the new `yyin` |
| end of `bad.tiny` | `bad.tiny` | `bad.tiny` | 1 | 4, 18 | `yywrap` prints, adds, closes; `left` becomes 0, `open_next` finds nothing and `yywrap` returns 1 |
| `yylex` returns 0 | closed | past the end | 0 | 0, 0 | `main` prints the total |

The `yylex` in `main` was called once. All the work of moving between files happened inside that call, between the end of one file and the start of the next. That is the job `yywrap` was made for, and the reason it is called "wrap": at the end of a file, the scanner can wrap around to more input.

### When it goes wrong

**The program hangs when given a file name.** The file did not open, `yyin` was left `NULL`, and `yylex` is reading the keyboard. Test every `fopen`.

**``undefined reference to `yywrap'``.** The specification has neither `%option noyywrap` nor a `yywrap` function, and you did not link `-lfl`. Add the option (or the function). Section 4.7 has the full message.

**The output file is empty, and the output appeared on the screen.** The actions use `printf`, which always writes to standard output. Write `fprintf(yyout, ...)` or `ECHO`.

**`yywrap` returns 0 and the program never ends.** It said there was more input without changing `yyin`. At the end of the last file it must return 1.

**The last file's counts are missing.** The totals are printed in `main` and the per-file line in `yywrap`; if `yywrap` returns 1 before printing, the last file is never reported. Print first, then decide.

### In one line each

1. `yyin` and `yyout` are `FILE *` globals; set them before the first `yylex`, or they become `stdin` and `stdout`.
2. Test `fopen`; a `NULL` `yyin` silently means the keyboard.
3. `ECHO` and the default rule write to `yyout`; `printf` does not.
4. At the end of `yyin`, `yylex` calls `yywrap`: 1 means stop, 0 means "I have opened more".
5. `%option noyywrap` makes `yywrap` always 1, so no library is needed; without it, supply `yywrap` or link `-lfl`.

**Check yourself**

<!-- q: idea=yyin type=predict wrong="it reports that the file is missing" -->
1. A student's `main` is `yyin = fopen(argv[1], "r"); yylex(); return 0;`. Predict what happens when it is run as `./lineno mistake.tiny` and `mistake.tiny` does not exist.

<details>
<summary>Answer</summary>

`fopen` returns `NULL`, so `yyin` is `NULL` when `yylex` is first called, and `yylex` sets it to `stdin`. The program waits for input from the keyboard and seems to hang; no message appears. Pressing Ctrl-D ends it with status 0. The tempting answer is a crash or an error message, but nothing in the program prints one, and a `NULL` `yyin` is not an error to Flex.

</details>

<!-- q: idea=yywrap type=predict wrong="the option is harmless when yywrap is defined" -->
2. Someone adds `%option noyywrap` to `multi.l`, next to `nounput noinput`, and builds it. What happens when Flex and GCC run, and what does the program print for two files?

<details>
<summary>Answer</summary>

Flex is happy: it does not read your C. It writes `#define yywrap() (/*CONSTCOND*/1)` into `lex.yy.c`, the line you saw with `grep` above. Then the C preprocessor meets your definition, `int yywrap(void)`, takes `yywrap(void)` for a use of that macro with one argument, and GCC stops: `error: macro "yywrap" passed 1 arguments, but takes just 0`, at the line of your definition. So nothing is printed, because nothing was built. Use `noyywrap` only when you do not write `yywrap`; when you do write it, leave the option out.
<!-- risk: message from a scratch build of multi.l with noyywrap added, not pasted in the chapter -->

</details>

<!-- q: idea=yyin type=apply wrong="ECHO goes to the screen" -->
3. Write a one-rule specification, with `main`, that copies a file to another file changing every tab into four blanks; the two names come from the command line.

<details>
<summary>Answer</summary>

```lex
%option noyywrap nounput noinput
%%
\t    { fprintf(yyout, "    "); }
%%
int main(int argc, char *argv[])
{
    if (argc != 3) {
        fprintf(stderr, "usage: %s in out\n", argv[0]);
        return 1;
    }
    if ((yyin = fopen(argv[1], "r")) == NULL || (yyout = fopen(argv[2], "w")) == NULL) {
        perror("fopen");
        return 1;
    }
    yylex();
    return 0;
}
```

One rule is enough: every other character goes to the default rule, which copies it to `yyout`, so the file is copied unchanged except for the tabs.

</details>

## 4.6 Start conditions: removing comments and scanning strings

### A pattern nobody can read

Look again at the block-comment rule in `tiny.l`:

```text
"/*"([^*]|"*"+[^*/])*"*"+"/"
```

It is correct, and section 4.4 explained it piece by piece, but you would not want to write it from memory in an exam, and it has two real weaknesses. It cannot say where an unclosed comment started without a second rule and some luck with longest match. And it does not know about strings: in C, `printf("/* not a comment */");` contains no comment, but a scanner that only knows the comment pattern will cut the string in half. String literals have the same problem in reverse: inside quotes, `\"` does not end the string and `\n` means one newline character, and writing all of that as one pattern is worse than the comment.

Chapter 2 solved both problems by changing what the scanner *does* for a while: once `skip` saw `/*`, it ran a different loop until `*/`. Lex has the same idea, built in. It is called a start condition, and it is what this section teaches.

### Changing boards

In the board game of chapter 3, the DFA is a board and each square says where each input card sends you. A start condition gives the scanner several boards. The normal board knows identifiers, numbers, operators and `/*`. When the scanner meets `/*`, an action says "from now on, play on the comment board". The comment board knows only three things: `*/`, which sends you back to the normal board, a newline, and anything else, which is thrown away. On the comment board an identifier means nothing, so `rate` inside a comment is just characters. A string board works the same way, with its own rules for `\"` and `\n`.

The picture breaks in one useful way. Flex does not really build several boards. It builds one big DFA in which each start condition has its own starting square, and switching boards only changes which starting square the *next* walk begins on. A walk that is already under way is never interrupted: you switch between matches, never in the middle of one.

### Declaring and switching start conditions

<!-- teaches: start-condition = start condition, %x, %s, BEGIN, INITIAL, <<EOF>> -->
A *start condition* is a name, declared in the definitions section, for a set of rules that are active only while the scanner is in that condition. There are two kinds of declaration:

- `%x COMMENT` declares an *exclusive* start condition: while in `COMMENT`, only rules marked `<COMMENT>` are active, and every unmarked rule is switched off.
- `%s NAME` declares an *inclusive* one: while in it, rules marked `<NAME>` are active *and so are* all unmarked rules.

For comments and strings you nearly always want `%x`: inside a comment, you do not want the identifier rule to fire.

A rule is marked by writing the condition in angle brackets in front of its pattern: `<COMMENT>"*/"`. One rule can belong to several, `<COMMENT,STR>\n`. Unmarked rules belong to the condition every scanner starts in, called `INITIAL`.

In an action, `BEGIN(COMMENT);` switches to `COMMENT` and `BEGIN(INITIAL);` switches back. `BEGIN` does not stop the action or return; it sets which board the next walk will use.

Finally, `<<EOF>>` is a special pattern that matches the end of the input. `<COMMENT><<EOF>>` matches the end of the input *while inside a comment*, which is exactly the "comment not closed" case. Its action must end the scan, with `return` or `yyterminate();`, or switch to new input; if it does neither, the scanner keeps hitting the same end of file.
<!-- risk: an <<EOF>> action that neither returns, calls yyterminate nor switches input loops (flex manual, "End-of-File Rules") -->

Here is what Flex does with those words, in the generated file of the comment remover below:

<!-- code: id=start-inside mode=fragment -->
```text
#define BEGIN (yy_start) = 1 + 2 *
#define INITIAL 0
#define COMMENT 1
```

`INITIAL` is 0 and `COMMENT` is 1, and `BEGIN(COMMENT)` becomes `(yy_start) = 1 + 2 * (COMMENT)`: it just stores a number that picks a starting square. (The `2 *` is there because each condition has two starting squares, one for walks that begin at the start of a line, used by `^` patterns, and one for the rest.) You will see these lines in your own `lex.yy.c` in a moment.

### A comment remover with start conditions

Section 2.4 wrote `decomment.c`, a comment remover in C, and fed it five lines of Tiny. Here is the same job in Lex, with one improvement that the C version did not have: it leaves strings alone. Like the C version, it replaces each block comment by one blank and keeps every newline, so that the line numbers of the code after a comment do not change. Save it as `decomment.l`:

<!-- code: id=decomment-write -->
```console
$ cat > decomment.l <<'END'
%{
/* decomment.l: remove C comments; keep strings and line numbers */
#include <stdio.h>
int start;               /* the line where the open comment began */
%}
%option noyywrap nounput noinput yylineno
%x COMMENT
%%
"/*"                 { start = yylineno; BEGIN(COMMENT); }
<COMMENT>"*/"        { putchar(' '); BEGIN(INITIAL); }
<COMMENT>\n          { putchar('\n'); }
<COMMENT>[^*\n]+     { }
<COMMENT>"*"         { }
<COMMENT><<EOF>>     { fprintf(stderr, "line %d: comment not closed\n", start);
                       return 1; }
"//".*               { }
\"([^"\\\n]|\\.)*\"  { ECHO; }
.|\n                 { ECHO; }
%%
int main(void)
{
    return yylex();
}
END
```

The rules, by board.

**On the normal board** (the unmarked rules): `/*` notes the line and switches to the comment board. `"//".*` drops a line comment, leaving its newline to the last rule, which copies it. The string rule matches a whole string literal, a quote, then any number of pieces that are either a character other than a quote, backslash or newline, or a backslash followed by any character, then a closing quote, and copies it unchanged. Because the string is matched whole, a `/*` inside it is never seen by the comment rule. The last rule copies every other character.

**On the comment board** (the `<COMMENT>` rules): `*/` prints the one blank and switches back. A newline is printed, to keep the line count. Runs of characters that are neither `*` nor newline are dropped, many at a time; a single `*` that is not part of `*/` is dropped on its own. `<<EOF>>` reports the line where the comment began and returns 1.

`main` returns what `yylex` returns: 0 at a normal end of input, or the 1 returned by the `<<EOF>>` action, so the exit status says whether a comment was left open.

Here is section 2.4's input with one line added, a `printf` whose string contains comment marks:

<!-- code: id=decomment-run -->
```console
$ cat > comments.tiny <<'END'
/* Tiny: the running statement */
init = 10;      // starting value
rate = 2.5; /* per
               second */ pos = init + rate * 60;
x = a/**/b / c;  /* rate **/ y = 1;
printf("/* not a comment */ %d\n", pos); // done
END
$ flex decomment.l
$ gcc -Wall -Wextra lex.yy.c -o decomment
$ ./decomment < comments.tiny

init = 10;
rate = 2.5;
  pos = init + rate * 60;
x = a b / c;    y = 1;
printf("/* not a comment */ %d\n", pos);
$ grep -n 'define BEGIN\|define INITIAL\|define COMMENT' lex.yy.c
117:#define BEGIN (yy_start) = 1 + 2 *
489:#define INITIAL 0
490:#define COMMENT 1
```

The first five lines of output are exactly what section 2.4's `decomment` printed: a blank for the comment on line 1, the line comment gone from line 2, the comment across lines 3 and 4 replaced by its newline and one blank, `a b` for `a/**/b`, and `/ c` kept as a division. The sixth line keeps its string whole; only the `// done` is gone. The C version of section 2.4 would have taken `/* not a comment */` for a comment and cut it out of the string.

### Walking line 5 board by board

Line 5 is `x = a/**/b / c;  /* rate **/ y = 1;`. Follow the board and the rule chosen at each step:

| Input | Board before | Rule that wins | Board after | Printed |
|---|---|---|---|---|
| `x = a` | normal | `.|\n`, five times | normal | `x = a` |
| `/*` | normal | `"/*"` (2 beats `.`'s 1) | comment | nothing |
| `*/` | comment | `<COMMENT>"*/"` (2 beats `"*"`'s 1) | normal | one blank |
| `b / c;` and two blanks | normal | `.|\n`, one character each; at `/` the next character is a blank, so `"/*"` and `"//".*` do not match | normal | `b / c;  ` |
| `/*` | normal | `"/*"` | comment | nothing |
| ` rate ` | comment | `<COMMENT>[^*\n]+` | comment | nothing |
| first `*` | comment | `<COMMENT>"*"`: `"*/"` does not match, because the next character is another `*` | comment | nothing |
| `*/` | comment | `<COMMENT>"*/"` | normal | one blank |
| ` y = 1;` and the newline | normal | `.|\n` | normal | ` y = 1;` and the newline |

The two hard cases from section 2.4, `/**/` and `**/`, are settled by longest match on the comment board: wherever `*/` can match, its two characters beat the single `*`. You do not need the `prev` variable of the C version.

The unclosed comment, as the input that must be rejected:

<!-- code: id=decomment-open exit=1 -->
```console
$ printf 'x = 1;\n/* open\ny = 2;\n' | ./decomment
x = 1;


line 2: comment not closed
```

The comment opened on line 2 swallowed `y = 2;`, its newlines were kept, and at the end of the input the scanner was still on the comment board, so `<COMMENT><<EOF>>` fired, reported line 2, and the exit status was 1.

### Scanning strings: a board that builds a value

A string literal in C is not just text between quotes. In `"pos = %d\n"`, the two characters `\` and `n` stand for one newline character, and in `"say \"hi\""` a backslash-quote is a quote inside the string, not its end. A scanner that hands a string to the next station must hand over its *value*, with the escapes worked out. This is the second classic use of a start condition. On the string board, each escape is a rule of its own, and the actions build the value in an array. Save this as `strings.l`:

<!-- code: id=strings-write -->
```console
$ cat > strings.l <<'END'
%{
/* strings.l: find C string literals and work out their values */
#include <stdio.h>
char buf[256];           /* the value of the string being read */
int len;                 /* characters in buf */
int start;               /* the line where it began */

void add(char c)
{
    if (len < (int) sizeof buf - 1)
        buf[len++] = c;
}
%}
%option noyywrap nounput noinput yylineno
%x STR
%%
\"               { len = 0; start = yylineno; BEGIN(STR); }
<STR>\"          { buf[len] = '\0';
                   printf("line %d: string of %d characters: [%s]\n", start, len, buf);
                   BEGIN(INITIAL); }
<STR>\\n         { add('\n'); }
<STR>\\t         { add('\t'); }
<STR>\\\"        { add('"'); }
<STR>\\\\        { add('\\'); }
<STR>\\.         { fprintf(stderr, "line %d: unknown escape %s\n", yylineno, yytext); }
<STR>[^\\"\n]+   { for (int i = 0; i < yyleng; i++) add(yytext[i]); }
<STR>\n          { fprintf(stderr, "line %d: string not closed\n", start);
                   BEGIN(INITIAL); }
.|\n             { }
%%
int main(void)
{
    yylex();
    return 0;
}
END
```

The patterns on the string board need reading slowly, because the backslash is both the thing being matched and Lex's own escape character. `\\n` is a backslash followed by the letter `n`: two characters of input, and the action adds one newline. `\\\"` is a backslash followed by a quote. `\\\\` is two backslashes. `\\.` is a backslash followed by any other character, an escape this scanner does not know, reported and dropped. Each known escape ties with `\\.` at length 2, and wins because it is written first. `[^\\"\n]+` is a run of ordinary characters, added all at once using `yyleng`. A newline inside a string is an error in C, so it reports the string as not closed and goes back to the normal board.

On the normal board there is only the opening quote and a rule that ignores everything else, because this program reports strings and nothing more.

<!-- code: id=strings-run -->
```console
$ cat > strings.txt <<'END'
printf("pos = %d\n", pos);
puts("say \"hi\" to C:\\tmp");
s = "/* not a comment */";
t = "left open;
u = "bad \q escape";
END
$ flex strings.l
$ gcc -Wall -Wextra lex.yy.c -o strings
$ ./strings < strings.txt
line 1: string of 9 characters: [pos = %d
]
line 2: string of 18 characters: [say "hi" to C:\tmp]
line 3: string of 19 characters: [/* not a comment */]
line 4: string not closed
line 5: unknown escape \q
line 5: string of 11 characters: [bad  escape]
```

Line 1's string is nine characters, not ten: `\n` became one real newline, which is why the closing bracket is printed on the next line. Line 2 has 18 characters: each `\"` became one quote and `\\` one backslash. Line 3 shows that comment marks are ordinary characters on the string board. Line 4's string runs into the newline and is reported against line 4, where it started, because the message uses `start`, not `yylineno`. On line 5 the unknown escape `\q` is reported and dropped, and the string is still finished: 11 characters, with two blanks where `\q` was.

### Why start conditions exist

You could do all of this without them. Section 2.4 did, in C, and Flex lets an action read characters itself with `input()` and loop, the way `skip` did. The trouble is that such an action is hand-written scanning inside a generated scanner: you are back to counting characters, pushing them back and checking for the end of file, and `%option noinput` has to go. Start conditions keep the whole scanner in one notation, patterns and actions, and let longest match and rule order do the fiddly work. The cost is small: a few more DFA states, and one more thing for a reader of your specification to keep in mind, which board each rule is on.

### When it goes wrong

**Identifiers are found inside comments.** The condition was declared with `%s` (inclusive), so every unmarked rule is active on the comment board too. Use `%x`.

**After the first comment everything disappears.** The rule for `*/` forgot `BEGIN(INITIAL)`, so the scanner never leaves the comment board.

**`BEGIN(COMMENT)` gives `‘COMMENT’ undeclared`.** The `%x COMMENT` line is missing from the definitions section, or is spelled differently.

**`unrecognized rule` at or just after a `<COMMENT>` rule.** A blank between `>` and the pattern, as in `<COMMENT> "*/"`; the pattern must follow the `>` at once.

**An unclosed comment goes unreported.** There is no `<COMMENT><<EOF>>` rule, so the scanner just stops at the end of the input, on the comment board, with no message.

### In one line each

1. `%x NAME` declares an exclusive start condition: in it, only `<NAME>` rules are active.
2. `%s NAME` is inclusive: unmarked rules stay active too.
3. `BEGIN(NAME)` switches board for the next walk; `BEGIN(INITIAL)` switches back.
4. `<NAME><<EOF>>` catches the end of the input inside a condition, for "comment not closed" and "string not closed".
5. Flex builds one DFA with a starting square per condition; `BEGIN` only chooses where the next walk starts.

**Check yourself**

<!-- q: idea=start-condition type=predict wrong="the string is treated as a comment" -->
1. Predict what `decomment` prints for the line `s = "a // b"; // note`.

<details>
<summary>Answer</summary>

`s = "a // b"; ` (with a blank at the end). On the normal board at `"`, the string rule matches the whole literal `"a // b"`, ten characters, far longer than `.`'s one, so it is copied unchanged and its `//` is never seen as a comment. After `; `, the `//` starts a real line comment, which `"//".*` drops. The tempting answer cuts the line at the first `//`.

</details>

<!-- q: idea=start-condition type=predict wrong="inclusive and exclusive behave the same here" -->
2. Change `%x COMMENT` to `%s COMMENT` in `decomment.l`. For `a /* b c */ d` the output does not change. Predict what each version prints for `a /*"x*/ b "y" */ c`, and explain the difference.

<details>
<summary>Answer</summary>

With `%x`: `a   b "y" */ c`. The comment is `/*"x*/`, which becomes one blank (between the two blanks the input already had), and the rest of the line is ordinary code, copied, including the stray `*/`.

With `%s`: `a "x*/ b "  c`. Right after `/*`, the scanner is on the comment board, but with `%s` the unmarked rules are active there too. At the `"`, `<COMMENT>[^*\n]+` can match only `"x` (it stops at the `*`), while the unmarked string rule matches `"x*/ b "`, eight characters, and wins by longest match. Its action is `ECHO`, so a piece of the comment is printed, and its `*/` is swallowed as part of a "string". The scanner is still in the comment until the real `*/` near the end, so `y"` and the blank after it are dropped, and that `*/` prints the one blank. For `a /* b c */ d` nothing changed only because no unmarked rule ever matched longer than a comment rule. Inclusive conditions let normal-board rules fire inside comments; for comments and strings, use `%x`.
<!-- risk: both outputs came from scratch runs of decomment.l with %x and with %s, not pasted in the chapter -->

</details>

<!-- q: idea=start-condition type=apply wrong="needs a second program" -->
3. Add one rule to `strings.l` so that a string still open at the end of the input is reported. Write the rule.

<details>
<summary>Answer</summary>

`<STR><<EOF>> { fprintf(stderr, "line %d: string not closed\n", start); return 0; }`. It fires only when the input ends while the scanner is on the string board; returning ends the scan. Without it, Flex's default end-of-file action stops the scan silently.

</details>

<!-- q: idea=start-condition type=fix wrong="the comment pattern is wrong" -->
4. A student's comment remover, using `%x COMMENT`, prints only the first line of a file and then nothing, although there is only one comment, on line 1. Their `*/` rule is `<COMMENT>"*/" { putchar(' '); }`. What is wrong?

<details>
<summary>Answer</summary>

The rule never switches back: it needs `BEGIN(INITIAL);`. After the first comment the scanner stays on the comment board for the rest of the input, where every character except newlines is dropped, so the rest of the file vanishes (the newlines still print, as empty lines).

</details>

## 4.7 When flex or gcc complains: the usual errors and their cures

### Three programs, three kinds of complaint

A Lex program passes through three programs before it runs, and each can refuse it. Flex reads your specification and complains about the patterns and the layout. GCC compiles `lex.yy.c` and complains about the C: your actions, your declarations, your `main`. The linker, run by GCC at the end, complains about functions that nobody defined. Then the program runs, and can still go wrong in ways no tool reports.

In the lab exam the clock is running, and the commonest way to lose twenty minutes is to read a message from one stage as if it came from another: to hunt through your C for a fault Flex found in a pattern, or to stare at your patterns because of a message about `yywrap`. This section makes each usual mistake really happen, so that you have seen every message once, and then sums them up in a table to keep.

### Who is talking: how to read a message

<!-- teaches: lex-errors = Flex error, linker error -->
The first word of a message tells you which program wrote it.

- `file.l:7: ...` with no column number: **Flex**. The number is a line of your `.l` file.
- `file.l:7:13: error: ...` or `warning: ...`: **GCC**, complaining about C that came from line 7, column 13, of your `.l` file. Flex writes lines like `#line 7 "file.l"` into `lex.yy.c` in front of each piece of your C, and GCC uses them to point you back to your own file.
- `lex.yy.c:1132:16: warning: ...`: **GCC**, about code that Flex wrote. You cannot fix that code; the cure is an option or a declaration in your specification, as with `nounput noinput` in section 4.1.
- `undefined reference to ...` followed by `ld returned 1 exit status`: **the linker**. Every file compiled; a function is used and defined nowhere.

Read the first message first. A single fault often causes several messages, and the later ones are usually echoes of the first.

### Flex: the specification itself

**An action's braces do not match.** Here the number rule's block never closes:

<!-- code: id=err-brace exit=1 -->
```console
$ cat > brace.l <<'END'
%option noyywrap nounput noinput
%%
[0-9]+   { printf("num %s\n", yytext);
[ \t\n]  { }
%%
int main(void) { yylex(); return 0; }
END
$ flex brace.l
brace.l:7: EOF encountered inside an action
```

Flex reads C inside an action by counting braces, so it read everything to the end of the file as part of the first action. The line number, 7, is one past the last line of the six-line file, which is *not* where the fault is; the message itself, `EOF encountered inside an action`, tells you to look for an unclosed `{` above it. Cure: close the block on the rule's line, or on a later line, as long as the braces balance.

**The action is on the next line.** An action must start on the same line as its pattern:

<!-- code: id=err-nextline exit=1 -->
```console
$ cat > nextline.l <<'END'
%option noyywrap nounput noinput
%%
[0-9]+
         { printf("num %s\n", yytext); }
%%
int main(void) { yylex(); return 0; }
END
$ flex nextline.l
nextline.l:5: unrecognized rule
nextline.l:5: fatal parse error
```

To Flex, `[0-9]+` alone on a line is a rule with an empty action, which is allowed, and the indented line under it is not the start of a rule. Flex gives up with `unrecognized rule` and `fatal parse error`, and it names line 5, the `%%` after the action, not line 4: the line Flex names is where it gave up, so look at the rules just above it. The cure is to move the `{` up onto the pattern's line.

**A name in braces that was never defined.**

<!-- code: id=err-undef exit=1 -->
```console
$ cat > undef.l <<'END'
%option noyywrap nounput noinput
digit    [0-9]
%%
{digits}+   { printf("num %s\n", yytext); }
.|\n        { }
%%
int main(void) { yylex(); return 0; }
END
$ flex undef.l
undef.l:4: undefined definition {digits}
undef.l:4: unrecognized rule
undef.l:4: unrecognized rule
```

The definition is `digit`, the rule uses `{digits}`. The first message is the real one; the two `unrecognized rule` lines after it are echoes. Cure: spell it as defined.

**The first `%%` is missing.** Then Flex reads the rules as if they were definitions, and the result is a flood:

<!-- code: id=err-nopct exit=1 -->
```console
$ cat > nopct.l <<'END'
%option noyywrap nounput noinput
[0-9]+  { printf("num %s\n", yytext); }
%%
int main(void) { yylex(); return 0; }
END
$ flex nopct.l
nopct.l:2: bad character: [
nopct.l:2: bad character: 0
nopct.l:2: bad character: -
nopct.l:2: bad character: 9
nopct.l:2: bad character: ]
nopct.l:2: bad character: +
nopct.l:2: bad character: {
nopct.l:2: unknown error processing section 1
nopct.l:2: unknown error processing section 1
nopct.l:2: bad character: (
nopct.l:2: bad character: "
nopct.l:2: unknown error processing section 1
nopct.l:2: bad character: %
nopct.l:2: unknown error processing section 1
nopct.l:2: bad character: \
nopct.l:2: unknown error processing section 1
nopct.l:2: bad character: "
nopct.l:2: bad character: ,
nopct.l:2: unknown error processing section 1
nopct.l:2: bad character: )
nopct.l:2: bad character: ;
nopct.l:2: bad character: }
```

Every character of the rule is reported as a `bad character`, because a definition line must start with a name. When you see a screen of `bad character` and `unknown error processing section 1` on one line, count your `%%` lines: there must be one before the first rule.

### GCC: the C that came from your actions

**A blank inside a pattern.** You want to match the two words `end if`, and write them as they look:

<!-- code: id=err-space exit=1 -->
```console
$ cat > space.l <<'END'
%option noyywrap nounput noinput
%%
end if   { printf("endif\n"); }
.|\n     { }
%%
int main(void) { yylex(); return 0; }
END
$ flex space.l
$ gcc -Wall -Wextra lex.yy.c -o space
space.l: In function ‘yylex’:
space.l:3:6: error: expected ‘(’ before ‘{’ token
    3 | end if   { printf("endif\n"); }
      |      ^
      |      (
```

Flex accepted it, and that is the trap. A blank ends a pattern, so the pattern is `end`, and the action is everything after it: `if   { printf("endif\n"); }`, a C `if` with no condition. Flex copies actions without understanding them, so only GCC can notice, and GCC reports `expected ‘(’ before ‘{’ token` at `space.l:3:6`, the rule's line, at the `{` where it expected the `(` of the `if`. Cure: quote the pattern, `"end if"`, or escape the blank, `end\ if`. Whenever GCC reports a syntax error on a rule's line, look first for a blank in the pattern.

**A library function with no header.** `lex.yy.c` includes `stdio.h`, `string.h`, `stdlib.h` and `errno.h` for its own use, so `printf`, `strcmp`, `strcpy` and `atoi` work in actions without your including anything. Anything else needs its header in the `%{ ... %}` block:
<!-- risk: the headers flex 2.6.4 includes in lex.yy.c (stdio, string, errno, stdlib); other versions may differ -->

<!-- code: id=err-upper mode=run -->
```console
$ cat > upper.l <<'END'
%option noyywrap nounput noinput
%%
[a-z]    { putchar(toupper(yytext[0])); }
%%
int main(void) { yylex(); return 0; }
END
$ flex upper.l
$ gcc -Wall -Wextra lex.yy.c -o upper
upper.l: In function ‘yylex’:
upper.l:3:11: warning: implicit declaration of function ‘toupper’ [-Wimplicit-function-declaration]
    3 | [a-z]    { putchar(toupper(yytext[0])); }
      |           ^~~~~~~
upper.l:1:1: note: include ‘<ctype.h>’ or provide a declaration of ‘toupper’
  +++ |+#include <ctype.h>
    1 | %option noyywrap nounput noinput
$ echo 'pos = init + rate * 60;' | ./upper
POS = INIT + RATE * 60;
```

With GCC 13 this is only a warning, and the program works, which makes it tempting to ignore. Do not: GCC even names the header to add, and from GCC 14 on, calling an undeclared function is an error and the build fails. Cure: `#include <ctype.h>` in a `%{ ... %}` block at the top.

**Other GCC messages you will meet.** `‘COMMENT’ undeclared`: a start condition used in `BEGIN` but not declared with `%x` (section 4.6). `‘count’ undeclared`: a variable declared in the user code section or in `main` instead of the `%{ ... %}` block (section 4.3). `implicit declaration of function ‘error’`: a function of your own, defined in the user code section and used in an action, with no declaration above the rules; declare it in the `%{ ... %}` block, as `tiny.l` does.

### The linker: functions nobody wrote

**No `noyywrap` and no `yywrap`.** Here is `first.l` with `noyywrap` taken out, made with `sed` as in section 4.1:

<!-- code: id=err-wrap exit=1 varies -->
```console
$ sed 's/noyywrap //' first.l > nowrap.l
$ flex nowrap.l
$ gcc -Wall -Wextra lex.yy.c -o nowrap
/usr/bin/ld: /tmp/ccrwXqSH.o: in function `yylex':
lex.yy.c:(.text+0x4ea): undefined reference to `yywrap'
collect2: error: ld returned 1 exit status
```

Flex and the compiler were content; the linker was not. `yylex` calls `yywrap` at the end of the input (section 4.5), and no file defines it. Cure: `%option noyywrap`, as in every program here. On a lab machine you will also see people add `-lfl` (or `-ll`) to the `gcc` command, which links Flex's library and its one-line `yywrap`; that works only where the library is installed, and the option works everywhere.

**No `main`.** A specification with an empty user code section and no library:

<!-- code: id=err-main exit=1 -->
```console
$ cat > nomain.l <<'END'
%option noyywrap nounput noinput
%%
[0-9]+   { printf("num %s\n", yytext); }
%%
END
$ flex nomain.l
$ gcc -Wall -Wextra lex.yy.c -o nomain
/usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/13/../../../x86_64-linux-gnu/Scrt1.o: in function `_start':
(.text+0x1b): undefined reference to `main'
collect2: error: ld returned 1 exit status
```

Flex writes `yylex`, never `main`. The message mentions `_start`, the C library's real entry point, which calls `main`. Cure: write `main` in the user code section.

### When it builds but misbehaves

**`flex scanner jammed`.** `%option nodefault` removes the default rule: instead of copying an unmatched character, the scanner stops with an error. It is a good option while you develop, because it makes a gap in your rules loud. Here the rules forget everything but numbers and blanks:

<!-- code: id=err-jam exit=2 -->
```console
$ cat > jam.l <<'END'
%{
#include <stdio.h>
%}
%option noyywrap nounput noinput nodefault
%%
[0-9]+     { printf("num %s\n", yytext); }
[ \t\n]+   { }
%%
int main(void)
{
    yylex();
    return 0;
}
END
$ flex jam.l
jam.l:8: warning, -s option given but default rule can be matched
$ gcc -Wall -Wextra lex.yy.c -o jam
$ echo '12 x 3' | ./jam
num 12
flex scanner jammed
```

Flex warned at build time: `-s option given but default rule can be matched` (`-s` is the command-line form of `nodefault`), meaning "some character matches none of your rules". The scanner printed `num 12`, then met `x`, matched nothing, printed `flex scanner jammed` and ended the program with exit status 2, without returning to `main`. Cure: a rule for every character, such as a final `.|\n` that reports or ignores what it gets.

**Sitting there doing nothing.** Not an error: the program is reading the keyboard, because no input was given or a file failed to open and `yyin` stayed `NULL` (section 4.5).

**Changes have no effect.** `gcc` compiled an old `lex.yy.c`; run `flex` again (section 4.1).

**Keywords come out as identifiers.** The rule order is wrong and Flex said `rule cannot be matched` (section 4.2).

### The table to keep

| Message | From | Cause | Cure |
|---|---|---|---|
| `EOF encountered inside an action` | Flex | a `{` in an action is never closed | balance the braces |
| `unrecognized rule`, `fatal parse error` | Flex | action on the next line, a blank before a `<COND>` pattern, or an unquoted `+`, `*` alone | put the action on the pattern's line; quote special characters |
| `undefined definition {name}` | Flex | name misspelt or not defined | define it in the definitions section, spelt the same |
| a flood of `bad character` | Flex | the first `%%` is missing | add `%%` before the rules |
| `unrecognized %option` | Flex | an option misspelt | check the spelling: `noyywrap nounput noinput` |
| `rule cannot be matched` (warning) | Flex | a rule always loses a tie to an earlier one | move the special case above the general one |
| `expected ‘(’ before ‘{’ token` on a rule's line | GCC | a blank in a pattern made part of it C | quote the pattern |
| `‘yyunput’ defined but not used`, `‘input’ defined but not used` | GCC | helpers you do not call | `%option nounput noinput` |
| `implicit declaration of function` | GCC | a missing header or declaration | include the header in `%{ ... %}` or declare the function |
| `‘NAME’ undeclared` | GCC | a variable or start condition not declared above the rules | declare it in `%{ ... %}`, or with `%x` |
| ``undefined reference to `yywrap'`` | linker | no `noyywrap`, no `yywrap`, no `-lfl` | `%option noyywrap` |
| ``undefined reference to `main'`` | linker | no `main` | write `main` in the user code section |
| `flex scanner jammed` | the program | `nodefault` and a character no rule matches | add a catch-all rule |

### In one line each

1. `file.l:N:` with no column is Flex; with a column it is GCC on your C; `lex.yy.c:N:` is GCC on Flex's code; `undefined reference` is the linker.
2. Read the first message; the rest are often echoes.
3. Flex never reads your C: a blank in a pattern or a missing header shows up only in GCC.
4. `noyywrap` and your own `main` mean no library is needed.
5. `nodefault` makes gaps in your rules loud while you develop.

**Check yourself**

<!-- q: idea=lex-errors type=fix wrong="the regular expression is wrong" -->
1. Flex prints `ops.l:9: EOF encountered inside an action` for a file of 9 lines whose rules are on lines 3 to 6. Where do you look, and for what?

<details>
<summary>Answer</summary>

Look at the actions on lines 3 to 6, for a `{` that is never closed (or a `"` that is never closed inside an action, which hides the `}` after it). Line 9 is only where Flex gave up, at the end of the file; it is not where the fault is.

</details>

<!-- q: idea=lex-errors type=decide wrong="add -lfl" -->
2. Your program built at home with `gcc lex.yy.c -lfl`, and on the exam machine the same command says `cannot find -lfl`. What is the quickest cure that works on any machine?

<details>
<summary>Answer</summary>

Add `%option noyywrap` to the definitions section (and make sure the file has its own `main`), run `flex` again and build with plain `gcc -Wall -Wextra lex.yy.c -o name`. The library was only supplying `yywrap` (and perhaps `main`); with the option, nothing from it is needed. Trying `-ll` instead is a guess that may fail the same way.

</details>

<!-- q: idea=lex-errors type=predict wrong="flex reports the blank" -->
3. A rule reads `"if" then  { printf("IF-THEN\n"); }`. Which program complains, and what does it say?

<details>
<summary>Answer</summary>

Flex accepts it: the pattern is `"if"`, and the action is `then  { printf("IF-THEN\n"); }`. GCC then compiles that C, where `then` is an unknown name followed by a block, and reports two errors on the rule's line of the `.l` file: `‘then’ undeclared (first use in this function)`, then `expected ‘;’ before ‘{’ token`. The cure is `"if then"` or `"if"\ "then"`. The tempting answer has Flex reporting it, but Flex never reads the C in an action.

</details>

<!-- q: idea=lex-errors type=explain wrong="the scanner crashed" -->
4. With `%option nodefault`, a scanner prints `flex scanner jammed` and the shell shows exit status 2. Your `main` returns 0 and prints a total after `yylex`, but the total never appeared. Explain both facts.

<details>
<summary>Answer</summary>

A character matched none of the rules, and with `nodefault` there is no default rule to copy it. Flex's generated code then reports the fatal error and calls `exit(2)` from inside `yylex`, so `yylex` never returns, the code after it in `main` never runs, and the exit status is the 2 from that `exit`, not your 0. It is not a crash (no signal); it is a deliberate stop.

</details>

## 4.8 Practice: the lab set of Lex programs, written from the question alone

You can now write a Lex specification in its three parts and build it with `flex` and `gcc` (4.1), choose patterns and order rules knowing that the longest match wins and then the earlier rule (4.2), count with actions and `yyleng` (4.3), return tokens one at a time and fill a symbol table (4.4), read and write named files and move between files with `yywrap` (4.5), use start conditions for comments and strings (4.6), and read any complaint from Flex, GCC or the linker (4.7). The lab exam gives you a one-line question and a blank file. These tasks climb from predicting what this chapter's programs do, through the lab sheet's usual questions written alone, to a program you have not seen that uses the whole chapter, checked without anyone marking it. Work in `~/cdlab/ch4`. Write every specification from the question, without looking back, and keep the three commands, `flex`, `gcc -Wall -Wextra`, run, in your fingers.

### Guided

<!-- q: idea=lex-rule-order type=predict wrong="3.14.15 is one number" -->
**1. Predict the rules.** Using `rules` from section 4.2, predict exactly what this prints, bracket by bracket, then run it:

<!-- code: id=practice-rules answer -->
```console
$ printf 'iff while1 <== 3.14.15\n' | ./rules
```

<details>
<summary>Answer</summary>

```console
$ printf 'iff while1 <== 3.14.15\n' | ./rules
[id iff] [id while1] [LE][ASSIGN] [num 3.14].[num 15]
```

`iff` and `while1` are longer than the keywords, so the identifier rule wins both. At `<==`, `"<="` takes two characters, and the `=` left over goes to `"="`. `3.14.15` is cut from the left: the number rule takes `3.14` (the walk tries to continue at the second `.`, finds no arrow on it from an accepting square, and stops), the `.` matches no rule and is copied by the default rule, and `15` is a second number. The tempting answer takes `3.14.15` as one number or as an error, but this scanner has no rule for either; it only ever takes the longest match at the current point.

</details>

<!-- q: idea=yyin type=apply wrong="only main changes, and the rules need a file rule" -->
**2. Vowels from a file.** Change the vowel counter of section 4.3 into `vowelfile.l`: it takes a file name on the command line, counts that file's vowels and consonants, prints them with the file name, and prints an error and exits with status 1 if the file will not open. With no name, it reads standard input as before. Test it on `../ch2/prog.tiny` and on a file that does not exist.

<details>
<summary>Answer</summary>

Only `main` changes; the rules are those of `vowels.l`.

<!-- code: id=practice-vowelfile exit=1 -->
```console
$ cat > vowelfile.l <<'END'
%{
/* vowelfile.l: vowels and consonants in a named file, or standard input */
#include <stdio.h>
int vowels, consonants;
%}
%option noyywrap nounput noinput
%%
[aeiouAEIOU]   { vowels++; }
[a-zA-Z]       { consonants++; }
.|\n           { }
%%
int main(int argc, char *argv[])
{
    const char *name = "standard input";

    if (argc > 1) {
        name = argv[1];
        if ((yyin = fopen(name, "r")) == NULL) {
            perror(name);
            return 1;
        }
    }
    yylex();
    printf("%s: vowels %d, consonants %d\n", name, vowels, consonants);
    return 0;
}
END
$ flex vowelfile.l
$ gcc -Wall -Wextra lex.yy.c -o vowelfile
$ ./vowelfile ../ch2/prog.tiny
../ch2/prog.tiny: vowels 27, consonants 42
$ echo 'pos = init + rate * 60;' | ./vowelfile
standard input: vowels 5, consonants 6
$ ./vowelfile nosuch.tiny
nosuch.tiny: No such file or directory
```

The usual mistake is to forget the test of `fopen`: then `./vowelfile nosuch.tiny` reads the keyboard instead of complaining (section 4.5).

</details>

<!-- q: idea=lex-errors type=fix wrong="only one thing is wrong" -->
**3. Three faults.** A student hands in this specification for "print every keyword and identifier in the input". It does not work, and there are three separate faults. Without running anything, say which program reports each one (Flex, GCC, the linker, or nobody), what it says, and the cure.

```text
%{
#include <stdio.h>
%}
%option nounput noinput
%%
[a-z][a-z0-9]*    printf("identifier %s\n", yytext);
else if           printf("keyword else if\n");
"if"|"else"       printf("keyword %s\n", yytext);
.|\n              ;
%%
int main(void) { yylex(); return 0; }
```

<details>
<summary>Answer</summary>

**The blank in `else if`.** Flex reads the pattern `else` and the action `if printf("keyword else if\n");`. Flex does not read the C, so it is silent about that; GCC reports `expected ‘(’ before ‘printf’` on line 7, the rule's line, because a C `if` needs a condition in brackets. Cure: `"else if"`.

**The keyword rules are below the identifier rule.** Flex warns `rule cannot be matched` twice, for line 7 (the `else` that the blank left as the pattern ties with the identifier rule on `else`) and for line 8, and builds anyway; the program would print `identifier if` for every keyword. Cure: put the keyword rules first.
<!-- risk: three.l messages from a scratch run (two warnings for lines 7 and 8, GCC error at 7:14), not pasted -->

**No `noyywrap`, no `yywrap`, no `-lfl`.** The linker stops with ``undefined reference to `yywrap'``. Cure: `%option noyywrap nounput noinput`.

You meet them in the order the tools run: the two Flex warnings first, then GCC's error. Only once the GCC error is fixed does the linker get to run and report `yywrap`, so a student fixing one message at a time sees three rounds. The tempting answer finds only the first fault, because each tool stops at its own stage and hides the faults of the next.

</details>

### On your own

<!-- q: idea=lex-pattern type=apply unseen="signed-numbers" wrong="-7 is counted as a positive 7" -->
**4. Signed numbers.** Write `signs.l`: count the positive integers, negative integers, positive fractions and negative fractions in the input. A number may have a sign, `+` or `-`, directly in front of it; a fraction has digits on both sides of the point. Test it on `12 -7 3.5 -0.25 +4 0 -10.0` and a second line `60 -x 7.`, and decide first what `7.` and `-x` should count as.

<details>
<summary>Answer</summary>

<!-- code: id=practice-signs -->
```console
$ cat > signs.l <<'END'
%{
/* signs.l: count positive and negative integers and fractions */
#include <stdio.h>
int posint, negint, posfrac, negfrac;
%}
%option noyywrap nounput noinput
digit    [0-9]
%%
"+"?{digit}+               { posint++; }
-{digit}+                  { negint++; }
"+"?{digit}+"."{digit}+    { posfrac++; }
-{digit}+"."{digit}+       { negfrac++; }
.|\n                       { }
%%
int main(void)
{
    yylex();
    printf("positive integers  %d\n", posint);
    printf("negative integers  %d\n", negint);
    printf("positive fractions %d\n", posfrac);
    printf("negative fractions %d\n", negfrac);
    return 0;
}
END
$ flex signs.l
$ gcc -Wall -Wextra lex.yy.c -o signs
$ printf '12 -7 3.5 -0.25 +4 0 -10.0\n60 -x 7.\n' | ./signs
positive integers  5
negative integers  1
positive fractions 1
negative fractions 2
```

Longest match does the work. On `-0.25`, the negative-fraction rule matches five characters and beats the negative-integer rule's two. `0` counts as positive here; if the question says zero is neither, give `0` and `+0` a rule of their own above the others, with the same length they tie and the earlier rule wins. `7.` is not a fraction by the question's definition, so `7` is a positive integer and the `.` is ignored; `-x` contains no number at all. Those are decisions, and in the exam you say them out loud. The real limit: in `x-3` the `-` is subtraction, but this program counts a negative 3. A scanner cannot tell; in a compiler the parser decides, and the scanner hands over `-` and `3` separately.

</details>

<!-- q: idea=yyin type=decide unseen="printf-to-writef" wrong="a rule for printf alone is enough" -->
**5. printf becomes writef.** Write `writef.l`: copy the C file named first on the command line to the file named second, replacing every call name `printf` by `writef` and `scanf` by `readf`, and report how many of each were replaced. It must not touch `sprintf`, a variable called `printf_count`, or the word `printf` inside a string. Write a small `io.c` to test it, and explain why the specification needs a rule that does nothing but copy identifiers.

<details>
<summary>Answer</summary>

<!-- code: id=practice-writef -->
```console
$ cat > writef.l <<'END'
%{
/* writef.l: copy a C file, printf -> writef, scanf -> readf, and count them */
#include <stdio.h>
int nprintf, nscanf;
%}
%option noyywrap nounput noinput
%%
"printf"                     { fprintf(yyout, "writef"); nprintf++; }
"scanf"                      { fprintf(yyout, "readf"); nscanf++; }
[A-Za-z_][A-Za-z0-9_]*       { ECHO; }
\"([^"\\\n]|\\.)*\"          { ECHO; }
.|\n                         { ECHO; }
%%
int main(int argc, char *argv[])
{
    if (argc != 3) {
        fprintf(stderr, "usage: %s in.c out.c\n", argv[0]);
        return 1;
    }
    if ((yyin = fopen(argv[1], "r")) == NULL) {
        perror(argv[1]);
        return 1;
    }
    if ((yyout = fopen(argv[2], "w")) == NULL) {
        perror(argv[2]);
        return 1;
    }
    yylex();
    fclose(yyout);
    printf("printf replaced %d times, scanf %d times\n", nprintf, nscanf);
    return 0;
}
END
```

A test file with every trap in it, then the run:

<!-- code: id=practice-writef-run -->
```console
$ cat > io.c <<'END'
#include <stdio.h>
int main(void)
{
    int n, printf_count = 0;
    char buf[40];
    printf("number? ");
    scanf("%d", &n);
    sprintf(buf, "%d", n);
    printf("printf and scanf: %s %d\n", buf, printf_count);
    return 0;
}
END
$ flex writef.l
$ gcc -Wall -Wextra lex.yy.c -o writef
$ ./writef io.c io-new.c
printf replaced 2 times, scanf 1 times
$ cat io-new.c
#include <stdio.h>
int main(void)
{
    int n, printf_count = 0;
    char buf[40];
    writef("number? ");
    readf("%d", &n);
    sprintf(buf, "%d", n);
    writef("printf and scanf: %s %d\n", buf, printf_count);
    return 0;
}
```

The identifier rule is what protects `sprintf` and `printf_count`. Without it, the scanner would walk into `sprintf`, find no rule for `s`, copy it with the default rule, and then match `printf` from the next position, turning `sprintf` into `swritef`. With it, `sprintf` is matched whole (7 characters beat nothing) and copied unchanged; on `printf_count`, 12 characters beat `"printf"`'s 6. On a plain `printf` the two rules tie at 6, and `"printf"`, written first, wins. The string rule protects the words inside `"printf and scanf: ..."`: the whole string is one match, longer than anything inside it. The output name must differ from the input name; `"w"` empties the file first.

</details>

<!-- q: idea=lex-rule-order type=explain wrong="order does not matter because order only breaks ties" -->
**6. Chapter 3's language in Lex.** Write `abb.l`: for each input line, print `accepted:` or `rejected:` and the line, accepting exactly the lines in the language of `(a|b)*abb` from section 3.6. Test it on the same lines `dfa` was given: `babb`, `abb`, `abba`, an empty line, `aabbabb`, `abcb`. Then explain why the accepting rule must be written first, although rule order "only breaks ties", and compare what Flex built with the four-state DFA of chapter 3.

<details>
<summary>Answer</summary>

<!-- code: id=practice-abb -->
```console
$ cat > abb.l <<'END'
%{
/* abb.l: accept the lines in (a|b)*abb */
#include <stdio.h>
%}
%option noyywrap nounput noinput
%%
^(a|b)*abb\n    { printf("accepted: %s", yytext); }
^.*\n           { printf("rejected: %s", yytext); }
%%
int main(void)
{
    yylex();
    return 0;
}
END
$ flex abb.l
$ gcc -Wall -Wextra lex.yy.c -o abb
$ printf 'babb\nabb\nabba\n\naabbabb\nabcb\n' | ./abb
accepted: babb
accepted: abb
rejected: abba
rejected:
accepted: aabbabb
rejected: abcb
```

On a line in the language, both rules match the whole line with its newline, the same length, so it *is* a tie, and the earlier rule must be the accepting one. On any other line the first rule cannot match the whole line, so the second wins whatever the order. With the rules swapped, Flex would warn `rule cannot be matched` and every line would be rejected. The `\n` in both patterns makes each rule match a whole line, so that `abba` cannot be accepted for its first three characters. Inside, Flex built from `(a|b)*abb` the same kind of board you built by hand: the subset construction and minimisation of sections 3.3 and 3.5, joined with the board for the second rule. The four squares of chapter 3's minimal DFA are in there, labelled with rule 1 where `abb` has just been read.

</details>

### Chapter project: a summary of a C program

<!-- q: idea=start-condition type=build unseen="c-summary" -->
**7. `csum.l`.** Write a Lex program that reads a C program, from the file named on the command line or from standard input, and prints a summary in exactly this form:

```text
lines        N
keywords     N
identifiers  N
integers     N
floats       N
strings      N
operators    N
comments     N
```

The rules of counting:

- **lines**: newlines, including those inside comments.
- **keywords**: `int char float double void if else while for do return break`; any other name is an identifier.
- **integers**: runs of digits; **floats**: digits, a point and digits.
- **strings**: string literals, with `\"` and other escapes allowed inside; nothing inside a string is counted as anything else.
- **operators**: each of `== != <= >= && || ++ -- += -=` counts as one, and so does each single `+ - * / % = < > !`.
- **comments**: each `/* ... */` and each `// ...` counts as one; nothing inside a comment is counted, except its newlines as lines.
- A line that starts with `#` is skipped entirely, apart from its newline.
- An unclosed `/*` prints `line N: comment not closed` to standard error, N being the line where it began, still prints the summary, and the exit status is 1.

Save this test program as `sample.c`; it is the input your program will be checked on:

<!-- code: id=sample-write -->
```console
$ cat > sample.c <<'END'
#include <stdio.h>
/* average of the marks; "quotes" here are not a string */
int main(void)
{
    int n = 0, total = 0;
    float avg = 0.0;   // no if or while counted here
    while (n < 10) {
        total += n * 2;
        n++;
    }
    if (n != 0 && total >= 1)
        avg = total / 2.5;
    printf("/* not a comment */ avg = %f\n", avg);
    return 0;
}
END
```

Count the keywords, identifiers and operators of `sample.c` by hand first: that is the written paper's half of this question, and it tells you whether a wrong number is your program's fault or your counting's.

**Done when** `./csum sample.c` prints exactly the model's output below. You can test that without anyone marking it: in the understudy folder (the one that holds `book/`), in the terminal, run `python3 scripts/us_practice.py check compiler-design 4.8 --in ~/cdlab/ch4`. It runs `flex csum.l`, builds `lex.yy.c` and compares what `./csum sample.c` prints with the model. Then test it, as habit 3 asks, on an empty input and on `printf 'int x; /* open\n'`, which must give the message and exit status 1.

<details>
<summary>Answer</summary>

The model, as a script that writes the specification, builds it and runs it on `sample.c`:

<!-- code: id=csum after=sample-write check="flex csum.l && gcc -Wall -Wextra lex.yy.c -o csum && ./csum sample.c" -->
```bash
cat > csum.l <<'END'
%{
/* csum.l: a summary of a C program: lines, keywords, identifiers,
   constants, strings, operators and comments */
#include <stdio.h>
int lines, keywords, ids, ints, floats, strings, ops, comments;
int start;               /* the line where an open comment began */
%}
%option noyywrap nounput noinput
%x COMMENT
digit    [0-9]
%%
^#.*                 { /* a preprocessor line: skipped */ }
"/*"                 { comments++; start = lines + 1; BEGIN(COMMENT); }
<COMMENT>"*/"        { BEGIN(INITIAL); }
<COMMENT>\n          { lines++; }
<COMMENT>[^*\n]+     { }
<COMMENT>"*"         { }
<COMMENT><<EOF>>     { fprintf(stderr, "line %d: comment not closed\n", start);
                       return 1; }
"//".*               { comments++; }
\"([^"\\\n]|\\.)*\"  { strings++; }

"int"|"char"|"float"|"double"|"void"|"if"|"else"|"while"|"for"|"do"|"return"|"break" { keywords++; }
[A-Za-z_][A-Za-z0-9_]*     { ids++; }
{digit}+"."{digit}+        { floats++; }
{digit}+                   { ints++; }

"=="|"!="|"<="|">="|"&&"|"||"|"++"|"--"|"+="|"-=" { ops++; }
[-+*/%=<>!]                { ops++; }
\n                         { lines++; }
.                          { /* blanks and punctuation */ }
%%
int main(int argc, char *argv[])
{
    int status;

    if (argc > 1 && (yyin = fopen(argv[1], "r")) == NULL) {
        perror(argv[1]);
        return 1;
    }
    status = yylex();
    printf("lines        %d\n", lines);
    printf("keywords     %d\n", keywords);
    printf("identifiers  %d\n", ids);
    printf("integers     %d\n", ints);
    printf("floats       %d\n", floats);
    printf("strings      %d\n", strings);
    printf("operators    %d\n", ops);
    printf("comments     %d\n", comments);
    return status;
}
END
flex csum.l && gcc -Wall -Wextra lex.yy.c -o csum && ./csum sample.c
```
```text
lines        15
keywords     7
identifiers  14
integers     7
floats       2
strings      1
operators    12
comments     2
```

The hand count, to compare with yours. **Keywords** (7): `int` before `main`, `void`, `int`, `float`, `while`, `if`, `return`; the `if` and `while` in the line comment and everything in the block comment are not counted. **Identifiers** (14): `main`, then `n` and `total` in the declaration, `avg`, `n` in the `while`, `total` and `n` on line 8, `n` in `n++`, `n` and `total` in the `if`, `avg` and `total` on line 12, `printf` and `avg` on line 13; `stdio` and `h` are not counted because line 1 is skipped. **Integers** (7): `0`, `0`, `10`, `2`, `0`, `1`, and the `0` after `return`. **Floats** (2): `0.0` and `2.5`. **Operators** (12): three `=` in the declarations, `<`, `+=`, `*`, `++`, `!=`, `&&`, `>=`, `=`, `/`. **Comments** (2) and **strings** (1): the `/* not a comment */` is inside the string, and the `"quotes"` are inside the comment.

What the program is built from: the comment board of section 4.6, with `start` worked out from the line counter because there is no `yylineno` here; the string rule of section 4.6, which matches the whole literal so nothing inside it is seen; keywords before identifiers (4.2); floats before integers only for readability, since `2.5` beats `2` by longest match anyway; two-character operators beating their one-character prefixes by longest match; `yyin` from the command line, tested (4.5); and the `<<EOF>>` action's `return 1` passed on to the exit status through `status`, so that the summary is still printed. The usual mistakes: counting `/` twice in `//` because the line-comment rule is missing, counting `"quotes"` as a string because comments were not handled before strings, and counting `if` in `// no if or while` as a keyword.

</details>
