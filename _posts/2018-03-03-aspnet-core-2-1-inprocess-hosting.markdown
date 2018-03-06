---
layout: post
title: Notes from changing AspNet Core 2.1-preview1 app to IIS In-Process hosting
date: '2018-03-03T12:00:00.000-07:00'
author: Andrii Snihyr
tags:
- AspNet
- AspNet Core
- 2.1-preview1
- IIS
- In-Process
- hosting
modified_time: '2018-03-03T12:00:00.000-07:00'
---
The AspNet Core 2.1-preview1 is out and it brings a ton of interesting improvements. One of which is that now you are able to host your AspNet app inside the IIS process. This, according to [AspNet team][1] will increase the request throughput by around 4.4 times. I did it for my app and faced a few issues that I though is worth describing here along with a process that I followed.
<!--more-->
["Improvements to IIS hosting"][1] blog post from AspNet team lists steps needed to change your app hosting model. To my surprise there is only three of them:
1. Install [Windows Server Hosting bundle][2]
2. Install AspNet Core Module (ANCM) module for ISS
3. Set `AspNetCoreModuleHostingModel` to `inprocess` in your project file.

Installation of Windows Server Hosting bundle is pretty straight forward and quite fast, I didn't encounter any issues with it. One note here is that it is recommended to do `iisreset` after this step.

For the installation of ANCM there is a PowerShell script written by [@shirhatti][3] that you can get from the [GitHub repository][5].
Here is how you execute the script:
```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/shirhatti/ANCM-ARMTemplate/master/install-ancm.ps1 -OutFile install-ancm.ps1
.\install-ancm.ps1
```
**UPD:**
The [PR][4] with a fix for the script was merged, so I updated the URL to take the version of the script from official repository.

If the script executed successfully there should be an `AspNet Core Module` in the list of modules in IIS.

![ANCM]({{ "images/aspnet-core-2-1-inprocess-hosting/modules-list.png" | relative_url }} "ANCM in the ISS modules list")

Since this is a `Native` module the app pool can stay with "No managed code" option for CLR version.

Now lets get to what needs to change in the application code itself. First thing is to set `AspNetCoreModuleHostingModel` to `inprocess` in the project file. This is done by adding a `PropertyGroup` to the project file:
```xml
  <PropertyGroup>
    <AspNetCoreModuleHostingModel>inprocess</AspNetCoreModuleHostingModel>
  </PropertyGroup>
```

This will not change the behavior of the app when running locally, but it will instruct MSBuild to add `hostingModel="inprocess"` property to the `ANCM` entry in the `web.config`. The resulting `web.config` should look like this:
```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
    </handlers>
    <aspNetCore processPath="dotnet" arguments=".\InProcTest.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" hostingModel="inprocess" />
  </system.webServer>
</configuration>
```

If you've created your application from 2.0 template, as I did, you will have `UseContentRoot(Directory.GetCurrentDirectory())` entry in `Program.cs`. This should be removed as it will cause the application to fail. The reason is that when hosting inside `w3wp` GetCurrentDirectory will return `C:\windows\system32\inetsrv` while the app is in the `C:\inetpub\wwwroot\InProcTest` so ANCM will not find your app and will complain with the status code of 500. The 2.1 app is smart enough to figure out its own location anyway. :)

**UPD:**
Before, I mentioned that `UseIISIntegration()` can be removed from the `Program.cs` as it was not needed, this is not entirely correct.
[@jkotalik][6] pointed out that `UseIISIntegration()` is still necessary for `InProcess` hosting as it will register IIS specific [services and options][8] and will set the [content root path][9] correctly. The good news it that [WebHost.CreateDefaultBuilder][7] will add `UseIISIntegration()` by default, so no need to add it in `Program.cs`.

After this changes, run the app and it should load without issues.

Side note, with this model, you can run only one app per IIS process. This is a decision that AspNet team made and also currently a limitation from .Net Core as it does not have support for multiple AppDomains.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">No, one app per process</p>&mdash; Damian Edwards (@DamianEdwards) <a href="https://twitter.com/DamianEdwards/status/969322206596493312?ref_src=twsrc%5Etfw">March 1, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Kinda, but even if it did, I&#39;m not sure we&#39;d do it. The gymnastics System.Web pulls to do that aren&#39;t really seen as idiomatic guidance anymore, i.e. each app should be in its own process.</p>&mdash; Damian Edwards (@DamianEdwards) <a href="https://twitter.com/DamianEdwards/status/969326928615190528?ref_src=twsrc%5Etfw">March 1, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Happy hosting!

[1]: https://blogs.msdn.microsoft.com/webdev/2018/02/28/asp-net-core-2-1-0-preview1-improvements-to-iis-hosting/
[2]: https://download.microsoft.com/download/A/B/1/AB1AA972-8F2F-43AD-9A81-72E9245CB0F5/dotnet-hosting-2.1.0-preview1-final-win.exe
[3]: https://github.com/shirhatti
[4]: https://github.com/shirhatti/ANCM-ARMTemplate/pull/5
[5]: https://github.com/shirhatti/ANCM-ARMTemplate
[6]: https://github.com/jkotalik
[7]: https://github.com/aspnet/MetaPackages/blob/7511a4da7f1d1d9651d19801aadea77f557e0b11/src/Microsoft.AspNetCore/WebHost.cs#L185
[8]: https://github.com/aspnet/IISIntegration/blob/4e8a9d24939ae41c862dcabc5c07330fb4991727/src/Microsoft.AspNetCore.Server.IISIntegration/WebHostBuilderIISExtensions.cs#L61-L69
[9]: https://github.com/aspnet/IISIntegration/blob/4e8a9d24939ae41c862dcabc5c07330fb4991727/src/Microsoft.AspNetCore.Server.IISIntegration/WebHostBuilderIISExtensions.cs#L58