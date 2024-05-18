# C# Course

This are notes from the course `Ultimate C# Masterclass`, available on **Udemy**.


# Introduction

The course uses `Visual Studio` as IDE to run the samples. It offers `QuickWatch` to run expression to debug code.

C# files end with the extension `.cs`.

# C# Fundamentals

## A first c# program

A simple hello world in `c#`

```cs
Console.log("hello c#")
```

In `c#` code must belong to a class. If don't, it is a top level statement; basically a syntactic sugar to avoid boilerplate code, that under the hood is run in a class.

## From a text file to an executable program

In order to be run, `c#` code has to be processed by a compiler, that generate a `.dll` file, that stands for **Dynamic Link Library - DLL**. The structure of a simple project called `TodoList` will look like the following

```
.--- bin/Debug/net8.0
 |   |---- TodoList.exe 
 |   |---- TodoList.dll
 |   |---- TodoList.deps.json
 |   |---- TodoList.pdb
 |-- obj/
 |-- Program.cs
 |-- TodoList.csproj
```

If we ship the `net8.0/` folder, we could run the code executing the `TodoList.exe` app without the need of the IDE.


## Variables

Declare and define a variable are two different things

```cs
int x;     // declaration
x = 10     // initialization

int y = 20 // declaration and initialization
```

In c# keywords cannot be used as variable names, but we can add `@` to use them (e.g. `int @class = 10`). Names can contain letters, digits, and the underscore but the first character cannot be a digit. Moreover, c# is case sensitive. 

```cs
// valid variable names
string _word = "word";
int number100 = 100;

// invalid variable names
string string = "string";
int 1number = 1;
int number-one;
```

The convention states that variables names should start with a lower case letter and to use lower camel case. 

If we want to use implicit typed variables we can use the `var` keyword.


## User Input
To get user input

```cs
string userLine = Console.ReadLine();
Console.ReadKey();
```

...

# Object Oriented Programming

In `c#` we have `struct`, `class` and `record` and not just classes but they are basically the same.

```cs
class Rectangle
{
    int width;
    int height;
}
```


By default members of a class are private and if not specified it has a default constructor `Rectangle()`.
Names of public member fields should start with the capital letter, instead private members should start the `_`. 

...

# ...

# Generic Types and advanced use of methods

We can build a class that handle a generic type

```cs
public class Pair<T>
    {
        public T First { get; private set; }
        public T Second { get; private set; }
        
        public Pair(T first, T second)
        {
            First = first;
            Second = second;
        }
        
        public void ResetFirst()
        {
            First = default;
        }
        
        public void ResetSecond()
        {
            Second = default;
        }
        
    }
```

where the `default` keyword returns the default value of a type.

```cs
var default_int = default(int);      // will be 0
var default_string = default(string) // will be null

default_T = default // type is inferred
```

...


## Functions, Actions, Lambdas and Delegates

We can define not only variables with values but also functions `Func`.

Actions `Action` are functions that return nothing.

Lambdas `Lambda` are anonymous functions.

```cs
Func<int, bool> myFunction; // takes an int and returns a bool
Func<int> myFunction;       // takes no arguments and returns an int
Action<double> myAction;    // takes a double as argument
Func<string, int> lambda = stringa => stringa.Length;
```

A `delegate` is a type whose instances hold a reference to a method (or methods) with a particular parameter list and return type. 

```cs
delegate string ProcessString(string input);

string TrimTo5Letters(string input)
{
    return input.Substring(0, 5);
}

string ToUpper(string input)
{
    return input.ToUpper();
}

ProcessString processString1 = TrimTo5Letters;
ProcessString processString2 = ToUpper;

Console.WriteLine(processString1("hello"));
Console.WriteLine(processString2("world"));
```

We can also have a `multicast delegate`, i.e. a delegate variable that holds references to more than one function

```cs
delegate void Print (string input);

Print print1 = text => Console.WriteLine(text.ToUpper());
Print print2 = text => Console.WriteLine(text.ToLower());
Print multicast = print1 + print2;

Print print4 = text => Console.WriteLine(text.Substring(0,3));
multicast += print4;
multicast("Crocodile");
```

