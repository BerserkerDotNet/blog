---
layout: post
title: Finding Visual Studio Tools
date: '2018-02-16T12:00:00.000-07:00'
author: Andrii Snihyr
tags:
- visual studio
- visual studio tools
- visual studio 2017
- vswhere
- scripts
- CI
- build
- build script
- MSBuild
- VSTest
modified_time: '2018-02-16T12:00:00.000-07:00'
---
Recently I was migrating a CI script from VS 2015 tools to VS 2017 tools. The script was calling to [vstest.console.exe](https://msdn.microsoft.com/en-us/library/jj155796.aspx?f=255&MSPPError=-2147217396) to run integration tests, somewhat like this:

```powershell
$vsTest = "$env:VS140COMNTOOLS\..\IDE\CommonExtensions\Microsoft\TestWindow\VSTest.console.exe"
& $vsTest "Tests.dll" "/Parallel" "/Settings:$runconfig" "/Logger:trx" 2>&1 | Write-Host
```
I've omitted the error handling code here for simplicity. The call itself is quite simple, it relies on the fact that when you install Visual Studio build tools on the CI machine it adds an environment variable `VS140COMNTOOLS` that points to the Visual Studio tools folder, usually it is `C:\Program Files (x86)\Microsoft Visual Studio 14.0\Common7\Tools\`.
<!--more-->
To my surprise for VS 2017 there is no such environment variable available. The reason is that VS 2017 supports multiple side-by-side installations, which apparently makes it difficult to create environment variables for every single version installed. 
I started to look for a way to detect where exactly Visual Studio is installed on the machine and came across a tool that is shipped with the VS installer and does just that. This tool is [vswhere](https://github.com/Microsoft/vswhere). It is basically a substitution for former environment variables and allows to detect instances of the Visual Studio installation on the machine. The usage is quite simple, if I run it in powershell like this:
```powershell
& "${Env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe"
```

It will give me a lot of information about my installation of the VS:
```
instanceId: f531a37c
installDate: 12/1/2017 1:53:36 PM
installationName: VisualStudio/15.5.4+27130.2024
installationPath: C:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise
installationVersion: 15.5.27130.2024
productId: Microsoft.VisualStudio.Product.Enterprise
productPath: C:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\Common7\IDE\devenv.exe
isPrerelease: 0
displayName: Visual Studio Enterprise 2017
... (and a bit more)
```
But besides getting all this info at once, I can get only certain properties that I'm interested in, for example by adding `-property installationPath` as an argument, I'll get this one line `C:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise`, which is exactly what I need for the script. The tool can do even more, if I need a specific component to be present in the installation I can add `-requires <componentid>` to the arguments list and get only those instances that have the component installed.
The list of available components can be found [here](https://docs.microsoft.com/en-us/visualstudio/install/workload-component-id-vs-enterprise).
For example there is an MSBuild component (Microsoft.Component.MSBuild) that you can use to query when search from build script as well as `Microsoft.VisualStudio.Component.TestTools.Core` component that indicates the presence of vstest.console.

So for my script I ended up with this:
```powershell
function global:Invoke-VSTest {
  $path = & "${Env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * -requires Microsoft.VisualStudio.Component.TestTools.Core -property installationPath
  $path = join-path $path 'Common7\IDE\CommonExtensions\Microsoft\TestWindow\vstest.console.exe'

  if (test-path $path) {
    & $path $args
  } else {
    throw "VSTest not found at '$path'"
  }
}

Invoke-VSTest "Tests.dll" "/Parallel" "/Settings:$runconfig" "/Logger:trx" 2>&1 | Write-Host
```
Here I ask [vswhere](https://github.com/Microsoft/vswhere) to find latest installation of VS that has a component `Microsoft.VisualStudio.Component.TestTools.Core` (Testing tools core features) and return me a path to it's installation folder.

As a side note, I would love to have an environment variable that will point to the [vswhere](https://github.com/Microsoft/vswhere) folder, because right now I need to do something like this `"${Env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe"` to find it.

Happy scripting!