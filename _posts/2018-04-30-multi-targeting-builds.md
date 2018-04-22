---
layout: post
title: ! Multi targeting builds in one solution
published: false
---

I've recently starting working on a new project in my spare time, [DbUpgrader](http://github.com/davidwengier/dbupgrader), and I'm trying to work on it for at least a few minutes every night. I variously use a Macbook Pro or Windows machine, and sometimes I use Visual Studio 2017 but sometimes I'm just using Visual Studio Code and mucking around on the console. I'd like to also try out Visual Studio for Mac sometime soon. All of these different environments have their advantages and features, but I mostly want to make sure that I can work in all of them, on the same project, without issue.

Enter the [new project system](https://github.com/dotnet/project-system) in Visual Studio which allows for minimal .csproj files that remain easily editable MSBuild targets without having to compromise with having separate build scripts. The challenge I set myself was to see if I could create a single solution with projects that fulfilled the following needs:

* Opens in Visual Studio on Windows without error
* Builds in Visual Studio without issue
* Tests appear in the Test Explorer in Visual Studio and tests run as expected
* Works with `dotnet build` on Mac and Windows
* Works with `dotnet test` on Mac and Windows

This may seem easy but its slightly complicated by the fact that I want to support not on the full .NET Framework v4.6 on Windows, but also .NET Core on Mac and Windows, without the .NET 4.6 support being an issue on Mac. To support .NET 4.6 the shared libraries need tobe .NET Standard 1.3 or lower, but I also have some functionality and tests that use `Microsoft.Data.Sqlite` which is only .NET Standard 2.0. So on Windows I want a build for .NET 4.6 without Sqlite support, and a build for .NET Core with it, and on Mac a build for .NET Core with support and no errors relating to missing .NET Framework support.

