---
layout: post
category: notes
tags: [azure]
---

# How to install multiple versions of Azure SDK

If you use the GUI version of Web Platform Installer to install Azure SDK you will find it always default to latest version.

To install multiple versions of Azure SDK, you can use webpicmd.exe (usually found at `C:\Program Files\Microsoft\Web Platform Installer\WebpiCmd.exe` or already in your PATH):

    webpicmd /install /products:VS2013AzurePack.2.2,VS2013AzurePack.2.5 /AcceptEula


