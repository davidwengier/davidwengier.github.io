---
layout: post
title: ! Codify your coding standards with .editorconfig
---

Every dev team has coding standards. Sometimes they're established through convention, tradition, example and maybe sometimes there is even a formal document outlining them (hopefully in a living format that can be updated!). No matter how its done though, nobody wants to be the bad guy in code reviews or pull requests and pull people up for what are usually minor infractions, however at the same time nobody wants to see a codebase be neglected and let inconsistency creep in, or readability wane.

Visual Studio has many excellent rules and formatting options to enable it to be fully configured to match your coding standards and conventions, but in a team environment it can be a pain to keep everything in sync. There are "team settings file" options which work most of the time but its not perfect and it still requires everyone to configure Visual Studio to use that shared file any time they join a team, or reinstall their machine.

Fortunately there is a way to enforce some coding standards at tooling level without these concerns, in particular with Visual Studio 2017 it now honours the configuration in a .editorconfig file, which overrides an individual developers settings and tells the IDE how to behave on a per-repository basis. The .editorconfig file is simply committed to the root of the repository and from then on it dictates things like indentation, formatting, style and naming rules. Not all IDEs will support all of the same features but the list on [the official site](http://editorconfig.org/#download) is certainly impressive.

In this post I'll be talking about how to codify some specific .NET related rules for Visual Studio. For more detailed information the [official documentation](https://docs.microsoft.com/en-us/visualstudio/ide/create-portable-custom-editor-options) is great, though I might be biased since its where I submitted my first ever PR to the docs project.

## Naming Rules

Naming rules allow you to codify the standards around naming and casing of fields, properties, constants etc. in your codebase. Each naming rule needs a name, which is specified in lower case with underscores, a severity, and a style to apply. For example:

```
dotnet_naming_rule.public_members_must_be_pascal.severity = error
dotnet_naming_rule.public_members_must_be_pascal.symbols = public_symbols
dotnet_naming_rule.public_members_must_be_pascal.style = pascal_style
```

In this example `dotnet_naming_rule` denotes that we're defining part of a rule, `public_members_must_be_pascal` is the name of our rule, and we're going to apply it to symbols that match the `public_symbols` naming symbols which we'll define later. We want this rule to be enforced at all times so the `severity` is `error`, which means Visual Studio will treat violations the same as it treats compiler errors. Lastly we've said that things that match this rule should use the style defined in `pascal_style` which is the name we will give to our style.

## Naming Styles

Naming styles define how a developer should format symbols that match any applied rules. Like naming rules they have a name, and they can then specify prefixes, suffixes, word separators and capitalization rules. In this case we simple need to define the capitalization rule of `pascal_case` like so:

```
dotnet_naming_style.pascal_style.capitalization = pascal_case
```

Again `dotnet_naming_style` means we're defining a style and `pascal_style` is the name of the style which we used in the rule.

## Naming Symbols

The final piece of the puzzle tells Visual Studio which symbols the rule should apply to. For our `public_symbols` we need to specify the accessibility to be public, and that we want the rule to apply to properties, methods, fields, events and delegates. We could probably also add in classes, structs and enums to this.

```
dotnet_naming_symbols.public_symbols.applicable_kinds = property,method,field,event,delegate
dotnet_naming_symbols.public_symbols.applicable_accessibilities = public
```

Naming symbols also allow you to specify `required_modifiers` so that you can target static, readonly, async or const symbols differently.

## Putting it all together

Those three elements combined are what makes a rule fully codified and means Visual Studio can be the bad guy when it comes to enforcing coding standards. No more need to have arguments about whether constants are SHOUTING_AT_YOU or are ABitMoreSubtle, you can end the age old battle between `_fields` and `m_fields` etc.

Additionally naming symbols and styles can be used by multiple naming rules so you only need to define something like `pascal_style` once to apply a pascal case capitalization convention to a few different things.

Be warned however, if you're introducing this to a legacy code base you need to tread carefully and probably just take the hit and fix all of the issues it raises in the same commit. Even if you set the severity to `warning` or `suggestion` at the very least you'll be potentially filling up the error window with issues and it's never a good idea to give anyone a reason to ignore things in the error window.

The .editorconfig file can also be used to specify indentation styles, brace usage and style, `var` usage and even whether `this.` is required to be used, or where System using statements should go. If you can spend the time to fill out all of the possibilities it makes like much easier in a team, as your codebase is immune to the quirks of individual dev machine configurations, or in open source projects ensuring contributors always match the style of the project they're contributing to.

A full example of the .editorconfig file I'm currently using for my personal projects can be found in the DbUpgrader project [here](https://github.com/davidwengier/dbupgrader/blob/master/.editorconfig).

### Gotchas

Some gotchas with setting up editor config files that I've found so far:

* If you specify constants should be pascal case then VS won't error when a constant is all caps since thats still valid pascal case.
* Ordering of rules in files seems to be inconsistent so rules around private fields and constants sometimes overlap for private constants, and VS will think you're doing the wrong this.

I will update the post if I find others.