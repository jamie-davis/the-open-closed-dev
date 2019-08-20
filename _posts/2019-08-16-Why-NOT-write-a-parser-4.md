---
layout: post
title: Why NOT write a parser? - Parser BNF
---

This is a follow up from my [Why NOT write a parser?](https://jamie-davis.github.io/the-open-closed-dev/why-not-write-a-parser/) post. In this post I will take you through designing the parser.

I am going to assume you are familiar with the preceding [post](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-3/) about creating a lexical analyser. The code in this post will be relying on the ```LexicalAnalyser``` type and the tokens it produces.

The parser we are going to write will make sense of simple mathematical expressions like these:

```ps
1
32 - 5
(5 * 3) + 45
145 * ((63/2 + 5) - 16)
```

The way the parser will see this input is after lexical analysis where ```(5 * 3) + 45``` will have been turned into a stream of tokens:

```ps
OpenParens
NumericLiteral (5)
Operator (*)
CloseParens
Operator (+)
NumericLiteral (45)
```

This post is going to be pretty theoretical and is going to deal with defining a medium complexity parser. The bulk of this post will not be of interest for very simple DSLs, but if you need to parse expressions, you will encounter the same difficulties that we will address here. The biggest issue, as we will see, is allowing the parser to deal with invalid input.

If you prefer to skip straight to the implementation, please [do](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-5/).

We are going to parse this using a recursive descent parser that looks ahead by one token, goes from left to right, and interpretation of a token only depends on what precedes it (or LL(1) if you prefer). While we are going to use it on arithmetic expressions, the method is very general, provided we can interpret the tokens in the way I just stated. If you are putting together your own DSL, then you can design it with this method in mind.

I chose the mathematical expression problem becuase it does not require a lot of tokens, but the expressions can be arbitrarily complex, so that I can demonstrate how we can reduce such complexity into something manageable. Most DSLs I've created don't have a way to nest things to an arbitrary level, and they are usually much easier to think about than this, so if you can understand what's going on here, you will be well equipped.

Let's break the problem down a bit.

- The most basic element is the ```NumericLiteral``` token, which is simply a number. It represents only a numeric value. For brevity, from now on I'm going to write this as just the number i.e. ```NumericLiteral(100)``` is just ```100``.

- The next most basic is an ```Operator```, which takes two operands, let's call them ```left``` and ```right```. Valid operands are expressions such as ```NumericLiterals```, other operands, or parenthesised expressions. e.g. 32 - 5 has the operator "minus", where ```left``` is 35 (```NumericLiteral(35)```) and ```right``` is 5 (```NumericLiteral(5)```). For brevity I'm going to write this as the operator and the operands e.g. ```-[35,5]```.

- The final element is the parenthesised expression. This may contain any or all of the other elements.I'll just write this as a set of parenthesis.

So let's say we start reading tokens and the first thing we see is ```55```. That had better be the end of the stream, or else followed by an operator or this is invalid input. If it's followed by ```+``` and then by ```10``` we would conclude we got ```+[55,10]``` and be happy that this is a legal expression.

Let's say we start reading tokens and the first thing we see is ```(```. Next we are expecting some valid expression. If we saw ```55```, then ```+``` and then ```10``` and finally ```)``` we would say we got ```(+[55,10])``` and be happy that this is a legal expression. 

Equally, if we see ```100 * (55+10)```, it's just the second example as the second expression inside the same pattern as the first example or:

```text
*[100,(+[55,10])]
```

Hopefully that's hinting at the solution to the parsing problem. Let's call the first example ```operatorExp```, and the second one ```parensExp```, and we might write it like this:

```text
<operatorExp> ::= NumericLiteral Operator NumericLiteral
<parensExp>   ::= OpenParens <operatorExp> CloseParens
```

This is called "Backus-Naur form" (BNF) and it's a formal way of expressing a "grammar" for a language. We've defined two rules (called "productions") - ```<operatorExp>``` and ```<parensExp>```. Anything in the grammar that's not a ```<rule>``` is called a "non-terminal" and is something we can check against the input tokens.

Going back to ```<operatorExp>```, the third example demonstrated that the operands need not be just numbers - they could be expressions, or literals, or they could be parenthesised expressions. We can express that in BNF:

```text
<operatorExp> ::= (NumericLiteral | <parensExp> | <operatorExp>) Operator (NumericLiteral | <operatorExp> | <parensExp>)
<parensExp>   ::= OpenParens <operatorExp> CloseParens
```

The "|" symbol means "OR". We could write it a bit more efficiently:

```text
<term>        ::= NumericLiteral | <parensExp> | <operatorExp>
<operatorExp> ::= <term> Operator <term>
<parensExp>   ::= OpenParens <operatorExp> CloseParens
```

We just said a ```<term>``` is a number or a ```<parensExp>``` or an ```<operatorExp>```. An ```<operatorExp>``` is a ```<term>``` followed by an ```Operator``` followed by a ```<term>```. A ```<parensExp>``` is an ```OpenParens``` followed by an ```<operatorExp>``` followed by a ```CloseParens```.

That's incorrect though because a ```<parensExp>``` could contain more than just an ```<operatorExp>```, but it is not allowed to contain a term because ```5 + (10)``` does not make sense. There's also the possibililty that the whole expression is just a numeric literal. Let's fix it:

```text
<calc>        ::= <calcExp> | NumericLiteral
<calcExp>     ::= <parensExp> | <operatorExp>
<term>        ::= NumericLiteral | <calcExp>
<operatorExp> ::= <term> Operator <term>
<parensExp>   ::= OpenParens <calcExp> CloseParens
```

What good has this done us? Well it gives us a view of the parser without having to build it. We can't run it yet, but we can walk through it with our examples. Let's try ```(5 * 3) + 45```:

```text
                                                                       Input
Try <calc>                                                             (5 * 3) + 45
    Try <calcExp>                                                      (5 * 3) + 45
        Try <parensExp>                                                (5 * 3) + 45
            Try OpenParens ... matched - take "("                      5 * 3) + 45
            Try <calcExp>                                              5 * 3) + 45
                Try <parensExp>                                        5 * 3) + 45
                    Try OpenParens ... no match (5)                    5 * 3) + 45
                Try <operatorExp>                                      5 * 3) + 45
                    Try <Term>                                         5 * 3) + 45
                        Try NumericLiteral ... matched - take "5"      * 3) + 45
                    Try Operator ... matched - take "*"                3) + 45
                    Try <Term>                                         3) + 45
                        Try NumericLiteral ... matched - take "3"      ) + 45
                Matched <operatorExp>                                  ) + 45
            Matched <calcExp>                                          ) + 45
            Try CloseParens ... match - take ")"                       + 45
        Matched <parensExp>                                            + 45
    Matched <calcExp>                                                  + 45
Matched <calc>                                                         + 45
```

So that revealed a problem. We didn't match all of the input.

The problem is that ```<calcExp>``` tries ```<parensExp>``` before it tries ```<operatorExp>```. If it tried those productions the other way around, "(5 * 3)" would have matched the first ```<term>``` in ```<operatorExp>``` allowing us to go on and match "+" as the ```Operator``` non-terminal and then another ```<term>```, which would have been the "45".

Corrected:

```text
<calc>        ::= <calcExp> | NumericLiteral
<calcExp>     ::= <operatorExp> | <parensExp>
<term>        ::= NumericLiteral | <parensExp> | <calcExp>
<operatorExp> ::= <term> Operator <term>
<parensExp>   ::= OpenParens <calcExp> CloseParens
```

```text
                                                                            Input
Try <calc>                                                                  (5 * 3) + 45
    Try <calcExp>                                                           (5 * 3) + 45
        Try <operatorExp>                                                   (5 * 3) + 45
            Try <term>                                                      (5 * 3) + 45
                Try NumericLiteral ... no match (()                         (5 * 3) + 45
                Try <parensExp>                                             (5 * 3) + 45
                    Try OpenParens ... matched - take "("                   5 * 3) + 45
                    Try <calcExp>                                           5 * 3) + 45
                        Try <operatorExp>                                   5 * 3) + 45
                            Try <term>                                      5 * 3) + 45
                                Try NumericLiteral ... matched - take "5"   * 3) + 45
                            Matched <Term>                                  * 3) + 45
                            Try Operator ... matched - take "*"             3) + 45
                            Try <term>                                      5 * 3) + 45
                                Try NumericLiteral ... matched - take "3"   ) + 45
                            Matched <Term>                                  ) + 45
                        Matched <operatorExp>                               ) + 45
                    Matched <calcExp>                                       ) + 45
                    Try CloseParens ... matched - take ")"                  + 45
                Matched <parensExp>                                         + 45
            Matched <term>                                                  + 45
            Try Operator ... matched - take "+"                             45
            Try <term>                                                      45
                Try NumericLiteral .. matched - take "45"                   FINISHED
            Matched <term>                                                  FINISHED
        Matched <operatorExp>                                               FINISHED
    Matched <calcExp>                                                       FINISHED
Matched <calc>                                                              FINISHED
```

There is another issue, let's try ```1 + 2 + 3```:

```text
                                                                            Input
Try <calc>                                                                  1 + 2 + 3
    Try <calcExp>                                                           1 + 2 + 3
        Try <operatorExp>                                                   1 + 2 + 3
            Try <term>                                                      1 + 2 + 3
                Try NumericLiteral ... matched - take "1"                   + 2 + 3
            Matched <term>                                                  + 2 + 3
            Try Operator ... matched - take "+"                             2 + 3
            Try <term>                                                      2 + 3
                Try NumericLiteral ... matched - take "2"                   + 3
            Matched <term>                                                  + 3
        Matched <operatorExp>                                               + 3
    Matched <calcExp>                                                       + 3
Matched <calc>                                                              + 3
```

We didn't take the whole calculation. Looks like ```<term>``` does not match enough. If we try to rewrite ```<term>``` to try a complete expression before a literal, say:

```text
<term>        ::= <calcExp> | <parensExp> | NumericLiteral
```

We would create an infinite loop because ```<calcExp>``` contains ```<operatorExp>``` which starts with ```<term>```, so we would encounter ```<term>``` again before we could have taken anything from a string. e.g.

```text
Try <calc>
    Try <calcExp>
        Try <operatorExp>
            Try <term>
                Try <calcExp> ...again (2nd time)
                    Try <operatorExp> ... again (2nd time)
                        Try <term> ... again (2nd time)
                            Try <calcExp> ... again (3rd time)
                                Try <operatorExp> ... again (3rd time)
                                    Try <term> ... again (3rd time)
... and so on forever
```

This is called left-recursion and we need to avoid it. If we can't then our grammer is not LL(1) and this method of parsing will not work. There is a fairly simple solution if we treat the second term in an operator expression as a different production:

```text
<operatorExp> ::= <term> Operator <secondTerm>
<term>        ::= NumericLiteral | <parensExp>
<secondTerm>  ::= <parensExp> | <operatorExp> | NumericLiteral
```

There I've said that a ```<term>``` can only be a ```NumericLiteral``` or a ```<parensExp>``` - it cannot be an ```<operatorExp>``` (whereas ```<secondTerm>``` can be any). This means that matching ```1 + 2 + 3``` would match like it was ```1 + (2 + 3)``` and not like it was ```(1 + 2) + 3```.

Here's a dry run of that:

```text
                                                                               Input
Try <calc>                                                                     1 + 2 + 3
    Try <calcExp>                                                              1 + 2 + 3
        Try <operatorExp>                                                      1 + 2 + 3
            Try <term>                                                         1 + 2 + 3
                Try NumericLiteral ... matched - take "1"                      + 2 + 3
            Matched <term>                                                     + 2 + 3
            Try Operator ... matched - take "+"                                2 + 3
            Try <secondTerm>                                                   2 + 3
                Try <parensExp>                                                2 + 3
                    Try OpenParens ... no match                                2 + 3
                Try <operatorExp>                                              2 + 3
                    Try <term>                                                 2 + 3
                        Try NumericLiteral ... matched - take "2"              + 3
                    Matched <term>                                             + 3
                    Try Operator ... matched - take "+"                        3
                    Try <secondTerm>                                           3
                        Try <parensExp>                                        3
                            Try OpenParens ... no match                        3
                        Try <operatorExp>                                      3
                            Try <term>                                         3
                                Try NumericLiteral ... matched - take "3"      FINISHED
                            Matched <term>                                     FINISHED
                            Try Operator ... no match (used "3" is put back)   3
                        Try NumericLiteral ... matched - take "3"              3
                    Matched <secondTerm>                                       FINISHED
                Matched <operatorExp>                                          FINISHED
            Matched <secondTerm>                                               FINISHED
        Matched <operatorExp>                                                  FINISHED
    Matched <calcExp>                                                          FINISHED
Matched <calc>                                                                 FINISHED
```

Therefore, corrected:

```text
<calc>               ::= <calcExp> | NumericLiteral
<calcExp>            ::= <operatorExp> | <parensExp>
<operatorExp>        ::= <term> Operator <secondTerm>
<term>               ::= NumericLiteral | <parensExp>
<secondTerm>         ::= <parensExp> | <operatorExp> | NumericLiteral
<parensExp>          ::= OpenParens <calcExp> CloseParens
```

Another thing to think about is whether this will go haywire if the expression is invalid. Lets try an example:

```text
                                                                  Input
Try <calc>                                                        1 + +
    Try <calcExp>                                                 1 + +
        Try <operatorExp>                                         1 + +
            Try <term>                                            1 + +
                Try NumericLiteral ... matched - take "1"         + +
            Matched <term>                                        + +
            Try Operator ... matched take "+"                     +
            Try <secondTerm>                                      +
                Try <parensExp>                                   +
                    Try OpenParens ... no match                   +
                Try <operatorExp>                                 +
                    Try <term>                                    +
                        Try NumericLiteral ... No match           +
                        Try <parensExp>                           +
                            Try OpenParens ... No match           +
        Try <parensExp>                                           +
            Try OpenParens --- No match                           +
```

The match failed. If you go through the process with other invalid expressions, you get the similar results.

In the next post we will convert the BNF into code.

Thanks for reading,

Jamie.
