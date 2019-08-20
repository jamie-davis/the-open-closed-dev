---
layout: post
title: Why NOT write a parser? - Parser BNF to Code
---

This is a follow up from my [Why NOT write a parser?](https://jamie-davis.github.io/the-open-closed-dev/why-not-write-a-parser/) post. In this post I will take you through creating the parser.

I am going to assume you are familiar with the preceding [post](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-3/) about creating a lexical analyser. The code in this post will be relying on the ```LexicalAnalyser``` type and the tokens it produces. You might also be interested in the [grammar post](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-3/) which details how I designed the parser using BNF.

Using BNF is strictly optional, and only useful if your grammar allows for an arbitrary level of complexity. The parser we are going to write will make sense of mathematical expressions like these:

```ps
1
32 - 5
(5 * 3) + 45
145 * ((63/2 + 5) - 16)
```

There is essentially no limit to how complex an expression can become, and it was easier to make sense of it using BNF than attempting to do without in this case. It's also worth pointing out that the main difficulties are caused by handling invalid input, which could take us into an infinite recursion loop if we use a naive grammar.

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
    public class TestParser
    {
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
                .AddColumn(r => r.Result.Describe(), cc => cc.Heading("Result"))
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
Input       Result
----------- --------------------
3 *5        *[3,5]
5 + 6 + 7   +[5,+[6,7]]
(3 *5)+  49 +[(*[3,5]),49]
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

The first question I have is what should we return? The description string we want to approve will be useless outside of the unit tests, so that has to be a side issue.

My first thought is this:

```c#
    public ArithmeticExpression
    {
        public string Describe() //the side issue
        {
            ...
        }

        public bool IsValid {get;}

        public string ErrorMessage {get;}
    }
```

It's probably good enough to be going on with. I'm not going to think about how the result of the calculation would be derived. That's not the parser's job so we shouldn't consider it now.

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

The question of what ```calculation``` will contain is more interesting. This is the real result of parsing and we know it should be the root of an abstract syntax tree, so with that assumption we should invent a base class for the tree nodes:

```c#
namespace Parser.Parsing
{
    internal abstract class ExpressionNode
    {
        internal abstract string Describe();

        internal abstract IEnumerable<ExpressionNode> ContainedNodes();
    }
```

We will derive concrete nodes from this class and implement ```Describe()``` to produce the strings we will show in the approval report.

We can now define ``TryTakeCalc```:

```c#
        private static bool TryTakeCalc(TokenKeeper pos, out ExpressionNode calculation)
        {

        }
```

If you think back to the [```LexicalAnalyser```](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-3/), I said I like to use a class to represent my position in the input, and I think we have the same issue with the token stream. For that reason I like to create a class called ```TokenKeeper``` which is the same in principle as ```StringKeeper```. I'm not going to distract us by building ```TokenKeeper``` as part of this post, but the [implementation](https://github.com/jamie-davis/ParserDemo/blob/master/Parser/Parsing/TokenKeeper.cs) can be found in the git repo.

This is the BNF from the [BNF post](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-4/):

```text
<calc>               ::= <calcExp> | NumericLiteral
<calcExp>            ::= <operatorExp> | <parensExp>
<operatorExp>        ::= <term> Operator <secondTerm>
<term>               ::= NumericLiteral | <parensExp>
<secondTerm>         ::= <parensExp> | <operatorExp> | NumericLiteral
<parensExp>          ::= OpenParens <calcExp> CloseParens
```

Let's focus on the ```<calc>``` production to begin with:

```text
<calc>               ::= <calcExp> | NumericLiteral
```

This says a ```<calc>``` is either a ```<calcExp>``` or a ```NumericLiteral```. We can realise the production in code like this:

```c#
        private static bool TryTakeCalc(TokenKeeper pos, out ExpressionNode node)
        {
            if (TryTakeCalcExp(pos, out node)
                 || TryTakeNumericLiteral(pos, out node))
            {
                return true;
            }

            node = null;
            return false;
        }
```

Everything we have in a production can be implemented as a ```TryTake<production>``` method. Where we have a non-terminal, we can implement it like this:

```c#
        private static bool TryTakeNumericLiteral(TokenKeeper pos, out ExpressionNode node)
        {
            if (pos.IsNext(TokenType.NumericLiteral))
            {
                var token = pos.Take();
                node = new NumericLiteralNode(token);
                return true;
            }

            return false;
        }
```

Since this is a simple "it's a numeric token or it isn't" decision, we can just take it if it's there. However, if we are implementing a production we have to deal with the idea that we might have to backtrack, so implementing ```TryTakeCalcExp```:

```text
<calcExp>     ::= <operatorExp> | <parensExp>
```

Should be handled like this:

```c#
        private static bool TryTakeCalcExp(TokenKeeper pos, out ExpressionNode node)
        {
            var work = new TokenKeeper(pos);
            if (TryTakeOperatorExp(work, out node)
             || TryTakeParensExp(work, out node))
            {
                pos.Swap(work);
                return node;
            }

            node = null;
            return false;
        }
```

Here we are working with a copy of ```pos``` called ```work``` and using ```pos.Swap(work)``` when we have a successful match. You can hopefully see how mechanical this becomes if you map out your grammar.

We have not discussed the nodes as yet, but we'll get to them shortly. Let's code out the rest of the productions in our grammar first:

```c#
        private static bool TryTakeCalc(TokenKeeper pos, out ExpressionNode node)
        {
            if (TryTakeCalcExp(pos, out node)
                || TryTakeNumericLiteral(pos, out node))
            {
                return true;
            }

            node = null;
            return false;
        }

        private static bool TryTakeCalcExp(TokenKeeper pos, out ExpressionNode node)
        {
            if (TryTakeOperatorExp(pos, out node)
                || TryTakeParensExp(pos, out node))
            {
                return true;
            }

            node = null;
            return false;
        }

        private static bool TryTakeOperatorExp(TokenKeeper pos, out ExpressionNode node)
        {
            var work = new TokenKeeper(pos);
            if (TryTakeTerm(work, out var left)
             && TryTakeOperator(work, out var op)
             && TryTakeSecondTerm(work, out var right))
            {
                node = new OperatorExpNode(left, op, right);
                pos.Swap(work);
                return true;
            }

            node = null;
            return false;
        }

        private static bool TryTakeTerm(TokenKeeper pos, out ExpressionNode node)
        {
            if (TryTakeNumericLiteral(pos, out node)
             || TryTakeParensExp(pos, out node))
            {
                return true;
            }

            node = null;
            return false;
        }

        private static bool TryTakeSecondTerm(TokenKeeper pos, out ExpressionNode node)
        {
            if (TryTakeParensExp(pos, out node)
             || TryTakeOperatorExp(pos, out node)
             || TryTakeNumericLiteral(pos, out node))
            {
                return true;
            }

            node = null;
            return false;
        }

        private static bool TryTakeParensExp(TokenKeeper pos, out ExpressionNode node)
        {
            var work = new TokenKeeper(pos);
            if (TryTakeOpenParens(work)
             && TryTakeCalcExp(work, out var calc)
             && TryTakeCloseParens(work))
            {
                node = new ParensExpressionNode(calc);
                pos.Swap(work);
                return true;
            }

            node = null;
            return false;
        }

        private static bool TryTakeNumericLiteral(TokenKeeper pos, out ExpressionNode node)
        {
            if (pos.Next.TokenType == TokenType.NumericLiteral)
            {
                node = new NumericLiteralNode(pos.Take());
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

        private static bool TryTakeOpenParens(TokenKeeper pos)
        {
            if (pos.Next.TokenType == TokenType.OpenParens)
            {
                pos.Take();
                return true;
            }

            return false;
        }

        private static bool TryTakeCloseParens(TokenKeeper pos)
        {
            if (pos.Next.TokenType == TokenType.CloseParens)
            {
                pos.Take();
                return true;
            }

            return false;
        }
```

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
Input                    Valid Result                         Error Message
------------------------ ----- ------------------------------ ---------------------------------
1                        True  1
32-5                     True  -[32, 5]
(3 *5)+  49              True  +[(*[3, 5]), 49]
1 + 2 + 3                True  +[1, +[2, 3]]
145 * ((63/2 + 5) - 16)  True  *[145, (-[(/[63, +[2, 5]]),
                               16])]
2 * 2 * (2 * 2)          True  *[2, *[2, (*[2, 2])]]
1 + +                    False                                Unable to interpret calculation  
                                                              at "+ +"
(1 + 5                   False                                Unable to interpret calculation  
                                                              at "( 1 + 5"
1 2                      False                                Unable to interpret calculation  
                                                              at "2"
+ 1                      False                                Unable to interpret calculation  
                                                              at "+ 1"
((                       False                                Unable to interpret calculation  
                                                              at "( ("
)                        False                                Unable to interpret calculation  
                                                              at ")"
+                        False                                Unable to interpret calculation  
                                                              at "+"
                         False                                Unable to interpret calculation  
                                                              at ""
(                        False                                Unable to interpret calculation  
                                                              at "("
145 * ((63/2 + 5) - 16)) False                                Unable to interpret calculation  
                                                              at ")"
14.5 * ((6.3/2.4 + 5.5)  True  *[14.5, (-[(/[6.3, +[2.4, 5.
- 1.6)                         5]]), 1.6])]
```

The follow up post will discuss taking actions based on the abstract syntax tree.

Thanks for reading.

Jamie
