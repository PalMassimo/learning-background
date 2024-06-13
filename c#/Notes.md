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

## String interpolation
Instead of using string concatenation, we can use string interpolation

```cs
int a = 1, b = 2;

// string concatenation
Console.WriteLine("a = " + a + ", b = " + b + ".");

// string interpolation
Console.WriteLine($"a = {a}, b = {b}.");

// we can make operations in the string
Console.WriteLine($"a + b = {a+b}");
```

## Chars
The type `char` is indicated with single quote `'` instead of a double quote `"`

```cs
char a = '!';
string b = "!";
```

## Loops
In `c#` we have many ways to define loops

The `while` loop

```cs
while(condition)
{
    // ...
}
```

The `do...while` loop. The difference between the `while` loop is that the body of the loop is executed at least one

```cs
do {
    // ...
} while(condition)
```

Of course, we can define the classic `for` loop

```cs
for (int i = 0; i < 10; i++)
{
    Console.WriteLine(i);
}
```

# Object Oriented Programming Fundamentals

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

## Constructor overloading
We can define multiple constructors inside our class definitions. We can use the `this` keyword to avoid code repetition

```cs
class Rectangle
{
    int Width;
    int Height;

    public Rectangle(int width)
    {
        Width = width;
        Height = 10
    }

    public Rectangle(int width, int height)
    {
        Width = width;
        Height = height
    }
}
```

```cs
class Rectangle
{
    int Width;
    int Height;

    public Rectangle(int width) : this(width, 10)
    {
    }

    public Rectangle(int width, int height)
    {
        Width = width;
        Height = height;
    }
}
```

## Expression-bodied methods
A `statement` is something that does not evaluate to a value, like `Console.WriteLine()` or `if(condition) {}`

An `expression` is something that evaluates to a value, like `1+2` or `GetText()`

If we have methods that contains only one statement or expression, we can use expression-bodied method syntax, to write concise code. 

```cs
public class Rectangle
{
    int Width;
    int Height;

    public int calculateArea() => Width*Height;
}
```

## `this` keyword
We can use `this` to refer to the current instance of a class

```cs
public Rectangle(int Width, int Height)
{
    this.Width = Width;
    this.Height = Height;

    RectangleAreaCalculator calculator = new RectangleAreaCalculator(this);

    Console.WriteLine(calculator.compute())
}
```

## Optional Parameters
Optional parameters must appear after all required parameters.
The default value of an optional parameter must be compile-time constant.
In case of amibuity about what method invoke, `c#` will choose the one with no optional parameters. 

```cs
public Dog CreateDog(string name, int age = 0)
{
    return new Dog(name, age);
}
```

## `nameof`
We can use the built-in function `nameOf` to get the name of a variable

```cs
var x = 10;
var xName = nameOf(x); // "x"
```

## `readonly` and `const`
A `readonly` field can only be assigned at the declaration or in the constructor

```cs
public class Rectangle
{
    public readonly int Width;  // = 0 declaration
    public readonly int Height;

    public Rectangle(int width, int height)
    {
        Width = width;
        Height = height;
    }
}

var rectangle = new Rectangle(2, 3);
// rectangle.Width = 10; not allowed
```

Immutability means that once an object is created it will never be modified. In other words, an object is `immutable` when none if its fields can be modified.

Const modifier can be applied to both variables and fields. Those variables and fields must be assigned at declaration and can never be modified afterward. They must be given a compile-time constant value, so a value that can be evaluated during the compilation, before the application is run.

`const` variable names should always start with a capital letter. 


## Properties
Properties and fields are different. To go deeper review the lecture. 

| Fields                                  | Properties                                      |
| --------------------------------------- | ----------------------------------------------- |
| variable-like                           | method-like                                     |
| single access modifier                  | separate access modifiers for getter and setter |
| no separate getter and setter           | getter and setter may be removed                |
| cannot be overridden in derived classes | can be overriden in derived classes             |
| should be private                       | can safely be public                            |

<br/>

A class `Order` with two properties

```cs
public class Order
{
    public string Item { get; }
    
    private DateTime _date; // backin field
    
    public DateTime Date
    {
        get { return _date; }
        set
        {
            if (value.Year == DateTime.Now.Year)
            {
                _date = value;
            }                
        }
    }
    
    public Order(string item, DateTime date)
    {
        Item = item;
        Date = date;
    }
}
```

## Object Initializers
Object initializers provide another method to insantiate objects

```cs
var person = new Person()
{
 Name: "Luca",
 Age: 20
}
```

in this way we are calling the constructor `constructor Person()` and then we are setting the fields through the `set` methods, that must be public. We can provide as many fields as we want; the ones not specified they will be set to their default values.

If we want to use object initializers but not set the `set` method as public, we can use the `init` keyword. This allows to set the field only on the object initialization

