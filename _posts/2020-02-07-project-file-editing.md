---
layout: post
title: Project File Editing in Visual Studio
published: false
---

One of the things at Microsoft that I "own" is the editing of SDK-style project files for .NET. This is where you can right click on a project in Solution Explorer and click "Edit Project File", or indeed just double click on the project if you have your settings configured that way. The XML editor will pop up, you can make changes, save the file, and everything works as expected. The code for all of this is not open source (though, oddly, it used to be ¯\_(ツ)_/¯) so there is nothing I can point to, and its highly coupled with Visual Studio for Windows notions so it might not make sense anyway, but I thought I'd write down some thoughts anyway.

The challenge with project file editing is that it has to handle quite a few different scenarios:

1. Changes can be made to the in-memory project, that have to be written to the text buffer in memory (eg, project properties changes, excluding a file via Solution Explorer)
2. Changes can be made to the text buffer that need to be loaded into the in-memory project (eg, Replace In Files with the “open files” option off)
3. Changes can be made to the text buffer that need to NOT be loaded into the in-memory project (eg, user typing in the editor)
4. Changes can be made to the file, that need to be reloaded by the text buffer and in-memory project (eg, switch branch in git, or use an external editor)

