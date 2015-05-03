---
layout: post
category: notes
tags: [azure,msbuild]
---

## Problem
 - Azure Cloud Service only creates `*.cspkg` package file in its target `Publish`
 - Your solution has multiple projects including Azure Cloud Service and Windows Desktop applications that may also have `Publish` targets
 - Therefore you cannot simply specify target `Publish` in msbuild to build your solution
 
## First try of changing default target in cloud service project
 - Modify the `.ccproj` file's default target from `Build` to `MyBuild`
 - Then add following targets (the last target might be optional but if you found the `app.publish` folder is not created in output directoy then try to add it)
 
        <Target Name="MyBuild" DependsOnTargets="Build" Condition="'$(PublishOnBuild)'!='true'"></Target>
        <Target Name="MyBuild" DependsOnTargets="Publish" Condition="'$(PublishOnBuild)'=='true'"></Target>
        <Target Name="CopyPackageToDropLocation" AfterTargets="CorePublish" DependsOnTargets="CorePublish" Condition="'$(PublishOnBuild)'!=''">
          <CreateItem Include="$(PublishDir)\*.cscfg">
              <Output TaskParameter="Include" ItemName="ConfigFiles" />
          </CreateItem>
        <Message Text="Copying package and config to drop folder." />
        <Copy SourceFiles="$(PublishDir)$(Name).cspkg" DestinationFiles="$(OutDir)\app.publish\$(Name).cspkg" />
        <Copy SourceFiles="@(ConfigFiles)" DestinationFolder="$(OutDir)\app.publish" />
        </Target>
 - Now if you pass `/p:PublishOnBuild=true` to msbuild command line the cloud service project will be published.
 
 However, we later found that with this you are no longer able to debug the cloud service in local emulator, it reports certain error about not able to find directory `csx\debug`; if you change the default target back to `Build` then this issue disappears.
 
 A few notes when I created this approach:
  - I didn't change the default target to `Publish` because I don't want to publish the package everytime I build the solution inside Visual Studio.
  - I also tried to depend on target `Publish` from `Build`, but it seems this will create circular dependency loop between them.
 - This is all because I didn't know how to specify different targets for different projects inside the same solution file.

## Improved approach of specifying different targets for different projects inside the same solution file
Based on information [here](https://msdn.microsoft.com/en-us/library/ms171486.aspx) I can specify different targets for different projects inside the same solution file. 
Therefore I can simply revert changes to the `ccproj` file and use msbuild command line to publish it while still building other projects using default target:

    msbuild mysolution.sln /p:VisualStudioVersion=12.0 /p:TargetProfile=Cloud /t:MyCloudService:Publish;MyWinformProject;SolutionFolder\My_Windows_Service
    
A few notes:
 - If you specify `MyWinformProject:Build` then it will report some error, but if you use `MyWinformProject:Rebuild` then it will work; anyway, just keep in mind for other projects you just provide the project name if the default target is already `Build`.
 - For projects inside solution folders you must also provide the solution folder names
 - For project names with `.` you must convert them to `_`
 - You must specify all projects that you need to build; this the only drawback of this approach, as you have to remember to modify your build script to add new projects, fortunately this should only be necessary for projects that you need to release, such as a new Windows Service project, or a Console project that you need to release to users.
 
