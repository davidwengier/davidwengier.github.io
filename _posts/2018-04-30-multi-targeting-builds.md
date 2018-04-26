---
layout: post
title: ! Multi targeting builds in one solution
published: false
---

I've recently starting working on a new project in my spare time, [DbUpgrader](http://github.com/davidwengier/dbupgrader), and I'm trying to work on it for at least a few minutes every night. I variously use a MacBook Pro or Windows machine, and sometimes I use Visual Studio 2017 but sometimes I'm just using Visual Studio Code and mucking around on the console. I'd like to also try out Visual Studio for Mac sometime soon. All of these different environments have their advantages and features, but I mostly want to make sure that I can work in all of them, on the same project, without issue.

Enter the [new project system](https://github.com/dotnet/project-system) in Visual Studio which allows for minimal .csproj files that remain easily editable MSBuild targets without having to compromise with having separate build scripts. The challenge I set myself was to see if I could create a single solution with projects that fulfilled the following needs:

* Opens in Visual Studio on Windows without error
* Builds in Visual Studio without issue
* Tests appear in the Test Explorer in Visual Studio and tests run as expected
* Works with `dotnet build` on Mac and Windows
* Works with `dotnet test` on Mac and Windows

This may seem easy but its slightly complicated by the fact that I want to support not on the full .NET Framework v4.6 on Windows, but also .NET Core on Mac and Windows, without the .NET 4.6 support being an issue on Mac. To support .NET 4.6 the shared libraries need to be .NET Standard 1.3 or lower, but I also have some functionality and tests that use `Microsoft.Data.Sqlite` which is only .NET Standard 2.0. So on Windows I want a build for .NET 4.6 without Sqlite support, and a build for .NET Core with it, and on Mac a build for .NET Core with support and no errors relating to missing .NET Framework support.

## Multi-targeting means multi-builds

The easiest way to think about multi-targeting in the new project system is to remember this simple fact: Each target framework acts like its a duplicate of the whole project. Consider a .csproj file with the following declaration.

```
<TargetFrameworks>net46;netcoreapp2.0</TargetFrameworks>
```

When building this project MSBuild will run the build twice, ones for .NET Framework 4.6 (net46) and once for .NET Core (netcoreapp2.0). Knowing this helps explain the logic of how the project file should be laid out in order to change what is built for each target.

In my case I want the Sqlite code to only be built for netcoreapp2.0 because it needs to target .NET Standard 2.0, and net46 is not quite at that level. The full table of versions and what they support is [on GitHub](https://github.com/dotnet/standard/blob/master/docs/versions.md) but suffice to say that net46 maps to .NET Standard 1.3.

Armed with this information we know that we need to exclude the Sqlite dependencies and files when building for net46 and this is done with a `Condition` attribute on the relevant spots in the project file.

```
  <ItemGroup Condition="'$(TargetFramework)' == 'netcoreapp2.0'">
    <ProjectReference Include="..\..\src\DbUpgrader.Sqlite\DbUpgrader.Sqlite.csproj" />
  </ItemGroup>
```

Here I am instructing the project system to only reference the Sqlite project if the target framework of the build is netcoreapp2.0. This is where thinking about the targets as separate builds makes sense. When its passing through this file building for net46 MSBuild will see that the condition is not met, and simply skip over this part of the file. No reference will be added. When building for netcoreapp2.0 the reference will be added.

## Excluding files

Thats all well and good for the reference but obviously if the reference is there then there must be files that use it. Because the new project system doesn't need specific file inclusions its unlikely that you would have a node that can have a condition added to it, so we need to be a bit creative.

You can use an `exclude` attribute on a `<Compile>` element alongside the normal `include` but I found the usage of that a bit ugly, and since by default there isn't any `<Compile>` elements needed in the Sdk projects it seemed a bit clunky to add one. The solution I settled on was to simply update the `DefaultItemExcludes` property that already exists, and is already used by the default project.

```
  <PropertyGroup Condition="'$(TargetFramework)' == 'net46'">
    <DefaultItemExcludes>$(DefaultItemExcludes);Integration\Sqlite\**\*</DefaultItemExcludes>
  </PropertyGroup>
```

Since we're now telling MSBuild to _exclude_ items we don't want, we flip the condition so its based on net46. These two things combined mean we get the project including everything we want when building for .NET Core, and not including the wrong things when building for .NET Framework.

## Targeting the targets

If the conditions so far have been based on the frameworks being targeted, then how do you make the targets conditional? To do that you need something at a higher level and fortunately the operating system fills this role perfectly. We can tell MSBuild to build .NET Core and .NET Framework on Windows, just .NET Core on a Mac, and everything will flow correctly from there based on whichever target is being built at the time. The conditions look very similar too.

```
  <TargetFrameworks Condition="'$(OS)' != 'Unix'">net46;netcoreapp2.0</TargetFrameworks>
  <TargetFrameworks>netcoreapp2.0</TargetFrameworks>
```

Two things to note here: The first thing obviously is that the OS for Mac is "Unix". This surprised me, but is not a big deal. I guessed originally that it would be "Mac" and when that didn't work I simply added a `<Message Text="$(OS)" />` to my project file and observed what the output was.

Secondly you'll notice that there is only a condition on one of the elements. This was not what I tried first, and whilst having conditions on both worked fine in the `dotnet build` world (on Mac and Windows) Visual Studio itself got very confused. I posted about it on Twitter and the very helpful [David Kean](https://twitter.com/davkean) who works for Microsoft on the new project system [pointed](https://twitter.com/davkean/status/987820416579223552) me to [this GitHub issue](https://github.com/dotnet/project-system/issues/1829) explaining that I'd hid a bug. It wasn't a big deal to remove one condition 
