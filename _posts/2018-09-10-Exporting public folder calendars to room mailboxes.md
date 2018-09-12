---
layout: post
title: Exporting public folder calendars to room mailboxes
---

I got a task to come up with a method to import room calendars in public folders to room resource mailboxes instead.
A lot of the methods I found online involved a lot of manual hand cranking, and that't not viable for 100s of calendars.

The outlook com-object didn't get me far so I started digging around the EWS API, and an hour or two later - it was finished.

## How to use it

First, create the resource mailbox that you want to migrate to.
After the resource is created, run the script:  

{% highlight powershell %}

# Get credentials 
$ExchangeCredential = Get-Credential -Message "Enter exchange user credential"
$Office365AdminCredential = Get-Credential -Message "Enter Office 365 admin credential"

# Create a splat because you don't have a 32:9 screen
$Splat = @{

    ExchangeCredential = $ExchangeCredential
    Office365AdminCredential= $Office365AdminCredential
    PublicFolderPath = 'HR\Rooms\Meeting room 1' 
    RoomMailAddress = meetingroom1@contoso.com 
    ChangePermissions = $True    

}

# Run script
.\Import-CalendarFromPublicFolder.ps1 @Splat
 


{% endhighlight %}

## The script

* [Github link to script](https://gist.github.com/AlexAsplund/93285b6a3c62be559eeec3abec4f3c4b)
* You need to install the [EWS API](https://www.microsoft.com/en-us/download/details.aspx?id=42951) before running this script.
* You need to have permission to change permissions for the '-RoomMailAddress' in Office 365.
* The account used for '-ExchangeCredential' needs to be able to read/write to the public folder calendar.




****
----