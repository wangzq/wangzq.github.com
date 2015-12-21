---
layout: post
category: notes
tags: [azure]
---

If you use the GUI version of Web Platform Installer to install Azure SDK you will find it always default to latest version.

To install multiple versions of Azure SDK, you can use webpicmd.exe (usually found at `C:\Program Files\Microsoft\Web Platform Installer\WebpiCmd.exe` or already in your PATH):

    webpicmd /install "/products:VS2013AzurePack.2.2,VS2013AzurePack.2.5" /AcceptEula

Update on 2015.12.21: it seems webpi package feed will only keep the last 2 versions of Azure SDK for Visual Studio, as of today, latest version is 2.8 so it only keeps 2.7 and 2.8; to install 2.6 you will have to use manual steps as described in the [microsoft download page](https://www.microsoft.com/en-us/download/details.aspx?id=46892); however, it seems if you just need to open the cloud service projects without errors, following are 3 minimum steps you need to do:

 1. Install the latest Azure SDK as usual
 1. Install the MicrosoftAzureLibsForNet-x64.msi
 1. Install MicrosoftAzureTools.VS.120 (for VS2013) or MicrosoftAzureTools.VS.140 (for VS2015)