```cs
public class Person
{
 Name { get; set; }
 Age { get; init; }
}
```

## Computed Properties
Sometimes properties don't wrap any field but return a computed result instead.

```cs
class Rectangle
{
 // describes a get only property
 public Description = $"Width: {_widt}, height: {Height}";
}
```
properties should never be performance heavy.

We should use computed properties when they represent data, we should use methods if we want to represent actions.

## Static Methods and Classes
A static class can only have static methods, but a non static class can have static methods. 

```cs
static class Calculator
{
 static int Add(int a, int b) => a + b;
}
```

Static methods cannot access static fields. 

All const fields are implicitly `static`.

To initialize the static variables we can define them inline or using the `static` constructor

```cs
public class Rectangle
{
 private static DateTime _firstInstance; // = DateTime.Now;
 
  static Constructor()
  {
   _firstInstance = DateTime.Now;
  }
}
```

## Namespaces
**Namespaces** declare a scope which contains a set of related types.

Type not placed in any named namespace belongs to the `global namespace`. 

Namespaces names should match the folders structure

To use namespace

```cs
namespace MyPersonalNamespace
{
    public class MyClass
    {
        // ...
    }
}
```

From c# version 10 we can use *file scoped namespace declaration*

```cs
namaspace MyPersonalNamespace;

public class MyClass
{
    // ...
}
```

In both cases, the class `MyClass` belongs to the namespace `MyPersonalNamespace`.

To use it in another namespace

```cs
using MyPersonalNamespace;

var myClass = new MyClass();
```

## Global using directives
Instead of import a namespace in every single file we can use global directorives

```cs
global using MyNamespace
```

but the `global` directive must be before all `using` directives.

If some using directives are not needed in the code is because the IDE generates a file containing global directives of the most used namespaces.

Global directives are suggested if we repeat a namespace very often in the project. In such cases, consider to create a file containing all global directives.

## Time Measure
We can use `System.Diagnostic` to measure code performance

```cs
using System.Diagnostic;

var stopwatch = Stopwatch.StartNew();

for(int i = 0; i < 100; i++)
{
    Console.WriteLine("something");
}

stopwatch.Stop();

Console.WriteLine($"Time in milliseconds {stopwatch.ElapsedMilliseconds}");

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

# LINQ
**Language Integrated Query - LINQ** is a set of technologies that allow simple and efficient querying over different kinds of data. It offers two syntax: `method syntax` and `query syntax`. The first is the most common and easy to use

Method syntax

```cs
var evenNumbers = numbers.Where(n => n % 2 == 0);
```

Query syntax
```cs
var evenNumbers = from number in numbers
                  where number % 2 == 0
                  select number;
```

LINQ can work with other types of collections like databases or XML files.

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

## Structs
The main difference between classes and struct is that classes are reference types, struct are value types instead. 

`struct` derive from `System.ValueType`, which in turn derives from `System.Object`. Structs can have fields, constructors and methods. 

A `struct` is a value type, so when you assign a struct to another struct, a complete copy of the value is made, hence changes to the new struct do not affect the original. Structs are stored on the stack, so it is better to not use them to store big amount of data. Value types hold their data directly. 

```cs
struct Point
{
    int X { get; set; }
    int Y { get; set; }

    public Point(x, y)
    {
        X = x;
        Y = y;
    }
} 
```

### `struct` versus `class`
Structures are value types, instead classes are reference types. Only reference types can be assigned to `null`, so a `struct` cannot be `null`. 

All `struct` are sealed, i.e. cannot be inherited: the following code does not compile

```cs
struct DerivedPoint : Point
{

}
```

Since struct are not inheritable, we cannot define abstract methods. However, struct can implement interfaces. 

Interfaces are reference types, so if a struct is passed to a method expecting a parameter of an interface type, this struct will have to be boxed.

A `struct` will always have a default constructor. 

A `struct` cannot have finalizers.

A `struct` cannot have a cycle in its definition: the following code does not compile

```cs
public struct Point {
    // Point cannot reference Point
    public Point ClosestPoint { get; } 
}
```

Usually we choose a `struct` over a `class` if 
- value semantic
- the type is small and data-centric, with little or no behavior
- value-based equality check
- type not frequently boxed
- type will not store data of reference types
- immutable data

...

# Collections
Collections are types implementing specific interfaces that define methods for collection manipulation. Main collections in `c#` are `IList`, which derives from `ICollection` which derives from `IEnumerable`. 

## `IEnumerable`
`IEnumerable` is an interface which allows to iterate a type with a `foreach` loop. Therefore, we cannot use `foreach` loop with a type which does not implement the `IEnumerable` interface.

The `foreach` loop is just a syntactic sugar to iterate over a collection. When compiled, it will be transformed in something like the following

