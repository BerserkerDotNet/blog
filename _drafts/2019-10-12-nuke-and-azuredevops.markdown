---
layout: post
title: Using NUKE with Azure Pipelines to build and deploy NuGet packages
date: '2019-10-10T10:00:00.000-07:00'
author: Andrii Snihyr
tags:
- .Net
- C#
- NUKE
- AzureDevOps
- Azure Pipelines
- VSTS
- Build
- Release
- CI
- Pipeline
- NuGet
modified_time: '2019-10-10T10:00:00.000-07:00'
---
Recently, I found this very neat build tool called [NUKE][1], that I really liked, and I want to share my experience using it to build and deploy packages to NuGet.
<!--more-->

### Goals
Use [NUKE][1] to build .Net Core project and produce artifacts (NuGet packages). Then, using [Azure Pipelines][4] push NuGet packages to [NuGet.org][6].
The project in question is ([BlazorState.Redux][5]). It is a fairly simple structure. There are two C# projects, `BlazorState.Redux` and `BlazorState.Redux.Storage` to build and package, tests to run, and samples project to build.

### What is NUKE?

[NUKE][1](/njuːk/) is a cross-platform build automation system with C# DSL. Lets break it down.

First, name, I'll simply leave this here:
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">new-make :)</p>&mdash; 👻 (@matkoch87) <a href="https://twitter.com/matkoch87/status/1182733102440992772?ref_src=twsrc%5Etfw">October 11, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Second, cross-platform, it is build with .NET core, so it runs on every platform, you can do your builds on Raspberry Pi if you want to.

Third, C# DSL, this is the best part. All the build stuff can be done in C#, using easy to understand fluent syntax. Moreover, it can be debugged. No more frustration over why this XML does not do what I want. 

If you're interested in learning more about [NUKE][1], I encourage you to go see this [video][2] from DotNext conference. Also, there is a good [documentation][3] available.

### What is Azure Pipelines?

[Azure Pipelines][4] is a cloud service that you can use to automatically build and test your code project and make it available to other users. It works with just about any language or project type.

[Azure Pipelines][4] combines continuous integration (CI) and continuous delivery (CD) to constantly and consistently test and build your code and ship it to any target.

### Setting up build project
Setting up build project with [NUKE][1] is extremely easy. Here are the steps:
1. Install [NUKE][1] global tool by running `dotnet tool install Nuke.GlobalTool --global`
2. In the repository root, run `nuke :setup` and follow the on-screen instructions.
![NUKE setup]({{ "images/nuke-and-azuredevops/setup.png" | relative_url }} "Nuke setup")

That is it. A new `_build` project should be added to the solution. Depending on what you answered to the question "Do you need help getting started with a basic build?" [NUKE][1] will generate either empty build project or it will generated basic build targets, such as "Clean", "Restore", "Compile".
Run `nuke` command in the root to build the project. If no parameters are passed, [NUKE][1] will invoke default target specified in the build project.

### Defining targets
From the setup we already have basic targets defined ("Clean", "Restore", "Compile"), what's left is to define "Test" and "Package" targets.
[NUKE][1] has an [extension][8] for various IDEs that adds code snippets to help with target definition. If installed, `ntarget` snippet is available to define target skeleton.
Lets start with a "Test" target:
<p>
<script src="https://gist.github.com/BerserkerDotNet/27933681256d27a27b273ffc9e1a7377.js"></script>
    <noscript>
        <pre>
Target Test => _ => _
    .DependsOn(Compile)
    .Executes(() =>
    {
        DotNetTest(s => s
            .SetProjectFile(Solution.GetProject("BlazorState.Redux.Tests"))
            .SetConfiguration(Configuration)
            .SetLogger("trx")
            .SetResultsDirectory(ArtifactsDirectory / "TestResults")
            .EnableNoBuild()
            .EnableNoRestore());
    });
        </pre>
    </noscript>
</p>

The snippet above defines a target called "Test", this target depends on "Compile" target. This means that "Compile" will always be executed before executing "Test".
In the target's body, ["dotnet test"][10] command is invoked using [NUKE's][9] fluent API. To be able to publish test results to AzureDevOps, `SetLogger` is set to "trx" and results will be copied to `artifacts\TestResults` directory.
Since our target depends on "Compile" and transitively on "Restore", `EnableNoBuild` and `EnableNoRestore` options are passed to make sure we don't restore or build twice.

