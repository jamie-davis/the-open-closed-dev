---
layout: post
title: Why NOT write a parser? - Lexical Analyser
---

This is a follow up from my [Why NOT write a parser?](https://jamie-davis.github.io/the-open-closed-dev/why-not-write-a-parser/) post. In this post I will take you through creating a lexical analyser.

Before we start, I'm going to assume that you have followed the instructions in [part 2](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-2/), and that you have the ```Parser``` and ```ParserTests``` projects, and that you got a sample test working.

The first step in parsing is to convert our input into a stream of "tokens" so that the parser has a more constrained input dataset to deal with and does not have to deal with analysing character data. This is partly for performance, but it also makes the parser easier to write and undestand. This conversion is called lexical analysis.

There are many ways to make this happen, and I am going to show you my default method. For demonstration purposes, we are going to make a simple calculator, so that we can analyse and compute strings like this:

```ps
(66 + 3 / 9) + 5
```

Looking at this as a string we have some obvious token types:

- Numbers i.e. 66, 3, 9, and 5
- Operators  i.e. +, and /
- Parentheses i.e. ( and )

This is a small and easy to digest set of tokens. Arguably too simple becuase, other than numbers, it does not require us to look ahead more than one character to identify a token. I imagine that if you find a reason to write a parser and ended up here, you are probably building some sort of domain specific language (DSL). Relatively simple, as in it's not a compiler, but most likely including keywords as well as literals, which will not be identifiable with single character look ahead.

Keep that in mind as we go on, because the method I'm going to demonstrate is probably overkill for the use case in this post, but will almost certainly make more sense for a real DSL.

The main processing of the lexical analyser will be to repeatedly try to extract tokens from the string until the string is empty. If you imagine that the string is a queue of characters, we will go through each of our token types, and peek at the start of the string to see if it's a match. If it is, we will chop the used characters off and create the token. And then we will repeat the process. Sort of like this:

```ps
while (string not empty)
{
    if (try to take a number)
    {
        remove the number from the string
        emit the number token
    }
    else if (try to take an operator)
    {
        remove the operator from the string
        emit the operator token
    }
    else if (try to take a parenthesis)
    {
        remove the parenthesis from the string
        emit the parenthesis token
    }
    else
    {
        this is not sensible input
        emit an error and stop
    }
}
```

The real implementation will have to deal with several issues:

1. Remembering our position in the string.

2. Advancing in the string if we matched something, but not if we didn't.

3. Extracting the matched characters from the string to return.

4. Skipping whitespace. (Usually.)

We could make that work as a single piece of code, but it's not the best way. I prefer to give some other class the job of remembering our position in the string, and looking at characters. Here's the same thing in a more concrete form (although this is still pseudo-code):

```c#
StringKeeper pos = new StringKeeper(string);
Token token;

while (!pos.Finished)
{
    if (TryTakeNumber(pos, out token))
    {
        yield return token;
    }
    else if (TryTakeOperator(pos, out token))
    {
        yield return token;
    }
    else if (TryTakeParens(pos, out token))
    {
        yield return token;
    }
    else
    {
        yield return new ErrorToken();
        break;
    }
}
```

The idea is that if one of the ```Try``` methods extracts a token, ```pos``` will have been updated. If it doesn't, ```pos``` is the same as it was before. The functions will indicate whether they extracted something by returning ```true``` or ```false```. This implies that any backtracking in ```pos``` should be handled by the ```Try``` method.

The pseudo code invented two important things:

- A class for handling the string data called ```StringKeeper```. (This is the name I give it. I don't claim it's the best name, but it *is* kind of keeping our string for us.) Rather than copying and pasting an old version of ```StringKeeper``` I built one from scratch and used it as a test driven development demonstration which you can find [here](https://jamie-davis.github.io/the-open-closed-dev/tdd-demo/).

- A type called ```Token``` which represents a token extracted from the string. Let's not worry about that in detail just yet, but the purpose of ```Token``` is to be a base class for all of our concrete token types. Therefore a ```NumericToken``` is going to be a different type than ```OperatorToken```, ```ParensToken``` or ```ErrorToken``` but they are all going to be derived from ```Token```.

Let's think about the ```Try``` functions for a minute, particularly the ```TryTakeNumber``` function because it is the only one that might actually need to backtrack if the number turns out to be invalid. More pseudo-code:

```c#
Token TryTakeNumber(StringKeeper pos, out Token token)
{
    StringKeeper work = new StringKeeper(pos);
    string number;
    while (!work.Finished && work.DigitNext)
    {
        number += work.TakeTheNextCharacter;
    }

    if (number is empty || (!work.Finished && !IsValidWayToEndANumber(work.Next)))
    {
        //not a number
        return false;
    }

    token = new NumericToken(number);
    pos = work;
    return true;
}
```

You should be able to see the idea:

- Work on a copy of pos so that we are not editing it as we go.

- Keep extracting digits until you run out of them. If there were no digits, or the next character is something that invalidates the number, exit. (And pos is stil the same as it was.)

- If we are happy with the number, create the token and replace pos with work. Return true.

We sort of avoid the backtracking problem by making a copy of the ```pos``` we are given and commiting to the position we reach when extracting digits if we are happy with what we find. This way, even if we fail, we don't mess up ```pos```, as long as we remember to update it, or we will end up in an infinite loop.

Alternatively, you could always work directly with ```pos``` and reset it back if you decide not to move on. I prefer the "commit to it on success" pattern because I'm uncomfortable with the thought that I might abandon a ```Try``` in numerous ways and forget to do the backtracking. To detect that issue in the unit tests, I'd need to guarantee to capture all of the possible exit conditions in all the various ```Try``` methods. I believe that I'm more likely to test for success properly than for failure, and therefore I'm less likely to let bugs through using a "commit" pattern.

Let's write some real code. Start with a unit test:

```c#
    public class TestLexicalAnalyser
    {
        [Fact]
        public void TokensAreExtracted()
        {
            //Arrange
            var testStrings = new [] {
                "1 + 2 + 3"
            };

            var output = new Output();

            //Act
            foreach (var testString in testStrings)
            {
                var result = LexicalAnalyser.ExtractTokens(testString);

                output.WrapLine($@"Analysis of ""{testString ?? "NULL"}"":");
                output.FormatTable(result.AsReport(rep => rep
                                                    .AddColumn(r => r.TokenType, cc => cc.LeftAlign())
                                                    .AddColumn(r => r.Text, cc => {}))
                );

                output.WriteLine();
                output.WriteLine();
            }

            //Assert
            output.Report.Verify();
        }
    }
```

This is an [approval](https://jamie-davis.github.io/the-open-closed-dev/approval-style-testing/) style test using my [TestConsole](https://github.com/jamie-davis/TestConsole/wiki) library. We added this library to the test project in the [preparation article](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-2/).

Just quickly, to familiarise you, the ```Output``` object is a utility from ```TestConsole``` that formats text into a readable report format. At the end of the test the line:

```c#
output.Report.Verify();
```

Compares the output generated in this run to the "approved" version. If it doesn't match (or doesn't exist, which it doesn't yet) it will show the two versions side by side using a text compare tool. We configured this in [preparation article](https://jamie-davis.github.io/the-open-closed-dev/Why-NOT-write-a-parser-2/), so that should be working.

The ```for``` loop iterates over the ```testStrings``` array, and runs each one through the lexical analyser:

```c#
var result = LexicalAnalyser.ExtractTokens(testString);
```

The rest of the content of the ```for``` loop is formatting. This should be the only test we actually need, because it takes the place of quite a few traditional tests. (The main advantage of approval style.) It could be considered cheating, but the output is very readable, and the workflow is very fast. I've not found a downside for this type of testing where it is appropriate, and I don't feel it makes the test suite less effective - just easier to use. 

Now let's create the ```LexicalAnalyser``` class:

```c#
    public static class LexicalAnalyser
    {
        public static IEnumerable<Token> ExtractTokens(string input)
        {
            var pos = new StringKeeper(input);

            while (!pos.Finished)
            {
                Token nextToken = null;
                if (TryTakeOperator(pos, out nextToken)
                || TryTakeNumeric(pos, out nextToken)
                || TryTakeParens(pos, out nextToken))
                {
                    yield return nextToken;
                    continue;
                }

                yield return new ErrorToken(pos, "Invalid token");
                break;
            }
        }
    }
```

I've condensed the pseudo-code a bit here, so that we only have one ```if``` statement. [Short-circuit evaluation](https://en.wikipedia.org/wiki/Short-circuit_evaluation) will stop additional ```Try``` methods from running if one returns ```true```, so this is safe.

Let's implement ```TryTakeNumeric``` in terms of the currently theoretical ```StringKeeper```:

```c#
    private static bool TryTakeNumeric(StringKeeper pos, out Token nextToken)
    {
        var work = new TokenKeeper(pos);
        var number = string.Empty;
        while (!work.Finished && work.NextIn("0123456789."))
        {
            number += work.Take();
            if (!decimal.TryParse(number, out _))
            {
                var errorData = string.Empty;
                while (!work.Finished && !work.WhiteSpaceNext() && !work.NextIn(Terminators))
                {
                    errorData += work.Take();
                }

                if (errorData != string.Empty)
                {
                    //we know that this is a badly formed number, so we should return an error token
                    nextToken = new ErrorToken(number + errorData, "Invalid number");
                    work.Swap(pos);
                    return true;
                }
            }
        }

        if (number == string.Empty)
        {
            //if we didn't extract anything, this isn't a number at all.
            nextToken = null;
            return false;
        }

        nextToken = new NumericToken(number);
        work.Swap(pos);
        return true;
    }

    private static bool TryTakeOperator(StringKeeper pos, out Token nextToken)
    {
        nextToken = null;
        return false;
    }

    private static bool TryTakeParens(StringKeeper pos, out Token nextToken)
    {
        nextToken = null;
        return false;
    }
```

The next step is to build a version of ```StringKeeper```, which you can see being done [here](https://jamie-davis.github.io/the-open-closed-dev/tdd-demo/).

If we now run the ```TokensAreExtracted``` test, you would see this output:

```ps
Analysis of "1 + 2 + 3":
Token Type     Text
-------------- -------
NumericLiteral 1
Error          + 2 + 3

```

So the test string turned out not to be the best. Let's be more thorough about testing ```TryTakeNumeric``` and change the test strings to:

```c#
            var testStrings = new [] {
                "100",
                "100+",
                "100 ",
                "100L",
            };
```

Giving:

```ps
Analysis of "100":                                                                             
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
                                                                                               
                                                                                               
Analysis of "100+":                                                                            
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
Error          +                                                                               
                                                                                               
                                                                                               
Analysis of "100 ":                                                                            
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
Error                                                                                          
                                                                                               
                                                                                               
Analysis of "100L":                                                                            
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
Error          L                                                                               
```

I don't like the result of "100L" which I think should be an invalid number. After fixing that the output looks like this:

```ps
Analysis of "100":                                                                             
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
                                                                                               
                                                                                               
Analysis of "100+":                                                                            
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
Error          +                                                                               
                                                                                               
                                                                                               
Analysis of "100 ":                                                                            
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
                                                                                               
                                                                                               
Analysis of "100L":                                                                            
Token Type   Text                                                                              
------------ ----                                                                              
Error        100L                                                                              
```

After approving that version (my diff tool has a feature to move the changes over to the approved file), I can add tests for the other token types:

```c#
            var testStrings = new [] {
                "100",
                "100+",
                "100 ",
                "100L",
                "+-*/",
                "()"
            };
```

A bit more coding gives:

```ps
Analysis of "100":                                                                             
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
                                                                                               
                                                                                               
Analysis of "100+":                                                                            
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
Operator       +                                                                               
                                                                                               
                                                                                               
Analysis of "100 ":                                                                            
Token Type     Text                                                                            
-------------- ----                                                                            
NumericLiteral 100                                                                             
                                                                                               
                                                                                               
Analysis of "100L":                                                                            
Token Type   Text                                                                              
------------ ----                                                                              
Error        100L                                                                              
                                                                                               
                                                                                               
Analysis of "+-*/":                                                                            
Token Type   Text                                                                              
------------ ----                                                                              
Operator     +                                                                                 
Operator     -                                                                                 
Operator     *                                                                                 
Operator     /                                                                                 
                                                                                               
                                                                                               
Analysis of "()":                                                                              
Token Type   Text                                                                              
------------ ----                                                                              
OpenParens   (                                                                                 
CloseParens  )                                                                                 
```

So hopefully you can see that we are still driving out the code with the unit tests, and the fact that we are using a single test for that doesn't go against the principles of unit testing. The fact that the output is easy to interpret is what makes the technique attractive.

Now we reach the point where we think about cases not covered yet - such as empty input and sequences of tokens:

```c#
            var testStrings = new [] {
                "100",
                "100+",
                "100 ",
                "100L",
                "+-*/",
                "()",
                "",
                null,
                "(3 *5)+  49"
            };
```

This adds the following:

```ps
Analysis of "":                                                                                
Token Type   Text                                                                              
------------ ----                                                                              
                                                                                               
                                                                                               
Analysis of "NULL":                                                                            
Token Type   Text                                                                              
------------ ----                                                                              
                                                                                               
                                                                                               
Analysis of "(3 *5)+  49":                                                                     
Token Type     Text                                                                            
-------------- ----                                                                            
OpenParens     (                                                                               
NumericLiteral 3                                                                               
Operator       *                                                                               
NumericLiteral 5                                                                               
CloseParens    )                                                                               
Operator       +                                                                               
NumericLiteral 49                                                                              
```

Here's the full code:

```c#
    public static class LexicalAnalyser
    {
        private const string Operators = @"+-*/";

        private const string Terminators = "()" + Operators;
        private static Token OpenParens = new OpenParensToken();
        private static Token CloseParens = new CloseParensToken();

        public static IEnumerable<Token> ExtractTokens(string input)
        {
            var pos = new StringKeeper(input);
            pos.SkipWhiteSpace();

            while (!pos.Finished)
            {
                Token nextToken = null;
                if (TryTakeOperator(pos, out nextToken)
                || TryTakeNumeric(pos, out nextToken)
                || TryTakeParens(pos, out nextToken))
                {
                    yield return nextToken;
                    pos.SkipWhiteSpace();
                    continue;
                }

                yield return new ErrorToken(pos, "Invalid token");
                break;
            }
        }

        private static bool TryTakeNumeric(StringKeeper pos, out Token nextToken)
        {
            var work = new StringKeeper(pos);
            var number = string.Empty;
            while (!work.Finished && work.NextIn("0123456789."))
            {
                number += work.Take();
            }

            if (number == string.Empty)
            {
                nextToken = null;
                return false;
            }

            var errorData = string.Empty;
            while (!work.Finished && !work.WhiteSpaceNext() && !work.NextIn(Terminators))
            {
                errorData += work.Take();
            }

            if (!decimal.TryParse(number, out _) || errorData != string.Empty)
            {
                //we know that this is a badly formed number, so we should return an error token
                nextToken = new ErrorToken(number + errorData, "Invalid number");
                work.Swap(pos);
                return true;
            }

            if (number == string.Empty)
            {
                //if we didn't extract anything, this isn't a number at all.
                nextToken = null;
                return false;
            }

            nextToken = new NumericToken(number);
            work.Swap(pos);
            return true;
        }

        private static bool TryTakeOperator(StringKeeper pos, out Token nextToken)
        {
            if (pos.NextIn(Operators))
            {
                var op = pos.Take();
                nextToken = new OperatorToken(op);
                return true;
            }

            nextToken = null;
            return false;
        }

        private static bool TryTakeParens(StringKeeper pos, out Token nextToken)
        {
            if (pos.Next == '(')
            {
                var op = pos.Take();
                nextToken = OpenParens;
                return true;
            }
            else if (pos.Next == ')')
            {
                var op = pos.Take();
                nextToken = CloseParens;
                return true;
            }

            nextToken = null;
            return false;
        }
    }
```

Have a look at ```TryTakeOperator``` and ```TryTakeParens```. They are simple than ```TryTakeNumeric``` because they only need to consider the next character. You may also notice a few other changes to handle invalid numbers properly, and skip white space at the right time. Those changes were all driven out by incorrect test output, even though we only have one actual test method (or, you could argue that we actually have 9 test cases, or if you really think about it, it would have needed even more without approval style testing).

I've not shown the token classes in the post, but you can see them in the [github repo](https://github.com/jamie-davis/ParserDemo) for the demo project. Chances are that it will have moved on since I wrote this post, but you should be able to find everything.

The lexical analyser seems to work well enough for now. The next post will deal with writing the parser.

Thanks for reading,

Jamie.
