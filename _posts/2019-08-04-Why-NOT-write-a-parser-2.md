---
layout: post
title: Why NOT write a parser? - Preparation
---

This is a follow up from my [Why NOT write a parser?](https://jamie-davis.github.io/the-open-closed-dev/why-not-write-a-parser/) post. In this post I will take you through creating a dot net core library and unit test library to hold the parser code. Future posts will fill out the projects.

So first let's prepare the projects using the dotnet cli. Run these commands in an empty folder:

1. Create the solution file

```ps
dotnet new sln -n Parser
```

2. Create the library project

```ps
dotnet new classlib -o Parser -n Parser
```

3. Create the test project

```ps
dotnet new xunit -o ParserTests -n ParserTests
```

4. Add the projects to the solution

```ps
dotnet sln Parser.sln add Parser\Parser.csproj
dotnet sln Parser.sln add ParserTests\ParserTests.csproj
```

5. Reference the library from the test project

```ps
dotnet add ParserTests\ParserTests.csproj reference Parser\Parser.csproj
```

6. Add TestConsole and FluentAssertions to the test project

```ps
dotnet add ParserTests\ParserTests.csproj package TestConsole
dotnet add ParserTests\ParserTests.csproj package FluentAssertions
```

Next, let's decide what compare tool TestConsole is going to use when an [approval](https://jamie-davis.github.io/the-open-closed-dev/approval-style-testing/) style test fails.

For this we need to create a configuration file somewhere above the level of our test project. The library will walk up the file tree looking for any folder with a file called ".testconsole" containing a single line like this:

```ps
reporter=winmerge
```

This is my file as I run on windows and want to use [WinMerge](http://winmerge.org/) as my compare tool. The other directly supported tools are [Kompare](https://kde.org/applications/development/org.kde.kompare) (use "reporter=kompare") and [Meld](https://meldmerge.org/) (use "reporter=meld").

If you want to use a different file compare tool you can add your own by implementing ```TestConsoleLib.Testing.IApprovalsReporter``` like this:

```c#
    [ApprovalsReporter("winmerge")]
    public class WinMergeReporter : IApprovalsReporter
    {
        public string FileName => "WinMergeU";
        public string Arguments => "$1 $2";
    }
```

- The ```[ApprovalsReporter("winmerge")]``` attribute provides the name to be used in the .testconsole file.

- The ```FileName``` property provides the name of the executable that should be run.

- The ```Arguments``` property provides a template for the command line arguments. ```$1``` will be replaced with the file path for the new output, ```$2``` will be replaced with the file path of the old (or Approved) output.

You will also need to let TestConsole know about the assembly in which you define your reporters by calling its ```CompareUtil.RegisterReporters``` method, passing in your ```Assembly``` reference.

Finally, let's prove that the unit test frameworks are functional. Find the ```UnitTest1.cs``` file and edit it to look like this:

```c#
using System;
using Xunit;
using TestConsoleLib.Testing;

namespace ParserTests
{
    public class UnitTest1
    {
        [Fact]
        public void Test1()
        {
            "Hi".Verify();
        }
    }
}
```

Next build everything and run the test. If you are not using an IDE, run this at the solution folder:

```ps
dotnet test
```

If everything is configured correctly, your file compare tool should open. For example:

![Winmerge](https://jamie-davis.github.io/the-open-closed-dev/images/parser-2-prep.png)

If you check your souce directory, you will see two files adjacent to the unit test:

```ps
UnitTest1.Test1.approved.txt
UnitTest1.Test1.received.txt
```

These are the files being displayed.

With WinMerge, to approve the results I just click the "All Right" button, highlighted here:

![Winmerge](https://jamie-davis.github.io/the-open-closed-dev/images/parser-2-prep-2.png)

After saving the files are identical:

![Winmerge](https://jamie-davis.github.io/the-open-closed-dev/images/parser-2-prep-3.png)

Once you make the files match, re-run the ```dotnet test``` command, and this time the test should pass, and the ```UnitTest1.Test1.received.txt``` file should be deleted.

If all has gone well, our test suite is ready for real tests and we can remove ```UnitTest1.cs``` and ```UnitTest1.Test1.approved.txt```.

The final step we need is to allow the test library to access ```internal``` items from the Parser library, which allows us to hide our implementation details but still have a separate test project. To do this add a file called ```Assembly.cs``` to the Parser folder, with the following contents:

```c#
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("ParserTests")]
```

The next post will deal with constructing a lexical analyser.

Thanks for reading,

Jamie.
