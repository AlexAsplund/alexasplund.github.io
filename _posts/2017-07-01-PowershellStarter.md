---
layout: post
title: Powershell starter - Run powershell scripts as a Windows service easily
categories: project
project: PowershellStarter
github: https://github.com/AlexAsplund/Powershell starter
projectdescription: A service made in C# that launches powershell.exe -file <path>.
tags: [Powershell, C#, Windows, Services,]
---

I wanted something that would run powershell scripts as a service easily and I never was a fan of running it with taskscheduler.
This windows service reads the path of the script and it's arguments from the app.conf file so you can run any powershell script that you want (i think).
It also redirects output and errors to eventlog and terminates itself if the script would stop running.
I have tested it on Windows Server 2012 and Windows 10.