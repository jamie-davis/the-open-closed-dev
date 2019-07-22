---
layout: post
title: An Introduction
---

As a developer, the two things I care about the most are:

1. Automated unit Tests
2. the SOLID principles.

I hope I don't need to convince anyone about the merits of test driven development. For me this is the only way to code, and if you decide not to adopt the practice, you should make sure you understand what it is you are giving up, and that your code is going to suffer as a result... unless it's genuinely trivial. Is your code trivial? I can't remember a time when I thought mine was.

TDD gives you a way to produce working code faster and keep it working always, whereas the SOLID principles help you produce code that's easy to understand and work with (i.e. maintenance friendly). These are two aspects of quality that are incredibly cheap to adopt but can have a profoundly positive impact on the software. Skimping on either comes with a cost, which is usually manifests as software that's hard to change and prone to failure. Make the code easy to work on, so that it doesn't feel risky to change, and the spaghetti will stay away. (Spaghetti as a food, I approve of; spaghetti _code_ will strangle your system, and possibly your developers.)

These are the SOLID principles (italics from wikipedia):

***

1. Single responsibility Principle: _A class should only have a single responsibility, that is, only changes to one part of the software's specification should be able to affect the specification of the class._ **My take** - generally, if I feel I'm writing an "indirect" test, it's a responsibility and should be in a class of its own. An example that springs to mind is when I was writing a line breaking algorithm and I felt I was writing some tests that just showed whether word splitting was working. I refactored the word splitting into a new class and the code for both responsibilities became dramatically simpler.

2. Open-closed Principle: _Software entities should be open for extension, but closed for modification._ **More on this later.**

3. Liskov substitution principle: _Objects in a program should be replaceable with instances of their subtypes without altering the correctness of that program._ **My take:** Inheritance is the tightest coupling between classes and can cause subtle problems, especially as a system grows. Understand the implications and only use inheritence if all reasonable alternatives are untenable, ideally limit its scope so that you have control over subclasses.

4. Interface segregation principle: _Many client-specific interfaces are better than one general-purpose interface._ **My take:** This is pretty simple to understand, and makes a lot of sense when you concentrate on the "client" code. Giant, multi-purpose interfaces are confusing and allow mistakes where the client erroneously relies on a method it was not intended to call. Splitting interfaces by use case makes you focus on how the functionality should be consumed, which always improves the design.

5. Dependency inversion principle: _One should depend upon abstractions, not concretions._ **My take:** This should not need any explanation. Dependency injection is an example of this principle in action. (But that's not the only way to invert control.)

***

I'd like to return to the open-closed principle. This and Liskov's seem to be the least well understood principles, in my opinion. Liskov's because it's a problem that takes time to manifest, and when it does it can have a large impact. The open-closed principle, on the other hand, sounds confusing. How can a software entity be both closed for modification and also open for extension?

The answer to that question is really (and this is my opinion) that the software is in two parts - one closed, and one open. The closed part is the stuff you do not expect to change as the system grows, wheras the open part is the way in which you add new functionality. Scope is also important, as the principle will probably be applied to solve a specific problem.

Let's imagine that we have a requirement to produce reports, and we expect that the set of reports will expand over time. Let's also imagine that the reports share technical requirements such as access to data sources, and perhaps there is some consistency in the formatting of the reports, such as page headers and footers and perhaps the page breaking algorithm.

This could be an opportunity to put the open-closed principle into practice. We could classify the reports themselves as the "open" part - in that we extend the system by adding new reports. We could then classify the functionality that enables the reports as the closed part. For example (_this is just made up off the cuff_):

**Closed**

`ReportManager` - Orchestrates the reporting process

`ReportFormatter/IReportFormatter` - Provides generic formatting facilities, including pagination and common header and footers.

`ReportData/IReportData`- Provides an interface to retrieve data to drive the reports.

`ColumnDefinition` - You can imagine that the generic formatting facility is going to need some sort of config to tell it how to put the report together.


**Open**

`SalesReport`

`ExpensesReport`

...and so on

Our main goal would be to reduce the amount of code needed for the reports themselves, and accept a slightly higher difficulty tarrif for the closed part in order to achieve that. The payoff is that over time we will produce a large number of reports, and they will be vastly cheaper and faster to produce as a result. Over the long term we save a lot of effort and bugs, for a relatively small up-front cost. Can you see how we are extending the functionality without the risk of introducing bugs? If we need to add the `QuarterlyPerformanceReport`, we won't be touching any of the other reports or the formatting features or the data access features, so we can reasonably assume that testing the new report will adequately validate our work.

Clearly, the "closed" part is going to need other types of extension, such a new formatting features. You might even visualise this as another opportunity to apply the principle so that we can add "formatters" without having to mess with working code. This is where having an open-closed mindset helps you. You could imagine adding a new report is just implementing `IReport`, and adding a new type of formatter is implementing `IFormatter`.

The decision whether to do this or not may not be clear cut. After all, you shouldn't be embarking on something like this unless the way in which it gets extended reflects a genuine requirement. In this example, we would need to know that we have a lot of reports. Even if we don't know for sure that the list will grow over time, if the number we need to produce is high enough, the effort will be justified by the simplification of the reports. If we don't have enough reports though, this would be over-engineering. The reality is that sometimes this type of solution is not justified. Other times, the only way to get the benefits is to refactor to it.

For the rest of this series, I am going to produce a library based on an idea I have implemented before - a way to seperate and isolate validation code, such that it will automagically be applied where it's needed. I will outline the idea in a new post, and then I will start to build it. As I go, I will try to call out the techniques I use to make it happen that I think are useful to know.

Thanks for reading.

Jamie
