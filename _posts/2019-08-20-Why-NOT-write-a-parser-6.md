---
layout: post
title: Why NOT write a parser? - TDD Parser Code
---

This is a follow up from my [Why NOT write a parser?](https://jamie-davis.github.io/the-open-closed-dev/why-not-write-a-parser/) post. In this post I will take you through creating the parser using TDD.

I am going to assume you are familiar with the preceding [post](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-3/) about creating a lexical analyser. The code in this post will be relying on the ```LexicalAnalyser``` type and the tokens it produces.

The parser we are going to write will make sense of mathematical expressions like these:

```ps
1
32 - 5
(5 * 3) + 45
145 * ((63/2 + 5) - 16)
```

There is essentially no limit to how complex an expression can become, and this means it is quite a challenge to understand. You might want to look at the [BNF article](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-4/) where I break the problem down. However, in this post, I am going to build the parser without referring to the analysis, relying on TDD to help me arrive at something functionally equivalent to the parser I (kind of) mechanically derived [here](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-5/). My aim for this is to produce a (possibly) smaller codebase through test driven development. I'm not saying the TDD process is better - but a lot of the parsers you are likely to build will be simple enough not to need a BNF syntax, so the process I'm going to show here is good enough, and I will have covered all of the bases, so to speak.

The way the parser will see input is after lexical analysis where an expression such as ```(5 * 3) + 45``` will have been turned into a stream of tokens:

```ps
OpenParens
NumericLiteral (5)
Operator (*)
CloseParens
Operator (+)
NumericLiteral (45)
```

To start with, let's get a unit test ready (this is going to seem awfully familiar if you've read the lexical analyser [post](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-3/):

```c#
        [Fact]
        public void InputIsConverted()
        {
            //Arrange
            var testStrings = new [] {
                "(3 *5)+  49"
            };

            //Act
            var results = testStrings
                .Select(s => new { String = s, Result = ArithmeticParser.Parse(s)})
                .ToList();

            //Assert
            var output = new Output();
            var report = results.AsReport(rep => rep
                .AddColumn(r => r.String, cc => cc.Heading("Input"))
                .AddColumn(r => r.Result.IsValid, cc => { })
                .AddColumn(r => r.Result.Describe(), cc => cc.Heading("Result"))
                .AddColumn(r => r.Result.ErrorMessage, cc => {})
            );

            output.FormatTable(report);
            output.Report.Verify();
        }
    }
```

This is not compiled as yet. In the [grammar post](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-3/) I tried to illustrate what comes out of the parsing process using a format like this:  ```+[*[3,5],49]```. While it's not as accessible as the original text, it's a concise way to show what the parser extracted and I think it's easier to read (at least for a programmer) than attempting to express the abstract syntax tree as a tree, or writing potentially dozens of unit tests to pick apart the expected tree.

Using an approval style test and a good text format is much easier to live with than large numbers of terse and rather hard to read unit tests. This is definitely a case where the approval style gives huge benefits in the form of reduced development time and, more importantly, in improved understanding.

We are looking to approve something a bit like this:

```text
                          Is
Input                     Valid Result                          Error Message
------------------------- ----- ------------------------------- -------------------------------
1                         True  1
32-5                      True  -[32, 5]
(3 *5)+  49               True  +[(*[3, 5]), 49]
```

We could make each calculation a test of it's own, maybe like this:

```c#
        [Fact]
        public void InputIsConverted()
        {
            //Arrange
            var testString = "(3 *5)+  49";

            //Act
            var result = ArithmeticParser.Parse(testString);

            //Assert
            result.Describe().Should().Be("+[*[3,5],49]");
        }
```

It's almost as easy to add a new test case, and almost as easy to read the output. However, what I like about the report format is that you can more easily understand what the parser is doing when you can see all of the test cases, and all of the formatted results. If you need to work on the parser but do not happen to be the author, then the report is a better starting point because it has better readability. I won't dispute that there may be a case for a mixture of both styles.

The last point I'd like to discuss is that we've not yet made any allowance for errors in the format. We may want to test those seperately, or we may want to alter the report a little bit to include testing of badly formed expressions (such as ```5 * +```) where we need to prove that the error is spotted. We will defer this for now and try to get some of the valid path done first.

The next step is to implement the parser itself. Our parsing function needs roughly this structure:

```ps
Tokenise the string
Try to extract the root production from the tokens
if it works and there's no remaining input
    return the result
else
    return the error
```

The first question I have is what should we return? I've been down this road [before](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-5/), and this is what I ended up with:

```c#
    public class ArithmeticExpression
    {
        private ExpressionNode _calculation;

        internal ArithmeticExpression(ExpressionNode calculation)
        {
            _calculation = calculation;
            IsValid = true;
        }
        internal ArithmeticExpression(string errorMessage)
        {
            ErrorMessage = errorMessage;
            IsValid = false;
        }

        public string Describe()
        {
            return _calculation?.Describe();
        }

        public bool IsValid {get;}

        public string ErrorMessage {get;}
    }
```

In the interest of functional equivalence, let's go with that.

The skeletal parser function looks like this:

```c#
    public static class ArithmeticParser
    {
        public static ArithmeticExpression Parse(string input)
        {
            var tokens = LexicalAnalyser.ExtractTokens(input ?? string.Empty);
            var pos = new TokenKeeper(tokens);
            if (TryTakeCalc(pos, out var calculation))
            {
                return new ArithmeticExpression(calculation);
            }
            else
            {
                var errorMessage = $"Unable to interpret calculation at \"{pos.RemainingData()}\"";
                return new ArithmeticExpression(errorMessage);
            }
        }

    }
```

You can see that it implies that we can create an ```ArithmeticExpression``` in two ways:

- with whatever comes back from ```TryTakeCalc``` in the ```calculation``` argument when ```TryTakeCalc``` returns true.

- with an error message argument when ```TryTakeCalc``` returns false.

We can deal with the error message like this:

```c#
        public ArithmeticExpression(string errorMessage)
        {
            ErrorMessage = errorMessage;
            IsValid = false;
        }
```

The question of what ```calculation``` will contain is more interesting. This is the real result of parsing and we know it should be the root of an abstract syntax tree, so with that assumption we should invent a base class for the tree nodes. Again, I have this ready:

```c#
namespace Parser.Parsing
{
    internal abstract class ExpressionNode
    {
        internal abstract string Describe();

        internal abstract IEnumerable<ExpressionNode> ContainedNodes();
    }
```

We will derive concrete nodes from this class and implement ```Describe()``` to produce the strings we will show in the approval report, and ```ContainedNodes``` so that we can iterate over the whole tree.

We can now define ``TryTakeCalc```:

```c#
        private static bool TryTakeCalc(TokenKeeper pos, out ExpressionNode calculation)
        {
            throw new NotImplementedException();
        }
```

If you think back to the [```LexicalAnalyser```](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-3/), I said I like to use a class to represent my position in the input, and I think we have the same issue with the token stream. For that reason I like to create a class called ```TokenKeeper``` which is the same in principle as ```StringKeeper```. I'm not going to distract us by building ```TokenKeeper``` as part of this post, but the [implementation](https://github.com/jamie-davis/ParserDemo/blob/master/Parser/Parsing/TokenKeeper.cs) can be found in the git repo.

Let's put some simple strings into our unit test:

```c#
            var testStrings = new [] {
                "1",
                "32-5"
            };
```

I've chosen those strings because they show the minimalist and basic structure of a calculation. Let's try to pull out "1" and have "32-5" be an error:

```c#
        private static bool TryTakeCalc(TokenKeeper pos, out ExpressionNode calculation)
        {
            if (pos.Next.TokenType == TokenType.NumericLiteral)
            {
                var literal = pos.Take();
                calculation = new NumericLiteralNode(literal);
                return true;
            }

            calculation = null;
            return false;
        }
```

Along with:

```c#
    internal class NumericLiteralNode : ExpressionNode
    {
        private decimal _value;

        public NumericLiteralNode(Token token)
        {
            _value = decimal.Parse(token.Text);
        }

        internal override string Describe()
        {
            return _value.ToString();
        }

        internal override IEnumerable<ExpressionNode> ContainedNodes()
        {
            yield break;
        }
    }
```

Giving:

```text
      Is
Input Valid Result Error Message
----- ----- ------ ----------------------------------------
1     True  1
32-5  False        Unable to interpret calculation at "- 5"
```

That looks correct. "1" was pulled out successfully, as was "32" but some tokens were not used and therefore the expression is in error.

Hopefully this is easy to understand:

```c#
            if (pos.Next.TokenType == TokenType.NumericLiteral)
            {
                var literal = pos.Take();
                calculation = new NumericLiteralNode(literal);
                return true;
            }

            calculation = null;
            return false;
```

If our next token is a number, consume it and return a ```NumericLiteralNode```, otherwise don't consume anything and return false.

Let's broaden our horizon a little and try to extract "32-5" correctly:

```c#
        private static bool TryTakeCalc(TokenKeeper pos, out ExpressionNode calculation)
        {
            if (TryTakeOperatorExp(pos, out calculation)
             ||TryTakeLiteral(pos, out calculation))
            {
                return true;
            }

            calculation = null;
            return false;
        }

        private static bool TryTakeOperatorExp(TokenKeeper pos, out ExpressionNode node)
        {
            var work = new TokenKeeper(pos);
            if (TryTakeLiteral(work, out var left)
             && TryTakeOperator(work, out var op)
             && TryTakeLiteral(work, out var right))
            {
                pos.Swap(work);
                node = new OperatorExpNode(left, op, right);
                return true;
            }

            node = null;
            return false;
        }

        private static bool TryTakeOperator(TokenKeeper pos, out OperatorToken op)
        {
            if (pos.Next.TokenType == TokenType.Operator)
            {
                op = pos.Take() as OperatorToken;
                return true;
            }

            op = null;
            return false;
        }

        private static bool TryTakeLiteral(TokenKeeper pos, out ExpressionNode node)
        {
            if (pos.Next.TokenType == TokenType.NumericLiteral)
            {
                var literal = pos.Take();
                node = new NumericLiteralNode(literal);
                return true;
            }

            node = null;
            return false;
        }
    }