"Package" target looks pretty much the same:
<p>
<script src="https://gist.github.com/BerserkerDotNet/ef4a59da41ff982d190b766087070b60.js"></script>
<noscript>
<pre>
Target Package => _ => _
    .DependsOn(Test)
    .Executes(() =>
    {
        DotNetPack(s => s
            .SetProject(Solution)
            .SetConfiguration(Configuration)
            .EnableNoBuild()
            .EnableNoRestore()
            .EnableIncludeSymbols()
            .EnableIncludeSource()
            .SetOutputDirectory(ArtifactsDirectory));
    });
</pre>
</noscript>
</p>

Here ["dotnet pack"][11] command is invoked with options to `EnableIncludeSymbols` and `EnableIncludeSource` that will generate `.symbols.nupkg` and allow to debug this NuGet package.
The output directory is set to the artifacts folder in the root of the repository. This information will be useful later when setting up AzureDevOps release pipeline.

Finally, when we done defining targets, lets update default target from "Compile" to "Package". In `Build.cs` change `Main` to invoke "Package". Like this: `public static int Main () => Execute<Build>(x => x.Package);`

### Configuring build pipeline
In [Azure Pipelines][4] build pipeline is defined in the YAML file.
Here is the right setup to author YAML files:

<img src="https://i.redd.it/0lg04ovga0m11.jpg" alt="Authoring YAML" width="350" />

Fortunately, [NUKE][1] reduces number of lines we need to write to very minimum.
First, standard pipeline header that defines the trigger branch and agents pool.

<p>
<script src="https://gist.github.com/BerserkerDotNet/e03f43a9d543e1e97f05c6623dd81f59.js"></script>
<noscript>
    <pre>
trigger:
- master

pool:
  vmImage: 'ubuntu-latest'
    </pre>
</noscript>
</p>

Interesting thing to note here is that the agent pool is set to `ubuntu-latest`. we're cross platform, why not to put that in use :)

Next, is the build command itself:
<p>
<script src="https://gist.github.com/BerserkerDotNet/cf6e6d729f930c243012cd2f8d93bbcd.js"></script>
<noscript>
    <pre>
- task: PowerShell@2
  displayName: 'Build & Package'
  inputs:
    filePath: 'build.ps1'
    </pre>
</noscript>
</p>

This is it.When no parameters passed [NUKE][1] will invoke default target, which is "Package" (see "Defining targets" section).Since "Package" depends on "Test" and "Test" depends on "Compile", [NUKE][1] will invoke all the targets in the chain, starting from "Clean".

Finally, we need to publish test results and artifacts:
<p>
<script src="https://gist.github.com/BerserkerDotNet/55a9d42bbf273726daabe168beb4a4df.js"></script>
<noscript>
    <pre>
- task: PublishTestResults@2
  inputs:
    testResultsFormat: 'VSTest'
    testResultsFiles: '*.trx'
    searchFolder: '$(System.DefaultWorkingDirectory)/artifacts/TestResults'
    mergeTestResults: true
- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifacts'
  inputs:
    PathtoPublish: 'artifacts'
    ArtifactName: 'drop'
    publishLocation: 'Container'
    </pre>
</noscript>
</p>

Here we publish both test results and NuGet packages from "artifacts" folder.

Here is the complete pipeline:
<p>
<script src="https://gist.github.com/BerserkerDotNet/424f6f4f4f5036aed789148943d272ab.js"></script>
<noscript>
    <pre>
trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: PowerShell@2
  displayName: 'Build & Package'
  inputs:
    filePath: 'build.ps1'
- task: PublishTestResults@2
  inputs:
    testResultsFormat: 'VSTest'
    testResultsFiles: '*.trx'
    searchFolder: '$(System.DefaultWorkingDirectory)/artifacts/TestResults'
    mergeTestResults: true
- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifacts'
  inputs:
    PathtoPublish: 'artifacts'
    ArtifactName: 'drop'
    publishLocation: 'Container'
    </pre>
</noscript>
</p>

Save this as a `azure-pipelines.yml` file in the root of the repository.

### Configuring release pipeline

### Conclusion

[1]: https://nuke.build/
[2]: https://www.youtube.com/watch?v=7gEqxzD6hbs
[3]: https://nuke.build/docs/getting-started/philosophy.html
[4]: https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started/what-is-azure-pipelines?view=azure-devops
[5]: https://github.com/BerserkerDotNet/BlazorState.Redux
[6]: https://www.nuget.org/
[7]: https://www.appveyor.com/
[8]: https://nuke.build/docs/running-builds/from-ides.html
[9]: https://nuke.build/api/Nuke.Common/Nuke.Common.Tools.DotNet.DotNetTasks.html#Nuke_Common_Tools_DotNet_DotNetTasks_DotNetTest_Nuke_Common_Tooling_Configure_Nuke_Common_Tools_DotNet_DotNetTestSettings__
[10]: https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-test?tabs=netcore21