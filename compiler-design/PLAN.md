# Compiler Design: the lab and exam coding book
for: a B.Tech computer science student who can write C programs and has met compiler-design theory in lectures, but has never written a scanner, a parser or a Lex/Yacc program alone, and must do so in the lab exam and work the algorithms by hand in the semester paper
run: c
kind: system

## Part I · See the whole compiler at once

### 1 Follow one statement through a compiler
1.1 What happens between prog.c and a.out: watching gcc stop after each stage | teaches: compile-pipeline, preprocessor, assembler, linker
1.2 The phases of a compiler, traced on pos = init + rate * 60 | teaches: compiler-phase, front-end, back-end | fig
1.3 Tokens, lexemes and patterns | teaches: token, lexeme, pattern
1.4 The symbol table: the notebook every phase shares | short | teaches: symbol-table-idea
1.5 Passes, interpreters, cross compilers and compiler-construction tools | short | teaches: compiler-pass, interpreter, cross-compiler
1.6 How a lab exam program is written, built, run and marked | short | teaches: lab-exam-cycle
1.7 Practice: trace a new statement through every phase by hand | practice | level: 2

## Part II · Scanning: from characters to tokens

### 2 Write a lexical analyzer in C
2.1 Reading source one character at a time, with one character of lookahead | teaches: lookahead, ungetc | needs: token
2.2 Recognising identifiers and telling keywords apart | teaches: identifier-scan, keyword-table
2.3 Numbers, operators and the longest-match rule | teaches: number-scan, longest-match
2.4 Skipping white space and both kinds of C comment | teaches: comment-skip
2.5 Transition diagrams: drawing the scanner before writing it | teaches: transition-diagram | needs: pattern | fig
2.6 Input buffering: buffer pairs and sentinels | short | teaches: buffer-pair, sentinel
2.7 Lexical errors and line numbers | teaches: lexical-error
2.8 A symbol table in C: insert, search, display, delete with a hash table | teaches: symbol-table, hash-chaining | level: 3
2.9 Practice: a complete scanner for a C subset that fills a symbol table | practice | level: 3

### 3 Regular expressions and finite automata
3.1 Regular expressions as patterns: union, concatenation, star and the shorthands | teaches: regex, regular-definition | needs: pattern
3.2 From a regular expression to an NFA: Thompson's construction | teaches: nfa, thompson-construction | fig
3.3 Epsilon-closure and the subset construction, by hand and in C | teaches: epsilon-closure, subset-construction | level: 3
3.4 A DFA straight from a regular expression: nullable, firstpos, lastpos, followpos | teaches: followpos, direct-dfa | fig | level: 2
3.5 Minimising a DFA by splitting groups | teaches: dfa-minimisation | level: 2
3.6 A table-driven DFA recogniser in C | teaches: dfa-simulation | needs: transition-diagram | level: 3
3.7 Practice: one regular expression carried to a minimal DFA and a C recogniser | practice | level: 3

### 4 Lex and Flex programs
4.1 The first Lex program: the three parts of a .l file, flex, gcc, run | teaches: lex-structure, yylex, yytext | needs: regex
4.2 Patterns and actions: longest match first, then the earlier rule | teaches: lex-rule-order, lex-pattern
4.3 Counting programs: lines, words, characters, vowels and consonants | teaches: lex-counting, yyleng
4.4 A Lex scanner for keywords, identifiers, numbers and operators | teaches: lex-token-scanner | needs: token, symbol-table
4.5 Reading files and ending input: yyin, yyout, yywrap and noyywrap | teaches: yyin, yywrap
4.6 Start conditions: removing comments and scanning strings | teaches: start-condition | level: 2
4.7 When flex or gcc complains: the usual errors and their cures | teaches: lex-errors
4.8 Practice: the lab set of Lex programs, written from the question alone | practice | level: 3

## Part III · Parsing: from tokens to structure

### 5 Grammars ready for parsing
5.1 Context-free grammars, derivations and parse trees | teaches: cfg, derivation, parse-tree | needs: token | fig
5.2 Ambiguity, and how precedence and associativity remove it | teaches: ambiguity, precedence-grammar | level: 2
5.3 Eliminating left recursion, by hand and in C | teaches: left-recursion | level: 3
5.4 Left factoring, by hand and in C | teaches: left-factoring | level: 3
5.5 FIRST sets, by hand and in C | teaches: first-set | level: 3
5.6 FOLLOW sets, by hand and in C | teaches: follow-set | level: 3
5.7 Practice: transform and compute for exam grammars | practice | level: 3