```

Two things to notice there... ```TryTakeCalc``` has changed to this:

```c#
            if (TryTakeOperatorExp(pos, out calculation)
             || TryTakeLiteral(pos, out calculation))
            {
                return true;
            }

            calculation = null;
            return false;
```

Our existing literal code is now in ```TryTakeLiteral``` and we try that after we've tried the new routine ```TryTakeOperatorExp```. Because "32-5" starts with a number, ```TryTakeLiteral``` would match it, so we need to try the more complex expression first.

```TryTakeOperatorExp``` is interesting:

```c#
            var work = new TokenKeeper(pos);
            if (TryTakeLiteral(work, out var left)
             && TryTakeOperator(work, out var op)
             && TryTakeLiteral(work, out var right))
            {
                pos.Swap(work);
                node = new OperatorExpNode(left, op, right);
                return true;
            }

            node = null;
            return false;
```

It makes a copy of the input ```TokenKeeper``` which it uses until it has extracted a whole operator expression before it "commits" to the tokens it consumes like this:

```c#
                pos.Swap(work);
```

Therefore, if it can't extract a whole expression, ```pos``` will remain unchanged.

And finally we need to define the operator expression node:

```c#
    internal class OperatorExpNode : ExpressionNode
    {
        private ExpressionNode _left;
        private OperatorToken _op;
        private ExpressionNode _right;

        public OperatorExpNode(ExpressionNode left, OperatorToken op, ExpressionNode right)
        {
            _left = left;
            _op = op;
            _right = right;
        }

        internal override string Describe()
        {
            return $"{_op.Text}[{_left.Describe()}, {_right.Describe()}]";
        }

        internal override IEnumerable<ExpressionNode> ContainedNodes()
        {
            yield return _left;
            yield return _right;
        }
    }
