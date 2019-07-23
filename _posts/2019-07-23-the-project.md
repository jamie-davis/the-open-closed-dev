---
layout: post
title: The project
---

Some time ago I built a validation mechanism into a WPF application. We had literally hundreds of fields that need validation, and they could be displayed in various combinations. This was a WPF application using the MVVM pattern, and screens were composed from reusable components that could be put together in different combinations, and it was common for a component to be part of multiple windows. I didn't want the validations to get in the way of the freedom of composition<sup id="a1">[1](#f1)</sup>.

When you think about validation, the kind that springs immediately to mind is probably what I call "first order" validations - mandatory fields, maximum length, formatting, range checking etc. There are also "second order" or "cross validations" where the validation needs to take into account more than one field - e.g. "If PaymentType is CreditCard then OrderTotal must be greater than 10".

Another aspect that I considered important is that I wanted to be able to be able to show the same field on multiple components without having to duplicate the validation code. I could always make the code shared, but the onus was still on the developer to know where the validations are needed. That is to say, if I add a new validation on a field, I have to diligently go and find all of the components that show the field and manually implement a call to the appropriate validation.

These were the key points I had in mind:

1. Validations are specified in terms of fields. e.g. Customer Name is mandatory.

2. Validation is a responsibility, so integral to the component is probably not the best place for them.

3. Validations need testing, and the simplest possible test is to call the validation method directly.

4. Adding components with existing fields requires you to find out what validations are needed and possible.

5. Components contain fields, and can be composed into larger units. The set of validations needed for the unit will depend on which components are present.

Looking at this, I thought that if you had a library of validations, and you knew which fields were present on whatever components were active, you could automatically run the validations that applied. And furthermore, this would mean adding a new validation to the library would be all that's needed to get it to run where needed. At no point would the developer be required to manually work out where the validation needed to be called. Adding fields to components would also automatically add their validations as well with no extra effort.

This led me to the following: 

(Note, it's been a long time since I last saw this work, so the below is what sounds correct to me today, rather than the battle tested design it was in reality.)

1. Validations are static functions on static classes. e.g.

```
// Note, off the cuff, but this is concept...
[Validation]
public static class CustomerValidations
{
    [Validation]
    public ValidationResult SurnameMandatory([Field("Customer.Surname")] string surname)
    {
        return string.IsNullOrWhiteSpace(surname)
            ? ValidationResult.Error("Surname is mandatory")
            : ValidationResult.Valid;
    }
}
```

2. Validations are specified in terms of fields, so the model being validated should identify the fields it contains. e.g.

```
public class CustomerViewModel
{
    public string Surname { get; set; }
}
``` 

Here the assumption would be that all properties on that view model are "Customer" fields, by convention. In fact, in the system, the properties had the same name as the database fields and the validation was simply specified in those terms. Where the database field's name was too terrible, there was an attribute to override it so the ViewModel property had a sensible name.

3. This was a WPF MVVM project so there was integration with `INotifyPropertyChanged` and `IDataErrorInfo` so that the validations would be executed when needed and would present the results in a WPF friendly way.

My plan for this project is to reproduce the intent of the validation mechanism. It worked very well, and I'd like access to something like it for every project going forward. My plan is to gradually build the functionality out, and when I need an interesting technique I will post about it here, and hopefully one day someone will find it useful.

Thanks for reading.

Jamie

<b id="f1">1</b> I didn't want to force a screen to have _component A_ and _component B_ just because they reference each other. They should only have both because they need both. [↩](#a1)

