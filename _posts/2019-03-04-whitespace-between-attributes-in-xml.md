---
layout: post
title: Benefits of Open Source - XML serializers in .NET
published: false
---

I learned an interesting fact today: There are no XML serializers in .NET that will let you inject whitespace between attributes. Well for some values of "interesting" anyway.

## The problem

There are a lot of XML files in the project I work on (specifically XAML files but we'll ignore that for now as it's not relevant) and they need to be largely consistent with each other, but also have minor differences. In order to make comparisons between them easier I wrote a small program to clean up and sort the XML nodes in these files but was getting frustrated with my formatting options.

I could turn on auto indenting in the `XmlWriterSettings` class via the `Indent` property and get output like this:

```xml
<Root Name="Foo" Type="Bar">
  <Child Name="Baz" Count="5" />
</Root>
```

I'm happy with the indentation there, but some of the elements have a lot of attributes and those lines could get long. I could turn on the `NewLineOnAttributes` property and get output like this:

```xml
<Root
  Name="Foo"
  Type="Bar">
  <Child
    Name="Baz"
    Count="5" />
</Root>
```

I don't like that at all. It would be worth pointing out here that this is all highly subjective and by no means should anybody care this much about XML formatting.

Having said that, I wanted to acheive this output:


```xml
<Root Name="Foo"
      Type="Bar">
  <Child Name="Baz"
         Count="5" />
</Root>
```

First attribute on the same line as the node, attributes wrapped to line up with the first attribute. Beauty. There are `WriteWhitespace` methods for an `XmlWriter` but I quickly discovered, as mentioned earlier, that if you use them while writing out attributes they throw an exception. They also somewhat annoyingly turn off automatic indentation as soon as they're used, but that's not a big deal.

Normally I would call this a day and admit defeat here, but this time I thought I'd at least go and find out how the code is written and why this behavior exists. Thanks to the fact that the .NET Framework is open source, this was a relatively straight forward thing to do.

## Reading the code

If you haven't seen it already you should check out (https://referencesource.microsoft.com/)[https://referencesource.microsoft.com/]. It's the source code to the .NET Framework available in your browser, easily navigable, and includes all of the comments the developers wrote so is much better than trying to decompile it yourself.