```

Giving:

```text
      Is             Error
Input Valid Result   Message
----- ----- -------- -------
1     True  1
32-5  True  -[32, 5]
```

Looking good so far. We have established a pattern for both taking a simple token (```TryTakeLiteral```) and taking something more complex (```TryTakeOperatorExp```). These patterns will be repeated to build the whole parser.

Let's add another type of expression to the test:

```c#
            var testStrings = new [] {
                "1",
                "32-5",
                "(100 * 3)"
            };
```

Giving:

```text
          Is
Input     Valid Result   Error Message
--------- ----- -------- ------------------------------------------------
1         True  1
32-5      True  -[32, 5]
(100 * 3) False          Unable to interpret calculation at "( 100 * 3 )"
```

Let's try:

```c#
        private static bool TryTakeCalc(TokenKeeper pos, out ExpressionNode calculation)
        {
            if (TryTakeOperatorExp(pos, out calculation)
             || TryTakeLiteral(pos, out calculation)
             || TryTakeParensExp(pos, out calculation))
            {
                return true;
            }

            calculation = null;
            return false;
        }

        private static bool TryTakeToken(TokenType tokenType, TokenKeeper pos)
        {
            if (pos.Next.TokenType == tokenType)
            {
                pos.Take();
                return true;
            }

            return false;
       }

        private static bool TryTakeParensExp(TokenKeeper pos, out ExpressionNode node)
        {
            var work = new TokenKeeper(pos);
            if (TryTakeToken(TokenType.OpenParens, work)
             && TryTakeCalc(work, out var calc)
             && TryTakeToken(TokenType.CloseParens, work))
            {
                pos.Swap(work);
                node = new ParensExpressionNode(calc);
                return true;
            }

            node = null;
            return false;
        }
