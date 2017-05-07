---
layout: post
title: Adding references to Visual Studio extension project
date: '2017-04-15T20:35:00.001-07:00'
author: Andrii Snihyr
tags:
- Visual Studio extension
- FileNotFoundException
- Visual Studio
- references
- assembly
- WPF
- ProvideBindingPath
modified_time: '2017-04-15T20:35:52.560-07:00'
---

This might sound like a very simple thing to do, just right click -> "Add References" -> Select an assembly and done. Even easier, just add a NuGet package.
Well, not quite. When I added a NuGet package to one of my extension projects I got a runtime error that took me some time to figure out. So I though it might be a good idea to write about it.
<!--more-->

The package I added was Extended WPF Toolkit, I like the controls they have. With a package in place I added a WatermarkTextBox control to my XAML like this:
```xml
<UserControl x:Class="Ext1.ToolWindow1Control"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:xctk="http://schemas.xceed.com/wpf/xaml/toolkit"
             mc:Ignorable="d">
    <Grid>
        <StackPanel Orientation="Vertical">
            <xctk:WatermarkTextBox Watermark="Enter your name..." Text="{Binding UserName}"/>
            <Button Content="Submit" Command="{Binding DoSubmit}"/>
        </StackPanel>
    </Grid>
</UserControl>
```
When I ran the app, I got an exception:

![FileNotFoundException]({{ "images/adding-references-to-visual-studio/FileNotFoundException.png" | relative_url }} "Could not load file or assembly")

What?! FileNotFound exception! Okay, lets take a look maybe our newly added assembly is not being copied to the output folder. No it is there:

![Dll in output folder]({{ "images/adding-references-to-visual-studio/DLLInOutputFolder.png" | relative_url }} "Dll is in output folder")

By examining an exception details more carefully I found that when I open my extension (toolwindow in this case), Visual Studio attempts to load a toolkit assembly, but the problem is that it is a Visual Studio loading this assembly not my extension and so the AppBase path is a path to a Visual Studio folder.
Here are Fusion logs from the exception:
```
LOG: Appbase = file:///C:/Program Files (x86)/Microsoft Visual Studio/2017/Community/Common7/IDE/
LOG: Attempting download of new URL .../Community/Common7/IDE/Xceed.Wpf.Toolkit.DLL.
LOG: Attempting download of new URL .../Community/Common7/IDE/Xceed.Wpf.Toolkit/Xceed.Wpf.Toolkit.DLL.
LOG: Attempting download of new URL .../Community/Common7/IDE/PublicAssemblies/Xceed.Wpf.Toolkit.DLL.
LOG: Attempting download of new URL .../Community/Common7/IDE/PublicAssemblies/Xceed.Wpf.Toolkit/Xceed.Wpf.Toolkit.DLL.
```

They clearly indicate that Visual Studio is not looking at the correct place. Now when it is clear why Visual Studio can't find the assembly, lets take a look on how it can be fixed.

**Approach #1.** Force the assembly to be a reference in the extension assembly itself.
Since Xceed.Wpf.Toolkit.dll is only used from XAML markup, extension assembly will not have a reference to it.

![No reference in output dll]({{ "images/adding-references-to-visual-studio/no_reference.png" | relative_url }} "Output dll does not have a reference to the toolkit dll")

To force it to be a reference, types from this assembly have to be used from C#.
Something as simple as: `typeof(WatermarkTextBox)` will do.

```csharp
[MethodImpl(MethodImplOptions.NoOptimization | MethodImplOptions.NoInlining)]
private void MakeARefrenceToNecessaryTypes()
{
    var type = typeof(WatermarkTextBox);
}
```

This code ideally needs to be somewhere in the package initialization, to guaranty it is executed before any XAML components are loaded. The way it works now is that when extension assembly is loaded all its references are loaded as well. The benefit is that only the assemblies that are really required by my extension are loaded and nothing more.
There are few downsides to this approach. I need to add to the list when I'm adding new stuff. Also compiler optimizations can take away things that are not really used in my code. That is why I decorated it with MethodImpl attribute.

**Approach #2.** Set [ProvideBindingPath](https://msdn.microsoft.com/en-us/library/microsoft.visualstudio.modeling.shell.providebindingpathattribute.aspx) attribute on a package class.
This attribute will add a directory where the package is installed to the Visual Studio probing list.
So when VS will attempt to load toolkit assembly it will find it.
This looks like much cleaner solution on the surface, but there are consequences. By adding the entire folder of my extension to the probing list, I'm exposing every assembly that is there, and so it might hit a performance if there are too many assemblies or even accidentally break another extension as it might get a version of the assembly from my package. In theory everything that is in that folder is used by the extension anyway, so the scenario of crashing some other extension is remote. But still it is good to be aware of the possibility.
Also, there is a way to specify a SubPath with this attribute though, that limits the exposure.

**Approach #3.** Subscribe to AppDomain.AssemblyResolve event and resolve it manually.
This is somewhat similar to the previous approach, but requires a bit more code.
Here is an example:

```csharp
protected override void Initialize()
{
    AppDomain.CurrentDomain.AssemblyResolve += CurrentDomain_AssemblyResolve;
    ToolWindow1Command.Initialize(this);
    base.Initialize();
}

private Assembly CurrentDomain_AssemblyResolve(object sender, ResolveEventArgs args)
{
    try
    {
        AssemblyName name = new AssemblyName(args.Name);
        return AppDomain.CurrentDomain.Load(name);
    }
    catch
    {
        return null;
    }
}

protected override void Dispose(bool disposing)
{
    AppDomain.CurrentDomain.AssemblyResolve += CurrentDomain_AssemblyResolve;
    base.Dispose(disposing);
}
```

In this case when assembly resolution fails, the event will fire and allow me to resolve the assembly manually. The difference is that now it executes in the context of my extension and so the base path is set to the root of the extension. In the above implementation it still have a problem of exposing all of the assemblies that are in the root of an extension, but you can fine grain the list and allow only the assemblies you need.

In my case I decided to choose an approach #2, It is the simplest and the cleanest approach with no maintenance required.

Another thing I want to mention regarding references in the Visual Studio extension projects is that any assembly you want to directly reference in your extension project must be signed.
The reason is simple, extension assembly is  signed and since it is signed it cannot reference an unsigned assemblies. You will get a following error if you'll attempt to reference an unsigned assembly:
*Could not load file or assembly 'MVVM.Essentials.Desktop, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null' or one of its dependencies.* A strongly-named assembly is required.  

Happy coding :)