### 6 Top-down parsers
6.1 A recursive-descent parser in C | teaches: recursive-descent | needs: first-set, left-recursion | level: 3
6.2 Building the LL(1) parsing table | teaches: ll1-table | needs: first-set, follow-set
6.3 Is a grammar LL(1)? Finding the conflict | teaches: ll1-conflict | level: 2
6.4 A table-driven predictive parser in C, with its stack trace | teaches: predictive-parser | fig | level: 3
6.5 Error recovery: panic mode and synch entries | short | teaches: panic-mode
6.6 Practice: grammar to table to parser for a new language | practice | level: 3

### 7 Bottom-up parsers
7.1 Shift-reduce parsing and handles, with a C program | teaches: shift-reduce, handle | level: 3
7.2 Operator-precedence parsing: relations, table and a C parser | teaches: operator-precedence | level: 3
7.3 LR(0) items, closure and goto: the canonical collection | teaches: lr0-item, closure, goto-function | fig
7.4 The SLR(1) parsing table | teaches: slr-table | needs: follow-set
7.5 CLR(1) and LALR(1) tables by hand | teaches: lr1-item, clr-table, lalr-table | level: 2
7.6 Shift-reduce and reduce-reduce conflicts | teaches: lr-conflict | level: 2
7.7 The LR driver in C: one table, one stack | teaches: lr-driver | level: 3
7.8 Practice: the full bottom-up question set | practice | level: 3

### 8 Yacc and Bison programs
8.1 The first Yacc program: the parts of a .y file and linking it with Lex | teaches: yacc-structure, yyparse, yylval | needs: yylex, cfg
8.2 Validating arithmetic expressions | teaches: yacc-validator
8.3 A calculator: values, precedence and associativity | teaches: yacc-precedence, yacc-value | needs: precedence-grammar | level: 3
8.4 Recognising languages: identifiers, a^n b^n and other classics | teaches: yacc-language
8.5 Reading Bison's conflict reports with -v and the .output file | teaches: yacc-conflict-report | needs: lr-conflict | level: 2
8.6 Infix to postfix with semantic actions | teaches: semantic-action | level: 3
8.7 Practice: the lab set of Yacc programs, written from the question alone | practice | level: 3

## Part IV · Translating, optimising and generating code

### 9 Syntax-directed translation and intermediate code
9.1 Syntax-directed definitions: synthesised and inherited attributes, annotated parse trees | teaches: sdd, synthesised-attribute, inherited-attribute | needs: parse-tree | fig
9.2 Three-address code and its kinds of instruction | teaches: three-address-code
9.3 Quadruples, triples and indirect triples, in C | teaches: quadruple, triple, indirect-triple | level: 3
9.4 Generating three-address code for expressions with Yacc | teaches: tac-generation | needs: yacc-value | level: 3
9.5 Boolean expressions, if and while: labels and backpatching | teaches: control-flow-tac, backpatching | level: 2
9.6 Type checking and type conversion | short | teaches: type-checking
9.7 Practice: from source to three-address code for a new program | practice | level: 3

### 10 Optimisation, code generation and the run-time picture
10.1 Basic blocks, leaders and flow graphs, in C | teaches: basic-block, leader, flow-graph | needs: three-address-code | fig | level: 3
10.2 Local optimisation: constant folding, propagation, common subexpressions, dead code | teaches: constant-folding, cse, dead-code | level: 3
10.3 The DAG of a basic block | teaches: dag | needs: quadruple | fig | level: 2
10.4 Loop optimisation: code motion, induction variables, strength reduction | teaches: code-motion, strength-reduction | level: 2
10.5 Peephole optimisation | short | teaches: peephole
10.6 A simple code generator: register and address descriptors, getreg | teaches: code-generator, getreg | level: 3
10.7 Activation records and storage allocation | short | teaches: activation-record
10.8 Practice: the mini compiler, end to end | practice | level: 3 | needs: tac-generation, lex-token-scanner