```

So ```TryTakeCalc``` now tries ```TryTakeOperatorExp``` then ```TryTakeLiteral``` then the new ```TryTakeParensExp```. The interesting thing about ```TryTakeParensExp``` is that it recursively calls ```TryTakeCalc```. This is what is meant by the term "recursive descent".

This gives:

```text
          Is                Error
Input     Valid Result      Message
--------- ----- ----------- -------
1         True  1
32-5      True  -[32, 5]
(100 * 3) True  (*[100, 3])
```

Seems like progress. We can now parse the three basic elements. Let's add a new test to prove we are not done:

```text
              Is
Input         Valid Result      Error Message
------------- ----- ----------- ------------------------------------------------
1             True  1
32-5          True  -[32, 5]
(100 * 3)     True  (*[100, 3])
100 + (3 * 5) False             Unable to interpret calculation at "+ ( 3 * 5 )"
```

Simple enough to support:

```c#
        private static bool TryTakeOperatorExp(TokenKeeper pos, out ExpressionNode node)
        {
            var work = new TokenKeeper(pos);
            if (TryTakeLiteral(work, out var left)
             && TryTakeOperator(work, out var op)
             && TryTakeCalc(work, out var right))
            {
                pos.Swap(work);
                node = new OperatorExpNode(left, op, right);
                return true;
            }

            node = null;
            return false;
        }
```

And sure enough:

```text
              Is                      Error
Input         Valid Result            Message
------------- ----- ----------------- -------
1             True  1
32-5          True  -[32, 5]
(100 * 3)     True  (*[100, 3])
100 + (3 * 5) True  +[100, (*[3, 5])]
```

So let's try the other way around:

```c#
            var testStrings = new [] {
                "1",
                "32-5",
                "(100 * 3)",
                "100 + (3 * 5)",
                "(3 * 5) + 100"
            };
```

Giving:

```text
              Is
Input         Valid Result            Error Message
------------- ----- ----------------- ------------------------------------------
1             True  1
32-5          True  -[32, 5]
(100 * 3)     True  (*[100, 3])
100 + (3 * 5) True  +[100, (*[3, 5])]
(3 * 5) + 100 False                   Unable to interpret calculation at "+ 100"
```

From here on out we need to take some care not to create an infinite loop. For example:

```c#
        private static bool TryTakeOperatorExp(TokenKeeper pos, out ExpressionNode node)
        {
            var work = new TokenKeeper(pos);
            if (TryTakeCalc(work, out var left)
             && TryTakeOperator(work, out var op)
             && TryTakeCalc(work, out var right))
            {
                pos.Swap(work);
                node = new OperatorExpNode(left, op, right);
                return true;
            }

            node = null;
            return false;
        }