```cs
var words = new string[]{"aaa", "bbb", "ccc"};
foreach (var word in words)
{
    Console.WriteLine(word);
}
```

```cs
IEnumerator wordsEnumerator = words.GetEnumerator();
object currentWord;

while(wordsEnumerator.MoveNext())
{
    currentWord = wordsEnumerator.Current;
    Console.WriteLine(currentWord);
}
```

`IEnumerable` does not expose any method to modify collection: we can only enumerate items. This interface only contains a single method that returns an object implementing the non-generic IEnumerator interface. In programming, an `enumerator` is a mechanism that allows iterating over a collection.

Let's implement the `IEnumerable` non generic type:

```cs
var customCollection = new CustomCollection(["aaa", "bbb"]);

foreach(var item in customCollection)
{
    Console.WriteLine(item);
}

Console.ReadKey();
class CustomCollection : IEnumerable
{
    string[] Words { get; }

    public CustomCollection(string[] words)
    {
        Words = words;
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return new WordEnumerator(Words);
    }
}

public class WordEnumerator : IEnumerator
{
    private int _currentPosition;
    private readonly string[] _words;

    public WordEnumerator(string[] words)
    {
        _words = words;
        _currentPosition = -1;
    }

    public object Current => _words[_currentPosition];

    public bool MoveNext()
    {
        _currentPosition++;

        if(_currentPosition < _words.Length)
        {
            return true;
        }
        else
        { 
            return false; 
        }
    }

    public void Reset()
    {
        _currentPosition = 0;
    }
}
```

The only inconvinient of this approach is that the `item` is of the type `object` and not of the type `string`. To fix this, we have to implement the generic type `IEnumerable<T>` interface. 

```cs
public interface IEnumerable<out T> : IEnumerable
{
    IEnumerator<T> GetEnumerator();
}
```

Since `IEnumerable<T>` interface implements `IEnumerable`, we have to define both `IEnumerator<T> GetEnumerator` and `IEnumerator GetEnumerator`. However, this will bring us to define two methods with the same name and the same arguments

```cs
public IEnumerator GetEnumerator(){ ... } // because of IEnumerable
public IEnumerator<string> GetEnumerator() { ... } // because of IEnumerable<T>
```

In such cases we have to use **explicit interface implementation**, used to resolve cases where two implemented interfaces declare different members with the same name and parameters:

```cs
IEnumerator IEnumerable.GetEnumerator(){ ... } // explicit 
public IEnumerator<string> GetEnumerator() { ... } // implicit
```

note that we cannot use any access modifiers when we use explicit interface implementation, so we had to remove the `public` access modifier.

Now, to make the code calling the first or another `GetEnumerator`, if we use the `var` keyword the implicit will be called. If we want to invoke the explicit we have to use an explicit type. This explains why we couldn't have an access modifier specified for this method. An explicit interface implementation doesn't have an access modifier since it isn't accessible as a member of the type it's defined in. Instead, it is only accessible when called to an instance of the interface.

```cs
var customCollection = new CustomCollection(new string[]{"aaa", "bbb"});
var enumerator = customCollection.GetEnumerator();
IEnumerable customCollection = customCollection.GetEnumerator();
```

To make the code works we end up to the following code

```cs
class CustomCollection : IEnumerable<string>
{
    string[] Words { get; }

    public CustomCollection(string[] words)
    {
        Words = words;
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return new WordEnumerator(Words);
    }

    public IEnumerator<string> GetEnumerator()
    {
        return new WordEnumerator(Words);
    }
}
```

We have to implement also the `Dispose()` method, probably because of dot net developers mistake.

```cs
public class WordEnumerator : IEnumerator<string>
{
    private int _currentPosition;
    private readonly string[] _words;

    public WordEnumerator(string[] words)
    {
        _words = words;
        _currentPosition = -1;
    }

    public string Current => _words[_currentPosition];

    object IEnumerator.Current => Current;

    public bool MoveNext()
    {
        _currentPosition++;

        if (_currentPosition < _words.Length)
        {
            return true;
        }
        else
        {
            return false;
        }
    }

    public void Reset()
    {
        _currentPosition = 0;
    }

    public void Dispose()
    {
     
    }
}
```

## Indexers
An **indexer** allows an object to be indexed like an array, so something like

```cs
var item = customClass[key];
customClass[anotherKey] = value;
```

The key usually is an `int`, but can be any object as a string (e.g. dictionaries). To implement such feature in our `CustomCollection` class

```cs
class CustomCollection : IEnumerable<string>
{
    string[] Words { get; }

    public string this[int index]
    {
        get { 
            return Words[index];
        }
        set
        {
            Words[index] = value;
        }
    }
}
```

## Collection Initializers
We want to be able to use the collection initializer also for our `CustomCollection`

