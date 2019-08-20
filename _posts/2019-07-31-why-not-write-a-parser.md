---
layout: post
title: Why NOT write a parser?
---

Building a parser is one of those hard problems that you think twice about before attempting, but I think it's good to have the know-how in your toolkit because now and then, they do make sense. This is particularly true if you are thinking like a library author<sup id="a1">[1](#f1)</sup>.

An example of where I've needed a parser was in an automatic configuration feature, where the config object could be annotated with a ```[ConfigSource(something)]``` attribute:

```text
[ConfigSource("default(true)")]
public bool IntegrationsActive { get; set; }
```

```text
[ConfigSource("storage.connectionstring(accounts)")]
public string AccountsStorageConnectionString { get; set; }
```

```text
[ConfigSource(@"api.base(mainwebSite, ""/home/integration/authorise"")")]
public string WbsiteAuthUri { get; set; }
```

```text
[ConfigSource("sql.adminconnection(portal)")]
public string DbUpConnectionString { get; set; }
```

The attributes were parsed and the config was automatically populated with values appropriate to the environment. Rather than having to manually find the values just to get your environment running, all you needed to know was the environment name. It also doubled as reliable documentation of what each setting meant.

From here on, I'm going to outline the method I use to build parsers in C#. The principles are transferrable, and the code should not involve any magical C# instructions that don't have equivalents in other languages<sup id="a2">[2](#f2)</sup>. I should warn you that I'm going to describe a way to build a recursive descent parser with backtracking. This means it's fine for simple grammars but would not be suitable for many modern programming languages - if you are building a compiler you'll probably need to go deeper<sup id="a3">[3](#f3)</sup>.

Parsing takes place in three main steps:

1. Lexical analysis produces a stream of tokens.

2. The parser takes the tokens and produces an abstract syntax tree (AST).

3. The AST is converted into the actions implied by the input.

#### Lexical Analysis

Briefly, lexical analysis coverts our essentially free format string into a set of classified tokens. For example, consider a simple arithmetical expression:

```1 + (3 * 5)```

A lexical analyser would break this down into a set of tokens:

```text
number      ("1")
operator    ("+")
openparens  ("(")
number      ("3")
operator    ("*")
number      ("5")
closeparens (")")
```

The lexical analyser needs to deal with identifying the tokens in the input, ignoring white space, and can also spot tokens that don't match our rules (i.e. strings that don't conform to one of our tokens).

This allows us to write the parser purely in tems of the token types ("number", "operator", "openparens", and "closeparens"), instead of checking character by character.

#### Parsing

Parsing converts our stream of tokens into an *abstract syntax tree*. Recursive descent parsers convert the input into a heirarchy. Reusing the example from above:

```1 + (3 * 5)```

We might convert that into:

```text
      ADD
       |
   ----------
  |          |
VAL(1)     MULT
             |
          -------
         |       |
       VAL(3)  VAL(5)
```

#### Actions

The final step is to deduce the actions required based on the AST. What we do will depend on why we are parsing the input. For a compiler, the input describes a program and it would emit low level code to order. If we just want to compute the answer, we can get away with asking the root tree node for its value and have each node type understand how to return it.

For example:

A VAL(n) node would return n.

A MULT node would always have two children and would multiply whatever they return as values.

An ADD node would always have two children and would add whatever they return as values.

```text
      ADD = 1 + 15 = 16
             |
     ----------------
    |                |
VAL(1) = 1    MULT = 3 * 5 = 15
                     |
                ------------
               |            |
           VAL(3) = 3   VAL(5) = 5
```

#### How?
I will discuss how to build the code for each of the three steps in follow on posts:

1. [Preparation](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-2/)

2. [Lexical Analyser](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-3/)

3. [Parser BNF](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-4/)

4. [Parser BNF To Code](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-5/)

5. [TDD Parser (i.e. what if you don't use BNF?)](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-6/)

Thanks for reading.

Jamie


<b id="f1">1</b> Always think like a library author. :grin: [↩](#a1)

<b id="f2">2</b> It's a sweeping generalisation, that may not be true for your language but it's true for most. [↩](#a2)

<b id="f3">3</b> To be more precise, I'm going to discuss LL(1) (**L**eft-to-right, **L**eftmost-derivation, look ahead by **1** token) parsers with backtracking. This is only good for unambiguous context free grammars, which is a fine way to build a simple domain specific language, but it's probably too limited for the next great programming language.

I'm self educated on this stuff, so there's a chance I'll misuse terminology from time-to-time - for example, I often say "clause" when you are supposed to say "production". I'll let you research what that means.

If you do want to go deeper, there is a ton of stuff on the internet. Have a look at Backus–Naur form (BNF) as a way to describe a grammar, and tools like LEXX and YACC which can do a lot of what we will do manually.

For grammars that are not LL(1), you will need something more complex than recursive descent.
 [↩](#a3)