```

Looks innocent enough but ther first thing ```TryTakeCalc``` calls is ```TryTakeOperatorExp```, giving us a loop. This is called "left-recursion", and you can't allow it. The issue we are dealing with here is that an operator expression is two terms separated by an operator, but we shouldn't think of the two terms as being equivalent. To understand what I mean consider the difference between these expressions:

```text
1 + 2

1 + 2 + 3
```

We can only have an operator expression consisting of two terms, but there are three in the second example. We can't have a 3 term operator expression, and a 4 term and a 5 term and so on, we need to make sure it's always 2. Therefore we either need to pretend the second example is ```1 + (2 + 3)``` or ```(1 + 2) + 3```. If we assume the first one (```1 + (2 + 3)```) then we can say that only the second term is allowed itself to be an operator expression (or any type of expression), and the first term is only allowed to be either a number or a parenthesised expression. This will ensure we don't get left-recursion. However, right-recursion is allowed and will give us the result we want.

Therefore:

```c#
        private static bool TryTakeOperatorExp(TokenKeeper pos, out ExpressionNode node)
        {
            var work = new TokenKeeper(pos);
            if (TryTakeFirstTerm(work, out var left)
             && TryTakeOperator(work, out var op)
             && TryTakeCalc(work, out var right))
            {
                pos.Swap(work);
                node = new OperatorExpNode(left, op, right);
                return true;
            }

            node = null;
            return false;
        }

        private static bool TryTakeFirstTerm(TokenKeeper pos, out ExpressionNode node)
        {
            if (TryTakeLiteral(pos, out node)
             || TryTakeParensExp(pos, out node))
            {
                return true;
            }

            node = null;
            return false;
        }
```

And:

```text
              Is                      Error
Input         Valid Result            Message
------------- ----- ----------------- -------
1             True  1
32-5          True  -[32, 5]
(100 * 3)     True  (*[100, 3])
100 + (3 * 5) True  +[100, (*[3, 5])]
(3 * 5) + 100 True  +[(*[3, 5]), 100]
```

Let's prove we are getting what we need by adding some more expressions:

```c#
            var testStrings = new [] {
                "1",
                "32-5",
                "(100 * 3)",
                "100 + (3 * 5)",
                "(3 * 5) + 100",
                "1 + 2 + 3",
                "1 + (3 * (5 + 6))"
            };
```

And:

```text

Input             Valid Result                  Message
----------------- ----- ----------------------- -------
1                 True  1
32-5              True  -[32, 5]
(100 * 3)         True  (*[100, 3])
100 + (3 * 5)     True  +[100, (*[3, 5])]
(3 * 5) + 100     True  +[(*[3, 5]), 100]
1 + 2 + 3         True  +[1, +[2, 3]]
1 + (3 * (5 + 6)) True  +[1, (*[3, (+[5, 6])])]
```

I find it amazing how little code you actually need to accomplish this.

We are not done yet, however. We need to see how the parser behaves when it gets invalid input:

```c#
            var testStrings = new [] {
                "1",
                "32-5",
                "(100 * 3)",
                "100 + (3 * 5)",
                "(3 * 5) + 100",
                "1 + 2 + 3",
                "1 + (3 * (5 + 6))",
                "1 +",
                "(1 + 3",
                "(",
                ")",
                "((1 + 2) * 3",
                "(1 + 2) * 3)",
            };
```

And:

```text
                  Is
