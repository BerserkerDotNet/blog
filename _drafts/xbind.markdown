---
layout: post
title: Reducing boilerplate code in UWP with T4 templates
author: Andrii Snihyr
date: '2017-08-13T14:05:00.001-07:00'
tags:
- T4
- Binding
- x:Bind
- C#
- UWP
- MVVM
modified_time: '2017-08-13T14:05:00.001-07:00'
---

I've been using [x:Bind][1] for quite some time. It is a great way to improve performance of your application and also get a compile time insurance that your binding expression is correct. But to use [x:Bind][1] with MVVM I have to write some boilerplate code in the application, which is boring, so lets see how this can be improved.
<!--more-->

First, lets start with why and what kind of boilerplate code have to be written and then we can get to how it can be done easily with minimum effort.
#### Why?
Unlike your usual `Binding` that uses `DataContext` as a source of properties, [x:Bind][1] uses a page or a user control to look for those.
Here is what Microsoft recommends to do to make the view model visible for [x:Bind][1]:

>To expose your view model to {x:Bind}, you will typically want to add new fields or properties to the code behind for your page or user control.
>
> -- <cite>[Microsoft Docs][2]</cite>

#### What?
I'm not a fan of adding anything to the code behind of my views, but looks like in this case it has to be an exception. Definitely I'm not going to add the hole bunch of properties to my code behind, but I would need to add at least one. I need to add a property that will represent my view model. I can't really use `DataContext` as it is of type `object` and casting every time is out of the question.

#### How?
I think there is no better way to deal with boilerplate code in C# then using a [T4][3] templates. In this case the template will have to find all the views that I have and generate a `partial` class that contains a property of `ViewModel` to which I can bind in my markup. So, I went searching on the internet for examples, but I couldn't find exactly what I was looking for. Plus almost all examples use reflection to find types and generate classes. Which is not to streamline....

So I decided to write my own template for that. 
So the first step is to get all the views in the project, preferably without using the reflection. Turns out that [T4][3] exposes `Host` that can help you navigate around the project, like this:
```csharp
	var views = Directory.GetFiles(Host.ResolvePath("..\\Views"))
				.Select(f => Path.GetFileNameWithoutExtension(f))
				.Where(f => !f.EndsWith(".xaml"))
				.ToList();
```
Here `Host.ResolvePath("..\\Views")` will evaluate to my views folder. I need to exit to the root folder using `..` because my template is in the root level folder called `Generated`. 

![FolderStructure]({{ "images/t4-with-x-bind/folder_structure.png" | relative_url }} "Folder structure where T4 template is under folder called 'Generated'")

From there I grab all the files in the views folder, filter those that I don't need (xaml.cs) and I have a list of views.
Nice and simple, and most importantly without any reflection in between.

Second step is to find out the namespace of the project. After all I want to re-use this template in many of my projects. That is a bit more difficult than grabbing a bunch of files. I know there is `DefaultNamespace` property of the project and since I have access to the Host, I should be able to get to that property. After an hour of biting my head over a non-human friendly documentation, I came up with this:
```csharp
private string GetNamespace()
{
	var serviceProvider = Host as IServiceProvider;
	var dte = serviceProvider?.GetService(typeof(DTE)) as DTE;
	var activeProjects = dte?.ActiveSolutionProjects as object[];

	if(activeProjects == null || activeProjects.Length == 0)
		return "";

	var project = activeProjects[0] as Project;
	var namespaceProperty = project?.Properties.Item("DefaultNamespace");

	if(namespaceProperty == null)
		return "";

	return (string)namespaceProperty.Value;
}
```

So now when I known all that I need, I can put it together to have a class that I need:

```csharp
namespace <#=projectNamespace#>.Views
{
<# foreach(var view in views){#>
  public partial class <#= view#> : System.ComponentModel.INotifyPropertyChanged
  {
	public <#= view#>()
	{
		InitializeComponent();
		DataContextChanged += OnDataContextChanged;
	}
	public <#=viewModelsNamespace#>.<#= view#>ViewModel <#=viewModelPropertyName#>
	{
		get
		{
			return DataContext as <#=viewModelsNamespace#>.<#= view#>ViewModel;
		}
	}
	private void OnDataContextChanged(Windows.UI.Xaml.FrameworkElement sender, Windows.UI.Xaml.DataContextChangedEventArgs args)
	{
		PropertyChanged?.Invoke(this, new System.ComponentModel.PropertyChangedEventArgs(nameof(<#=viewModelPropertyName#>)));
	}
	public event System.ComponentModel.PropertyChangedEventHandler PropertyChanged;
	}
<#}#>
}
``` 

Where `viewModelsNamespace` and `viewModelPropertyName` are:
```csharp
var viewModelsNamespace = "ViewModels"; // Namespace for view models. elative to the project namespace.
var viewModelPropertyName = "VM"; // Name of the ViewModel property hat will apear in the view class.
```
The resulting output of the template looks like this:

```csharp
namespace DummyProject.Universal.Views
{
	public partial class MainPage : System.ComponentModel.INotifyPropertyChanged
	{
		public MainPage()
		{
			InitializeComponent();
			DataContextChanged += OnDataContextChanged;
		}

		public ViewModels.MainPageViewModel VM
		{
			get
			{
				return DataContext as ViewModels.MainPageViewModel;
			}
		}

		private void OnDataContextChanged(Windows.UI.Xaml.FrameworkElement sender, Windows.UI.Xaml.DataContextChangedEventArgs args)
		{
			PropertyChanged?.Invoke(this, new System.ComponentModel.PropertyChangedEventArgs(nameof(VM)));
		}

		public event System.ComponentModel.PropertyChangedEventHandler PropertyChanged;
	}
}
```

And finally in my view I can bind like this:
```xml
<Button Command="{x:Bind VM.Save}" />
```

The full source code of the template available on the [UWP.T4.TemplatesKit][4] GitHub repository. All you need to do to use it, is just copy it over to your project.

[1]: https://docs.microsoft.com/en-us/windows/uwp/xaml-platform/x-bind-markup-extension
[2]:https://docs.microsoft.com/en-us/windows/uwp/xaml-platform/x-bind-markup-extension#property-path
[3]:https://en.wikipedia.org/wiki/Text_Template_Transformation_Toolkit
[4]: https://github.com/BerserkerDotNet/UWP.T4.TemplatesKit/blob/master/Templates/ConcreateViewModelPage.tt