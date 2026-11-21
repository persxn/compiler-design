# Chapter 2: Write a lexical analyzer in C

<!-- contents -->
- [2.1 Reading source one character at a time, with one character of lookahead](#21-reading-source-one-character-at-a-time-with-one-character-of-lookahead)
- [2.2 Recognising identifiers and telling keywords apart](#22-recognising-identifiers-and-telling-keywords-apart)
- [2.3 Numbers, operators and the longest-match rule](#23-numbers-operators-and-the-longest-match-rule)
- [2.4 Skipping white space and both kinds of C comment](#24-skipping-white-space-and-both-kinds-of-c-comment)
- [2.5 Transition diagrams: drawing the scanner before writing it](#25-transition-diagrams-drawing-the-scanner-before-writing-it)
- [2.6 Input buffering: buffer pairs and sentinels](#26-input-buffering-buffer-pairs-and-sentinels)
- [2.7 Lexical errors and line numbers](#27-lexical-errors-and-line-numbers)
- [2.8 A symbol table in C: insert, search, display, delete with a hash table](#28-a-symbol-table-in-c-insert-search-display-delete-with-a-hash-table)
- [2.9 Practice: a complete scanner for a C subset that fills a symbol table](#29-practice-a-complete-scanner-for-a-c-subset-that-fills-a-symbol-table)
<!-- /contents -->

## 2.1 Reading source one character at a time, with one character of lookahead

### Where does `rate` end?

In chapter 1 you cut `pos = init + rate * 60;` into lexemes by eye, and section 1.3 turned them into tokens. Your eye did something no program can: it saw the whole line at once. The first program of the compiler-design lab asks you to do the same cut in C, "write a lexical analyzer that reads a program and prints its tokens", and a C program does not see a line at once. It sees one character, then the next, then the next.

Try it the program's way. You have read `r`, `a`, `t`, `e`. Is the identifier finished? You cannot tell. The next character might be `2`, and then the name is `rate2`. You only know that `rate` has ended when you have read the character after it, the space or the `*`. By then you have read one character too many, and that character belongs to the next lexeme. Somebody has to keep it, or it is lost.

That one problem shapes every hand-written scanner. This section solves it in about twenty lines of C, and the solution has a name you will use for the rest of the book.

### Taking cards off a pile

Picture the scanner station on the assembly line from section 1.1. Its input comes in on a belt, one character at a time. To find out what the next character is, the station has to take it off the belt; there is no looking without taking. When it takes a character that turns out to belong to the next lexeme, it puts it back on the belt, in front, so that it is the first thing taken next time.

That putting back is exactly what C lets you do, and it is the whole trick. The picture stops being true in two places. First, C promises that you can put back one character; it does not promise two (the next sections show this is enough). Second, the belt is not really a belt: behind `getchar` the C library reads the file in large blocks and hands you characters out of a block in memory. Section 2.6 opens that box.

### What getchar and ungetc really do

`getchar()` returns the next character of standard input. Its return type is `int`, not `char`, and the reason matters for a scanner: it must be able to return every possible byte value and also one extra value, `EOF`, which means "there are no more characters". `EOF` is a negative `int` (on Linux with glibc it is -1), so it can never be confused with a real character, which `getchar` returns as a value from 0 to 255.
<!-- checked: how=run date=2026-11-21 ref="scratch gcc 13.3 glibc build printed EOF=-1; C11 7.21.1p3: EOF is a negative int" h=e58478ba -->

`ungetc(c, stdin)` pushes the character `c` back onto standard input. The next `getchar()` returns `c`, and the one after that carries on where reading had got to. Nothing happens to the file; the character is only kept in the input stream's memory until it is read again.

<!-- teaches: lookahead = lookahead, lookahead character -->
The character a scanner reads past the end of a lexeme, to find out that the lexeme has ended, is called the *lookahead* character. Every scanner in this book reads exactly one lookahead character at the end of an identifier or a number, and either uses it or gives it back.

<!-- teaches: ungetc = ungetc, pushback -->
Giving it back with `ungetc` is called *pushback*. The C standard guarantees one character of pushback on a stream; a second `ungetc` before the next read may fail. A scanner that needs to "unread" two characters therefore has to keep the extra one itself, which is one reason Tiny's numbers (section 2.3) are designed so that one character is always enough.
<!-- checked: how=known date=2026-11-21 ref="C11 7.21.7.10p3: one character of pushback is guaranteed" h=9bdcdef9 -->

Here is the cut of `rate*60` traced one call at a time, as the program below does it. The column on the right says what is waiting on the belt after each step.

| Step | Call | Returns | What the scanner does | Waiting on the belt |
|---|---|---|---|---|
| 1 | `getchar()` | `r` | a letter: start a name | `ate*60` |
| 2 | `getchar()` | `a` | letter or digit: add it | `te*60` |
| 3 | `getchar()` | `t` | add it | `e*60` |
| 4 | `getchar()` | `e` | add it | `*60` |
| 5 | `getchar()` | `*` | not a letter or digit: the name `rate` is complete | `60` |
| 6 | `ungetc('*')` | | give the lookahead back | `*60` |
| 7 | `getchar()` | `*` | the next lexeme starts here | `60` |

Step 5 is the moment you could not avoid: the scanner had to take the `*` off the belt to learn that `rate` was over. Step 6 is the cure.

### The first scanner: words and everything else

Make this chapter's folder first. Open a terminal in your home folder:

<!-- code: id=ch2-setup -->
```console
$ mkdir cdlab/ch2
$ cd cdlab/ch2
```

Save this program as `words.c` in `~/cdlab/ch2`. It is not yet a real scanner; it knows only two kinds of lexeme, a *word* (a letter followed by letters and digits) and a single other character, and it throws white space away.

<!-- code: id=words mode=compile -->
```c
#include <stdio.h>
#include <ctype.h>

int main(void)
{
    int c;
    char word[64];
    int n;

    c = getchar();
    while (c != EOF) {
        if (isalpha(c)) {
            n = 0;
            while (isalnum(c)) {
                if (n < 63)
                    word[n++] = c;
                c = getchar();
            }
            word[n] = '\0';
            ungetc(c, stdin);
            printf("word   %s\n", word);
        } else if (!isspace(c)) {
            printf("other  %c\n", c);
        }
        c = getchar();
    }
    return 0;
}
```

Build it and give it the running statement on standard input. `printf` writes the line into a pipe, `|`, and `words` reads the other end of the pipe as if you had typed it:

<!-- code: id=words-run -->
```console
$ gcc -Wall -Wextra words.c -o words
$ printf 'pos = init + rate * 60;\n' | ./words
word   pos
other  =
word   init
other  +
word   rate
other  *
other  6
other  0
other  ;
```

Every program in this chapter is built and run like this. From now on the book shows the program, says what input it was given, and shows what it printed, without repeating the two commands.

### Walking through what just happened

Follow the variable `c` from the top. `c = getchar()` before the loop reads `p`. The outer loop asks one question per lexeme: what kind of character is `c`?

`p` is a letter, so the inner loop starts. It copies `p` into `word`, reads `o`, copies it, reads `s`, copies it, and reads the space. The space is not a letter or digit, so the inner loop stops, holding the space in `c`. That space is the lookahead. `word[n] = '\0'` ends the string, `ungetc(c, stdin)` gives the space back, and the program prints `word   pos`.

Now the last line of the outer loop, `c = getchar()`, reads again, and gets the space it just gave back. The next round of the outer loop sees a space, which is white space, so it does nothing, and the next `getchar` reads `=`. That is printed as `other  =`. And so on to the end.

Look at the last four lines of output. `60` came out as two separate characters, `6` and `0`, because this program has no idea what a number is. Section 2.3 teaches it. The `if (n < 63)` line is there so that a name of a hundred letters cannot run past the end of `word`, which has room for 63 characters and the `'\0'`; the extra letters are read and dropped. Section 2.7 comes back to what a scanner should say about such a name.

Two calls of `isalnum` and `isalpha` deserve a note. They take an `int`, and they are defined for every value `getchar` can return, `EOF` included, so `isalnum(EOF)` is simply false and the inner loop ends cleanly at the end of the input. And `ungetc(EOF, stdin)` is allowed too: it does nothing. So a name at the very end of a file, with no newline after it, needs no special code.

### Change one thing: take out the ungetc

Copy the program to `nolook.c` and delete the `ungetc` line, nothing else. Give it the same statement, this time written without spaces, `pos=init+rate*60;`:

<!-- code: id=nolook stdin="pos=init+rate*60;\n" -->
```c
#include <stdio.h>
#include <ctype.h>

int main(void)
{
    int c;
    char word[64];
    int n;

    c = getchar();
    while (c != EOF) {
        if (isalpha(c)) {
            n = 0;
            while (isalnum(c)) {
                if (n < 63)
                    word[n++] = c;
                c = getchar();
            }
            word[n] = '\0';
            printf("word   %s\n", word);
        } else if (!isspace(c)) {
            printf("other  %c\n", c);
        }
        c = getchar();
    }
    return 0;
}
```
```text
word   pos
word   init
word   rate
other  6
other  0
other  ;
```

The `=`, the `+` and the `*` are gone, and no message says so. Walk it: after `pos` the inner loop stops holding `=` in `c`. Nobody gives it back, and the outer loop's `c = getchar()` overwrites it with `i`. Every character that comes straight after a name is swallowed.

Now give `nolook` the spaced version, `pos = init + rate * 60;`. The character swallowed after each name is then a space, which the program was going to throw away anyway, and the output is identical to the correct program's. This is why the bug survives: on the neat, spaced test you typed first, it works. Habit 3 of this book, test on a normal input, an empty input and an input that must be rejected, exists for exactly this; and the input with no spaces counts as normal, since C allows it.

There is a second way to fix it, without `ungetc`: keep the lookahead in `c` and skip the `c = getchar()` at the bottom whenever the word branch ran. That works, and some textbooks write scanners that way, with a global variable that always holds the next character. This book uses `ungetc` because, from section 2.2 on, the scanner is a function that the parser calls once per token, and a function that starts with a clean `getchar()` every time is easier to get right than one that must remember to look at a variable first.

### When it goes wrong

**`char c` instead of `int c`.** On a PC it compiles without a warning, even with `-Wall -Wextra`. There `char` is signed, so the byte 255 in a file turns into -1 and stops the loop early as if the file had ended; on machines where `char` is unsigned, such as many ARM boards, `c` can never equal -1 and the loop at the end of the file never stops (there `-Wextra` does warn: "comparison is always true due to limited range of data type"). Always `int c` for anything `getchar` returns.
<!-- checked: how=run date=2026-11-21 ref="scratch gcc 13.3 -Wall -Wextra: no warning by default; with -funsigned-char, -Wtype-limits warns; CHAR_MIN=-128 on x86-64" h=85ca4e30 -->

**Characters vanish after names.** You forgot the `ungetc`, as in `nolook`. Test on input with no spaces.

**Two characters pushed back.** You read past the end of a lexeme by two characters and call `ungetc` twice. On glibc it may happen to work, but the standard promises only one, so an examiner's machine owes you nothing. Redesign so that one is enough, or keep the second character in a variable of your own.
<!-- checked: how=run date=2026-11-21 ref="scratch glibc build: two ungetc calls both succeeded; C11 7.21.7.10p3 guarantees one" h=ebf7d427 -->

**The array overflows.** A long name written into `word` without the `n < 63` guard writes past the end of the array. It may run fine in your test and crash at the examiner's desk. Keep the guard.

### In one line each

1. A scanner reads one character at a time and learns that a lexeme has ended only by reading the character after it.
2. That extra character is the lookahead; `ungetc` pushes it back so the next `getchar` returns it.
3. `getchar` returns an `int` so that `EOF` can be told apart from every real character.
4. C guarantees one character of pushback, which is all a scanner for Tiny or C needs.

**Check yourself**

<!-- q: idea=lookahead type=predict wrong="nolook loses the digits too" -->
1. Without running it, say what `nolook` prints for the input `x1=y2*3;`. Which characters are lost, and why are the digits in `x1` and `y2` safe?

<details>
<summary>Answer</summary>

```text
word   x1
word   y2
other  3
other  ;
```

The `=` after `x1` and the `*` after `y2` are lost: each is the lookahead that ends a name, and without `ungetc` it is overwritten by the bottom `getchar`. The digits `1` and `2` are safe because they are part of the names (`isalnum` accepts digits after the first letter), so the inner loop copies them before it stops. The `3` and `;` come after an operator, not after a name, so they are read normally. If you said the digits are lost, you thought a word could only hold letters; read the inner loop's test again.

</details>

<!-- q: idea=ungetc type=explain wrong="ungetc writes the character back into the file" -->
2. A friend says "`ungetc` writes the character back into the file, so if I `ungetc` an `x` the file now has an `x` in it". Explain in two sentences what `ungetc` really changes.

<details>
<summary>Answer</summary>

`ungetc` changes nothing on disk; it puts the character into the stream's own memory, in front of what is still to be read, so the next `getchar` returns it. That is also why you could push back a character the file never contained, and why only one character of pushback is guaranteed: there is only a small space set aside for it.

</details>

<!-- q: idea=lookahead type=fix wrong="the guard n < 63 is enough to make the loop end" -->
3. A student writes the inner loop as `while (isalnum(c)) { word[n++] = c; c = getchar(); if (n == 63) break; }` and puts `ungetc(c, stdin);` after it. A name of 70 letters is scanned. What comes out, and what is the fault?

<details>
<summary>Answer</summary>

The loop stops after 63 letters with the 64th letter in `c` (it was read just before the `break`), and `ungetc` pushes that letter back. The next round of the outer loop starts a new word with it, so one name of 70 letters comes out as two words, one of 63 letters and one of 7. The fault is that the length limit ends the lexeme; the limit should only stop the copying, while the loop keeps reading letters to the true end of the name, as `words.c` does with `if (n < 63)`. A scanner must never split one lexeme into two because its array is full.

</details>

## 2.2 Recognising identifiers and telling keywords apart

### `while` is not a name

Give `words` from section 2.1 the line `while (rate2 > if_x) whiley = if;` and it prints `word   while` and `word   if`, exactly as it prints `word   rate2`. To the reading loop they are all the same thing: a letter followed by letters and digits. To a parser they are completely different. `while` starts a loop, `if` starts a choice, and `rate2` is a variable. Section 1.3 left this wrinkle for now: keywords match the identifier pattern, yet each one must come out as its own token.

There is a second fault in that output. `if_x` comes out as `word if`, then `other _`, then `word x`, because `words` does not know that C (and Tiny) let a name contain underscores. And there is a trap close by: `whiley` must come out as one identifier, not as the keyword `while` followed by a `y`.

This section fixes all three, and turns the reading loop into the shape every scanner in this book has: a function the parser can call to get one token.

### One rule for spelling, one list of exceptions

Think about how a station on the assembly line would sort names if some of them were reserved. It would not keep a separate procedure for each reserved word. It would read each name in full by the one spelling rule, then glance at a short card pinned above the bench listing the reserved words. On the card: the name leaves as that keyword. Not on the card: it leaves as `id`.

<!-- teaches: identifier-scan = identifier scanning -->
That is the method, and it has two halves. *Identifier scanning* reads the longest run of characters that fits the identifier pattern, a letter or underscore followed by letters, digits and underscores, and stops at the first character that does not fit, which is the lookahead and goes back.

<!-- teaches: keyword-table = keyword table, reserved word -->
The *keyword table* is the card: a list of the *reserved words* of the language, each with its token. After a name has been read in full, the scanner looks it up there. Only a name that matches an entry character for character becomes a keyword.

The order of the two halves is the whole point. Because the name is read in full before the lookup, `whiley` is compared as `whiley`, finds no match, and becomes an identifier, just as C itself treats it. If the scanner checked for keywords while reading, it would see `w-h-i-l-e` match after five letters and cut too early. Section 2.3 gives this "read as far as the pattern allows" idea its proper name.

There is another way you will see in textbooks: draw a separate path for every keyword in the scanner's diagram (section 2.5 shows what such diagrams look like), so that `w`, `h`, `i`, `l`, `e` leads to a "keyword while" state. It works, but a language with thirty keywords needs thirty paths, all tangled with the identifier path, and adding one keyword means redrawing. The table costs one line per keyword and a few `strcmp` calls per name. A third way, which the Dragon book uses, is to put the keywords into the symbol table before scanning starts, marked as keywords; section 2.8 builds the symbol table, and you can do the same with it there.

The picture of a card breaks in one place: the real check is exact. `While` with a capital W is not on the card, because C and Tiny are case-sensitive, so `While` is an identifier. Some languages (Pascal, SQL) ignore case, and their scanners lower-case each name before the lookup.

### The scanner becomes a function

Section 1.1's assembly line had each station finish its whole job before handing the result on. A real scanner does not work like that, and this is where the picture breaks. The parser does not wait for a finished list of tokens; whenever it needs the next token it calls the scanner, which reads just far enough to make one token and returns it. So the scanner is a function. In this book it is called `gettoken`. It returns the token name as a number, and leaves the characters of the lexeme in a global array called `lexeme` for whoever wants them.

Until chapter 6 there is no parser, so `main` stands in for it: it calls `gettoken` over and over and prints each token, until `gettoken` returns `END`, the token this book uses for "no more input".

Save this as `idkw.c`. It was given the line `while (rate2 > if_x) whiley = if;`:

<!-- code: id=idkw stdin="while (rate2 > if_x) whiley = if;\n" -->
```c
#include <stdio.h>
#include <ctype.h>
#include <string.h>

enum { ID, IF, WHILE, OTHER, END };
const char *name[] = { "id", "if", "while", "other", "end" };

struct { const char *word; int token; } keyword[] = {
    { "if", IF },
    { "while", WHILE },
};
#define NKEY 2

char lexeme[64];

int gettoken(void)
{
    int c, n = 0;

    c = getchar();
    while (isspace(c))
        c = getchar();
    if (c == EOF)
        return END;
    if (isalpha(c) || c == '_') {
        while (isalnum(c) || c == '_') {
            if (n < 63)
                lexeme[n++] = c;
            c = getchar();
        }
        lexeme[n] = '\0';
        ungetc(c, stdin);
        for (int i = 0; i < NKEY; i++)
            if (strcmp(lexeme, keyword[i].word) == 0)
                return keyword[i].token;
        return ID;
    }
    lexeme[0] = c;
    lexeme[1] = '\0';
    return OTHER;
}

int main(void)
{
    int t;

    while ((t = gettoken()) != END)
        printf("%-6s %s\n", name[t], lexeme);
    return 0;
}
```
```text
while  while
other  (
id     rate2
other  >
id     if_x
other  )
id     whiley
other  =
if     if
other  ;
```

### Reading the program, part by part

**The token names.** `enum { ID, IF, WHILE, OTHER, END }` gives each token name a number: `ID` is 0, `IF` is 1, and so on. Numbers are what a parser will compare, quickly, with `==` or a `switch`. The array `name` holds the same names as strings in the same order, so `name[t]` turns a token number back into text for printing. Keep the two lists in step: add a token to one and not the other, and every name after it prints wrongly.

**The keyword table.** `keyword` is an array of small structures, each a word and its token. The structure has no name of its own because nothing else needs it. Writing word and token side by side in one row is safer than two separate arrays: you cannot add `"while"` and forget `WHILE`. `NKEY` is the number of rows. Tiny has only two keywords for now, `if` and `while`, which chapter 9 needs; they are reserved from the start so that nobody writes a Tiny program that uses `while` as a variable. The C subset in section 2.9 needs eight, and that is a change of the table alone.

**Skipping white space.** `gettoken` begins by reading characters until one is not white space. `isspace` is true for the space, tab, newline and a few other blank characters, and false for `EOF`, so at the end of the input the loop stops with `c == EOF` and the function returns `END`. Section 2.4 adds comments to what is skipped.

**The identifier.** The test that starts a name is `isalpha(c) || c == '_'`: a letter or an underscore. The test that continues it is `isalnum(c) || c == '_'`: a letter, a digit or an underscore. The difference between the two tests is why `2x` can never start a name. The loop is the one from section 2.1, the lookahead goes back with `ungetc`, and only then, with the whole name in `lexeme`, comes the lookup.

**The lookup.** `strcmp(lexeme, keyword[i].word) == 0` is true only when the two strings are equal in every character and in length. On a match the function returns that keyword's token; after the loop, with no match, it returns `ID`.

**Everything else.** Any other character becomes a one-character lexeme with the token `OTHER`, for now. Section 2.3 replaces `OTHER` with real tokens for numbers and operators.

### Walking `whiley` through it

Here is the call of `gettoken` that produced `id whiley`, one step at a time.

| Step | `c` | `lexeme` afterwards | What happens |
|---|---|---|---|
| 1 | space | (previous token's) | white space: read on |
| 2 | `w` | | a letter: start a name |
| 3 | `h`, `i`, `l`, `e` | `whil`, then `while` | each is a letter: add it |
| 4 | `y` | `whiley` | still a letter: add it; the name is not over just because it spells a keyword so far |
| 5 | space | `whiley` | not a name character: the name is complete; push the space back |
| 6 | | `whiley` | compare with `if`: no; with `while`: no (the lengths differ) |
| 7 | | `whiley` | return `ID` |

Now the call that produced `if if`: steps 2 to 5 collect `if` and stop at `;`, which goes back; step 6 finds `if` equal to the first row; the function returns `IF`. Same code, different table result.

### When it goes wrong

**Keywords come out as `id`.** You compared before ending the string, so `lexeme` still had the tail of a longer, earlier name in it: `while` after `whiley` would be compared as `whiley`. The line `lexeme[n] = '\0';` must come before the lookup.

**`whiley` comes out as `while` and `id y`.** The scanner checked for keywords during reading, or compared only the first few characters (with `strncmp` and the keyword's length). Read the whole name first, compare it whole.

**`if_x` comes out as three tokens.** The underscore is missing from one of the two tests. It must be in both: a C name may start with `_` and may contain it.

**Every token after `while` prints the wrong name.** The `enum` and the `name` array are out of step: one has a token the other lacks, or they are in a different order. Write them one under the other and count.

### In one line each

1. Read the longest run that fits the identifier pattern, push the lookahead back, then look the name up.
2. A name becomes a keyword only if it equals a row of the keyword table exactly; otherwise it is `id`.
3. The scanner is a function, `gettoken`, that returns one token name per call and leaves the characters in `lexeme`.
4. Token names are numbers from an `enum`; a parallel array of strings prints them.

**Check yourself**

<!-- q: idea=keyword-table type=predict wrong="ifwhile is two keywords" -->
1. Predict what `idkw` prints for `if ifwhile While _if;`.

<details>
<summary>Answer</summary>

```text
if     if
id     ifwhile
id     While
id     _if
other  ;
```

`ifwhile` is read whole, eight letters, and matches no row, so it is one identifier, not two keywords. `While` differs from `while` in its first character, so it is an identifier. `_if` starts with an underscore, which is allowed, and as a whole it is not a keyword. Only the first `if` is exactly a row of the table.

</details>

<!-- q: idea=keyword-table type=fix wrong="strncmp with the keyword's length is the same as strcmp" -->
2. A student's lookup reads `if (strncmp(lexeme, keyword[i].word, strlen(keyword[i].word)) == 0)`. Find an input on which the scanner gives the wrong token, and say what it prints.

<details>
<summary>Answer</summary>

Any identifier that begins with a keyword: `iffy`, `whiley`, `if_x`. `strncmp` compares only as many characters as the keyword has, so for `iffy` it compares `if` with `if`, says equal, and the scanner prints `if     iffy`: the keyword token with the whole name as its lexeme. The parser would then see an `if` statement where the programmer wrote a variable. `strcmp` also checks that both strings end at the same place, which is what "the whole name equals the keyword" means.

</details>

<!-- q: idea=identifier-scan type=apply -->
3. Tiny is to get a third keyword, `print`, with its own token. Write down every line of `idkw.c` that changes, and say what `idkw` will print for `print printer;` afterwards.

<details>
<summary>Answer</summary>

Three places change. The enum gets a new name, for example `enum { ID, IF, WHILE, PRINT, OTHER, END };`. The `name` array gets `"print"` in the same position: `{ "id", "if", "while", "print", "other", "end" }`. The table gets a row `{ "print", PRINT },` and `NKEY` becomes 3. Then:

```text
print  print
id     printer
other  ;
```

If you added the row to the table but not to the enum and the `name` array, it will not even compile, because `PRINT` is not defined; if you added `PRINT` to the enum in a different position from `"print"` in `name`, the output prints the wrong names.

</details>

## 2.3 Numbers, operators and the longest-match rule

### Two characters that are one token

`idkw` from section 2.2 still prints the 60 of the running statement as `other 6` and `other 0`, two lexemes where there is one. Operators have the same problem the other way round. Chapter 9 gives Tiny `if` and `while`, and their conditions need `<=`, `>=`, `==` and `!=`. Each of those is two characters and one token. But `<` on its own is a token too, and so is `=`. When the scanner reads `<`, it cannot yet know whether it has the token `<` or the first half of `<=`.

It is the problem of section 2.1 again, in a new place. And it comes with a rule you will need every time two patterns could both match: when `<` and `<=` both fit, which wins? This section answers that, adds numbers and every Tiny operator, and gives you the scanner that later chapters build on.

### Read as far as you can

You already apply the rule when you read English. Faced with the word "therefore", you do not stop after "the" because "the" is a word; you read on as long as the letters still make a word, and take the longest one. A scanner does the same. At each point it reads the longest run of characters that still forms a lexeme of some token, and only then stops.

<!-- teaches: longest-match = longest match, maximal munch -->
That is the *longest-match rule*, also called *maximal munch*: the next lexeme is always the longest prefix of the remaining input that matches some pattern. It is why `whiley` in section 2.2 was one identifier, why `60` is one number, and why `<=` is one token. When two patterns match the same longest lexeme, as `while` matches both the keyword and the identifier pattern, a second rule chooses between them; in this scanner that rule is the keyword table.

The English picture breaks in an important way. A human reader who runs into nonsense backs up and tries a different split. A scanner never does. It takes the longest lexeme it can, hands it on, and moves forward, even if a shorter choice would have let the program make sense. Your own compiler shows this. Here `a+++++b` could be read as `a++ + ++b`, which is a correct C expression, but GCC's scanner reads greedily, `a`, `++`, `++`, `+`, `b`, and the parser is then handed `a++ ++`, which is nonsense:

<!-- code: id=munch mode=error -->
```c
int main(void)
{
    int a = 1, b = 2, c;

    c = a+++++b;
    return c;
}
```
```text
munch.c: In function 'main':
munch.c:5:12: error: lvalue required as increment operand
    5 |     c = a+++++b;
      |            ^~
```

The caret points at the second `++`. It is the parser and semantic analyzer complaining, but it was the scanner that cut the characters that way, without looking back. Written with spaces, `a++ + ++b`, the same expression compiles. This is also a favourite viva and paper question: "how is `x+++y` tokenised?" Answer by the rule: `x`, `++`, `+`, `y`.

### Numbers in Tiny

<!-- teaches: number-scan = number scanning, unsigned number -->
Tiny's numbers are unsigned: one or more digits, then optionally a decimal point followed by one or more digits. So `60`, `100` and `2.5` are numbers; `-7` is the operator `-` followed by the number `7`, which is how C sees it too, and the parser puts them together. *Number scanning* is identifier scanning with a different alphabet and one decision in the middle:

1. Read digits for as long as they come.
2. If the lookahead is not a `.`, the number is complete: push the lookahead back.
3. If it is a `.`, keep it and read one more character. A digit means there is a fraction: read digits for as long as they come, then push back the character that ended them.
4. Anything else after the `.` breaks Tiny's rule: `5.` is not a number.

Why forbid `5.`, which C allows? Because of section 2.1's limit. Suppose Tiny accepted `5` followed by some token that begins with a dot. Then on `5.x` the scanner would read `5`, `.` and `x` before it could know that the number was only `5`, and it would have to give back two characters, the `.` and the `x`, where C promises only one. A language designer who writes "a digit must follow the point" makes the scanner easy. Tiny's rule costs the programmer one character (`5.0`) and saves the scanner a buffer. C's own numbers also allow an exponent, as in `6.02e23`; Tiny leaves it out, and the Dragon book's diagram for "unsigned number" with an exponent appears in section 2.5's questions.

### Operators: one switch and one helper

Single-character operators need no lookahead at all: a `+` is complete the moment it is read. So they go into one `switch`. The four two-character operators share one shape: read the first character; then read one more; if it is `=`, both characters form the token; if not, push it back and the first character alone is the token. That shape is a small function, `two`, used four times. Its three arguments are the second character it hopes for, the token to return if it comes, and the token to return if it does not.

`!` is the odd one. Tiny has no "not" operator, so `!` alone fits no pattern. The scanner returns the token `ERROR` for it, and for `5.`; section 2.7 makes those errors say what went wrong and where.

### The scanner, version three

Save this as `scan-ops.c`. Everything from `idkw.c` is still there; numbers, the real operator tokens and the helper `add`, which appends one character to `lexeme` and keeps it ended with `'\0'`, are new. The program was given three lines, the running statement, a `while` header with a division, and a line with a mistake in it:

<!-- code: id=scan-ops stdin="pos = init + rate * 60;\nwhile (pos>=100) pos = pos/2.5;\nx = a<b != 5.;\n" -->
```c
#include <stdio.h>
#include <ctype.h>
#include <string.h>

enum { ID, NUM, IF, WHILE, ASSIGN, EQ, NE, LT, LE, GT, GE,
       PLUS, MINUS, TIMES, DIVIDE, LPAREN, RPAREN, SEMI, ERROR, END };

const char *name[] = { "id", "num", "if", "while", "=", "==", "!=",
                       "<", "<=", ">", ">=", "+", "-", "*", "/",
                       "(", ")", ";", "error", "end" };

struct { const char *word; int token; } keyword[] = {
    { "if", IF },
    { "while", WHILE },
};
#define NKEY 2

char lexeme[64];
int len;

void add(int c)                 /* append c to the lexeme */
{
    if (len < 63)
        lexeme[len++] = c;
    lexeme[len] = '\0';
}

int two(int second, int yes, int no)   /* one character of lookahead */
{
    int c = getchar();

    if (c == second) {
        add(c);
        return yes;
    }
    ungetc(c, stdin);
    return no;
}

int gettoken(void)
{
    int c;

    len = 0;
    lexeme[0] = '\0';
    c = getchar();
    while (isspace(c))
        c = getchar();
    if (c == EOF)
        return END;
    if (isalpha(c) || c == '_') {
        while (isalnum(c) || c == '_') {
            add(c);
            c = getchar();
        }
        ungetc(c, stdin);
        for (int i = 0; i < NKEY; i++)
            if (strcmp(lexeme, keyword[i].word) == 0)
                return keyword[i].token;
        return ID;
    }
    if (isdigit(c)) {
        while (isdigit(c)) {
            add(c);
            c = getchar();
        }
        if (c == '.') {
            add(c);
            c = getchar();
            if (!isdigit(c)) {          /* Tiny: a digit must follow '.' */
                ungetc(c, stdin);
                return ERROR;
            }
            while (isdigit(c)) {
                add(c);
                c = getchar();
            }
        }
        ungetc(c, stdin);
        return NUM;
    }
    add(c);
    switch (c) {
    case '+': return PLUS;
    case '-': return MINUS;
    case '*': return TIMES;
    case '/': return DIVIDE;
    case '(': return LPAREN;
    case ')': return RPAREN;
    case ';': return SEMI;
    case '=': return two('=', EQ, ASSIGN);
    case '<': return two('=', LE, LT);
    case '>': return two('=', GE, GT);
    case '!': return two('=', NE, ERROR);
    }
    return ERROR;
}

int main(void)
{
    int t;

    while ((t = gettoken()) != END)
        printf("%-6s %s\n", name[t], lexeme);
    return 0;
}
```
```text
id     pos
=      =
id     init
+      +
id     rate
*      *
num    60
;      ;
while  while
(      (
id     pos
>=     >=
num    100
)      )
id     pos
=      =
id     pos
/      /
num    2.5
;      ;
id     x
=      =
id     a
<      <
id     b
!=     !=
error  5.
;      ;
```

The first eight lines are the token stream of section 1.3, now made by a program: `id = id + id * num ;`. Each token name in the left column is the printed form of a number from the `enum`; the parser in chapter 6 will compare those numbers, never the strings.

### Walking the hard cases

**`pos>=100`, written with no spaces.** The identifier loop reads `p`, `o`, `s` and stops at `>`, which goes back. The next call reads `>`, falls through to the `switch`, adds it to `lexeme` and calls `two('=', GE, GT)`. That reads `=`, which is the hoped-for second character, so it is added and `GE` comes back with `lexeme` holding `>=`. The next call reads `1`, `0`, `0` and stops at `)`, which goes back. Three tokens, no spaces needed, because no identifier character can be part of an operator and no operator character can be part of a number.

**`a<b`.** The call for `<` reads `b` as its lookahead inside `two`. It is not `=`, so `two` pushes `b` back and returns `LT` with `lexeme` holding just `<`. The `b` comes out as the next identifier.

**`2.5`.** Digits loop: `2`, then the lookahead is `.`. Keep it, read `5`: a digit, so the fraction loop reads `5` and stops at `;`, which goes back. `lexeme` is `2.5`.

**`5.;`.** Digits loop: `5`, then `.`, kept. The next character is `;`, not a digit. The `;` is pushed back, and the function returns `ERROR` with `lexeme` holding `5.`. The `;` is still there for the next call, which is why the line ends with a normal `; ;`. One pushback was enough, as promised.

### Change one thing: no lookahead for `=`

Suppose the `case '=':` line simply returned `ASSIGN`, like `+` returns `PLUS`. Then `x==y` would come out as `id x`, `= =`, `= =`, `id y`: two assignments in a row. The scanner would say nothing, because each `=` is a perfectly good lexeme. The error would appear only in the parser, as a syntax error about an unexpected `=`, far from its real cause. A missing lookahead in the scanner always shows up like this: as a strange error one phase later.

### When it goes wrong

**`<=` comes out as `<` and `=`.** The `<` case returns at once instead of calling `two`. Every operator that is the first character of a longer operator needs the lookahead.

**The character after `<` disappears.** In `a<b`, the `b` goes missing: `two` read it and did not push it back when it was not `=`. Every path out of `two` that does not use the character must call `ungetc`.

**`12abc` comes out as `num 12` and `id abc`.** That is what this scanner does, and it is wrong for C, which (as `60x` showed in section 1.3) calls it one malformed number. Section 2.7 adds the check.

**`-5` is one token in your output.** You let the number loop accept a leading `-`. Then `x-5` scans as `id x`, `num -5`, with no operator between them, and the parser sees two operands side by side. Leave signs to the parser.

### In one line each

1. Longest match: the next lexeme is the longest prefix of the input that fits a pattern; the scanner never backs up to try a shorter one.
2. A Tiny number is digits, optionally `.` and more digits; a digit must follow the point, so one character of lookahead is enough.
3. Single-character operators need no lookahead; `<=`, `>=`, `==` and `!=` need one, handled by `two`.
4. A character that fits no pattern gives the token `ERROR`.

**Check yourself**

<!-- q: idea=longest-match type=predict wrong="the scanner splits so that the expression makes sense" -->
1. Using the longest-match rule, how does a C scanner cut `x---y`? And `x----y`? Which of the two does GCC accept, and why?

<details>
<summary>Answer</summary>

`x---y` is `x`, `--`, `-`, `y`: the longest operator at `-` is `--`, and the third `-` is alone. That is `(x--) - y`, a correct expression, so GCC accepts it. `x----y` is `x`, `--`, `--`, `y`, which is `x-- -- y`: decrementing the result of `x--`, which is not a variable, and then a `y` with no operator before it. It is rejected, even though `x-- - -y` would have been correct. The scanner does not look for a split that makes sense; it takes the longest lexeme each time.

</details>

<!-- q: idea=number-scan type=predict wrong="3.14.15 is one bad number" -->
2. Predict the tokens `scan-ops` prints for `y=3.14.15;`. Be careful at the second point.

<details>
<summary>Answer</summary>

```text
id     y
=      =
num    3.14
error  .
num    15
;      ;
```

The number loop reads `3`, the `.`, `1`, `4`, and stops at the second `.`: the fraction loop only takes digits. `3.14` is complete, and the second `.` goes back. The next call reads `.`, which is not a letter, digit or operator Tiny knows, so the `switch` finds no case and the function returns `ERROR` with lexeme `.`. Then `15` and `;`. If you thought `3.14.15` was one bad number, remember that a scanner ends a lexeme as soon as the pattern stops fitting.

</details>

<!-- q: idea=longest-match type=apply -->
3. C has the operators `+`, `++` and `+=`. Write the `case '+':` of a C scanner using one character of lookahead, in the style of `two`, assuming tokens `PLUS`, `INC` and `PLUSEQ`.

<details>
<summary>Answer</summary>

```c
case '+':
    c = getchar();
    if (c == '+') {
        add(c);
        return INC;
    }
    if (c == '=') {
        add(c);
        return PLUSEQ;
    }
    ungetc(c, stdin);
    return PLUS;
```

`two` handles one possible second character; here there are two, so the case is written out. Still only one character is read ahead, and it is pushed back on the one path that does not use it. A common slip is to call `getchar()` once in each `if`, which reads a different character in the second test.

</details>

## 2.4 Skipping white space and both kinds of C comment

### A comment the scanner tried to read

Tiny programs, like C programs, may carry comments of both kinds: `//` to the end of the line, and `/*` up to the next `*/`, possibly over several lines. `scan-ops` from section 2.3 has never heard of them. It is still in `~/cdlab/ch2`; give it a line with a comment:

<!-- code: id=comment-fail -->
```console
$ printf 'rate = 2.5; // per second\n' | ./scan-ops
id     rate
=      =
num    2.5
;      ;
/      /
/      /
id     per
id     second
```

The comment came out as two division signs and two identifiers. A parser handed `/ / per second` would report a syntax error on a line that is perfectly correct.

Section 1.3 said that in C the preprocessor removes comments before the compiler proper sees them. Tiny has no preprocessor, so its scanner must do that job itself, as the scanners of most languages do. This section writes the code that does it, first as a program of its own (removing comments from a file is a lab question in its own right), then as the function the scanner calls before every token.

### Blanks and notes in the margin

On the assembly line, picture the scanner station sweeping its bench clear before each cut: blanks, tabs and newlines go in the bin, and so do notes someone scribbled in the margin. What is left on the bench is the first character of the next lexeme.

The picture is right that nothing of a comment reaches the parser. It breaks in one place: a comment is not nothing. It still separates the lexemes on either side of it. In C, `a/**/b` is two identifiers, `a` and `b`, never the one name `ab`; the C standard says each comment is replaced by one space. So the sweeping must leave a gap where the note was, and in a scanner it does, automatically, because the scanner ends the lexeme `a` when it meets the `/`, before it ever finds out a comment starts there.
<!-- checked: how=known date=2026-11-21 ref="C11 5.1.1.2p1 phase 3: each comment is replaced by one space character" h=062ea806 -->

### How the scanner tells a comment from a division

<!-- teaches: comment-skip = comment skipping, line comment, block comment -->
*Comment skipping* starts from one character that is ambiguous: `/` may be the division operator, or the first half of `//` (a *line comment*) or `/*` (a *block comment*). One character of lookahead decides, just as it did for `<` and `<=`:

| After `/` comes | It is | What the scanner does |
|---|---|---|
| `/` | a line comment | read and drop characters up to the newline (or the end of the input) |
| `*` | a block comment | read and drop characters up to the first `*/` |
| anything else | a division | push that character back; `/` is a token |

A line comment is easy to end: the newline. A block comment is harder, because its end is two characters, `*` then `/`, and the scanner sees one at a time. It must remember whether the character it saw last was a `*`. Think of it as two squares on a board. On the square "inside" every character keeps you there, except `*`, which moves you to the square "just saw a star". On that square, `/` ends the comment, another `*` keeps you on the same square, and anything else sends you back to "inside". A variable called `prev`, holding the previous character, is that board in C: the comment ends when `prev` is `*` and the new character is `/`.

Two rules of C come with block comments, and Tiny keeps them. They do not nest: `/* one /* two */ c */` ends at the first `*/`, and ` c */` is left over as ordinary text. And a block comment that never ends runs to the end of the file, which is a mistake in the program the scanner should report (section 2.7 does; here the rest of the file is simply eaten).

### A comment remover

Save this as `decomment.c`. It copies its input to its output, leaving out the comments: a line comment is dropped up to its newline, which is kept; a block comment becomes one space, and any newlines inside it are kept, so that the line numbers of the code after it do not change. It was given this five-line Tiny program:

```text
/* Tiny: the running statement */
init = 10;      // starting value
rate = 2.5; /* per
               second */ pos = init + rate * 60;
x = a/**/b / c;  /* rate **/ y = 1;
```

<!-- code: id=decomment stdin="/* Tiny: the running statement */\ninit = 10;      // starting value\nrate = 2.5; /* per\n               second */ pos = init + rate * 60;\nx = a/**/b / c;  /* rate **/ y = 1;\n" -->
```c
#include <stdio.h>

int main(void)
{
    int c, d, prev;

    while ((c = getchar()) != EOF) {
        if (c != '/') {
            putchar(c);
            continue;
        }
        d = getchar();
        if (d == '/') {                 /* a // comment: drop it up to the newline */
            while ((c = getchar()) != '\n' && c != EOF)
                ;
            if (c == '\n')
                putchar('\n');
        } else if (d == '*') {          /* a block comment becomes one space */
            prev = 0;
            while ((c = getchar()) != EOF && !(prev == '*' && c == '/')) {
                if (c == '\n')
                    putchar('\n');      /* keep the line count */
                prev = c;
            }
            putchar(' ');
        } else {                        /* a division sign: give d back */
            putchar(c);
            ungetc(d, stdin);
        }
    }
    return 0;
}
```
```text

init = 10;
rate = 2.5;
  pos = init + rate * 60;
x = a b / c;    y = 1;
```

### Walking through the five lines

**Line 1** is one block comment. The `/` is read, the lookahead `d` is `*`, and the inner loop reads `T`, `i`, `n` ... with `prev` following one character behind. At ` */` the loop sees `prev == '*'` and `c == '/'` and stops. One space is printed for the whole comment. The newline after it is ordinary text and is copied, so line 1 of the output is a single space and line 2 is still line 2.

**Line 2** ends in a line comment. At `//` the inner loop drops `starting value` and stops at the newline, which is printed. The spaces before the comment are ordinary text and stay.

**Lines 3 and 4** hold one block comment that crosses a line. The newline inside it is printed (so `pos = ...` stays on line 4, where the programmer wrote it); the rest of the comment is dropped and replaced by one space, which is why line 4 starts with two spaces: the space for the comment and the space the programmer typed after `*/`.

**Line 5** has the two hard cases. `a/**/b` becomes `a b`: the comment `/**/` has nothing inside it, and the loop still ends correctly, because after `/*` it reads `*` (with `prev` still 0, so no end yet) and then `/` with `prev == '*'`. The `/ c` is a division: after the `/` the lookahead is a space, which is neither `/` nor `*`, so the `/` is printed and the space is pushed back to be copied next. Last, `/* rate **/`: on the first `*` `prev` becomes `*`; on the second `*` it stays `*`; on the `/` the comment ends.

### The tempting loop that fails

Most first attempts at the block comment look like this: read until a `*`, then check that the next character is `/`. Save it as `badcomment.c`; it handles only block comments, which is all it needs to show the fault. It was given two lines, `x = 1; /* rate **/ y = 2;` and `z = 3;`:

<!-- code: id=badcomment stdin="x = 1; /* rate **/ y = 2;\nz = 3;\n" -->
```c
#include <stdio.h>

int main(void)
{
    int c, d;

    while ((c = getchar()) != EOF) {
        if (c == '/') {
            d = getchar();
            if (d == '*') {
                while ((c = getchar()) != EOF)
                    if (c == '*' && getchar() == '/')
                        break;
                putchar(' ');
                continue;
            }
            ungetc(d, stdin);
        }
        putchar(c);
    }
    return 0;
}
```
```text
x = 1;
```

Everything after the comment is gone, both lines of it, and no message says why. Walk the inner loop over `**/`. The first `*` passes the test `c == '*'`, so `getchar()` reads the next character to compare with `/`. That character is the second `*`; it is not `/`, so the loop goes on, but the second `*` has now been read and thrown away. The next character is `/`, and `c == '*'` is false for it. The `*/` was there, but the loop looked at its two halves in two different rounds. From then on it hunts for a `*/` that never comes, and swallows the rest of the file. The `prev` version cannot do this: every character is looked at once, as `c`, and then kept as `prev` for the next round.

The fault is worth remembering because it survives casual testing: `/* rate */` works perfectly in `badcomment`. Only a comment ending in two or more stars breaks it, and such comments are common: `/*****/` banners at the top of every lab file end that way.

### The same loop, inside the scanner

In the scanner, comment skipping is a function. It sweeps away white space and comments and returns the first character of the next lexeme, which `gettoken` then classifies exactly as before. It is the comment remover with `putchar` taken out and a `return` where the remover would print:

<!-- code: id=skip-sketch mode=fragment -->
```c
int skip(void)     /* skip blanks and comments; return the next character */
{
    int c, prev;

    for (;;) {
        c = getchar();
        if (isspace(c))
            continue;
        if (c != '/')
            return c;
        c = getchar();
        if (c == '/') {                 /* a // comment runs to the newline */
            while (c != '\n' && c != EOF)
                c = getchar();
        } else if (c == '*') {          /* a block comment runs to the first */
            prev = 0;
            c = getchar();
            while (c != EOF && !(prev == '*' && c == '/')) {
                prev = c;
                c = getchar();
            }
        } else {                        /* just a division sign */
            ungetc(c, stdin);
            return '/';
        }
    }
}
```

`gettoken` starts with `c = skip();` in place of its white-space loop, and the `case '/'` in its `switch` stays as it is: when `skip` returns `/`, it really is a division. The `for (;;)` matters: after one comment there may be white space, then another comment, then more white space, and `skip` keeps sweeping until it meets something that is none of these. Section 2.7 puts this function into the finished scanner, with two additions that need its newlines: line counting, and an error for a comment that is never closed.

### When it goes wrong

**Everything after a comment vanishes.** Either the comment was never closed (the scanner then rightly eats the rest; section 2.7 makes it say so), or your loop is the `badcomment` loop, which misses `**/`. Test with `/***/` and `/* x **/`.

**`a/**/b` comes out as one identifier `ab`.** Only possible in a comment remover that prints nothing for a block comment. Print one space. A scanner does not have this problem, as you saw.

**Division signs disappear.** After reading `/` and a character that is neither `/` nor `*`, the code forgot to push that character back, so `x / y` loses its space (harmless) and `x/y` loses the `y` (not harmless). The same lookahead rule as `two` in section 2.3.

**Line numbers are wrong after a comment.** The comment remover dropped the newlines inside a block comment. Keep them, or every line number reported after the comment is too small.

**C only: `"http://example"` loses half a string.** C has string literals, and `//` inside quotes is not a comment. A scanner must recognise the string first and read it whole, so that its characters never reach the comment test. Tiny has no strings; the C subset in section 2.9 is where it matters.

### In one line each

1. After `/`, one character of lookahead decides: `/` starts a line comment, `*` a block comment, anything else is division.
2. A line comment ends at the newline; a block comment at the first `*/`, found by remembering the previous character.
3. Block comments do not nest, and one that never ends eats the rest of the file.
4. In the scanner, `skip` sweeps blanks and comments in a loop and returns the first character of the next lexeme.

**Check yourself**

<!-- q: idea=comment-skip type=predict wrong="nested comments end at the matching */" -->
1. Predict what `decomment` prints for `a = b /* one /* two */ c */ ;`. Then say what `gettoken` (with `skip`) would hand on for the same line.

<details>
<summary>Answer</summary>

```text
a = b   c */ ;
```

The comment starts at the first `/*` and ends at the first `*/`; the second `/*` is just text inside it. What follows, ` c */ ;`, is ordinary text and is printed. A scanner with `skip` would hand on `id a`, `= =`, `id b`, `id c`, `* *`, `/ /`, `; ;`: the left-over `*/` is not a comment end any more, just two operators, and the parser then rejects the line. If you expected the whole line up to the last `*/` to vanish, you assumed comments nest; in C and Tiny they do not.

</details>

<!-- q: idea=comment-skip type=fix wrong="badcomment fails on every comment" -->
2. Which of these comments does `badcomment` handle correctly, and which make it swallow the rest of the file: `/* a */`, `/**/`, `/***/`, `/* a * b */`? Explain using the rule "each character must be tested once".

<details>
<summary>Answer</summary>

Three are handled: `/* a */`, `/**/` and `/* a * b */`. Only `/***/` swallows the rest. Walk `/**/`: after `/*` the loop reads `*` as `c`, the test's `getchar()` reads `/`, and the comment ends correctly. In `/* a * b */` the `*` after `a` makes the test read the space after it; that space is thrown away untested, which does no harm. In `/***/` the loop reads the first `*` as `c`, the test reads the second `*` (not `/`) and throws it away, and the next round reads `/` as `c`, which is not `*`. The `*/` was there, but its `*` was read by the test's `getchar()` and never tested as `c`. So the comments that break the loop are exactly those with two or more stars straight before the closing `/`. If you said `/**/` fails too, count again: it has only one star before its `/`.

</details>

<!-- q: idea=comment-skip type=decide -->
3. A classmate's scanner counts lines by counting `\n` characters in `gettoken` only, after `skip` has returned. Will its line numbers be right for code that follows a multi-line block comment? What is the better place to count?

<details>
<summary>Answer</summary>

No. Every newline inside a comment, and every newline that is plain white space, is read and thrown away inside `skip`, where the classmate is not counting, so the count stays too low and every later error is reported on the wrong line. Newlines are only ever seen by whatever reads characters. So the count belongs in the one function that reads characters: wrap `getchar` in a small function that adds one to the line count whenever it returns `\n`, and have `skip` and `gettoken` both call it. That is exactly what section 2.7 does.

</details>

## 2.5 Transition diagrams: drawing the scanner before writing it

### A scanner you can see

`gettoken` has grown a case at a time over three sections, and each case was found by thinking hard about one kind of lexeme. In the lab exam there is no time to think that hard while typing, and in the written paper there is no program at all: the question says "draw the transition diagram for relational operators" or "for identifiers and unsigned numbers", and the marks go to a drawing. Both needs have one answer. Before writing the code, draw what the scanner does, one picture per kind of token: the pattern of section 1.3 for that token name, drawn so that a program can follow it step by step. Then the code is a copy of the picture.

A drawing also finds mistakes that code hides. Look at `scan-ops` from section 2.3 and ask: after a `!`, what happens if the next character is not `=`? The answer is buried in one argument of a call to `two`. In a drawing it is a missing arrow, and you see it at once.

### The board game again

The book's picture for this is a board game. Each square of the board says, for every card you might turn up next, which square that card sends you to. You start on the square marked "start" and turn up one card at a time. Some squares are winning squares: landing there ends the game with a prize, which here is a token. In a scanner the cards are the input characters, and turning up a card is `getchar`.

<!-- teaches: transition-diagram = transition diagram, state, edge, accepting state, retract -->
A *transition diagram* is that board, drawn with a few fixed conventions:

- A circle is a *state*: a point in the scanning of a lexeme, which remembers everything that matters about the characters read so far. The state marked by an arrow labelled "start" is where every lexeme begins.
- An arrow from one state to another is an *edge*. Its label says which characters take that edge: one character (`=`), a class of characters (`letter`, `digit`), or `other`, meaning "any character that no other edge out of this state names".
- A double circle is an *accepting state*: reaching it means a lexeme has been found, and the state is labelled with the action, usually "return this token".
- A star `*` beside an accepting state means *retract*: the last character read was the lookahead and does not belong to the lexeme, so it goes back to the input. In C, a star is an `ungetc`.

The picture breaks in two places, and both lead to chapter 3. A board game lets you be on one square at a time, and the boards in this section have exactly one arrow for each card from each square, so you always know where to go. Chapter 3 meets boards where one card has two arrows, and you stand on several squares at once. And a scanner has one board per kind of token here, tried one after another; chapter 3 shows how to join all of them into a single board, and section 3.6 drives such a board from a table instead of from code.

### Diagram for Tiny's operators

Here is the board for the seven Tiny tokens that start with `<`, `>`, `=` or `!`. In the drawing a highlighted box is an accepting state (the book's figures have no double circles), and a star in its label means retract.

<!-- fig: id=relop-diagram
title: Transition diagram for < <= > >= = == !=
kind: flow
dir: right
round s0: 0\nstart
box s1: 1
box s2: 2\nreturn LE
box s3: 3 *\nreturn LT
box s4: 4
box s5: 5\nreturn GE
box s6: 6 *\nreturn GT
box s7: 7
box s8: 8\nreturn EQ
box s9: 9 *\nreturn ASSIGN
box s10: 10
box s11: 11\nreturn NE
s0 -> s1: <
s1 -> s2: =
s1 -> s3: other
s0 -> s4: >
s4 -> s5: =
s4 -> s6: other
s0 -> s7: =
s7 -> s8: =
s7 -> s9: other
s0 -> s10: !
s10 -> s11: =
hot: s2 s3 s5 s6 s8 s9 s11
-->
![Flow chart, left to right. 0 start leads to 1 (<). 0 start leads to 4 (>). 0 start leads to 7 (=). 0 start leads to 10 (!). 1 leads to 2 return LE (=). 1 leads to 3 * return LT (other). 4 leads to 5 return GE (=). 4 leads to 6 * return GT (other). 7 leads to 8 return EQ (=). 7 leads to 9 * return ASSIGN (other). 10 leads to 11 return NE (=). 2 return LE, 3 * return LT, 5 return GE, 6 * return GT, 8 return EQ, 9 * return ASSIGN and 11 return NE are highlighted.](fig/02-lexical-analyzer/relop-diagram.svg)

*Figure 2.5a: Tiny's relational and assignment operators as a transition diagram. Highlighted states accept; a star means the last character read is pushed back.*

Read it as the scanner would, on `<=`. Start in state 0. The card `<` sends you to state 1. The card `=` sends you to state 2, which accepts: return `LE`. No star, because both characters belong to the lexeme. Now on `<b`: state 0, card `<` to state 1, card `b`: there is no `=` edge for it, so it takes `other` to state 3, which returns `LT` and has a star, because the `b` was read only to find out that the lexeme had ended. Push it back.

State 10 is the place the drawing helps. Its only edge is `=`. A `!` followed by anything else has nowhere to go, and that is the drawing's way of saying that the input fits no pattern: no edge, no token. Section 2.7 decides what the scanner should say then. Draw the diagram for a new token, look at every state, and ask what happens on each character with no edge. That one question finds most scanner bugs before they are typed.

### Diagrams for identifiers and numbers

The other two diagrams the paper asks for are identifiers and numbers. The numbering carries on from Figure 2.5a, as the Dragon book does, so that every state in the scanner has its own number.

<!-- fig: id=id-num-diagram
title: Transition diagrams for identifiers and Tiny numbers
kind: flow
dir: right
round s12: 12\nstart
box s13: 13\nloop on letter, digit or _
box s14: 14 *\nreturn lookup()
round s15: 15\nstart
box s16: 16\nloop on digit
box s17: 17
box s18: 18 *\nreturn NUM
box s19: 19\nloop on digit
box s20: 20 *\nreturn NUM
note e: no other edge out of 17: `5.` is an error
s12 -> s13: letter or _
s13 -> s14: other
s15 -> s16: digit
s16 -> s18: other
s16 -> s17: .
s17 -> s19: digit
s19 -> s20: other
s17 ..> e
hot: s14 s18 s20
-->
![Flow chart, top to bottom. 12 start leads to 13 loop on letter, digit or _ (letter or _). 15 start leads to 16 loop on digit (digit). 13 loop on letter, digit or _ leads to 14 * return lookup() (other). 16 loop on digit leads to 18 * return NUM (other). 16 loop on digit leads to 17 (.). 17 leads to 19 loop on digit (digit). 17 leads to no other edge out of 17: 5. is an error by a dashed arrow. 19 loop on digit leads to 20 * return NUM (other). 14 * return lookup(), 18 * return NUM and 20 * return NUM are highlighted.](fig/02-lexical-analyzer/id-num-diagram.svg)

*Figure 2.5b: Identifiers (states 12 to 14) and Tiny's unsigned numbers (states 15 to 20). A state labelled "loop on" keeps you on the same square for as long as those cards come; other books draw it as an arrow from the circle back to itself.*

Compare state 13 with the identifier loop in `gettoken`. The loop on state 13 is the `while (isalnum(c) || c == '_')` loop; the edge `other` to 14 is the loop ending; the star on 14 is the `ungetc` after the loop; and "return lookup()" is the keyword table. The number diagram is the number code of section 2.3 drawn out: state 16 is the digit loop, state 17 is "just read a point", state 19 is the fraction loop. State 17 has no `other` edge, which is Tiny's rule "a digit must follow the point".

### From the drawing to C: one case per state

There are two ways to turn a diagram into code, and you have already used the first. In `gettoken`, a state is a place in the program: a loop for a state with an edge to itself, an `if` for a choice between edges. It is short, but the states are not written down anywhere, so it is easy to lose one.

The second way writes the states down. One variable, `state`, holds the number of the square you are on. A loop repeats a `switch (state)`; each `case` is one circle of the diagram. A case that is not accepting reads one character and sets `state` to wherever that character's edge leads. An accepting case returns its token, after an `ungetc` if it has a star. Nothing needs to be invented: you go round the diagram circle by circle and copy.

Save this as `relop.c`. Its `relop` function is Figure 2.5a, case for case; `main` feeds it the characters of its input, skipping blanks, and prints each token. It was given the line `<= < >= > == = != !x<5`:

<!-- code: id=relop stdin="<= < >= > == = != !x<5\n" -->
```c
#include <stdio.h>

/* Figure 2.5a in C: one case per state, one variable for the state */
const char *relop(void)
{
    int state = 0, c = 0;

    for (;;) {
        switch (state) {
        case 0:
            c = getchar();
            if (c == '<')
                state = 1;
            else if (c == '>')
                state = 4;
            else if (c == '=')
                state = 7;
            else if (c == '!')
                state = 10;
            else {
                ungetc(c, stdin);       /* no arrow for c: not ours */
                return NULL;
            }
            break;
        case 1:
            c = getchar();
            state = (c == '=') ? 2 : 3;
            break;
        case 2:
            return "LE";
        case 3:
            ungetc(c, stdin);           /* the * on state 3: retract */
            return "LT";
        case 4:
            c = getchar();
            state = (c == '=') ? 5 : 6;
            break;
        case 5:
            return "GE";
        case 6:
            ungetc(c, stdin);
            return "GT";
        case 7:
            c = getchar();
            state = (c == '=') ? 8 : 9;
            break;
        case 8:
            return "EQ";
        case 9:
            ungetc(c, stdin);
            return "ASSIGN";
        case 10:
            c = getchar();
            if (c != '=') {
                ungetc(c, stdin);       /* no "other" arrow out of 10 */
                return "ERROR";
            }
            state = 11;
            break;
        case 11:
            return "NE";
        }
    }
}

int main(void)
{
    int c;
    const char *t;

    while ((c = getchar()) != EOF) {
        if (c == ' ' || c == '\n')
            continue;
        ungetc(c, stdin);
        t = relop();
        if (t == NULL)
            printf("%c: no arrow from state 0\n", getchar());
        else
            printf("%s\n", t);
    }
    return 0;
}
```
```text
LE
LT
GE
GT
EQ
ASSIGN
NE
ERROR
x: no arrow from state 0
LT
5: no arrow from state 0
```

### Walking `!x<5` through the cases

The `!` takes state 0 to state 10. In case 10 the next character is `x`, which is not `=`. There is no edge, so the function pushes `x` back and returns `ERROR`. Back in `main`, the next character is that same `x`. It goes back too, and `relop` is called; case 0 has no edge for `x`, pushes it back and returns `NULL`, and `main` reads it and reports it. Then `<` goes 0 to 1; in case 1 the next character is `5`, not `=`, so state 3; case 3 pushes the `5` back and returns `LT`. Last, the `5` has no edge from state 0.

Look at the stars once more. Only states 3, 6 and 9 push back, the three states the diagram marks with a star. If you put an `ungetc` in case 2 as well, `<=` would return `LE` and then the `=` would be read again as the start of the next lexeme, giving `LE` followed by `ASSIGN`. If you leave the `ungetc` out of case 3, `a<b` loses its `b`. The stars in the drawing are exactly the `ungetc` calls in the code, no more and no fewer.

Why bother with this longer style when `two` did the same job in three lines? In the lab, because it can be written from the drawing without a single decision, under time pressure, and an examiner can check it state by state against the drawing on your answer sheet. And because it is the first step to section 3.6, where the `switch` is replaced by a table with one row per state and one column per kind of character, and the same loop runs any diagram you put in the table.

### When it goes wrong

**A state with no answer for some character.** You drew `!` to state 10 and `=` to 11 and stopped. In the code, case 10 then has no plan for `!x`. Decide for every state: which characters have edges, and is "no edge" an error or an `other` edge to an accepting state?

**A star in the wrong place.** A star belongs only on an accepting state reached by an `other` edge (or any edge whose character is not part of the lexeme). Stars on states reached by `=` push back a character that belongs to the token.

**Two edges with the same label out of one state.** Then the board does not say where to go. For the scanners of this chapter that is a mistake in the drawing; chapter 3 shows what such boards mean and how to remove the choice.

**The paper's operator set is not Tiny's.** The Dragon book's own relop diagram is for a Pascal-like language: `=` is equality, `<>` is not-equal, and there is no `==` or `!=`. Draw the operators the question lists, not the ones you remember.

### In one line each

1. A transition diagram is a board of states and edges; the scanner starts at "start" and follows the edge for each character.
2. An accepting state returns a token; a star on it means the last character is pushed back.
3. A character with no edge out of the current state means the input fits no pattern (unless an `other` edge exists).
4. In C, one variable holds the state and one `case` copies each circle; the stars are exactly the `ungetc` calls.

**Check yourself**

<!-- q: idea=transition-diagram type=apply wrong="= and <> need no lookahead" -->
1. Draw the Dragon book's transition diagram for `relop` in a language whose relational operators are `<`, `<=`, `<>`, `=`, `>` and `>=` (`=` here means equality). Number the states from 0 and mark the stars.

<details>
<summary>Answer</summary>

| State | Edges | Accepts |
|---|---|---|
| 0 (start) | `<` to 1, `=` to 5, `>` to 6 | |
| 1 | `=` to 2, `>` to 3, other to 4 | |
| 2 | | return LE |
| 3 | | return NE |
| 4 * | | return LT |
| 5 | | return EQ |
| 6 | `=` to 7, other to 8 | |
| 7 | | return GE |
| 8 * | | return GT |

The two states with stars are 4 and 8, the ones reached by `other`. State 5 has no star and no edges out: `=` is complete as soon as it is read, because in this language no operator starts with `=` and goes on, so no lookahead is needed. Tempting slips: drawing a lookahead after `=` out of habit from C, and putting a star on state 3 (`<>`: both characters belong to the lexeme).

</details>

<!-- q: idea=transition-diagram type=fix wrong="the extra ungetc is harmless" -->
2. A student copies Figure 2.5a into C but writes `case 8: ungetc(c, stdin); return "EQ";`. What does the program print for `a==b` when its scanner calls this function at the `=`, and which token appears that the programmer never wrote?

<details>
<summary>Answer</summary>

At the `=`, state 0 goes to 7; the next character is the second `=`, so state 8. Case 8 now pushes that second `=` back and returns `EQ`. The next call reads the pushed-back `=` again: state 7, then the lookahead `b`, which is other, so state 9 pushes `b` back and returns `ASSIGN`. The stream is `id a`, `EQ`, `ASSIGN`, `id b`: an extra assignment token. State 8 has no star in the drawing because the second `=` is part of the lexeme `==`; pushing it back makes it count twice.

</details>

<!-- q: idea=transition-diagram type=apply -->
3. C's unsigned numbers may have an exponent: `60`, `2.5`, `6.02E23`, `1E-9`, `5E+3`. Extend Figure 2.5b's number diagram (states 15 to 20) to accept an optional exponent part: `E`, then an optional `+` or `-`, then one or more digits. Which new states need a star?

<details>
<summary>Answer</summary>

Keep 15, 16, 17 and 19. From 16 and from 19 add an edge `E` to a new state 21. From 21: `+` or `-` to 22, `digit` to 23. From 22: `digit` to 23. From 23: `digit` back to 23, `other` to a new accepting state 24, which returns NUM with a star. States 18 and 20 (other out of 16 and 19) keep their stars. So the stars are on 18, 20 and 24: every accepting state reached by `other`. States 21 and 22 have no `other` edge: `5E` and `5E+` followed by a non-digit are errors. This is the Dragon book's "unsigned number" diagram, except that it lets a number end right after its integer part too.

</details>

## 2.6 Input buffering: buffer pairs and sentinels

Your scanner calls `getchar` once per character, and section 2.1 admitted that the belt behind it is not a belt: the C library reads the file in large blocks and hands out characters from memory. A compiler that must be fast does this itself, and the paper asks how: "explain input buffering with buffer pairs and sentinels".

<!-- teaches: buffer-pair = buffer pair, lexemeBegin, forward -->
A *buffer pair* is one array of 2N characters used as two halves of N each, where N is a disk block (4096 is common). Each half is filled with one read from the file. Two pointers move through it: `lexemeBegin` marks the first character of the current lexeme, and `forward` runs ahead until a lexeme is found. When `forward` reaches the end of one half, the other half is refilled with the next block and `forward` carries on there. Two halves, not one, so that a lexeme crossing the end of a block is still whole in memory: the old half is not overwritten until the scanner has finished with it.
<!-- checked: how=known date=2026-11-21 ref="text says common, not fixed; 4096 is the usual ext4 block and x86-64 page size" h=d77f0e20 -->

Trace it with N = 8 on the running statement. The first half holds `pos = in`, the second `it + rat`. The scanner is at `init`: `lexemeBegin` points at the `i` in position 6, and `forward` moves to 7 (`n`), then to 8, the end of the first half. The second half is already loaded, so `forward` goes on to `i`, `t`, and then the space, which ends the lexeme `init`, spread over both halves. Next, at `rate`, `forward` runs off the end of the second half; the first half is reloaded with `e * 60;` and `forward` wraps round to the start of the array. This only works while a lexeme is shorter than N: a name longer than one half would need the half that `lexemeBegin` still points into.

<!-- teaches: sentinel = sentinel -->
Now count the tests. Each time `forward` moves, the scanner must ask two questions: "is this the end of a half?" and "which character is it?". The trick is a *sentinel*: one extra place at the end of each half holding a special character (call it `eof`) that cannot appear in a program. Then the scanner asks only "which character is it?", and only when the answer is `eof` does it ask why: the end of a half, or the real end of the input.

<!-- code: id=sentinel-sketch mode=fragment -->
```c
switch (*forward++) {
case EOF_MARK:                       /* the sentinel */
    if (forward == end_of_first_half + 1) {
        reload_second_half();
        forward = start_of_second_half;
    } else if (forward == end_of_second_half + 1) {
        reload_first_half();
        forward = start_of_first_half;
    } else {
        /* a sentinel inside a half: the real end of the input */
        return END;
    }
    break;
/* cases for the other characters */
}
```

That is the Dragon book's loop, in C. In the common case, an ordinary character, it costs one test instead of two, and a scanner runs this loop for every character of every program.

**The usual mistake** in the paper is to say the sentinel marks only the end of the file. It marks the end of each half; the end of the input is a sentinel found somewhere else.

**Check yourself**

<!-- q: idea=sentinel type=predict wrong="the sentinel removes the end-of-buffer check altogether" -->
1. A file of 100 characters is scanned with N = 8. Without sentinels, about how many "end of half?" tests are made? With sentinels, how many times is the "which half ended?" question asked?

<details>
<summary>Answer</summary>

Without sentinels, one end test for every move of `forward`: about 100 (more, counting the characters read again after a pushback). With sentinels, the extra question is asked only when an `eof` is met: once at the end of each filled half, 100 / 8 rounded up, so 13 halves, plus once for the real end, about 13 or 14 times. The check is not removed; it moves from every character to every block.

</details>

<!-- q: idea=buffer-pair type=explain wrong="one buffer would do" -->
2. Why does a buffer pair use two halves instead of one buffer of N refilled whenever it is used up?

<details>
<summary>Answer</summary>

With one buffer, refilling it when `forward` reaches its end would overwrite the start of the lexeme that `lexemeBegin` still points to, as with `init` crossing positions 7 and 8 above. With two halves, only the half the scanner has finished with is refilled, so a lexeme that crosses a block boundary is still whole. The cost is that no lexeme can be longer than N.

</details>

## 2.7 Lexical errors and line numbers

### "error 5." Where?

`scan-ops` from section 2.3 printed `error  5.` for the bad number, and nothing else. In a five-line test that is enough. In a two-hundred-line program it is useless: which line? What is wrong with it? And what happened after it, did the scanner give up? Compare GCC on the broken statement of section 1.3: `lex-error.c:5:25: error: invalid suffix "x" on integer constant`. File, line, column, what is wrong, and then it carries on and reports the next problem too.

The examiner will give your scanner a wrong input on purpose (habit 3 of this book is to test with one before they do), and what it prints then is marked as carefully as the token list. This section decides which errors a scanner can find at all, what it should do after one, and how it knows which line it is on. Then it puts every piece of the chapter into the finished scanner, `tiny-scan.c`.

### What a scanner can and cannot catch

<!-- teaches: lexical-error = lexical error, panic mode -->
A *lexical error* is a place where the remaining input does not begin with any lexeme: no pattern matches. Only that. Here is what that covers in Tiny, and what it does not.

| Input | Error? | Found by | Why |
|---|---|---|---|
| `x = @;` | yes | scanner | `@` starts no Tiny lexeme |
| `y = 5.;` | yes | scanner | a point must be followed by a digit |
| `z = 9lives;` | yes | scanner | a number cannot run straight into letters (C says the same of `60x`) |
| `a = !b;` | yes | scanner | `!` is a lexeme only as the start of `!=` |
| `/* never closed` | yes | scanner | the comment pattern needs a `*/` |
| `pos = = 5;` | no | parser | every lexeme is fine; the order is wrong |
| `whlie (x) y = 1;` | no | parser | `whlie` is a good identifier; a name followed by `(` and a statement is wrong grammar |
| `pos = rat * 60;` | no | semantic analyzer | `rat` is a good identifier; only the symbol table knows it was never given a value |

The last three rows are the favourite trap of the paper. The scanner sees one lexeme at a time and knows only patterns, so a misspelled keyword is just a name to it. The Dragon book's example is `fi (a == f(x))`: the scanner cannot tell whether `fi` is a mistyped `if` or a call of a function called `fi`, so it hands on `id` and lets the parser decide.

### What to do after an error

Stopping at the first error is the easy choice and the wrong one: the programmer fixes one mistake, runs again, and meets the next. A scanner should report the error and carry on, so that one run lists every lexical error. To carry on, it must get back to a point where a lexeme can start. The simplest way is called *panic mode*: throw away characters until one of them can begin a good token, and start again there. In `tiny-scan` that is almost free. The bad character, or the bad number, has already been read when the error is found; the next call of `gettoken` simply starts fresh with whatever comes next.

The Dragon book lists four finer repairs a scanner may try: delete one character, insert a missing one, replace one with another, or swap two neighbours. A scanner that tries them might turn `5.;` into `5.0;` and go on as if it had been written that way. Real compilers rarely bother; GCC reports and moves on, and so does `tiny-scan`. Learn the four for the paper.

The scanner hands nothing to the parser for a bad lexeme in this book: it returns the token `ERROR`, `main` prints the message instead of a token, and at the end, if there was any error, the program says how many and exits with status 1 instead of 0. A non-zero exit status is how a program tells the shell (and `make`, and a marking script) that it failed.

### Counting lines

The line number is the count of newlines read so far, plus one. Section 2.4's last question showed where the counting belongs: in the one place that reads characters. So `tiny-scan` wraps `getchar` in a small function, `next`, that adds one to `line` whenever it returns `'\n'`. Every part of the scanner, `skip` and `gettoken` alike, calls `next` instead of `getchar`.

But there is a second place a character can travel: back. When the lookahead that ends a name is a newline, `ungetc` gives it back, and the next `next` reads it again and counts it again. Counted twice, the line number drifts one line too far every time that happens. Here is the fault, made real. `next` counts, `back` forgets to uncount; each word is printed with the line it started on. It was given three lines, `pos`, `init` and `rate`:

<!-- code: id=linebug stdin="pos\ninit\nrate\n" -->
```c
#include <stdio.h>
#include <ctype.h>

int line = 1;

int next(void)
{
    int c = getchar();

    if (c == '\n')
        line++;
    return c;
}

void back(int c)
{
    ungetc(c, stdin);      /* the newline goes back, the count does not */
}

int main(void)
{
    int c, n, start;
    char word[64];

    while ((c = next()) != EOF) {
        if (!isalpha(c))
            continue;
        start = line;
        n = 0;
        while (isalnum(c)) {
            if (n < 63)
                word[n++] = c;
            c = next();
        }
        word[n] = '\0';
        back(c);
        printf("line %d: %s\n", start, word);
    }
    return 0;
}
```
```text
line 1: pos
line 3: init
line 5: rate
```

`pos` is right; `init` is reported on line 3 and `rate` on line 5. After `pos`, the newline is read (line becomes 2), pushed back (still 2), and read again (3). The cure is one line in `back`: if the character going back is a newline, `line--`. Notice how well this bug hides. Give the same program `pos = 1;` and `x = 2;` and every line is right, because the lookahead after `pos` is a space and the newline after `;` is never pushed back. The drift appears only when a line ends straight after a name or a number, which in Tiny means a statement split over two lines, as in the test file below.

The other number a scanner needs is the line where the current token *started*, which is not always the current `line`: an unclosed comment that starts on line 3 is only discovered at the end of the file. `tiny-scan` keeps it in `tokline`, set when each token's first character has been found, and set again at the start of each block comment.

### The finished scanner

Here is everything in the chapter in one program: section 2.2's keyword table, section 2.3's numbers, operators and longest match, section 2.4's `skip`, and this section's `next`, `back`, line numbers and error messages. Save it as `tiny-scan.c`. It is the first file of the mini compiler: chapter 4 writes the same scanner in Flex, as `tiny.l`, and from chapter 6 on a parser asks a scanner like this one for one token at a time.

<!-- code: id=tiny-scan mode=compile -->
```c
/* tiny-scan.c: a hand-written scanner for Tiny */
#include <stdio.h>
#include <ctype.h>
#include <string.h>

enum { ID, NUM, IF, WHILE, ASSIGN, EQ, NE, LT, LE, GT, GE,
       PLUS, MINUS, TIMES, DIVIDE, LPAREN, RPAREN, SEMI, ERROR, END };

const char *name[] = { "id", "num", "if", "while", "=", "==", "!=",
                       "<", "<=", ">", ">=", "+", "-", "*", "/",
                       "(", ")", ";", "error", "end" };

struct { const char *word; int token; } keyword[] = {
    { "if", IF },
    { "while", WHILE },
};
#define NKEY 2

char lexeme[64];   /* the characters of the current token */
int len;           /* how many characters lexeme holds */
int line = 1;      /* the line the next character is on */
int tokline;       /* the line the current token started on */
int errors;        /* lexical errors so far */

int next(void)     /* read one character, counting lines */
{
    int c = getchar();

    if (c == '\n')
        line++;
    return c;
}

void back(int c)   /* give one character back */
{
    if (c == '\n')
        line--;
    ungetc(c, stdin);
}

void add(int c)    /* append c to the lexeme */
{
    if (len < 63)
        lexeme[len++] = c;
    lexeme[len] = '\0';
}

void error(const char *msg)
{
    fprintf(stderr, "line %d: error: %s: %s\n", tokline, msg, lexeme);
    errors++;
}

int skip(void)     /* skip blanks and comments; return the next character */
{
    int c, prev;

    for (;;) {
        c = next();
        if (isspace(c))
            continue;
        if (c != '/')
            return c;
        c = next();
        if (c == '/') {                 /* a // comment runs to the newline */
            while (c != '\n' && c != EOF)
                c = next();
        } else if (c == '*') {          /* a block comment runs to the first */
            tokline = line;
            prev = 0;
            c = next();
            while (c != EOF && !(prev == '*' && c == '/')) {
                prev = c;
                c = next();
            }
            if (c == EOF) {
                add('/');
                add('*');
                error("comment not closed");
                return EOF;
            }
        } else {                        /* just a division sign */
            back(c);
            return '/';
        }
    }
}

int two(int second, int yes, int no)   /* one character of lookahead */
{
    int c = next();

    if (c == second) {
        add(c);
        return yes;
    }
    back(c);
    return no;
}

int gettoken(void)
{
    int c;

    len = 0;
    lexeme[0] = '\0';
    c = skip();
    tokline = line;
    if (c == EOF)
        return END;
    if (isalpha(c) || c == '_') {
        while (isalnum(c) || c == '_') {
            add(c);
            c = next();
        }
        back(c);
        for (int i = 0; i < NKEY; i++)
            if (strcmp(lexeme, keyword[i].word) == 0)
                return keyword[i].token;
        return ID;
    }
    if (isdigit(c)) {
        while (isdigit(c)) {
            add(c);
            c = next();
        }
        if (c == '.') {
            add(c);
            c = next();
            if (!isdigit(c)) {
                back(c);
                error("digit expected after '.'");
                return ERROR;
            }
            while (isdigit(c)) {
                add(c);
                c = next();
            }
        }
        if (isalpha(c) || c == '_') {
            while (isalnum(c) || c == '_') {
                add(c);
                c = next();
            }
            back(c);
            error("letters after a number");
            return ERROR;
        }
        back(c);
        return NUM;
    }
    add(c);
    switch (c) {
    case '+': return PLUS;
    case '-': return MINUS;
    case '*': return TIMES;
    case '/': return DIVIDE;
    case '(': return LPAREN;
    case ')': return RPAREN;
    case ';': return SEMI;
    case '=': return two('=', EQ, ASSIGN);
    case '<': return two('=', LE, LT);
    case '>': return two('=', GE, GT);
    case '!':
        if (two('=', NE, ERROR) == NE)
            return NE;
        error("'!' without '='");
        return ERROR;
    }
    error("unexpected character");
    return ERROR;
}

int main(void)
{
    int t;

    while ((t = gettoken()) != END)
        if (t != ERROR)
            printf("%3d  %-6s %s\n", tokline, name[t], lexeme);
    if (errors > 0) {
        fprintf(stderr, "%d lexical error(s)\n", errors);
        return 1;
    }
    return 0;
}
```

Now build it and test it three ways, as habit 3 asks: a normal program, an empty input, and an input that must be rejected. The normal program is the running statement with a loop after it, split over two lines on purpose so that a newline is the lookahead after `init`. `prog.tiny` and `bad.tiny` are written with `cat` and a here-document, as in chapter 1; keep them, because chapter 4 runs its Flex scanner on the same files.

<!-- code: id=tiny-scan-normal -->
```console
$ gcc -Wall -Wextra tiny-scan.c -o tiny-scan
$ cat > prog.tiny <<'END'
/* Tiny: the running statement, then a loop */
init = 10;
rate = 2.5;
pos = init
    + rate * 60;   // 160
while (pos >= 100)
    pos = pos - 7.25 / (rate != 0);
END
$ ./tiny-scan < prog.tiny
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

The empty input and the input that must be rejected:

<!-- code: id=tiny-scan-reject exit=1 -->
```console
$ ./tiny-scan < /dev/null
$ cat > bad.tiny <<'END'
x = 5. ;
y = 9lives @ 2;
z = !y; /* left open
w = 1;
END
$ ./tiny-scan < bad.tiny
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
```

### Reading the three runs

**The normal program.** The comment on line 1 produced nothing, and the first token is on line 2, as it should be. `pos = init` ends line 4 with a name, so the newline was the lookahead after `init`: read (line 5), pushed back by `back` (line 4 again), read once more by `skip` (line 5). The `+` is correctly on line 5, and every line after it is right; without the `line--` in `back`, everything from the `+` on would be one line too far. The `// 160` comment vanished. The token stream of the running statement, `id = id + id * num ;`, is in there, split over lines 4 and 5.

**The empty input.** `/dev/null` is a file that is always empty. `skip` reads `EOF` at once, `gettoken` returns `END`, the loop in `main` never runs, and the program prints nothing and exits with status 0. Nothing crashed and nothing was invented, which is all an empty test has to show. It is still worth running: a scanner that reads `lexeme` before setting it, or prints a token before checking for `END`, fails here.

**The bad input.** Each message gives the line where the bad lexeme started, what is wrong, and the lexeme itself. Follow the recovery. On line 1, `5.` is reported and the `;` after it is still scanned. On line 2 two errors are found in one statement and `2 ;` is still scanned after them. On line 3, `!y` gives an error for the `!`, then `y` is a normal identifier: panic mode threw away one character and started again. Then the block comment starts on line 3 and is never closed: it swallows line 4, `w = 1;`, and the error is reported against line 3, where the comment began, because `skip` set `tokline` there. Last, the count, and the shell's exit status 1, which `echo $?` would show.

The order of the lines on your screen is the order in which they were written: messages to the error stream, tokens to standard output, as they happen. If you redirect the tokens to a file, `./tiny-scan < bad.tiny > tokens.txt`, only the messages stay on the screen, which is exactly why messages go to `stderr`.

### Reading the program for the viva

The examiner will point at lines and ask why. Here are the answers, function by function.

**`next` and `back`.** The only places that read and unread characters, so the only places that count lines. `back` must undo exactly what `next` did.

**`add`.** Appends one character to `lexeme` and keeps it ended with `'\0'`. The guard `len < 63` keeps a long name from running past the array; the extra characters are still read, so the lexeme is never split (section 2.1's third question).

**`error`.** Prints to `stderr` and counts. It prints `tokline`, not `line`, so a multi-line lexeme is reported where it started.

**`skip`.** Section 2.4's function, now with `next` and `back`, and one addition: when `EOF` arrives inside a block comment it reports "comment not closed" and returns `EOF`, so `gettoken` returns `END`.

**`two`.** Section 2.3's one-character lookahead for `<=`, `>=`, `==` and `!=`.

**`gettoken`.** Resets the lexeme, calls `skip`, notes `tokline`, then follows the first character into one of three branches: identifier or keyword, number, operator. Each branch is a transition diagram of section 2.5, and each `back` is a star. Two branches report errors: the number branch for `5.` and for letters straight after digits, and the operator branch for `!` alone and for any character that starts nothing.

**`main`.** Stands in for the parser: calls `gettoken` until `END`, prints each good token with its line, and turns the error count into the exit status.

### When it goes wrong

**Line numbers too high, more so further down.** `back` does not undo the count of a newline, as in `linebug`. Test with a statement split after a name.

**Line numbers too low after a comment.** Some code reads characters with `getchar` directly, missing the count. Search the file for `getchar`: it should appear only inside `next`.

**One error, then nonsense for the rest of the line.** The error path did not push back the character that ended the bad lexeme, so the next token starts one character late. In `tiny-scan`, the `5.` path pushes back the `;`, which then appears as a token.

**No error reported for an unclosed comment.** `skip` returns `EOF` silently. Check for `EOF` inside the comment loop and report it.

**The program exits with status 0 after errors.** A marking script that checks the status will count your rejected input as accepted. Return 1 when `errors > 0`.

### In one line each

1. A lexical error is input where no pattern matches; misspelled keywords and wrong order are not lexical errors.
2. After an error, report it with its line and carry on from the next character (panic mode).
3. Count lines in the one function that reads characters, and uncount a newline that is pushed back.
4. Report the line where the token started, send messages to `stderr`, and exit with status 1 if anything was wrong.

**Check yourself**

<!-- q: idea=lexical-error type=decide wrong="whlie is a lexical error" -->
1. For each input, say whether `tiny-scan` reports an error, and if not, which later phase would: `a = b # c;`, `while (a) b = 1`, `wihle (a) b = 1;`, `c = 1.5.2;`.

<details>
<summary>Answer</summary>

`a = b # c;`: lexical error, `#` starts no Tiny lexeme; `tiny-scan` reports "unexpected character: #" and goes on to `c ;`. `while (a) b = 1` (no semicolon): no lexical error; the parser finds the missing `;`. `wihle (a) b = 1;`: no lexical error; `wihle` is a perfectly good identifier, and the parser finds that a name followed by `(a)` is not a statement. `c = 1.5.2;`: the scanner reads `1.5`, stops at the second point, then reports "unexpected character: ." and reads `2` as a number: one lexical error. If you called `wihle` a lexical error, remember that a scanner knows only patterns, not intentions.

</details>

<!-- q: idea=lexical-error type=predict -->
2. Predict exactly what `tiny-scan` prints, and its exit status, for this two-line input: `pos = rate` on line 1 and `  * 60 $;` on line 2.

<details>
<summary>Answer</summary>

```text
  1  id     pos
  1  =      =
  1  id     rate
  2  *      *
  2  num    60
line 2: error: unexpected character: $
  2  ;      ;
1 lexical error(s)
```

Exit status 1. `rate` ends line 1 with a newline as its lookahead; `back` uncounts it, so `rate` is on line 1 and `*` correctly on line 2. The `$` is reported on line 2 and the scan goes on to `;`. If you put `*` on line 3, you forgot that `back` undoes the count.

</details>

<!-- q: idea=lexical-error type=fix wrong="the fault is in next" -->
3. A student's scanner gives the right line for every token in `prog.tiny` except those after line 4, which are all one line too high. They insist `next` is correct. Where would you look, and what one line would you expect to be missing?

<details>
<summary>Answer</summary>

In `back`. Line 4 is the line that ends with a name (`pos = init`), the one place in `prog.tiny` where a newline is the lookahead and gets pushed back. If `back` does not do `if (c == '\n') line--;`, that newline is counted twice and every later token is one line too high, exactly the pattern described. `next` is indeed fine; the count goes wrong on the way back, not on the way in.

</details>

## 2.8 A symbol table in C: insert, search, display, delete with a hash table

### From `id pos` to `<id,1>`

`tiny-scan` prints `id pos`, but the token section 1.3 promised is `<id,1>`: the token name and a pointer to the name's line in the symbol table, the notebook of section 1.4. Every time the scanner meets an identifier it must look the name up, and add it if it is new. In `prog.tiny`, `pos` appears four times and `rate` three; each appearance is a lookup, and in a real program of a few thousand lines there are tens of thousands of them. The notebook has to be quick to search.

The lab asks for the notebook on its own, too. "Implement a symbol table with insert, search, display and delete operations", often with "using hashing", is one of the most common exam questions in the subject, and it is a pure C question: structures, pointers, `malloc` and `free`. This section writes that program, `symtab.c`, works its hash table by hand the way the paper wants, and connects it to the scanner.

### Why a plain list is too slow

The obvious notebook is an array of names, searched from the top. Looking up a name means comparing it with every name already written, until one matches or the list ends. With n names that is up to n calls of `strcmp` per lookup. A program with 1,000 names and 20,000 identifier occurrences costs up to 20 million comparisons, and most of them are for names that are not even close.

A dictionary does not make you start at "a" every time. You open it near the right place, because the word itself tells you where to look. A hash table does the same with arithmetic.

### The notebook with tabs

Picture the compiler's notebook with eleven tabs down its edge, numbered 0 to 10. To find which tab a name lives behind, you do a small sum on the name's letters, and the answer is the tab number. You turn straight to that tab, and behind it there is only a short list of names, the ones whose sums gave the same number. Search that short list and you are done.

<!-- teaches: symbol-table = hash table, bucket, hash function -->
In C, the notebook is a *hash table*: an array of *buckets*, here eleven, each holding the start of a list. The small sum is the *hash function*: it turns a name into a bucket number from 0 to 10, the same number every time for the same name. A *symbol table* in this book is exactly this, a hash table of records, one per name, each holding the name and what the compiler knows about it.

The picture breaks in two places. Behind each tab the names are in no useful order: not alphabetical, not in the order they appear in the program. Display the whole table and the names come out bucket by bucket, which looks random. And the book's own warning about the notebook still holds: a real compiler keeps one page per scope, opened at the start of a block and thrown away at its end. Tiny has a single scope, so one table is enough here.

### The hash function, by hand

`symtab.c` uses the simplest hash function the exam asks for: add up the character codes of the name and take the remainder when dividing by the table size, 11. Do it for the running statement's three names, using the ASCII codes `p` 112, `o` 111, `s` 115 and so on:

| Name | Sum of codes | Sum mod 11 | Bucket |
|---|---|---|---|
| `pos` | 112 + 111 + 115 = 338 | 338 = 30 × 11 + 8 | 8 |
| `init` | 105 + 110 + 105 + 116 = 436 | 436 = 39 × 11 + 7 | 7 |
| `rate` | 114 + 97 + 116 + 101 = 428 | 428 = 38 × 11 + 10 | 10 |
| `rat` | 114 + 97 + 116 = 327 | 327 = 29 × 11 + 8 | 8 |

`rat` lands in bucket 8, where `pos` already is. Two different names in one bucket is a *collision*. It cannot be avoided: there are millions of possible names and eleven buckets. What matters is what the table does about it.

<!-- teaches: hash-chaining = chaining, collision, load factor -->
*Chaining* is the answer used here: each bucket holds a linked list, a chain, of every record whose name hashes to it. A collision just makes a chain one record longer. Search hashes the name, then walks only that bucket's chain. With n names spread over m buckets, an average chain has n/m records; that ratio is the *load factor*. Keep it small (around 1) and a lookup costs one hash and a comparison or two, however many names there are. The table size is a prime so that sums which share a factor with the size do not all crowd into the same few buckets.

The sum of codes is a weak hash function, and it is worth knowing why. Any two names with the same letters in a different order, such as `pos` and `sop`, always collide, and short names all have small sums. Better hash functions mix the letters, for example `h = h * 31 + c` for each character (the one in K&R section 6.6), so that order matters. For the lab and the paper, use the function the question gives; if it gives none, the sum is acceptable, and saying why it is weak earns a mark in the viva.
<!-- checked: how=reworded date=2026-11-21 ref="claim narrowed to better hash functions; K&R 2nd ed. 6.6 hash uses *s + 31 * hashval" h=8dca07f9 -->

### The program

Each record is a `struct entry`: the name, its type, and `next`, a pointer to the following record in the same chain. `bucket` is an array of eleven such pointers, one per chain; as a global array it starts out all `NULL`, so every chain begins empty.

The four operations every exam lists are four functions. `search` walks one chain. `insert` refuses a name that is already there (one record per name, as section 1.4 required) and puts a new record at the front of its chain, which takes the same time however long the chain is. `delete` unlinks a record and frees it. `display` prints every bucket with its chain. A fifth function, `clear`, frees every record before the program ends.

`main` reads commands, one per line: `insert NAME TYPE`, `search NAME`, `delete NAME`, `display` and `quit`. Commands with words are easier to read in a test than a numbered menu; the menu version the lab manual may ask for comes after the walk-through. Save this as `symtab.c`. It was given these commands, in this order: insert `pos`, `init`, `rate` (all `float`) and `rat` (`int`); display; search `rate`; search `speed`; insert `pos` again; delete `pos`; display; delete `pos` again; quit.

<!-- code: id=symtab stdin="insert pos float\ninsert init float\ninsert rate float\ninsert rat int\ndisplay\nsearch rate\nsearch speed\ninsert pos int\ndelete pos\ndisplay\ndelete pos\nquit\n" -->
```c
/* symtab.c: a symbol table kept in a hash table with chaining */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SIZE 11                 /* number of buckets; a prime */

struct entry {
    char name[32];
    char type[16];
    struct entry *next;         /* the next entry in the same bucket */
};

struct entry *bucket[SIZE];     /* globals start as NULL: every chain empty */

int hash(const char *s)
{
    int sum = 0;

    while (*s != '\0')
        sum += (unsigned char)*s++;
    return sum % SIZE;
}

struct entry *search(const char *name)
{
    struct entry *e;

    for (e = bucket[hash(name)]; e != NULL; e = e->next)
        if (strcmp(e->name, name) == 0)
            return e;
    return NULL;
}

int insert(const char *name, const char *type)
{
    int h = hash(name);
    struct entry *e;

    if (search(name) != NULL)
        return 0;               /* one entry per name */
    e = malloc(sizeof *e);
    if (e == NULL) {
        perror("malloc");
        exit(1);
    }
    snprintf(e->name, sizeof e->name, "%s", name);
    snprintf(e->type, sizeof e->type, "%s", type);
    e->next = bucket[h];        /* the new entry goes in front */
    bucket[h] = e;
    return 1;
}

int delete(const char *name)
{
    int h = hash(name);
    struct entry *e = bucket[h], *prev = NULL;

    while (e != NULL && strcmp(e->name, name) != 0) {
        prev = e;
        e = e->next;
    }
    if (e == NULL)
        return 0;               /* not in the table */
    if (prev == NULL)
        bucket[h] = e->next;    /* it was first in its bucket */
    else
        prev->next = e->next;   /* step over it */
    free(e);
    return 1;
}

void display(void)
{
    struct entry *e;

    for (int h = 0; h < SIZE; h++) {
        printf("%2d:", h);
        for (e = bucket[h]; e != NULL; e = e->next)
            printf(" -> %s %s", e->name, e->type);
        printf("\n");
    }
}

void clear(void)
{
    struct entry *e, *after;

    for (int h = 0; h < SIZE; h++) {
        for (e = bucket[h]; e != NULL; e = after) {
            after = e->next;
            free(e);
        }
        bucket[h] = NULL;
    }
}

int main(void)
{
    char cmd[16], name[32], type[16];
    struct entry *e;

    while (scanf("%15s", cmd) == 1) {
        if (strcmp(cmd, "insert") == 0 && scanf("%31s %15s", name, type) == 2) {
            if (insert(name, type))
                printf("inserted %s in bucket %d\n", name, hash(name));
            else
                printf("%s is already in the table\n", name);
        } else if (strcmp(cmd, "search") == 0 && scanf("%31s", name) == 1) {
            e = search(name);
            if (e != NULL)
                printf("found %s: type %s, bucket %d\n", e->name, e->type, hash(name));
            else
                printf("%s not found\n", name);
        } else if (strcmp(cmd, "delete") == 0 && scanf("%31s", name) == 1) {
            if (delete(name))
                printf("deleted %s\n", name);
            else
                printf("%s not found, nothing deleted\n", name);
        } else if (strcmp(cmd, "display") == 0) {
            display();
        } else if (strcmp(cmd, "quit") == 0) {
            break;
        } else {
            printf("unknown command %s\n", cmd);
        }
    }
    clear();
    return 0;
}
```
```text
inserted pos in bucket 8
inserted init in bucket 7
inserted rate in bucket 10
inserted rat in bucket 8
 0:
 1:
 2:
 3:
 4:
 5:
 6:
 7: -> init float
 8: -> rat int -> pos float
 9:
10: -> rate float
found rate: type float, bucket 10
speed not found
pos is already in the table
deleted pos
 0:
 1:
 2:
 3:
 4:
 5:
 6:
 7: -> init float
 8: -> rat int
 9:
10: -> rate float
pos not found, nothing deleted
```

### Walking through the output

**The four inserts.** Each `insert` computed the bucket by hand's arithmetic: `pos` 8, `init` 7, `rate` 10, `rat` 8. When `rat` was inserted, bucket 8 already pointed to `pos`. The two lines `e->next = bucket[h]; bucket[h] = e;` make the new record point to the old first record, then make the bucket point to the new one. So chain 8 is now `rat` then `pos`, which is what the first `display` shows: newest first.

Here is chain 8 before and after, as you would draw it in the paper:

| | `bucket[8]` points to | then | then |
|---|---|---|---|
| after inserting `pos` | `pos` | `NULL` | |
| after inserting `rat` | `rat` | `pos` | `NULL` |

**The searches.** `search rate` hashed `rate` to 10, found the chain's first record equal, and returned it. `search speed` hashed `speed` to bucket 1 (115 + 112 + 101 + 101 + 100 = 529 = 48 × 11 + 1), found that chain empty and returned `NULL`. Neither search looked at any other bucket, and that is the whole gain over a list.

**The second `insert pos`.** `insert` calls `search` first, finds `pos`, and returns 0 without adding anything. A symbol table with two records for `pos` would let two parts of the compiler write different facts about the same variable into two different places.

**The delete.** `delete pos` walks chain 8 with two pointers: `e`, the record being looked at, and `prev`, the one before it. It starts with `e` at `rat` and `prev` at `NULL`; `rat` is not `pos`, so `prev` becomes `rat` and `e` becomes `pos`. Found. Because `prev` is not `NULL`, the line `prev->next = e->next;` makes `rat` point past `pos`, to `NULL`; only then is `pos` freed. The second `display` shows chain 8 with `rat` alone.

Had the name been first in its chain, `rat` for example, `prev` would still be `NULL` when it was found, and the other branch, `bucket[h] = e->next;`, would make the bucket itself skip it. Those are the two cases every delete in a linked list must handle, and forgetting the first is the classic fault in the exam.

**The last delete.** `pos` is no longer in chain 8, so `e` runs off the end and the function returns 0: the program says so instead of failing.

### The menu the lab manual asks for

Many lab manuals word the question "write a menu-driven program". The functions stay exactly the same; only `main` changes, to print a menu and read a number. The heart of it:

<!-- code: id=symtab-menu mode=fragment -->
```c
int choice;

for (;;) {
    printf("1.Insert 2.Search 3.Display 4.Delete 5.Exit\nChoice: ");
    if (scanf("%d", &choice) != 1 || choice == 5)
        break;
    switch (choice) {
    case 1:
        printf("Name and type: ");
        if (scanf("%31s %15s", name, type) == 2 && !insert(name, type))
            printf("%s is already in the table\n", name);
        break;
    /* cases 2, 3 and 4 call search, display and delete the same way */
    }
}
```

Write the functions first and test them with the command version, where a test is one line of input; add the menu at the end if the question asks for it. A menu is pleasant for the examiner and slow for you to test.

### Into the scanner

The scanner needs one call: "find this name, and add it if it is new", which returns the record either way. In `tiny-scan`, the identifier branch of `gettoken` would call it just before `return ID;` and keep the record (or a number stored in it) as the token's attribute, so that the token becomes `<id,1>` instead of `id pos`. Section 2.9 builds exactly that for a C subset, with a record number in each entry so the scanner can print `<id,1>`, `<id,2>`, `<id,3>` in the order names first appear. It is the same hash table, with `search` and `insert` joined into one function.

### When it goes wrong

**Display crashes, or prints garbage, after a delete.** The record was freed but never unlinked, so its neighbour or its bucket still points to freed memory, and `display` follows the pointer. Unlink first (`prev->next = e->next` or `bucket[h] = e->next`), free second.

**Deleting the first name in a bucket does nothing.** The delete only handles the `prev` case. When the record is first, it is the bucket that must change. Test delete on a name at the front of a chain and on one behind it, as the walk above did.

**The same name appears twice in `display`.** `insert` does not search first.

**Every name lands in bucket 0, or the bucket numbers are negative.** The sum was taken over `char` values that are negative for non-ASCII bytes, or the hash returns `sum % SIZE` of a negative sum. The cast `(unsigned char)` in `hash` prevents it.

**The sanitizer, or `valgrind`, reports leaked memory.** Records were never freed at the end. Free every chain before `main` returns, as `clear` does; an examiner who runs your program under a leak checker sees the difference.

### In one line each

1. A hash table is an array of buckets; the hash function turns a name into a bucket number.
2. Two names in one bucket are a collision; chaining keeps a linked list per bucket.
3. Insert searches first, then adds at the front of the chain; search walks one chain only.
4. Delete keeps a pointer to the previous record and handles the first-in-chain case separately: unlink, then free.

**Check yourself**

<!-- q: idea=hash-chaining type=apply wrong="the later insert goes at the end of the chain" -->
1. By hand, with table size 11 and the sum-of-codes hash, insert `sum`, `i`, `n` and `avg` (codes: `s` 115, `u` 117, `m` 109, `i` 105, `n` 110, `a` 97, `v` 118, `g` 103) and draw every non-empty chain. Then delete `sum` and draw it again. Check your drawing with `./symtab`.

<details>
<summary>Answer</summary>

`sum` = 341 = 31 × 11, bucket 0. `i` = 105 = 9 × 11 + 6, bucket 6. `n` = 110 = 10 × 11, bucket 0: a collision with `sum`. `avg` = 318 = 28 × 11 + 10, bucket 10. Because new records go in front, chain 0 is `n` then `sum`. Deleting `sum` takes the `prev` branch (it is second in its chain), leaving chain 0 as `n` alone. The program agrees; `grep -v ':$'` hides the empty buckets:

<!-- code: id=symtab-hand -->
```console
$ printf 'insert sum int\ninsert i int\ninsert n int\ninsert avg float\ndisplay\ndelete sum\ndisplay\n' | ./symtab | grep -v ':$'
inserted sum in bucket 0
inserted i in bucket 6
inserted n in bucket 0
inserted avg in bucket 10
 0: -> n int -> sum int
 6: -> i int
10: -> avg float
deleted sum
 0: -> n int
 6: -> i int
10: -> avg float
```

If you drew chain 0 as `sum` then `n`, you inserted at the end; that is also a correct hash table, but not what `symtab.c` does, and the paper's answer must match the method the question names.

</details>

<!-- q: idea=hash-chaining type=fix wrong="the free is in the wrong place" -->
2. A student's delete is below. It works for some names and not for others. Which names does it fail on, and what happens to them?

<!-- code: id=bad-delete mode=fragment -->
```c
int delete(const char *name)
{
    int h = hash(name);
    struct entry *e = bucket[h], *prev = NULL;

    while (e != NULL && strcmp(e->name, name) != 0) {
        prev = e;
        e = e->next;
    }
    if (e == NULL)
        return 0;
    if (prev != NULL)
        prev->next = e->next;
    free(e);
    return 1;
}
```

<details>
<summary>Answer</summary>

It fails on every name that is first in its chain: `rat` in bucket 8 after the walk above, or any name alone in its bucket. For those `prev` is still `NULL`, so nothing is unlinked, yet the record is freed. `bucket[h]` still points to the freed memory, so the next `display`, `search` or `insert` in that bucket reads freed memory: it may print garbage, crash, or seem to work, which is the worst case. The missing line is `else bucket[h] = e->next;`. Names further down a chain are deleted correctly, which is why the student's tests passed.

</details>

<!-- q: idea=symbol-table type=decide -->
3. A classmate says: "A hash table is overkill for the lab. An array of 100 structs with a linear search is simpler and fast enough." When is the classmate right, and what would you answer if the examiner asked why a compiler uses hashing anyway?

<details>
<summary>Answer</summary>

For a lab program with ten names the classmate is right that the difference cannot be measured, and if the question does not say "hashing", an array is an acceptable answer. But a compiler looks up a name at every occurrence of every identifier. With n names a linear search costs up to n comparisons per lookup, so the total grows with n times the number of occurrences; a hash table with a load factor near 1 costs about one comparison per lookup whatever n is. On a program of thousands of names that is the difference between instant and noticeably slow. The array also has a fixed size of 100, which a real program can exceed; a chained hash table grows one record at a time.

</details>

## 2.9 Practice: a complete scanner for a C subset that fills a symbol table

You can now write a scanner from a blank file: read with one character of lookahead, cut identifiers and look them up in a keyword table, take the longest match for numbers and operators, sweep away white space and comments, draw each token as a transition diagram first, report lexical errors with line numbers, and keep names in a hash table. The tasks below climb from predicting what `tiny-scan` does, through extending it on your own, to the lab exam's most common scanner question, written whole. Work in `~/cdlab/ch2`, where `tiny-scan`, `symtab` and the test files from this chapter already are.

### Guided

<!-- q: idea=longest-match type=predict wrong="//half is a division and a name" -->
**1. Predict, then run.** Without running it, write down every line `tiny-scan` prints for these two lines, then check:

<!-- code: id=practice-predict answer -->
```console
$ printf 'x=y==z/2.5//half\n/***/w<=(1)!=_v;\n' | ./tiny-scan
```

<details>
<summary>Answer</summary>

```console
$ printf 'x=y==z/2.5//half\n/***/w<=(1)!=_v;\n' | ./tiny-scan
  1  id     x
  1  =      =
  1  id     y
  1  ==     ==
  1  id     z
  1  /      /
  1  num    2.5
  2  id     w
  2  <=     <=
  2  (      (
  2  num    1
  2  )      )
  2  !=     !=
  2  id     _v
  2  ;      ;
```

Line 1: the first `=` is followed by `y`, so `two` returns the assignment; the second `=` is followed by `=`, so `two` returns `==`. After `z`, the next call of `gettoken` starts with `skip`, which reads `/` and looks one character further: `2`, neither `/` nor `*`, so it pushes the `2` back and returns `/`, a division. After `2.5`, `skip` reads `/` and then `/` again: a line comment, so `//half` produces nothing at all. Line 2: `/***/` is a block comment whose last star sits right before the `/`, which the `prev` loop handles (`badcomment` would have swallowed the rest). Then `<=`, `(`, `1`, `)`, `!=` and `_v`, since a name may start with an underscore. If you printed `/`, `/` and `id half` on line 1, you treated the second `/` like the first; the difference is the character after it.

</details>

<!-- q: idea=keyword-table type=apply -->
**2. Grow Tiny.** Change `tiny-scan.c` so that Tiny also has the keyword `else` and the operator `%`. List every line you change or add, and give a one-line test that proves both work.

<details>
<summary>Answer</summary>

Four places. The enum gets `ELSE` and `MOD`, for example `enum { ID, NUM, IF, WHILE, ELSE, ASSIGN, ..., DIVIDE, MOD, LPAREN, ... };`. The `name` array gets `"else"` and `"%"` at exactly the same positions. The keyword table gets a row `{ "else", ELSE },` and `NKEY` becomes 3. The `switch` gets `case '%': return MOD;`. No lookahead is needed for `%` because Tiny has no operator that starts with `%` and goes on. A good test: `printf 'if (a) b = c %% 2; else elsewhere = 1;\n' | ./tiny-scan` (in `printf`, `%%` prints one `%`). It must show `else` as the keyword once and `elsewhere` as an `id`. The usual slip is to add `"%"` to `name` at a different position from `MOD` in the enum, which shifts every name printed after it.

</details>

<!-- q: idea=transition-diagram type=apply -->
**3. Draw `skip`.** Draw the transition diagram for what happens after `skip` reads a `/`: the three outcomes of section 2.4 (division, line comment, block comment), including the two-square part that finds `*/`. Number the states from 0 and mark the stars.

<details>
<summary>Answer</summary>

| State | Edges | Meaning |
|---|---|---|
| 0 (start) | `/` to 1 | |
| 1 | `/` to 2, `*` to 3, other to 6 | just read one `/` |
| 2 | newline to 0, other to 2 | inside a line comment |
| 3 | `*` to 4, other to 3 | inside a block comment |
| 4 | `/` to 0, `*` to 4, other to 3 | just saw a star |
| 6 * | | return `/` (division), push the lookahead back |

The comment states lead back to the start, not to an accepting state, because a comment produces no token: the scanner simply starts looking for one again. Only state 6 accepts, and it has a star, because the character after the `/` belongs to the next lexeme. The edge `*` from 4 back to 4 is what `badcomment` was missing. (An end of input in state 2 is the end of the program; in state 3 or 4 it is the error "comment not closed".)

</details>

### On your own

<!-- q: idea=number-scan type=apply unseen="hex-constants" -->
**4. Hexadecimal constants.** C writes numbers in base 16 as `0x` followed by hex digits: `0x1F`, `0XFF`, `0x0`. Extend the number branch of `tiny-scan`'s `gettoken` so that these come out as one `num` token. Decide, and justify, what it should do with `0x` followed by a character that is not a hex digit, as in `0xg` or `0x;`. Write the code for the branch.

<details>
<summary>Answer</summary>

After the first digit, if it was `0` and the next character is `x` or `X`, the number is hexadecimal; otherwise go on as before. The C library has `isxdigit`, true for `0` to `9`, `a` to `f` and `A` to `F`.

<!-- code: id=hex-branch mode=fragment -->
```c
if (isdigit(c)) {
    if (c == '0') {
        add(c);
        c = next();
        if (c == 'x' || c == 'X') {
            add(c);
            c = next();
            if (!isxdigit(c)) {
                back(c);
                error("hex digit expected after 0x");
                return ERROR;
            }
            while (isxdigit(c)) {
                add(c);
                c = next();
            }
            back(c);
            return NUM;
        }
    }
    while (isdigit(c)) {        /* the decimal path, as before */
        add(c);
        c = next();
    }
    /* ... fraction and letters-after-number checks as in tiny-scan ... */
}
```

`0x` with no hex digit after it must be an error, for the same reason as `5.`: reading `0x;` as `0` followed by the name `x` would need two characters of pushback (`x` and `;`), and C itself rejects it. Note that the decimal loop still works for `0` alone and for `012`, since after the `if` the `0` is already in `lexeme` and `c` holds the next character. The most likely mistake is to forget the "letters after a number" check on the hex path: then `0x1Fg` gives `num 0x1F` and `id g`, where C gives one error.

</details>

<!-- q: idea=longest-match type=decide -->
**5. A point at the start.** In C, `.5` is a number, and `.` alone is the member operator (`p.x`). Should your C-subset scanner accept `.5`, and can it do so with one character of lookahead? Describe the decision at a `.` in words, then say what `tiny-scan` does with `x = .5;` today.

<details>
<summary>Answer</summary>

It can, with one character. At a `.`, read the next character. If it is a digit, this is a number: keep both and continue in the fraction loop. If not, push that one character back and return the `.` as the member operator. Only one character was read ahead, so one pushback is enough; the trouble case of section 2.3 was the opposite order, digits then `.` then a non-digit, where a language that allowed it would need two. Whether to accept it is a design decision: the C standard says `.5` is a valid constant, so a scanner for C should; Tiny's definition starts every number with a digit, so `tiny-scan` rightly does not. Today `tiny-scan` reports `unexpected character: .` on that line and then scans `5` as a `num`: one lexical error, and the scan goes on.

</details>

<!-- q: idea=comment-skip type=explain -->
**6. Strings in C.** A C-subset scanner must read string literals like `"rate = %g\n"` as one token. Explain why the string branch must come before comment skipping can see the inside of the string, what should end a string, and what should happen at a newline or end of file before the closing quote.

<details>
<summary>Answer</summary>

`skip` runs only before a token starts. Once `gettoken` has seen the opening `"` it is inside the string branch, which reads characters itself until the closing quote; `skip` never sees them, so `"http://x"` keeps its `//` and `"/*"` keeps its `/*`. The danger is a scanner that removes comments in a separate pass first, like `decomment`, which would cut `"http://x"` in half. A string ends at the next `"` that is not escaped: after a backslash, the next character is taken as it is, so `\"` does not end the string and `\\` is one backslash. A newline before the closing quote is a lexical error in C (GCC 13 says `error: missing terminating " character`), and so is the end of the file; report it with the line where the string started (`tokline`) and carry on with the next line, as `tiny-scan` does after a bad number.

</details>

### Chapter project: a C-subset scanner with a symbol table

<!-- q: idea=symbol-table type=build -->
**7. The lab question, written whole.** Write `cscan.c` from a blank file. It reads a program in this C subset from standard input and prints one line per token, then the symbol table.

- **Keywords:** `int float char if else while for return`.
- **Identifiers:** a letter or `_`, then letters, digits and `_`. Each identifier goes into a hash table of 11 buckets with chaining and the sum-of-codes hash (section 2.8). The first time a name is seen it gets the next number, starting at 1, and the line where it was seen; every later appearance finds the same record.
- **Numbers:** digits, optionally `.` and more digits; a point must be followed by a digit.
- **Operators:** `+ - * / % = == != < <= > >= && || ! ++ --`. **Punctuation:** `( ) { } ; ,`.
- **Skipped:** white space, `//` comments and `/* */` comments.
- **Errors:** as in `tiny-scan` (same messages, to `stderr`), then carry on; exit status 1 if there were any.

Each token line is `printf("%3d  %-8s <%s>\n", line, lexeme, kind)` for keywords (kind `keyword`), operators (`op`) and punctuation (`punct`), and `printf("%3d  %-8s <%s,%s>\n", ...)` for identifiers (`<id,N>`, where N is the name's number) and numbers (`<num,LEXEME>`). After the last token print an empty line, the line `symbol table`, and then, bucket by bucket from 0 to 10 and front to back within each chain, `bucket %2d: %d %s (line %d)` with the bucket, the number, the name and the line.

Test it on this program, saved as `avg.c`, then on an empty input and on an input with errors:

```text
int main()
{
    int count = 0, i;
    float avg = 2.5;   // running average
    for (i = 0; i < 10; i++) {
        /* add the even values */
        if (i % 2 == 0 && count != 3)
            count = count + i;
    }
    return count;
}
```

**Done when** your program prints exactly the model's output for `avg.c`. You can test that without anyone marking it: in the understudy folder (the one that holds `book/`), in the terminal, run `python3 scripts/us_practice.py check compiler-design 2.9 --in ~/cdlab/ch2`. It builds your `cscan.c` and compares what it prints with the model below.

<details>
<summary>Answer</summary>

The model joins `tiny-scan` and `symtab`. The symbol table's `search` and `insert` become one function, `lookup`, which returns the record whether it found it or made it; the record carries the number and line instead of a type. The keyword table needs no tokens, since every keyword prints the same kind. Two-character operators use `also`, a variant of `two` that just reports whether the second character came: `+` may be followed by `+`, `=` `!` `<` `>` by `=`, and `&` `|` only by themselves (a single `&` or `|` is an error in this subset).

<!-- code: id=cscan check="gcc -Wall -Wextra cscan.c -o cscan && ./cscan" stdin="int main()\n{\n    int count = 0, i;\n    float avg = 2.5;   // running average\n    for (i = 0; i < 10; i++) {\n        /* add the even values */\n        if (i % 2 == 0 && count != 3)\n            count = count + i;\n    }\n    return count;\n}\n" -->
```c
/* cscan.c: a scanner for a C subset that fills a symbol table */
#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>
#include <string.h>

#define SIZE 11

struct entry {
    char name[32];
    int num;                    /* the attribute value: <id,num> */
    int line;                   /* where the name was first seen */
    struct entry *next;
};

struct entry *bucket[SIZE];
int entries;                    /* how many names so far */

const char *keyword[] = { "int", "float", "char", "if", "else",
                          "while", "for", "return" };
#define NKEY 8

char lexeme[64];
int len, line = 1, tokline, errors;

int hash(const char *s)
{
    int sum = 0;

    while (*s != '\0')
        sum += (unsigned char)*s++;
    return sum % SIZE;
}

struct entry *lookup(const char *name)   /* find the name, or add it */
{
    int h = hash(name);
    struct entry *e;

    for (e = bucket[h]; e != NULL; e = e->next)
        if (strcmp(e->name, name) == 0)
            return e;
    e = malloc(sizeof *e);
    if (e == NULL) {
        perror("malloc");
        exit(1);
    }
    snprintf(e->name, sizeof e->name, "%s", name);
    e->num = ++entries;
    e->line = tokline;
    e->next = bucket[h];
    bucket[h] = e;
    return e;
}

int next(void)
{
    int c = getchar();

    if (c == '\n')
        line++;
    return c;
}

void back(int c)
{
    if (c == '\n')
        line--;
    ungetc(c, stdin);
}

void add(int c)
{
    if (len < 63)
        lexeme[len++] = c;
    lexeme[len] = '\0';
}

void error(const char *msg)
{
    fprintf(stderr, "line %d: error: %s: %s\n", tokline, msg, lexeme);
    errors++;
}

void show(const char *token, const char *attr)   /* one line per token */
{
    if (attr[0] == '\0')
        printf("%3d  %-8s <%s>\n", tokline, lexeme, token);
    else
        printf("%3d  %-8s <%s,%s>\n", tokline, lexeme, token, attr);
}

int skip(void)
{
    int c, prev;

    for (;;) {
        c = next();
        if (isspace(c))
            continue;
        if (c != '/')
            return c;
        c = next();
        if (c == '/') {
            while (c != '\n' && c != EOF)
                c = next();
        } else if (c == '*') {
            tokline = line;
            prev = 0;
            c = next();
            while (c != EOF && !(prev == '*' && c == '/')) {
                prev = c;
                c = next();
            }
            if (c == EOF) {
                add('/');
                add('*');
                error("comment not closed");
                return EOF;
            }
        } else {
            back(c);
            return '/';
        }
    }
}

int also(int second)            /* add the next character if it is second */
{
    int c = next();

    if (c == second) {
        add(c);
        return 1;
    }
    back(c);
    return 0;
}

int scan(void)                  /* one token; 0 at the end of input */
{
    int c;
    char attr[16];

    len = 0;
    lexeme[0] = '\0';
    c = skip();
    tokline = line;
    if (c == EOF)
        return 0;
    if (isalpha(c) || c == '_') {
        while (isalnum(c) || c == '_') {
            add(c);
            c = next();
        }
        back(c);
        for (int i = 0; i < NKEY; i++)
            if (strcmp(lexeme, keyword[i]) == 0) {
                show("keyword", "");
                return 1;
            }
        snprintf(attr, sizeof attr, "%d", lookup(lexeme)->num);
        show("id", attr);
        return 1;
    }
    if (isdigit(c)) {
        while (isdigit(c)) {
            add(c);
            c = next();
        }
        if (c == '.') {
            add(c);
            c = next();
            if (!isdigit(c)) {
                back(c);
                error("digit expected after '.'");
                return 1;
            }
            while (isdigit(c)) {
                add(c);
                c = next();
            }
        }
        if (isalpha(c) || c == '_') {
            while (isalnum(c) || c == '_') {
                add(c);
                c = next();
            }
            back(c);
            error("letters after a number");
            return 1;
        }
        back(c);
        show("num", lexeme);
        return 1;
    }
    add(c);
    switch (c) {
    case '(': case ')': case '{': case '}': case ';': case ',':
        show("punct", "");
        return 1;
    case '*': case '/': case '%':
        show("op", "");
        return 1;
    case '+': case '-':
        also(c);                /* ++ or -- */
        show("op", "");
        return 1;
    case '=': case '!': case '<': case '>':
        also('=');              /* == != <= >= */
        show("op", "");
        return 1;
    case '&': case '|':
        if (also(c)) {          /* && or || */
            show("op", "");
            return 1;
        }
        break;
    }
    error("unexpected character");
    return 1;
}

void table(void)
{
    struct entry *e, *after;

    printf("\nsymbol table\n");
    for (int h = 0; h < SIZE; h++)
        for (e = bucket[h]; e != NULL; e = e->next)
            printf("bucket %2d: %d %s (line %d)\n", h, e->num, e->name, e->line);
    for (int h = 0; h < SIZE; h++)
        for (e = bucket[h]; e != NULL; e = after) {
            after = e->next;
            free(e);
        }
}

int main(void)
{
    while (scan())
        ;
    table();
    if (errors > 0) {
        fprintf(stderr, "%d lexical error(s)\n", errors);
        return 1;
    }
    return 0;
}
```
```text
  1  int      <keyword>
  1  main     <id,1>
  1  (        <punct>
  1  )        <punct>
  2  {        <punct>
  3  int      <keyword>
  3  count    <id,2>
  3  =        <op>
  3  0        <num,0>
  3  ,        <punct>
  3  i        <id,3>
  3  ;        <punct>
  4  float    <keyword>
  4  avg      <id,4>
  4  =        <op>
  4  2.5      <num,2.5>
  4  ;        <punct>
  5  for      <keyword>
  5  (        <punct>
  5  i        <id,3>
  5  =        <op>
  5  0        <num,0>
  5  ;        <punct>
  5  i        <id,3>
  5  <        <op>
  5  10       <num,10>
  5  ;        <punct>
  5  i        <id,3>
  5  ++       <op>
  5  )        <punct>
  5  {        <punct>
  7  if       <keyword>
  7  (        <punct>
  7  i        <id,3>
  7  %        <op>
  7  2        <num,2>
  7  ==       <op>
  7  0        <num,0>
  7  &&       <op>
  7  count    <id,2>
  7  !=       <op>
  7  3        <num,3>
  7  )        <punct>
  8  count    <id,2>
  8  =        <op>
  8  count    <id,2>
  8  +        <op>
  8  i        <id,3>
  8  ;        <punct>
  9  }        <punct>
 10  return   <keyword>
 10  count    <id,2>
 10  ;        <punct>
 11  }        <punct>

symbol table
bucket  3: 2 count (line 3)
bucket  3: 1 main (line 1)
bucket  6: 3 i (line 3)
bucket 10: 4 avg (line 4)
```

Read the symbol table at the end: `count` and `main` share bucket 3 (553 and 421 both leave 3 when divided by 11), with `count` in front because it was inserted later; `i` has number 3 and is reported on line 3, where it was first seen, even though it is used most on line 5. Every later `i` printed `<id,3>`: one record per name.

The empty input and the input with errors, after saving `avg.c` with a here-document:

<!-- code: id=cscan-tests exit=1 -->
```console
$ cat > avg.c <<'END'
int main()
{
    int count = 0, i;
    float avg = 2.5;   // running average
    for (i = 0; i < 10; i++) {
        /* add the even values */
        if (i % 2 == 0 && count != 3)
            count = count + i;
    }
    return count;
}
END
$ ./cscan < /dev/null

symbol table
$ printf 'int x = 3 # 4;\nfloat y = 1.;\nx = y & 1;\n' | ./cscan
  1  int      <keyword>
  1  x        <id,1>
  1  =        <op>
  1  3        <num,3>
line 1: error: unexpected character: #
  1  4        <num,4>
  1  ;        <punct>
  2  float    <keyword>
  2  y        <id,2>
  2  =        <op>
line 2: error: digit expected after '.': 1.
  2  ;        <punct>
  3  x        <id,1>
  3  =        <op>
  3  y        <id,2>
line 3: error: unexpected character: &
  3  1        <num,1>
  3  ;        <punct>

symbol table
bucket  0: 2 y (line 2)
bucket 10: 1 x (line 1)
3 lexical error(s)
```

The empty input prints only the heading of an empty table. The bad input reports its three errors with their lines, keeps scanning after each, still fills the table with `x` and `y`, and exits with status 1. The usual mistakes in this project: numbering names per occurrence instead of per name (then `i` would be 3, 5, 6 ...); forgetting to free the records (the sanitizer reports a leak); and letting a single `&` fall through to `op` instead of reporting it.

</details>