Input             Valid Result                  Error Message
----------------- ----- ----------------------- -----------------------------------------------
1                 True  1
32-5              True  -[32, 5]
(100 * 3)         True  (*[100, 3])
100 + (3 * 5)     True  +[100, (*[3, 5])]
(3 * 5) + 100     True  +[(*[3, 5]), 100]
1 + 2 + 3         True  +[1, +[2, 3]]
1 + (3 * (5 + 6)) True  +[1, (*[3, (+[5, 6])])]
1 +               False                         Unable to interpret calculation at "+"
(1 + 3            False                         Unable to interpret calculation at "( 1 + 3"
(                 False                         Unable to interpret calculation at "("
)                 False                         Unable to interpret calculation at ")"
((1 + 2) * 3      False                         Unable to interpret calculation at "( ( 1 + 2 )
                                                * 3"
(1 + 2) * 3)      False                         Unable to interpret calculation at ")"
```

Surprisingly, it is behaving acceptably. I went through the BNF based version of this process and errors caused left-recursion all over, but that's not the case here. I ran the BNF parser tests through our test driven parser and while it does not give customised error messages like the BNF one, it produces identical syntax trees and correctly rejects all of the invalid input with no left-recursion.

I can't say how much of this success is due to the fact I'd already been through the BNF based process, and my mental model was superior, but it makes me wonder if I missed an opportunity for simplification in the BNF. To check I'm going to reverse engineer a BNF definition from our test driven parser:

```text
<calc>        ::= <operatorExp> | NumericLiteral | <parensExp>
<parensExp>   ::= OpenParens <calc> CloseParens
<operatorExp> ::= <firstTerm> Operator <calc>
<firstTerm>   ::= NumericLiteral | <parensExp>
```

Compare that with the first version of the BNF:

```text
<calc>               ::= <calcExp> | NumericLiteral
<calcExp>            ::= <operatorExp> | <parensExp>
<operatorExp>        ::= <term> Operator <secondTerm>
<term>               ::= NumericLiteral | <parensExp> | <calcExp>
<secondTerm>         ::= <parensExp> | <operatorExp> | NumericLiteral
<parensExp>          ::= OpenParens <calcExp> CloseParens
```

It looks to me as though the BNF was fine except ```<term>``` still contains ```<calcExp>```

Next, the nodes. These are the elements in our abstract syntax tree. The need to handle errors in the input as if they were parsed means that we need a special type of error node:

They can be constructed from tokens:

```c#
    internal class NumericLiteralNode : ExpressionNode
    {
        private decimal _value;

        public NumericLiteralNode(Token token)
        {
            _value = decimal.Parse(token.Text);
        }

        internal override string Describe()
        {
            return _value.ToString();
        }

        internal override IEnumerable<ExpressionNode> ContainedNodes()
        {
            yield break;
        }
    }```

Or from other nodes or a mix:

```c#
    internal class OperatorExpNode : ExpressionNode
    {
        private ExpressionNode _left;
        private OperatorToken _op;
        private ExpressionNode _right;

        public OperatorExpNode(ExpressionNode left, OperatorToken op, ExpressionNode right)
        {
            _left = left;
            _op = op;
            _right = right;
        }

        internal override string Describe()
        {
            return $"{_op.Text}[{_left.Describe()}, {_right.Describe()}]";
        }

        internal override IEnumerable<ExpressionNode> ContainedNodes()
        {
            yield return _left;
            yield return _right;
        }
    }

    internal class ParensExpressionNode : ExpressionNode
    {
        private ExpressionNode _calc;

        public ParensExpressionNode(ExpressionNode calc)
        {
            _calc = calc;
        }

        internal override string Describe()
        {
            return $"({_calc.Describe()})";
        }

        internal override IEnumerable<ExpressionNode> ContainedNodes()
        {
            yield return _calc;
        }

    }```

And finally a specialist one for errors:

```c#
    internal class ErrorToken : Token
    {
        public override TokenType TokenType => TokenType.Error;

        public string ErrorMessage { get; }

        public ErrorToken(StringKeeper pos, string errorMessage)
        {
            Text = pos.TakeAll();
            ErrorMessage = errorMessage;
        }

        public ErrorToken(string atData, string errorMessage)
        {
            Text = atData;
            ErrorMessage = errorMessage;
        }
    }
```

With a little tweak to the unit test to allow for errors:

```c#
        [Fact]
        public void InputIsConverted()
        {
            //Arrange
            var testStrings = new [] {
                "1",
                "32-5",
                "(3 *5)+  49",
                "1 + 2 + 3",
                "145 * ((63/2 + 5) - 16)",
                "2 * 2 * (2 * 2)",
                "1 + +",
                "(1 + 5",
                "1 2",
                "+ 1",
                "((",
                ")",
                "+",
                "",
                "145 * ((63/2 + 5) - 16))",
                "14.5 * ((6.3/2.4 + 5.5) - 1.6)",
            };

            //Act
            var results = testStrings
                .Select(s => new { String = s, Result = ArithmeticParser.Parse(s)})
                .ToList();

            //Assert
            var output = new Output();
            var report = results.AsReport(rep => rep
                .AddColumn(r => r.String, cc => cc.Heading("Input"))
                .AddColumn(r => r.Result.IsValid, cc => { })
                .AddColumn(r => r.Result.Describe(), cc => cc.Heading("Result"))
                .AddColumn(r => r.Result.ErrorMessage, cc => {})
            );

            output.FormatTable(report);
            output.Report.Verify();
        }
    }
