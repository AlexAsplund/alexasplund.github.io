---
layout: post
title: Exporting public folder calendars to room mailboxes
---

I was tasked with comming up with a method to import room calendars in public folders to room resource mailboxes instead.

The outlook com-object didn't get me far so I started digging around the EWS API, and an hour or two later - it was finished.

## How to use it
```Powershell

$ExchangeCredential = Get-Credential -Message "Enter exchange user credential"
$Office365AdminCredential = Get-Credential -Message "Enter Office 365 admin credential"

.\Import-CalendarFromPublicFolder.ps1 -ExchangeCredential $ExchangeCredential -Office365AdminCredential $Office365AdminCredential -PublicFolderPath 'HR\Rooms\Meeting room 1' -RoomMailAddress meetingroom1@contoso.com -ChangePermissions

```
## The script

You will need to install the [EWS API](https://www.microsoft.com/en-us/download/details.aspx?id=42951) before running this script.

[Github link to script](https://gist.github.com/AlexAsplund/93285b6a3c62be559eeec3abec4f3c4b)


****
----