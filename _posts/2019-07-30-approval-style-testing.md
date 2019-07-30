---
layout: post
title: Approval style testing
---

Some years ago I came across a library called [ApprovalTests.NET](https://approvaltests.com/), and it has become a vital tool in all of my .NET test libraries.

For many tests, simple assertions are fine. For example<sup id="a1">[1](#f1)</sup>:

```c#
    [Test]
    public void WordBreaksAreCounted()
    {
        // Arrange
        var colFormat = new ColumnFormat("heading", typeof (string));
        const string value = "One two three four five six seven eight nine ten eleven.";

        //Act
        var addedBreaks = ColumnWrapper.CountWordwrapLineBreaks(value, colFormat, 20);

        //Assert
        Assert.That(addedBreaks, Is.EqualTo(2));
    }
```

Here there is just one assertion, and just one reason to fail. There are some types of test, where a significant amount of data is returned - perhaps in a collection for example. Consider this fragment:

```c#
    [Test]
    public void RightAlignedLinesAreExpandedToCorrectLength()
    {
        var c = new ColumnFormat("h", typeof (string), ColumnAlign.Right);
        c.SetActualWidth(20);
        var value = "Line" + Environment.NewLine + "Data" + Environment.NewLine + "More";
        var wrapped = ColumnWrapper.WrapValue(value, c, 20).Select(l => "-->" + l + "<--");
```

The content of ```wrapped``` at the end of the fragment is:

```c#
-->                Line<--
-->                Data<--
-->                More<--
```

We need a way to assert that this was indeed the output. This would work:

```c#
        Assert.That(wrapped, Is.EqualTo(new [] {"-->                Line<--","-->                Data<--","-->                More<--"}));
```

It's pretty ugly to read and especially maintain, but maybe okay for three lines. What if the output needed to be much longer?

Another scenario I'd like you to think about is where a collection of objects is returned, and we need to assert that the expected objects are present and have the correct values:

```c#
        [Test]
        public void DefaultRenderColumnIsGenerated()
        {
            var cols = typeof (TestType).GetProperties()
                .Select(p => p.Name == "StringCol" ? new ColumnFormat("My String", p.PropertyType) : null);
            var propFormats = FormatAnalyser.Analyse(typeof(TestTypeWithRenderable), cols, true);
```

```propFormats``` is a collection of these:

```c#
    internal class PropertyColumnFormat
    {
        public PropertyInfo Property { get; set; }
        public ColumnFormat Format { get; set; }

        internal PropertyColumnFormat(PropertyInfo property, ColumnFormat format)
        {
            Property = property;
            Format = format;
        }
    }
```

Also, a ```ColumnFormat``` has these properties:

```c#
    public string Heading { get; internal set; }
    public Type Type { get; private set; }
    public ColumnAlign Alignment { get; internal set; }
    public int DecimalPlaces { get; internal set; }
    public int ActualWidth { get; private set; }
    public string FormatTemplate { get; private set; }
    public string Width { get; private set; }
    public int FixedWidth { get; set; }
    public int MinWidth { get; set; }
    public int MaxWidth { get; set; }
    public double ProportionalWidth { get; set; }
```

The point of the test is to prove that an appropriate ```ColumnFormat``` is generated for each column in the data. Very tricky to assert in a sensible way. However, the actual test method looks like this:

```c#
    [Test]
    public void DefaultRenderColumnIsGenerated()
    {
        var cols = typeof (TestType).GetProperties()
            .Select(p => p.Name == "StringCol" ? new ColumnFormat("My String", p.PropertyType) : null);
        var propFormats = FormatAnalyser.Analyse(typeof(TestTypeWithRenderable), cols, true);
        Approvals.Verify(ReportFormats(propFormats));
    }
```

The key element is obviously the ```Approvals.Verify``` call. This is one of 8 similar tests in the test [fixture](https://github.com/jamie-davis/TestConsole/blob/master/src/TestConsole.Tests/OutputFormatting/Internal/TestFormatAnalyser.cs), and they all take a set of ```propFormats```, pass them into a formatting function, and "approve" the result.

This is what gets approved:

```
IntCol      = ColumnFormat("Int Col", System.Int32, Right, 2DP, Actual 0, , Requested )
StringCol   = ColumnFormat("My String", System.String, Left, 2DP, Actual 0, , Requested )
RenderCol   = ColumnFormat("Render Col", TestConsole.OutputFormatting.IConsoleRenderer, Left, 2DP, Actual 0, , Requested )
```

I won't bother explaining what the output means, but if this test should fail on my development machine, I would see this pop up:

![Winmerge](https://jamie-davis.github.io/the-open-closed-dev/images/winmerge-test-output-screenshot.png)

Here, the approvals process has compared the text produced by the unit test to a checked-in "approved" text file. Since it doesn't match in this case, it's taken the defined action to report the failure - which in my case is to run Winmerge on the two versions. On a build server, it wouldn't try to run a file compare tool, it would just fail the test.

This type of test allows us to assert that a large amount of data matches the expected result. From a code point of view, there is still only one "assert", but you might say this is stretching the definition of a single assert, and I would agree. However, the output is stunningly easy to interpret (at least, you should make sure it is), and most compare utilities will allow you to "move" the differences over, so you can approve the new version very quickly<sup id="a2">[2](#f2)</sup>.

The big question is, should everything be an approval test? The answer is definitely "no" - where it makes sense it is absolutely the best way, but more traditional assertions (I Like to use [FluentAssertions](https://fluentassertions.com/)) are faster and simpler. However I'd also encourage you to avoid putting more than one assertion in a unit test. Read up on the Single Assertion Principle - it makes your tests more informative and easier to understand, and it makes failures easier to diagnose.

For some sorts of problem, however, approval style testing can save a large number of tests, save a lot of programming time, and make tests much easier to understand. Going back to the first column wrapping example, this is the actual approved output:

```
RightAlignedLinesAreExpandedToCorrectLength

Original:
Line
Data
More

Wrapped:
   12345678901234567890
   ---------+---------+
-->                Line<--
-->                Data<--
-->                More<--
```

You can see the intention of the test, the input, and the result, making it very easy to confirm that the code has done what it is supposed to do. We've baked an easy to evaluate expected result for a complex problem into the test suite with very little effort.

Use approval testing. Not for everything, but where it helps.

Thanks for reading.

Jamie.

<b id="f1">1</b> This and the other test examples are based on tests in my [TestConsole](https://github.com/jamie-davis/TestConsole) library.[↩](#a1)

<b id="f2">2</b> I should really come clean and point out that I don't actually use ApprovalTests any more - not because there were any problems with it, just that I needed to build a test suite that could run .NET core code and ApprovalTests was not compatible at the time (I believe it is now, but they didn't make it in time for me). However, in a way it's a compliment to ApprovalTest that the concept is so good that I would build myself a netstandard alternative rather than live without it.

The full story is that I had a library for formatting text (almost) purely for the purpose of approving it with ApprovalTests. This library is called TestConsole and it's published to [nuget](https://www.nuget.org/packages/TestConsole). I've been using it with ApprovalTests for a long time. It was born out of my [ConsoleToolkit](https://www.nuget.org/packages/ConsoleToolkit) library, which had code to format text in tabular form. I extracted and adjusted the formatting code in order to make the first version of TestConsole. In time I needed to convert TestConsole to be a .NET Standard library, and there was no way to run the unit tests in a netcoreapp project unless I patched in the minimal Approval style features needed for its test suite. It turned out to be very straightforward to do that, so I have kept the features, and I've switched my other code away from ApprovalTests as a result.

As a kludge to avoid rewiring a large number of ApprovalTests dependencies, I made the syntax for verify exactly the same as in ApprovalTests. However, TestConsole exposes ```.Verify()``` as an extension method, and this is the intended assertion method. Have a look at TestConsole, as it makes this type of testing very easy.
[↩](#a2)
