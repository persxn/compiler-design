# Compiler Design practices worth teaching

Sources read 2026-11-21 as reading material only:
- [Run] commands run on the build machine on 2026-11-21: GCC 13.3, flex 2.6.4, GNU Bison 3.8.2
- [Flex manual] Flex manual, https://westes.github.io/flex/manual/ (Options: noyywrap; Generated scanner; Start conditions)
- [Bison manual] GNU Bison manual, https://www.gnu.org/software/bison/manual/ (Yacc compatibility; Precedence; Understanding the .output file)
- [GCC 14 porting] Porting to GCC 14, https://gcc.gnu.org/gcc-14/porting_to.html (implicit function declarations)
- [Dragon] Aho, Lam, Sethi, Ullman, Compilers: Principles, Techniques and Tools, 2nd ed. (the notation the exam papers use)

## Habits

1. Build every C program with `gcc -Wall -Wextra` and treat each warning as a bug to fix before the examiner sees it. From chapter 1.
2. Never name an output file after a source file: `gcc lexer.c -o lexer`, never `-o lexer.c`. From chapter 1.
3. Test each program on a normal input, an empty input and one input that must be rejected, and show all three when asked. From chapter 2.
4. Put `%option noyywrap` in every Lex file that has its own `main`, so it links with no library. From chapter 4.
5. In every Yacc file declare `int yylex(void);` and `void yyerror(const char *s);` before use. From chapter 8.
6. Use the Dragon book's notation (E, T, F, id, $ as end marker, FIRST/FOLLOW, I0, I1 ...) so hand work matches the exam paper. From chapter 3.
7. Check a hand-worked table (FIRST/FOLLOW, LL(1), SLR, DFA) against the book's program for the same grammar before trusting it. From chapter 5.
8. Read input with `fgets` or `getchar`, never `gets`. From chapter 2.

## Facts that change

- flex's generated scanner calls `yywrap()` at end of input; with no `%option noyywrap` and no `-lfl` (or `-ll`) the link fails with "undefined reference to `yywrap'". [Run, 2026-11-21]
- On many lab machines the flex library is linked as `-lfl`; `-ll` is the AT&T lex name and exists only where a libl alias is installed. [Flex manual, 2026-11-21]
- `yacc` on Linux is usually GNU Bison in Yacc mode (`bison -y`); `yacc -d file.y` writes `y.tab.c` and `y.tab.h`. Plain `bison -d file.y` writes `file.tab.c` and `file.tab.h` instead. [Run, 2026-11-21]
- GCC 13 only warns about calling `yylex` or `yyerror` with no declaration (`-Wimplicit-function-declaration`); from GCC 14 this is an error by default. [Run, 2026-11-21; GCC 14 porting, 2026-11-21]
- `bison -v` (or `yacc -v`) writes the parser's states and any conflicts to an `.output` file (`y.output` in Yacc mode). [Bison manual, 2026-11-21]
- `gets` was removed from C in C11; GCC warns, and the program may not link on newer C libraries. [GCC 14 porting, 2026-11-21]

## Handle with care

- `gcc prog.c -o prog.c`: overwrites the source with the binary; always give the output a different name. [Run, 2026-11-21]
- `cat > file.l <<'END'` and `>` redirection: replace a file that already exists; the book writes each file once per folder and uses a new folder per chapter. [Run, 2026-11-21]
- `rm lex.yy.c y.tab.*` or `rm *.c`: a wildcard can take the learner's own sources with the generated ones; the book removes generated files by their exact names only. [Run, 2026-11-21]
