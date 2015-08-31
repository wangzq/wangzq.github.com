---
layout: post
category: notes
tags: [msbuild]
---

## Problem
Today when I simply type `msbuild` trying to build the only solution file in current directory, it throws some strang error about not being able to find build configuration `Debug|BWS`. If I specify `/p:Configuration=Debug /p:Platform="Any CPU"` then it works.

## Root Causes
At first I didn't know that MSBuild will use all environment variables as properties by default, therefore I didn't check the environment variables, so I spent quite some time in debugging msbuild:

 * <http://stackoverflow.com/questions/14119498/building-a-solution-file-using-msbuild/14120447#1412044>
 * <http://blogs.msdn.com/b/visualstudio/archive/2010/07/06/debugging-msbuild-script-with-visual-studio.aspx>
 * <http://blogs.msdn.com/b/visualstudio/archive/2010/07/09/debugging-msbuild-script-with-visual-studio-2.aspx>
 * <http://blogs.msdn.com/b/msbuild/archive/2010/07/09/debugging-msbuild-script-with-visual-studio-3.aspx>

After setting the environment variable `msbuiltemitsolution` to `1` to first generate the `.metaproj` file, I then enabled msbuild debugger by using `reg add "HKLM\Software\Microsoft\MSBuild\12.0" /v DebuggerEnabled /d 'true' /f /reg:32`, then I used `msbuild /debug my.sln.metaproj` and I successfully step thru the file, and verified the `Platform` variable is indeed something incorrect.

After that I checked environment variables and found some other programs I installed a while back has setup a machine wide environment variable called `Platform` setting to `BMS`.


