---
layout: post
title: Script Capture Tag Helper
date: '2018-03-24T10:00:00.000-07:00'
author: Andrii Snihyr
tags:
- AspNet
- AspNet Core
- Tag Helper
- TagHelper
- tag
- helper
- razor
- sections
- MVC
modified_time: '2018-03-24T10:00:00.000-07:00'
---
Razor [sections][3]{:target="_blank"} allows us to write a block of code in one view and render it later in another. The typical example would be the `scripts` section that takes a script block from the page and renders it in the layout after all common scripts. This works fine until you try to use `sections` from the partial view or from the display template.
<!--more-->

#### Why even bother 
I wanted to add charts to the [Ether][2]{:target="_blank"} reporting tool that I wrote to track KPI metrics for one of my projects at work.
The tool generates a few different report types, each of which is a separate .NET class, but they all rendered by one Razor Page. To avoid me having to write switch/case to distinguish between different report types, I use [Display Template][7]{:target="_blank"} feature of Razor to do this for me. Naturally, different report types will require different charts to be displayed, some might not have them at all, so, I quickly dismissed the idea of including all necessary scripts into a base page. Those scripts need to be in specific display template, but here is a problem, [sections][3]{:target="_blank"} do not work inside partial view or display template.

#### Looking for the solution
I quickly remembered that I came across somewhat similar problem back in 2015 when working with [Sitecore CMS][8]{:target="_blank"}. There too, you cannot use sections as Sitecore does its custom rendering that completely overrides standard MVC rendering. The solution was found by my colleague at the time, [Derek Hunziker][5]{:target="_blank"}, he proposed to write an [HtmlHelper][6]{:target="_blank"} that will capture the markup on the page and render it on the layout. While this works fine in Sitecore or in regular AspNet MVC application, it would not work in AspNet Core application as it relays on `System.Web` features, like `ViewDataContainer` and `WebViewPage.OutputStack` that are no longer available and hardly would be in the "spirit" of AspNet Core application. While not having `WebViewPage.OutputStack` exposed as an API, AspNet Core has one feature that can actually capture markup, [Tag Helpers][1]{:target="_blank"}.

#### Will tag helpers help?
[Tag Helpers][1]{:target="_blank"} is an adequate addition to the Razor markup. In essence, they meant to replace HtmlHelper class with the more robust way of combining C# and Html. Here is a simple example of a tag helper:
<script src="https://gist.github.com/BerserkerDotNet/7daa1b1c950c837f90e848558c077679.js"></script>
The tag helper part is in `asp-page-handler` attribute that is added to the `form` html tag. This attribute is a hint to the Razor engine that this tag needs additional processing. The final result will look like this:
<script src="https://gist.github.com/BerserkerDotNet/a435a76c51ae5a1cb5437aa34c3d20d8.js"></script>
Tag helpers are an ideal solution in this case, as they are easy to use, elegant and certainly AspNet Core friendly. In fact, I think, it is hard to find a solution that would be easier to use, since it is literally just adding an attribute to an html tag.
To solve this, there need to be two tag helpers, one that will capture the script block and the second one that will render it.

#### Script Capture Tag Helper
Authoring custom tag helpers is remarkably easy, it can be done in three steps. The first step is to create a class that inherits base `TagHelper` class. Second, is to mark this class with `HtmlTargetElement` attribute specifying what html element to target, in this case, it is `script`, and which attributes to look for. This is needed for Razor to know when to apply tag helper. The third step is to override `Process` or `ProcessAsync` methods that will hold a tag helper logic.
<script src="https://gist.github.com/BerserkerDotNet/41e92b53874caa11fd1dba7b593172aa.js"></script>
Tag helper can bind attribute value to a C# property, this is done by marking the property with `HtmlAttributeName` and specifying attribute name. In the example above there is "Capture" property being bind to the corresponding html attribute.
If `ViewContext` instance needs to be accessed by the tag helper, it is done by adding a property to a tag helper class and decorating it with "ViewContext" attribute. this is specially injected as it needs to represent the current instance of the "ViewContext", other dependencies can be injected through normal dependency injection.
The content of the tag can be accessed through the `output.GetChildContentAsync` method, it returns an instance of `TagHelperContent`, to get a string representation of a content, `GetContent` method of `TagHelperContent` needs to be called. Since the content of the script block is captured for future rendering, it cannot be left as is, because it will be rendered in the middle of a page. To prevent this `output.SuppressOutput` method needs to be called.

<script src="https://gist.github.com/BerserkerDotNet/69806b9d25c1371e4021be23da880e79.js"></script>
Here I store captured content in `HttpContext.Items` dictionary using `capture` attribute value as a key.
This is simplified implementation, complete implementation can be found at [ScriptCaptureTagHelper.cs][9]

#### Script Render Tag Helper
Similar to capture tag helper, render tag helper is implemented in its own class that inherits from base `TagHelper` class, but instead of storing current content it reads the value from `HttpContext.Items` dictionary, using the same key value and calls `SetHtmlContent` on the output object:
<script src="https://gist.github.com/BerserkerDotNet/0e0f0ea73b156e63640f0f660da7fd52.js"></script>
This is simplified implementation, complete implementation can be found at [ScriptRenderTagHelper.cs][10]

#### Using it
`@addTagHelper *, ScriptCaptureTagHelper` needs to be added to the `_ViewImports.cshtml` file to register tag helpers. `ScriptCaptureTagHelper` is the name of the assembly where tag helpers are located.
To actually use tag helpers add a `capture` tag on the script block in the display template or in the partial view.
<script src="https://gist.github.com/BerserkerDotNet/7e2503e1080262b7c6f91c800664a673.js"></script>
In the view where the script needs to be rendered, add an empty `script` tag with the `render` attribute, specifying the same id as for `capture` attribute.
<script src="https://gist.github.com/BerserkerDotNet/6880d7bc609c0f8d7e53df46ee84908a.js"></script>

#### Where can I get it
The complete implementation is available on the [ScriptCaptureTagHelper GitHub][11] page. Also, there is a [ScriptCaptureTagHelper][12] NuGet package available for download.
Complete implementation has more features then just capturing single script block. IT allows capture of multiple blocks under ther same ID with ordering and preserves attributes, which makes possible to capture script reference tag.

Happy coding :)

[1]: https://docs.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/intro
[2]: https://github.com/BerserkerDotNet/Ether
[3]: https://docs.microsoft.com/en-us/aspnet/core/mvc/views/layout#sections
[4]: https://stackoverflow.com/search?q=Using+Sections+for+Partial+View%3F
[5]: https://twitter.com/dthunziker
[6]: http://www.layerworks.com/blog/2015/7/17/sitecore-javascript-renderings
[7]: https://exceptionnotfound.net/asp-net-mvc-demystified-display-and-editor-templates/
[8]: https://www.sitecore.com/
[9]: https://github.com/BerserkerDotNet/ScriptCaptureTagHelper/blob/master/ScriptCaptureTagHelper/ScriptCaptureTagHelper.cs
[10]: https://github.com/BerserkerDotNet/ScriptCaptureTagHelper/blob/master/ScriptCaptureTagHelper/ScriptRenderTagHelper.cs
[11]: https://github.com/BerserkerDotNet/ScriptCaptureTagHelper/blob/master/ScriptCaptureTagHelper/
[12]: https://www.nuget.org/packages/ScriptCaptureTagHelper/