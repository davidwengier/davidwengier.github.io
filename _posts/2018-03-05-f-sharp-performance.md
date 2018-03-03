---
layout: post
title: Looking into F# Performance
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

With this solution in mind I've always wondered then if the appeal of F# is that it can do these sorts of things without the normal performance or memory overhead that lambdas have in C#. Remember I'm a complete newbie to F# so I can only apologise if you're yelling at your computer right now.

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

Running the two algorithms for 10, 100, 1,000 and 10,000 elements each, you can clearly see that there is barely any difference in performance numbers, or memory used, and if anything the F# is slightly slower.

## Strong opinions, weakly held

So my hypothesis was wrong: Excellent! That means I get to learn something. Hypothesis number two based on the results (ie, cheating) is then that the benefit of F# is that it is _faster_ but since my C# code didn't have the overhead of lambdas that it was actually a good sign that it even kept up.

That one is a bit harder to test with a benchmark though, so I pulled out the big guns and decided to have a look at how the F# compiler actually handles the code. To do this I went to one of my favourite tools: [SharpLab](https://sharplab.io).

SharpLab is an online code decompiler, and even has an experimental runner, and the reason I love it is that it specifically turns off all "syntax sugar" when decompiling so you can see what the compiler is really doing. I used it extensively in preparing for my talk on Lowering and in fact even submitted a PR for because it didn't fully export the innards of `using` statements.

Feeding the F# code into SharpLab revealed the answer immediately! I used it to convert from F# to C# and the line for the above pipeline in F# looks like this:

```csharp
FSharpOption<string> option = _.@is(5, "buzz", n, _.@is(3, "fizz", n, null));
```
The full decompilation can be see [here on SharpLab](https://sharplab.io/#v2:DYLgZgzgNAJiDUAfYBTALgAgJYQzLAbjgPYBOGxArmgA7UDKapWAdgOYYsakoSXCYAvAFgAUBgnYwnDAFI8hEuUEYADBjQALFCzGT9GesQC2KDAAoLPPgIyIAfBgDyNNFmIsAdDBRgAhvxoAGp+wJQo5gBEkQCUMRjwFNR0aIzM7DF6kijAEGZY0taBngCSEEamGtq64gYSRQJZEjl5TQYAch4oYmKomGBYAF6DGAAeGLgqOBgAzBiRA8ORYxO96BgARpTDK5PYuACs81tLu2uYG37suxhTuADsx1dsy+MQPaLnGIuDAELbgwAwtoAMYAa1wXBEtQknRYZjaDm+QxGXERjhOqIw6M2zxkOJcbg83l8AQEITCZnMECYrA4LEyMMkXx+/x20P0AG0AIyeTzc1SqAC6dkcABkcGhPMY/DRkcM2UDQRCPpyADwAURYTAAngAFYisND2IVfGWsDB+UhsAi3NqsgGijASmmeLBoFDkeg6mkoYyeQEeCDEVCeADqzA9EvhbW9vv9gZYwdDACUUH4YNGIvEkVg2CwyN0maogA===)

Ignoring the slight differences in naming and the underscores there are two things immediately apparent:

1. The compiler has inlined the `fizz` and `buzz` methods entirely which tells me that there are some quite aggressive optimizations in the compiler
2. There are no lambdas! In fact the pipeline has simply been flipped around with the result of the first function passed in as an argument to the second.

## I'm very new at this

Clearly after seeing this code we wouldn't expect to see major differences between the F# and C# since they are so similar. In fact further this reveals the harsh truth of my lack-of-F# skills: I didn't use any partial application after all! Partial application would have resulted in the creation of new functions with less arguments, but their simply aren't any. What I thought was partial application must have been merely the automatic passing of arguments through pipelines.

A helpful comment from [Chris Carr](https://gist.github.com/davidwengier/bc95d4b0b52042ed5d54326d7810b30b#gistcomment-2364907) set me straight though; I just had to drop the parameters from the `fizz` and `buzz` definitions.

```fsharp
let fizz = is 3 "fizz" 
let buzz = is 5 "buzz" 
```

With that simple change the code now uses partial application and the compiler will generate classes to hold those methods in a very similar way to how C# compiles lambdas. This can be verfied again with SharpLab but unless you're used to reading lowered C# it can be confusing. Nevertheless with that change in place we can rerun the benchmark and the results get a lot clearer.

|   Method |   Max |         Mean |      Error |     StdDev |       Median |    Gen 0 |    Gen 1 |   Gen 2 |  Allocated |
|--------- |------ |-------------:|-----------:|-----------:|-------------:|---------:|---------:|--------:|-----------:|
| **DoFSharp** |    **10** |     **2.152 us** |  **0.0694 us** |  **0.2026 us** |     **2.114 us** |   **1.2054** |        **-** |       **-** |    **2.48 KB** |
| DoCSharp |    10 |     1.424 us |  0.0556 us |  0.1594 us |     1.366 us |   0.5245 |        - |       - |    1.08 KB |
| **DoFSharp** |   **100** |    **18.779 us** |  **0.5799 us** |  **0.8856 us** |    **18.483 us** |  **11.7493** |        **-** |       **-** |   **24.09 KB** |
| DoCSharp |   100 |    12.352 us |  0.1971 us |  0.1646 us |    12.331 us |   5.1270 |        - |       - |   10.51 KB |
| **DoFSharp** |  **1000** |   **203.328 us** |  **2.6131 us** |  **2.1821 us** |   **202.955 us** | **121.8262** |        **-** |       **-** |  **249.68 KB** |
| DoCSharp |  1000 |   134.318 us |  3.5127 us |  3.9043 us |   133.580 us |  55.6641 |        - |       - |  114.29 KB |
| **DoFSharp** | **10000** | **2,664.391 us** | **51.5494 us** | **61.3659 us** | **2,654.340 us** | **421.8750** | **218.7500** | **27.3438** | **2584.25 KB** |
| DoCSharp | 10000 | 1,792.485 us | 30.9460 us | 27.4328 us | 1,798.266 us | 285.1563 | 119.1406 | 27.3438 | 1230.77 KB |

With the partial application in place we can see that the F# code is quite a bit slower and more memory hungry than the C#. Its fairly safe to assume that the overhead is simply due to needing extra classes - instances of which take up heap memory - and calling into those classes - with virtual calls being slower than the previous static calls.

## So what did we learn?

So in summary it turns out that I still have a long way to go to understanding F#, but also that my gut reaction of "that must be using lambdas so it must be slow" is essentally correct.

No in the real world this level of "slow" is unlikely to be felt but at least this exercise hsa confirmed for me that whatever the reasons for using F# are, performance is not necessarily one of them, at least not without the usual standard of effort that would be required in other languages.