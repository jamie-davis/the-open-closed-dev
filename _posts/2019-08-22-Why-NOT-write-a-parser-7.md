---
layout: post
title: Why NOT write a parser? - Acting on the parser output
---

This is a follow up from my [Why NOT write a parser?](https://jamie-davis.github.io/the-open-closed-dev/why-not-write-a-parser/) post. In this post I will take you through deriving actions from the abstract syntax tree.

Once you have successfully parsed something, you need to be able to do something with the parser results. Generally speaking, you will end up with an "abstract syntax tree" (AST), which is just a different way to represent the original input and not in itself the end goal. In our case, we have the AST for a simple arithmetical expression, and what we really want is the answer. We came up with a simple text representation of the parser output for our unit tests in which:

```text
5 + 3 * 6
```

Becomes:

```text
+[5, *[3, 6]]
```

We need to be able to compute the answer - 23. Simple enough, it's 5 + the result of 3 * 6, and sufficient detail is present in the tree nodes.

Let's prepare a new unit test. It will be along the same lines as the parsing test:

```c#
        [Fact]
        public void ResultIsComputed()
        {
            //Arrange
            var testStrings = new [] {
                "5 + 3 * 6",
            };

            //Act
            var results = testStrings
                .Select(s => new { String = s, Result = ArithmeticParser.Parse(s).Compute()})
                .ToList();

            //Assert
            var output = new Output();
            output.FormatTable(results);
            output.Report.Verify();
        }
```

To make that compile, we need to add a ```Compute``` method to ```ArithmeticExpression```:

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

        public decimal Compute()
        {
            throw new NotImplementedException();
        }
    }
```

```_calculation``` is the root of the AST, and I suggest we make each tree node responsible for computing its result:

```c#
    internal abstract class ExpressionNode
    {
        internal abstract string Describe();

        internal abstract IEnumerable<ExpressionNode> ContainedNodes();

        internal abstract decimal Compute();
    }
```

An argument can be made that this is giving the nodes multiple responsibilities, which is not good according to the single responsibility principle, and I agree. However, let's press on with it and see how it turns out.

The easiest computation is the numeric literal, so let's implement that first:

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

        internal override decimal Compute()
        {
            return _value;
        }
    }
```

Next let's do ```OperatorExpNode```:

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

        internal override decimal Compute()
        {
            var left = _left.Compute();
            var right = _right.Compute();
            switch (_op.Text)
            {
                case "+":
                    return left + right;

                case "-":
                    return left - right;

                case "*":
                    return left * right;

                case "/":
                    return left / right;

                default:
                    return 0M;
            }
        }
    }
```

And finally:

```c#
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

        internal override decimal Compute()
        {
            return _calc.Compute();
        }
    }
```

```ArithmeticExpression``` just needs to call ```Compute()``` on the root node:

```c#
        public decimal Compute()
        {
            return _calculation.Compute();
        }
```

The test reports:

```text
String                               Result
--------- ---------------------------------
5 + 3 * 6                             23.00
```

Looks good. Until...

```text
String                               Result
--------- ---------------------------------
5 + 3 * 6                             23.00
5 * 3 + 6                             45.00
```

Clearly operator precedence is not working. That's no surprise, we've not done anything to accomodate it. The reason for the result is simple enough - we are effectively getting ```5 * (3 + 6)``` from the parser due to the way the process works (i.e. we get ```*[5, +[3, 6]]```). We need to re-arrange the nodes to correct the associativity of the operators.

One way to do this is to do the work in the ```OperatorExpression``` node itself:

```c#
        private static Dictionary<string, int> OperatorPrecedence = new Dictionary<string, int>
        {
            {"+", 1},
            {"-", 1},
            {"*", 0},
            {"/", 0},
        };
...
        public OperatorExpNode(ExpressionNode left, OperatorToken op, ExpressionNode right)
        {
            _left = left;
            _op = op;
            _right = right;

            if (_right is OperatorExpNode rightOpExp)
            {
                var rightPrecedence = OperatorPrecedence[rightOpExp._op.Text];
                var leftPrecedence = OperatorPrecedence[_op.Text];
                if (rightPrecedence > leftPrecedence)
                {
                    var myLeft = left;
                    var yourLeft = rightOpExp._left;
                    var yourRight = rightOpExp._right;

                    _right = yourRight;
                    _left = rightOpExp;

                    _op = rightOpExp._op;
                    rightOpExp._op = op;

                    rightOpExp._left = myLeft;
                    rightOpExp._right = yourLeft;
                }
            }
        }