```

We can give the parser a try. Here's the report for approval:

```text
                          Is
Input                     Valid Result                            Error Message
------------------------- ----- --------------------------------- -----------------------------
1                         True  1
32-5                      True  -[32, 5]
(3 *5)+  49               True  +[(*[3, 5]), 49]
1 + 2 + 3                 True  +[1, +[2, 3]]
145 * ((63/2 + 5) - 16)   True  *[145, (-[(/[63, +[2, 5]]), 16])]
2 * 2 * (2 * 2)           True  *[2, *[2, (*[2, 2])]]
1 + +                     True  Error: invalid operator
                                expression at 1 + +
(1 + 5                    True  Error: Unmatched open parenthesis
                                at ( 1 + 5
1 2                       False                                   Unable to interpret
                                                                  calculation at "2"
+ 1                       True  Error: Unexpected operator at + 1
((                        True  Error: Unmatched open parenthesis
                                at ( (
)                         True  Error: Unexpected close parens at
                                )
+                         True  Error: Unexpected operator at +
                          True  Error: Missing expression term at
145 * ((63/2 + 5) - 16))  False                                   Unable to interpret
                                                                  calculation at ")"
14.5 * ((6.3/2.4 + 5.5) - True  *[14.5, (-[(/[6.3, +[2.4, 5.5]]),
1.6)                            1.6])]
```

We also have the error node appearing as an expression so a tweak to the Parse method:

```c#
        public static ArithmeticExpression Parse(string input)
        {
            var tokens = LexicalAnalyser.ExtractTokens(input ?? string.Empty);
            var pos = new TokenKeeper(tokens);
            if (TryTakeCalc(pos, out var calculation) && pos.Finished)
            {
                var error = AllNodes(calculation).FirstOrDefault(n => n is ErrorNode);
                if (error != null)
                    return new ArithmeticExpression(error.Describe());

                return new ArithmeticExpression(calculation);
            }
            else
            {
                var errorMessage = $"Unable to interpret calculation at \"{pos.RemainingData()}\"";
                return new ArithmeticExpression(errorMessage);
            }
        }

        private static IEnumerable<ExpressionNode> AllNodes(ExpressionNode root)
        {
            yield return root;
            foreach (var node in root.ContainedNodes().SelectMany(n => AllNodes(n)))
            {
                yield return node;
            }
        }
```

Giving:

```text
                          Is
Input                     Valid Result                          Error Message
------------------------- ----- ------------------------------- -------------------------------
1                         True  1
32-5                      True  -[32, 5]
(3 *5)+  49               True  +[(*[3, 5]), 49]
1 + 2 + 3                 True  +[1, +[2, 3]]
145 * ((63/2 + 5) - 16)   True  *[145, (-[(/[63, +[2, 5]]),
                                16])]
2 * 2 * (2 * 2)           True  *[2, *[2, (*[2, 2])]]
1 + +                     False                                 Invalid operator expression at
                                                                1 + +
(1 + 5                    False                                 Unmatched open parenthesis at (
                                                                1 + 5
1 2                       False                                 Unable to interpret calculation
                                                                at "2"
+ 1                       False                                 Unexpected operator at + 1
((                        False                                 Unmatched open parenthesis at (
                                                                (
)                         False                                 Unexpected close parens at )
+                         False                                 Unexpected operator at +
                          False                                 Missing expression term at
145 * ((63/2 + 5) - 16))  False                                 Unable to interpret calculation
                                                                at ")"
14.5 * ((6.3/2.4 + 5.5) - True  *[14.5, (-[(/[6.3, +[2.4, 5.
1.6)                            5]]), 1.6])]
```

Thanks for reading.

Jamie
