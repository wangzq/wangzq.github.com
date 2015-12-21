---
layout: post
category: notes
tags: [azure,msbuild]
---

## Problem
Today I was working on creating a tfs build definition for a TFS server 2013 on-premise project using Azure SDK 2.6, which has a cloud service project that is using following configuration to create an additional virtual application in the web role's root web site (from its `ServiceDefinition.csdef`):

      <Sites>
        <Site name="Web">
          <VirtualApplication name="foo" physicalDirectory="..\..\..\foo">
          </VirtualApplication>

When using the approach from [here](/notes/2015/05/03/How-to-publish-azure-cloud-service-package-in-continuous-build/) to create the package for the Cloud Service, I got following error:

      ServiceDefinition.csdef: Cannot find the physical directory 'D:\foo' for virtual path Web/foo/

This can also be reproduced locally (to simulate the msbuild command-line parameters on the TFS build server):

      msbuild .\myservice.sln /p:TargetProfile=Dev /p:VisualStudioVersion=12.0 /t:MyCloudService:Publish /p:OutDir=c:\temp\output

NOTE: when you don't specify the `OutDir` property it actually works fine locally; however the TFS build server is always specifying this property to copy the output to a drop folder.

After spending a few hours searching for solutions, it turns out this is some known issue since Azure SDK 1.8:

From <http://blogs.msdn.com/b/davidhardin/archive/2013/01/18/building-and-packaging-virtual-applications-within-azure-projects.aspx>

        Prior to SDK 1.8 the csdef being used was located in a subfolder within the csx folder under the Azure project and was named ServiceDefinition.build.csdef. The folder name matched the configuration being built, normally either Debug or Release. It is important to note that this caused the physicalDirectory attribute in csdef to be with respect to a folder in the source tree while building locally in Visual Studio and also on the TFS build server. Additionally, the csdef file was not copied to the build output folder.

        Azure SDK 1.8 on the other hand copies the csdef file to the build output folder and changed the ResolveServiceDefinition target so that it looks for the file there. While doing a local Visual Studio build the output folder is a subfolder within the bin folder under the Azure project matching the configuration being built, typically Debug or Release. The important point for a local Visual Studio build is that this folder is an equal number of folders down in the source tree when compared to the previous csx subfolder so the SDK 1.8 change should be transparent to most developers.

        Unfortunately the build output folder on a TFS build server is very different. A step in the TFS build process needs to copy the build output to a drop folder for archival purposes. To facilitate this the TFS build templates uses two folder, one for the source and another for the build output. The obj and csx folders continue to get created within the source folders but instead of the build output going to the bin folder as is done by Visual Studio, TFS uses a build output folder named Binaries that is at the same, sibling folder level as the Source. The build output folder contents is also slightly different in that there is a _PublishedWebsites folder.

        …

        If you want to verify what is packaged, inside the .cspkg file there is a sitesroot folder which contains a folder for each application; 0 is the root site and 1 is the virtual application. 