```cs
var list = new List<int>(10) {1, 2};

var customCollection = new CustomCollection {"aa", "bb"}
```

The code will not compile because it is translated in the following

```cs
var customCollection = new CustomCollection();
customCollection.Add("aa");
customCollection.Add("bb");
```

Therefore, `CustomCollection` must have the empty constructor and the `Add()` method. Instead, if we want to be able to write `new CustomCollection(10){"aa", "bb"}` we have to have the `public CustomCollection(int i){}` constructor. 

To reach this we have to add the following code to `CustomCollection` class

```cs
class CustomCollection
{
    public CustomCollection()
    {
        Words = new string[10];
    }

    private int _currentIndex = 0;

    public void Add(string item)
    {
        Words[_currentIndex] = item;
        ++_currentIndex;
    }
}
```
## ...


## `yield` statement
If we want only some element of a collection that potentially has bilion of elements, it is a waste of resources build it from scratch everytime. To get only the elements we want to use we can use the `yield` statement

The code we want to execute

```cs
var evenNumbers = GenerateEvenNumbers();
evenNumbers.Skip(5).Take(10);
evenNumbers.First();
```

We can transform the following code

```cs
public IEnumerable<int> GenerateEvenNumbers()
{
    var evenNumbers = new List<int>();
    for (int i = 0; i < int.Max; i + 2)
    {
        evenNumbers = i;
    }
    return evenNumbers;
}
```

to the following

```cs
public IEnumerable<int> GenerateEvenNumbers()
{
    for (int i = 0; i < int.Max; i + 2)
    {
        yield return i;
    }
}
```

In this way will be generated only the needed elements and they are generated one at a time. 

The previous snippet won't be executed immediately when is reached because it is translated similar to the following, i.e. it is translated in an `Iterator` definition

```cs
IEnumerable<int> GenerateEvenNumbers()
{
    return new Iterator
    {
        currentIteration = 0;
        MethodGeneratingSequence = ...
    }
}
```

Two conditions must be met to make a method an iterator method:
- must have a `yield` statement inside
- must return either `IEnumerable` or `IEnumerator`

where the next execution will start after the `yield` statement.

**Note**: if we iterate an iterator with a new `foreach` loop it gets reset. 

If we want to write a method that given a list of integers it returns the elements before the first negative one, we can use the `yield break` statement.

```cs
IEnumerable<int> GetBeforeFirstNegative(IEnumerable<int> input)
{
    foreach(var number in input)
    {
        if(number >= 0)
        {
            yield return number;
        }
        else 
        {
            yield break;
        }
    }
}
```


## Interface Segregation Principle
The **Interface Segregation Principle** belongs to SOLID and it states that
- no code should be forced to depend on methods it does not use
- no class should be forced to implement methods from an interface when those methods do not fit in the class

## Read-Only Collections
Let's consider the following code

```cs
var planets = ReadPlanets();
planets.Clean();

List<string> ReadPlanets()
{
    var result = new List<string>
    {
        "Alderaan",
        "Coruscant",
        "Bespin"
    }
}
```

If we want to make a collection read-only a first approach could be changing the result type `IEnumerable<string>` to make the `planets.Clear()` a compiler error. However we can run the `Clear()` method could be run if we use casting

```cs
var planets = ReadPlanets();
var planetsAsList = (List<string>) planets;
planetsAsList.Clear();
```

Another approach is to use `ReadOnlyCollection`, that implements `IList` interface.

For dictionaries we have `ReadOnlyDictionary`

```cs
var dictionary = new Dictionary<string, int>
{
    ["aaa"] = 1
};

var readOnlyDictionary = new ReadOnlyDictionary<string, int>(dictionary);
```


## Queue
C# sdk offers the `Queue` class that implements queue data structure

```cs
var queue = new Queue<string>();
queue.Enqueue("a");
queue.Enqueue("b");
queue.Enqueue("c");

var first = queue.Dequeue();
var second = queue.Peek(); // get an element without removing it

```

We also can use the `PriorityQueue` that takes two arguments: the first is the type of the items composing the queue, the second is the type of priority

```cs
var priorityQueue = new PriorityQueue<string, int>();
priorityQueue.Enqueue("a", 5);
priorityQueue.Enqueue("b", 5);
priorityQueue.Enqueue("c", 2);
priorityQueue.Enqueue("d", 3);

priorityQueue.Dequeue(); // is "c"
```

hence the smaller the number, the higher the priority. If two items have the same priority, the element dequeued is the element put first. 

## Stack
C# offers also the `Stack` class

```cs
var stack = new Stack<string>();
stack.Push("a");
stack.Push("b");
stack.Push("c");

stack.Pop();  // "c"
stack.Peek(); // "b"
```

where again the `Peek` method get an element withouth removing it.







