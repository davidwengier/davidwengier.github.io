---
layout: post
title: F# Performance
published: false
---

After dabbling with F# and thinking about the code conventions and style that is idiomatic in that language, or at least in a FizzBuzz sample, it occurred to me that some of the ideals of F# could be seen as a potential code smell in C#.

Remebering this rather elegant looking snippet of F# from the previous post:

```fsharp
let fizzBuzzChecks n =
    None 
    |> fizz n 
    |> buzz n 
    |> Option.defaultValue (string n)
```

This pipelining of functions, and calling a function that has two parameters but only passing in one, is the type of F# I'm used to seeing in examples and has some very appealing qualities in terms of readability and maintainability, but something that always niggled at the back of my mind was "what about performance?". Being a C# dev I thought of how partial applcation would be implemented in that language, and immediately the thought of lambdas and Funcs come to mind, perhaps something like this:

```csharp
Func<int, Option<string>> fizzPartial(Option<string> result) => x => @is(3, "fizz", x, result);
```

This looks confusing, but you end up with a method `fizzPartial` that takes an option, and returns a function that takes an int. You would call it like this:

```csharp
Option<string> result = fizzPartial(result)(n);
```

With this solution in mind I've always wondered then if the appeal of F# is that it can do these sorts of things without the normal performance or memory overhead that lambdas have in C#. But hey, people like F# right, so it must have some benefit besides a more readable syntax, maybe this is it?

## Lets measure!

If you have a performance question there is only one thing to do: measure.

I whipped up a quick BenchmarkDotNet project with my original F# from the previous post, and the C# equivalent. I deliberately did a straight port of the code, and didn't rewrite it using Funcs as I felt that would be an unfair comparison. The code for the benchmark program is here on GitHub: [https://github.com/davidwengier/FSharpVsCSharp](https://github.com/davidwengier/FSharpVsCSharp)

What I expected to see was the F# being a lot faster than the C# because my hypothesis was that it was designed to this sort of thing elegantly, but more importantly efficiently. The results were surprising:

|   Method |   Max |         Mean |      Error |     StdDev |       Median |    Gen 0 |    Gen 1 |   Gen 2 |  Allocated |
|--------- |------ |-------------:|-----------:|-----------:|-------------:|---------:|---------:|--------:|-----------:|
| **DoFSharp** |    **10** |     **1.494 us** |  **0.0560 us** |  **0.1606 us** |     **1.429 us** |   **0.7496** |        **-** |       **-** |    **1.54 KB** |
| DoCSharp |    10 |     1.164 us |  0.0270 us |  0.0696 us |     1.138 us |   0.5245 |        - |       - |    1.08 KB |
| **DoFSharp** |   **100** |    **12.796 us** |  **0.2293 us** |  **0.2145 us** |    **12.783 us** |   **7.1716** |        **-** |       **-** |   **14.71 KB** |
| DoCSharp |   100 |    11.487 us |  0.2281 us |  0.2134 us |    11.411 us |   5.1270 |        - |       - |   10.51 KB |
| **DoFSharp** |  **1000** |   **140.903 us** |  **2.4491 us** |  **2.4053 us** |   **141.002 us** |  **75.9277** |   **0.2441** |       **-** |  **155.93 KB** |
| DoCSharp |  1000 |   124.636 us |  1.5742 us |  1.4725 us |   124.681 us |  55.6641 |        - |       - |  114.29 KB |
| **DoFSharp** | **10000** | **1,843.656 us** | **32.1990 us** | **30.1189 us** | **1,850.531 us** | **263.6719** | **138.6719** | **27.3438** | **1646.74 KB** |
| DoCSharp | 10000 | 1,671.483 us | 28.2928 us | 25.0808 us | 1,664.843 us | 285.1563 | 119.1406 | 27.3438 | 1230.77 KB |

Running the two algorithms for 10, 100, 1,000 and 10,000 elements each, you can clearly see that there is barely any difference in performance numbers, or memory used.

## Strong opinions, weakly held

So my hypothesis was wrong: Excellent! That means I get to learn something. Hypothesis number two based on the results (ie, cheating) is then that the benefit of F# is that it is _faster_ but since my C# code didn't have the overhead of lambdas that it was actually a good sign that it even kept up.

That one is a bit harder to test with a benchmark though, so I pulled out the big guns and decided to have a look at how the F# compiler actually handles the code.