In summary, it is due to the order of packaging and copying bits to drop folder, which is different in TFS build and running locally. However, while the approach mentioned above does help solve the error when I run the msbuild command locally with `OutDir` property specified, it is not copying the cspkg file at the path specified by the `OutDir` property. Later I used the information found in [this](http://blogs.blackmarble.co.uk/blogs/rfennell/post/2014/07/14/Building-Azure-Cloud-Applications-on-TFS.aspx) to fix this: it mentioned a few options, I found the last one which is adding a target to execute after Publish is most convenient (since I don't want to mess with the build process template which is a XAML file).

To summarize the changes:

 1. Add following to the end of the cloud service project:

      <!-- Virtual applications to publish -->
      <ItemGroup>
        <!-- Manually add PublishToFileSystem target to each virtual application .csproj file -->
        <!-- For each virtual application add a VirtualApp item to this ItemGroup:
        
        <VirtualApp Include="Relative path to csproj file">
          <PhysicalDirectory>Must match value in ServiceDefinition.csdef</PhysicalDirectory>
        </VirtualApp>
        -->
        <VirtualApp Include="..\foo\foo.csproj">
          <PhysicalDirectory>_PublishedWebsites\foo</PhysicalDirectory>
        </VirtualApp>
      </ItemGroup>
      <!-- Executes before CSPack so that virtual applications are found -->
      <Target
        Name="PublishVirtualApplicationsBeforeCSPack"
        BeforeTargets="CorePublish;CsPackForDevFabric"
        Condition="'$(PackageForComputeEmulator)' == 'true' Or '$(IsExecutingPublishTarget)' == 'true' ">
        <Message Text="Start - PublishVirtualApplicationsBeforeCSPack" />
        <PropertyGroup Condition=" '$(PublishDestinationPath)'=='' and '$(BuildingInsideVisualStudio)'=='true' ">
          <!-- When Visual Studio build -->
          <PublishDestinationPath>$(ProjectDir)$(OutDir)</PublishDestinationPath>
        </PropertyGroup>
        <PropertyGroup Condition=" '$(PublishDestinationPath)'=='' ">
          <!-- When TFS build -->
          <PublishDestinationPath>$(OutDir)</PublishDestinationPath>
        </PropertyGroup>
        <Message Text="Publishing '%(VirtualApp.Identity)' to '$(PublishDestinationPath)%(VirtualApp.PhysicalDirectory)'" />
        <MSBuild
          Projects="%(VirtualApp.Identity)"
          ContinueOnError="false"
          Targets="PublishToFileSystem"
          Properties="Configuration=$(Configuration);PublishDestination=$(PublishDestinationPath)%(VirtualApp.PhysicalDirectory);AutoParameterizationWebConfigConnectionStrings=False" />
        <!-- Delete files excluded from packaging; take care not to delete xml files unless there is a matching dll -->
        <CreateItem Include="$(PublishDestinationPath)%(VirtualApp.PhysicalDirectory)\**\*.dll">
          <Output ItemName="DllFiles" TaskParameter="Include" />
        </CreateItem>
        <ItemGroup>
          <FilesToDelete Include="@(DllFiles -> '%(RootDir)%(Directory)%(Filename).pdb')" />
          <FilesToDelete Include="@(DllFiles -> '%(RootDir)%(Directory)%(Filename).xml')" />
        </ItemGroup>
        <Message Text="Files excluded from packaging '@(FilesToDelete)'" />
        <Delete Files="@(FilesToDelete)" />
        <Message Text="End - PublishVirtualApplicationsBeforeCSPack" />
      </Target>

      <Target Name="CustomPostPublishActions" AfterTargets="AfterPublish" Condition="'$(TF_BUILD_DROPLOCATION)' != ''">
      <Exec Command="echo Post-PUBLISH event: Copying published files to: $(TF_BUILD_DROPLOCATION)" />
      <Exec Command="xcopy &quot;$(ProjectDir)bin\$(ConfigurationName)\app.publish&quot; &quot;$(TF_BUILD_DROPLOCATION)\app.publish\&quot; /y " />
      </Target>

 1. Add following to the web project foo.csproj (add after importing Web.Applications.targets):

      <!-- Performs publishing prior to the Azure project packaging -->
      <Target Name="PublishToFileSystem" DependsOnTargets="PipelinePreDeployCopyAllFilesToOneFolder">
        <Error Condition="'$(PublishDestination)'==''" Text="The PublishDestination property is not set." />
        <MakeDir Condition="!Exists($(PublishDestination))" Directories="$(PublishDestination)" />
        <ItemGroup>
          <PublishFiles Include="$(_PackageTempDir)\**\*.*" />
        </ItemGroup>
        <Copy SourceFiles="@(PublishFiles)" DestinationFiles="@(PublishFiles->'$(PublishDestination)\%(RecursiveDir)%(Filename)%(Extension)')" SkipUnchangedFiles="True" />
      </Target>

 1. Change the reference to the virtual application's location in `ServiceDefinition.csdef`:

      <VirtualApplication name="foo" physicalDirectory="_PublishedWebSites\foo">
 
Reminder: in the TFS build definition, use following msbuild arguments:

      /p:VisualStudioVersion=12.0 /p:TargetProfile=Dev /p:MyCloudService:Publish