```

In that code I've added a dictionary to provide the ordering (```OperatorPrecedence```) and I've altered the constructor of the node to re-arrange the expression. This certainly works, but it feels like an odd place to find the functionality, and it's not good to hide things as hard to understand as this. I feel like it's become clear that actually computing the result is going to be more useful outside of the parsing classes.

That leaves us with the need to convert the AST into something that can perform the calculation. Let's try it this way:

```c#
    internal static class Computer
    {
        private static Dictionary<string, int> OperatorPrecedence = new Dictionary<string, int>
        {
            {"+", 1},
            {"-", 1},
            {"*", 0},
            {"/", 0},
        };

        #region ComputationNodes

        abstract class ComputationNode
        {
            internal abstract decimal Compute();
        }

        sealed class BinaryOperation : ComputationNode
        {
            private readonly OperatorToken _op;
            private readonly ComputationNode _left;
            private readonly ComputationNode _right;

            public BinaryOperation(OperatorToken op, ComputationNode left, ComputationNode right)
            {
                _op = op;
                _left = left;
                _right = right;
            }

            internal override decimal Compute()
            {
                var left = _left.Compute();
                var right = _right.Compute();
                switch (_op.Text)
                {
                    case "+":
                        return left + right;

                    case "-":
                        return left - right;

                    case "*":
                        return left * right;

                    case "/":
                        return left / right;

                    default:
                        return 0M;
                }
            }
        }

        sealed class Literal : ComputationNode
        {
            private readonly decimal _value;

            public Literal(decimal value)
            {
                _value = value;
            }

            internal override decimal Compute()
            {
                return _value;
            }
        }

        #endregion

        internal static decimal Compute(ArithmeticExpression input)
        {
            var converted = Convert(input.Calculation);
            return converted?.Compute() ?? 0M;
        }

        private static ComputationNode Convert(ExpressionNode node)
        {
            switch (node)
            {
                case OperatorExpNode opNode:
                    return ConvertOperator(opNode);

                case ParensExpressionNode parensNode:
                    return ConvertParensNode(parensNode);

                case NumericLiteralNode numberNode:
                    return new Literal(numberNode.Value);

                default:
                    return null;
            }
        }

        private static ComputationNode ConvertParensNode(ParensExpressionNode parensNode)
        {
            return Convert(parensNode.Calculation);
        }

        private static ComputationNode ConvertOperator(OperatorExpNode opNode)
        {
            if (opNode.Right is OperatorExpNode rightOpExp)
            {
                var rightPrecedence = OperatorPrecedence[rightOpExp.Op.Text];
                var leftPrecedence = OperatorPrecedence[opNode.Op.Text];
                if (rightPrecedence > leftPrecedence)
                {
                    var myLeft = opNode.Left;
                    var yourLeft = rightOpExp.Left;
                    var yourRight = rightOpExp.Right;

                    var newRight = new OperatorExpNode(myLeft, opNode.Op, yourLeft);
                    var newOpExp = new OperatorExpNode(yourRight, rightOpExp.Op, newRight);
                    var desc = newOpExp.Describe();
                    return Convert(newOpExp);
                }
            }

            return new BinaryOperation(opNode.Op, Convert(opNode.Left), Convert(opNode.Right));
        }
    }
```

There's a lot going on in there. Let's break it down a bit.

The issue is that precedence is being ignored by the parser and we need to re-arrange the AST to correct that. So, for example:

```text
5 * 3 + 6 = *[5, +[3, 6]]
```

Should be:

```text
5 * 3 + 6 = +[*[5, 3], 6]
```

We have:

1. Swapped the outer operator and inner operator.
2. Retained the same arguments (5, 3, 6).
3. Fixed the grouping such that the inner operator takes the first two parameters instead of the last two.

This operation is what's going on in ```ConvertOperator``` when the precedence of two ```OperatorExpNode``` instances is backwards.

In addition to that we have created a new tree we can "execute" from two the two node types ```BinaryOperation``` and ```Literal```. We don't need a node type for a parenthesised expression in the "execution tree" because parenthesis was just a way to order calculations and as long as we have preserved the order, we don't need to represent the parenthesis directly.

What "execute" means in the context of our problem is to compute the value of the original expression. Your parser will probably have a different purpose - hopefully less involved - but it will still depend on the contents of the AST. For most of the parsers I've built, the interpretation of the tree is very straightforward, and rightly or wrongly, it's often built right into the AST tree nodes. This example was more complex, because I felt a trivial demonstration would have been easily dismissed, but hopefully this series of posts has demonstrated how to put together something reasonably challenging.

Thanks for reading.

Jamie