`Functions` and `Actions` are simply generic `delegates`: we can rewrite the `print1()` method with an `action`

```cs
Action<string> print1 = text => Console.WriteLine(text.ToUpper())
```

there is no need to use `delegates` but we can encounter because functions and actions came out later. 

...


# Advanced c# types 

## Reflections
**Reflection** is a mechanism that allows to write code that can inspect types used in the application.

Let's say we want to write a class method which stores in a file any object with its field and type. To know at runtime which are its field names and not only their values, we have to use `reflections`. Moreover we can know about access modifiers, methods and so on.

```cs
var house = new House("123 Maple Road", 170.6, 2);
var Converter = new ObjectToTextConverter(house);

class ObjectToTextConverter
{
    public string Convert(object obj)
    {
        Type type = obj.GetType();
        var properties = type
            .GetProperties()
            .Where(p => p.Name != "EqualityContract");

        return String.join(", ", 
            properties.Select(property => 
                $"{property.Name} is {property.GetValue(obj)}")
            );
    }
}
```


Some of the advantages of using reflections:
- loading `.dlls` at runtime
- instantiating objects or given types at runtime
- finding all classes with the given base type/implemented interface
- reading the attributes
- running a method by its name
- useful for debugging code
- creates new types at runtime

Disadvantages of using reflections
- hard to mantain and understand
- error prone approach
- big performance impact


## Attributes
**Attributes** add metadata to a type. They add information about a type or method to this type's existing metadata. 

All attribute class must derive from the `System.Attribute` class.

Let's say we want to make sure that certain class string fields must a min and a max number of characters.

```cs
var validPerson = new Person("John", "Smith");
var invalidDog = new Dog("Fido");

var validator = new Validator();
validator.validate(validPerson);
validator.validate(invalidDog);

public class Dog
{
    [StringLengthValidate(2,10)]
    public string Name { get; }

    public Dog(string name) => Name = name;
}

public class Person
{
    [StringLengthValidate(2, 25)]
    public string Name { get; }

    public string LastName { get; }

    public Person(string name, string lastName) => {
        Name = name;
        LastName = lastName;
    }
}

// use the attribute on class members
[AttributeUsage(AttributeTargets.Property)]
class StringLengthValidateAttribute: Attribute 
{
    public int Min { get; }
    public int Max { get; }

    public StringLengthValidateAttribute(int min, int max)
    {
        Min = min;
        Max = max;
    }
}

public class Validator
{
    public bool Validate(object obj)
    {
        var type = obj.GetType();
        var propertiesToValidate = type
            .GetProperties()
            .Where(property => 
                Attribute.IsDefined(property, typeof(StringLengthValidateAttribute)));
        
        foreach(var property in propertiesToValidate)
        {
            object? propertyValue  = property.GetValue(obj);
            if(propertyValue is not string)
            {
                throw new InvalidOperationException(
                    $"Attribute {nameof(StringLengthValidateAttribute)}"+
                    $" can only be applied to strings"
                );
            }

            var value = (string) propertyValue;
            var attribute = (StringLengthValidateAttribute) property.GetCustomAttributes(typeof(StringLengthValidateAttribute), true).First();

            if(value.Legth < attribute.Min || value.Length > attribute.Max)
            {
                Console.WriteLine($"Property {property.Name} is invalid");
                Console.WriteLine($"Value is {value}");
                return false;
            }
        }
    }
}

```

**Note**: all attribute classes must end with "Attribute".


Not all types are valid attributes parameters. The following does not compile

```cs
[Some(new List<int>{1, 2, 3})]
public class SomeClass 
{

}

public class SomeAttribute: Attribute
{
    public SomeAttribute(List<int> numbers)
    {
        Numbers = numbers
    }
}
```

Valid attributes parameters types are 
- simple types like `bool`, `string` or numeric types.
- `enums`
- `System.Object`
- single dimensional array of any of the above

So the following will work

```cs
[Some(new int[]{1, 2, 3})]
public class SomeClass 
{

}

public class SomeAttribute: Attribute
{
    public int[] Numbers { get; }

    public SomeAttribute(int[] numbers)
    {
        Numbers = numbers
    }
}
```
