## Chapter 1: Follow one statement through a compiler

- Risks: 9 items (9 marked, 0 destructive, 0 unrun, 0 dangling). Settled: 1 by PRACTICES.md, 2 run, 0 one look (source), 0 known for certain, 6 reworded as uncertain; each now a `checked` mark.
- One read: 2 case, 0 order, 1 page, 0 words, 0 repeats fixed.
- Fixed: 1.1 stage-compile now says what an ARM machine shows (fmul, fadd, bl printf); 1.1 driver says the cc1/collect2 folder varies; 1.1 no-include answer now says GCC 14 makes the implicit declaration an error and no program is made (was "some missing declarations"); 1.2 dump file numbers and dump text said to vary by GCC version, .LC0 layout said to be x86-64; 1.6 viva share of marks said to vary by lab; 1.3 comments "as you saw in pos.i" (pos.c has no comment) now says a comment you add is gone from pos.i; 1.3 'rat' found "nothing" in the table now "no declaration", agreeing with 1.4's answer; 1.3 answer said prnitf "reached the linker before anything objected", contradicting 1.1's warning, now "got only a warning and failed at the linker".
- Not run: none (1.4 local-variable nm claim and 1.1 prnitf on GCC 13 were run by hand; Ubuntu package names checked with apt-cache).
- Open: two FLAGs for "symbol table" in 1.2 (figure alt text) and 1.3 (token table) left: the term is given in plain words with (§1.4) at its first use in 1.2 and again in 1.3.
- Score 9: accuracy 9, depth 9, clarity 8, sequencing 8, practical 9, exercises 9, completeness 9, efficiency 8. Lowest: 1.2 symbol table used before 1.4, with a plain-words line and (§1.4).
- 2026-11-21 chapter 1 reviewed; open: two symbol-table look-ahead FLAGs in 1.2/1.3, plain words given

## Chapter 2: Write a lexical analyzer in C

- Risks: 7 items (7 marked, 0 destructive, 0 unrun, 0 dangling). Settled: 0 by PRACTICES.md, 3 run, 0 one look (source), 3 known for certain, 1 reworded as uncertain; each now a `checked` mark.
- One read: 0 case, 0 order, 1 page, 0 words, 0 repeats fixed.
- Fixed: 2.1 `char c` pitfall said it compiles without a warning everywhere; a scratch build with -funsigned-char shows -Wextra (-Wtype-limits) warns, so it now says "on a PC" and quotes the unsigned-char warning; 2.1 walk-through said a space is "neither a letter nor printable" (the code tests isspace) and now says "white space"; 2.8 "real compilers" use h*31+c narrowed to "better hash functions", citing K&R 6.6.
- Not run: none (EOF value, two ungetc calls on glibc and char signedness were run in a scratch folder, outside the chapter's programs).
- Open: 2.5 id-num-diagram alt text says "top to bottom" while the figure spec says dir: right (generated text, for us-draw); 2.3 scan-ops, 2.5 relop, 2.8 symtab, 2.9 cscan are long programs with no build-up (check NOTE), too big for this pass.
- Score 9: accuracy 9, depth 9, clarity 8, sequencing 9, practical 9, exercises 9, completeness 9, efficiency 8. Lowest: 2.7 and 2.8 long (2938, 2776 words) with long programs shown whole.
- 2026-11-21 chapter 2 reviewed; open: 2.5 figure alt text direction; four long programs without build-up

## Chapter 3: Regular expressions and finite automata

- Risks: 7 items (7 marked, 0 destructive, 0 unrun, 0 dangling). Settled: 0 by PRACTICES.md, 3 run, 0 one look (source), 4 known for certain, 0 reworded as uncertain; each now a `checked` mark.
- One read: 0 case, 0 order, 2 page, 0 words, 1 repeats fixed.
- Fixed: 3.1 unquoted `./match (a|b)*abb` said bash reports an error "near `(`"; a run shows ``syntax error near unexpected token `a'``, now quoted; 3.6 signed-char pitfall said `isalpha` "reads outside the array" (glibc's table covers -128..255), now says it is undefined behaviour because `<ctype.h>` takes only unsigned char values or EOF; 3.2 `abababb` said to go round the star "five times", now three (the star takes `abab`); 3.3 "it is how `match` ... works" (unchecked claim about glibc) now "many libraries are built on the same idea"; 3.7 task 4's question and task 3's whole answer were pasted twice, the second copy outside any `<details>`, exposing an answer: duplicate removed; repair for efficiency in 3.2: cut the numbering explanation said three times (after the output, in the viva list, and a closing paragraph) to once.
- Not run: none (bash message, flex's yy_last_accepting_state and char signedness run in a scratch folder; task 3's `a(a|b)*b` pipeline count run with `us_run.py try`).
- Open: 3.2 (3806 words), 3.3 (3558) and 3.5 (3042) above the lesson range; thompson, subset and minimise are long programs shown whole with no build-up (check NOTE), too big for this pass; 3.1 the quoting advice is said both after match-abb and in "When it goes wrong" (left: the second carries the real message).
- Score 9: accuracy 9, depth 9, clarity 8, sequencing 9, practical 9, exercises 9, completeness 9, efficiency 7. Lowest: 3.2 3806 words with a 189-line program shown whole; repeats cut, size remains.
- 2026-11-21 chapter 3 reviewed; open: 3.2/3.3/3.5 over length; thompson, subset, minimise without build-up
