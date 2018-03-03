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

For the installation of ANCM there is a PowerShell script written by [@shirhatti][3] that you can get from the GitHub repository:
```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/BerserkerDotNet/ANCM-ARMTemplate/master/install-ancm.ps1 -OutFile install-ancm.ps1
.\install-ancm.ps1
```
Note, that here I point to the forked version of the script in my own repository. The reason is that the script version that is in the blog, didn't work for me right away. There was an error extracting files from the package, a small issue that I fixed in this [PR][4]. When or if it will be merged to the main repository I will update this post.
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

If you've created your application from 2.0 template, as I did, you will have `UseContentRoot(Directory.GetCurrentDirectory())` entry in `Program.cs`. This should be removed as it will cause the application to fail. The reason is that when hosting inside `w3wp` GetCurrentDirectory will return `C:\windows\system32\inetsrv` while the app is in the `C:\inetpub\wwwroot\InProcTest` so ANCM will not find your app and will complain with the status code of 500. The 2.1 app is smart enough to figure out its own location. :)

`UseIISIntegration()` can also be removed as we are not going to proxy out app via IIS.

After this changes, run the app and it should load without issues.

Happy hosting!

[1]: https://blogs.msdn.microsoft.com/webdev/2018/02/28/asp-net-core-2-1-0-preview1-improvements-to-iis-hosting/
[2]: https://download.microsoft.com/download/A/B/1/AB1AA972-8F2F-43AD-9A81-72E9245CB0F5/dotnet-hosting-2.1.0-preview1-final-win.exe
[3]: https://github.com/shirhatti
[4]: https://github.com/shirhatti/ANCM-ARMTemplate/pull/5