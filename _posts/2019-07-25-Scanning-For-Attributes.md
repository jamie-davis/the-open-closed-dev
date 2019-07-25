---
layout: post
title: Scanning for types with attributes
---

In .NET I've used a useful technique many times before to avoid having to manually hook things together, by scanning for types in an assembly.

This gives a very declaritive feel to the code and adding a new example of some previously used pattern is a simple task involving only the creation of a new class. An example of this is in my [ConsoleToolkit](https://www.nuget.org/packages/ConsoleToolkit/) library, which helps you to build .NET command line applications. This is a sample command definition:

```c#
    [Command]
    [Description("Create an environment. The environment will be created in the current subscription. If the environment already exists in the configuration table, this command will fail.")]
    class CreateCommand
    {
        [Positional]
        [Description("The name of the environment. The resource group and all resource names will be derived from this name.")]
        public string Name { get; set; }

        [Option("configure")]
        [Description("Automatically run configure after creating the resources.")]
        public bool AutoRunConfigure { get; set; }

        [Option("export")]
        [Description("Just export the template JSON instead of executing it. The json will be written to the specified file.")]
        public string JsonExportFile { get; set; }

        [CommandHandler]
        public void Handle(IConsoleAdapter console, IErrorAdapter error)
        {
            ///command implementation
        }
    }
```

This was taken from an Azure deployment utility, and I've trimmed it a bit. It defines a command accepted by a console application, and the toolkit build the command line schema by scanning the application assembly looking for classes that have the ```[Command]``` attribute. It pulls information about the command parameters and options out of the class by looking for decorated properties, and calls the method with the ```[CommandHandler]``` attribute if the user's input matches the command. As a developer, you just need to declare the class and the command will be accepted. The key point is that the list of commands is determined automatically.

For my validation project, I'm using the same technique to find validation methods. Here's an example from a unit test:

```c#
    [Validation]
    public static class StringValidation
    {
        public static ValidationResult SurnameIsMandatory([Field("Customer.Surname")] string surname)
        {
            return ValidationResult.Valid; //Not decided how to create an error result yet
        }

        public static ValidationResult FirstNameIsMandatory([Field("Customer.FirstName")] string firstname)
        {
            return ValidationResult.Valid; //Not decided how to create an error result yet
        }

        public static string NotAValidator(string firstName, string surname)
        {
            return $"{surname}, {firstName}";
        }
    }

```

The rule will be, classes annotated with ```[Validation]``` may contain validation methods. Validation methods must be static, return ```ValidationResult```, and have at least one ```[Field(name)]``` parameter. (Clearly the example doesn't do any validation, but I've not decided on the best way to create a failure yet.) The declared fields will be used to associate the validation with the data it needs to run against, and the framework will organise calling the validation method when required. The task of adding a new validation becomes declaring a method on a validation.

We can use .NET reflection to find instances of the class very easily. This is how I'm finding them at the time of writing:

```c#
    public static class ValidationFinder
    {
        public static IEnumerable<ValidationInfo> ScanAssembly(Assembly assembly, Func<Type, bool> func = null)
        {
            var types = assembly.DefinedTypes.Where(t => t.IsClass && t.GetCustomAttribute<ValidationAttribute>() != null && !t.ContainsGenericParameters);
            if (func != null)
                types = types.Where(t => func(t));

            var methods = types.SelectMany(t => t.DeclaredMethods
                    .Where(m => m.IsStatic && m.IsPublic && m.ReturnType == typeof(ValidationResult) &&
                                !m.ContainsGenericParameters));

            return methods.Select(m => new ValidationInfo(m));
        }
    }
```

This is a ```netstandard2.0``` project, so it's using slightly different reflection syntax to older versions, but it's the intent that matters. Breaking it down, we begin with this linq query:

```c#
    var types = assembly.DefinedTypes.Where(t => t.IsClass && t.GetCustomAttribute<ValidationAttribute>() != null && !t.ContainsGenericParameters);
```

The ```Assembly``` instance provides us with a list of its types, and we are filtering them by the presence of a ```ValidationAttribute```, and adding the rules that the type must be a class without any unknown generic parameters. I've not specified that it must be a static class, but it would be ugly to mix validation methods into a class with other responsibilities. I don't know how people will use the library, so I decided against forcing them to use static classes<sup id="a1">[1](#f1)</sup>.

The next line is a bit odd:

```c#
    if (func != null)
        types = types.Where(t => func(t));
```

Here I'm allowing an optional filter function to be passed in - this is entirely so that I can write unit tests that have repeatable results<sup id="a2">[2](#f2)</sup>.


```c#
    var methods = types.SelectMany(t => t.DeclaredMethods
            .Where(m => m.IsStatic && m.IsPublic && m.ReturnType == typeof(ValidationResult) &&
                        !m.ContainsGenericParameters));
```

This is simply finding public static methods that return ```ValidationResult``` on the qualifying test classes. Finding the methods is the point of the exercise, and we essentially discard the containing class.

And here we are simply returning the result:

```c#
    return methods.Select(m => new ValidationInfo(m));
```

I like the declaritive approach to this - we don't need to maintain a list of validations anywhere because the methods can be discovered at runtime. I've used this technique before, and performance of reflection has never been a major concern, but I would try to do this as a one time initialisation step, rather than repeatedly. I would expect that connecting up the validations to the model will be more of a performance concern, but I'll address that when 

Thanks for reading.

Jamie

<b id="f1">1</b> It's your foot. [↩](#a1)

<b id="f2">2</b> This is the test in question:
```c#

    [Fact]
    public void ValidationsAreRecognised()
    {
        //Arrange
        var assembly = GetType().Assembly;

        //Act
        var validations = ValidationFinder.ScanAssembly(assembly, t => t?.Namespace == _testNamespace);

        //Assert
        var output = new Output();
        output.FormatTable(validations.Select(v => new {ContainingType = v.ContainingType.Name,  Method = v.Method.Name}));
        output.Report.Verify();
    }

```

This test is running a scan on the test assembly, and then checking the expected validations are extracted. I will need to write numerous test validations, and I don't want them to cause this test to fail, so I've incorporated a filtering mechanism so that I can limit the scope of the scan. This is an example of me being comfortable with allowing the requirement to test the code have an impact on the design - I will tolerate this sort of intrusion to give me access to the power of unit testing.

The test above shows one of my favourite unit testing techniques, which I'll post about later.
 [↩](#a1)
