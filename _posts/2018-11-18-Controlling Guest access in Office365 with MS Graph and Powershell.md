---
layout: post
title: Controlling Guest access in Office365 with MS Graph and Powershell
tags: [Microsoft Graph, Powershell, Modules, MyAAD, Microsoft Teams, Unified Groups, Azure Active Directory]
---

You know when you read documentation about a task or problem you need to solve and you think 'Thats easy!'. But you find yourself scratching yourself in the head 6 hours later... Sounds like the average IT-task right?


This time the problem that needed solving was that our organization needed to start allowing guest access to a handfull of teams/unified groups.

I started of by allowing guest access globaly on the tenant and restricting guest access to sharepoint sites, allowing guests to teams etc but quickly found out that this isn't working as i thought it would.

[Github link to MyAAD module](https://github.com/AlexAsplund/MyAAD)

# The solution that didn't work

I thought that I could set a policy to deny guest access to all groups and apply specific Object settings to the groups that would need guest access.
I successfully tried this for enabling guest access on all groups in a test tenant and restricting it for a few with object settings, so this should be no problem right?

## The solution I thought would work

[This script is stolen from the documentation.](https://docs.microsoft.com/en-us/office365/admin/create-groups/manage-guest-access-in-groups?view=o365-worldwide)


-----------

```Powershell
# This denies guest access to all groups on the tenant
Install-Module AzureADPreview -Force -AllowClobber
Import-Module AzureADPreview -Force

# Connect to AzureAD
Connect-AzureAD

$template = Get-AzureADDirectorySettingTemplate | ? {$_.displayname -eq "group.unified"}
$settingsCopy = Get-AzureADDirectorySetting -Id $settingsObjectID
$settingsCopy["AllowGuestsToAccessGroups"] = "false"

Set-AzureADDirectorySetting -Id $settingsObjectID -DirectorySetting $settingsCopy

(Get-AzureADDirectorySetting -Id $settingsObjectID).Values

# I thought this would allow guest invite on a specific room

$template = Get-AzureADDirectorySettingTemplate | ? {$_.displayname -eq "group.unified.guest"}
$settingsCopy = $template.CreateDirectorySetting()
$settingsCopy["AllowToAddGuests"] = $True
New-AzureADObjectSetting -TargetType Groups -TargetObjectId $GroupId -DirectorySetting $settingsCopy
```

-----------


This will be easy! I only need to create like 2 functions or something for managing this! Or so i thought...

What happens is that is that on the Directory setting ```"AllowToAddGuests" = "false"``` takes preceedence (or disables on a tenant level) over the object settings on ALL groups. At least this is the conclusion i came to after waiting for 24 hours (in case of propagation delay) and troubleshooting for a few hours.


I made the assumption that it worked similarly to 'AllowToAddGuests=true' but of course not, nothing in IT is. Especially for a function as important as this one. This will need another happy little workaround.


# The workaround

The only solution I found was to do this the other way around. To set the directory template to ```AllowToAddGuests=true``` but the group object settings on the rooms I didn't want to share to ```AllowToAddGuests=false```. But for not loosing track of a unified group when you have almost 2000 of them, i also set a object specific setting on the rooms that I did want to share to ```AllowToAddGuests=true```


## Directory settings

First of, I needed to change the settings of the directory template to ```AllowToAddGuests=false```. I did this with the AzureADPreview Module but you can do this with pure Graph API if you want to.

-----------

```Powershell
# This allows guest access to all groups on the tenant
Install-Module AzureADPreview -Force -AllowClobber
Import-Module AzureADPreview -Force

# Connect to AzureAD
Connect-AzureAD
$template = Get-AzureADDirectorySettingTemplate | ? {$_.displayname -eq "group.unified"}
$settingsCopy = Get-AzureADDirectorySetting -Id $settingsObjectID
$settingsCopy["AllowGuestsToAccessGroups"] = "false"
```

-----------


## Group Object settings

An object setting is a setting that is applied from a template in AzureAD to a specific object. In this case we will apply Object Settings to all groups to specify if it allows guests to be invited or not.

As I mentioned, we could just set ```AllowToAddGuests=false``` on all the rooms except those we want to share. But if we set it on all rooms we can easily collect missed groups.

You need a script to run regularly for enforcing this (jenkins in my case)

## The MyAAD module 

I wrote a whole module for this one that you can find on my github: [https://github.com/AlexAsplund/MyAAD](https://github.com/AlexAsplund/MyAAD)
The goal of that module was that it needed to be as simple and reusable as possible and it should only rely on the built in capabilities of Powershell.

Available commands:
```
    Get-MyAADAccessToken
    Get-MyAADDirectorySetting
    Get-MyAADDirectorySettingTemplate
    Get-MyAADGroupSetting
    Get-MyAADUnifiedGroups
    New-MyAADDirectorySetting
    New-MyAADGroupSetting
    Remove-MyAADGroupSetting
    Set-MyAADGroupGuestAccess
```


## First run

First of, you need to set the ```AllowToAddGuests=false``` on all groups that you don't want to enable guest invite on.
In my case this was easy, it was all groups except two:

-----------

```Powershell
Import-Module MyAAD
# username = Application Id
# password = Application Key
$ClientCredential = Get-Credential

# Get Access Token
$AccessTokenResponse = Get-MyAADAccessToken -ClientCredential $ClientCredential -TenantName contoso.onmicrosoft.com
$AccessToken = $AccessTokenResponse.access_token

# Fetch Unified Groups
$UnifiedGroups = Get-MyAADUnifiedGroups -AccessToken $AccessToken
# Set guest access to false on all groups
$UnifiedGroups | Set-MyAADGroupGuestAccess -AllowToAddGuests $False #-Force (if you want to clear all existing settings on the group)
# Set guest access to true on some groups
$SharedUnifiedGroups = @(
    '755a9096-0eb1-4063-b1be-be0792a09997',
    'b0f5851b-856a-4ddc-8d9c-e21e46a4bfbd'
)
$UnifiedGroups | ? {$_.Id -in $SharedUnifiedGroups} | Set-MyAADGroupGuestAccess -AllowToAddGuests $True -Force
```

-----------

## The scheduled scripts

### The first script for configuring newly created groups

This will need to be scheduled somewhere, I'm running this in Jenkins so that I can manage secrets and monitor easily.
Also, it only collects the groups that were created within the last 48 hours. Others might do with selecting every group every run(2nd script) but since we have almost 2000 groups It's a lot easier this way and the risk of getting throttled is smaller.

-----------

```Powershell    
# Get the credential into a credential object in your preferred secure way.
    
# All groups created in the last 48 hours will be selected
$CreatedAfter = (get-date).AddHours(-48)

# Get Access Token
$AccessTokenResponse = Get-MyAADAccessToken -ClientCredential $ClientCredential -TenantName contoso.onmicrosoft.com
$AccessToken = $AccessTokenResponse.access_token
# Get all groups
$UnifiedGroups = Get-MyAADUnifiedGroups -AccessToken $AccessToken | ? {[datetime]$_.createdDateTime -gt $CreatedAfter}
# Get all group settings
$UnifiedGroupsSettings = $UnifiedGroups | Get-MyAADGroupSetting -AccessToken $AccessToken 
# Get all groups without settings applied and apply settings
$NoObjectSetting = $UnifiedGroupsSettings | Where-Object {$_.Value.id -eq $null}
$NoObjectSetting | Set-MyAADGroupGuestAccess -AllowToAddGuests $false -AccessToken $AccessToken
```

-----------

### Second script for enforcing it on missed groups etc.

This one takes a lot longer but  should be run hourly or daily IMO.
It's like the first script but it takes all groups including old ones.

-----------

```Powershell
# Get the credential into a credential object in your preferred secure way.

# Get Access Token
$AccessTokenResponse = Get-MyAADAccessToken -ClientCredential $ClientCredential -TenantName contoso.onmicrosoft.com
$AccessToken = $AccessTokenResponse.access_token
# Get all groups
$UnifiedGroups = Get-MyAADUnifiedGroups -AccessToken $AccessToken
# Get all group settings
$UnifiedGroupsSettings = $UnifiedGroups | Get-MyAADGroupSetting -AccessToken $AccessToken 
# Get all groups without settings applied and apply settings
$NoObjectSetting = $UnifiedGroupsSettings | Where-Object {$_.Value.id -eq $null}
$NoObjectSetting | Set-MyAADGroupGuestAccess -AllowToAddGuests $false  -AccessToken $AccessToken
```

-----------

## Allowing guest invite for a room
This is easy, get the GUID of the group you want to set and:

-----------

```Powershell
# Force will delete old setting and replace with new.
Set-MyAADGroupGuestAccess -Id '755a9096-0eb1-4063-b1be-be0792a09997' -AllowToAddGuests $false -AccessToken $AccessToken -Force
```

-----------

# Conclusion

It took many hours more than expected and I will never assume something is straight forward again until next week. Also, this was a great learning experience for using Microsoft Graph, that seems to become a must if you want to deliver value to your organization.

Also, after getting to know Microsoft Graph a bit more it's become easier to use raw ```Invoke-RestMethod``` commands instead of relying on ADAL or other heavier moules.

This is also not the most secure way to do it, since a group can be created and guests can be invited during a short span of time, depending on how often script 1 runs. But for many, this is enough. I'm thinking about creating a script that removes guests from groups that don't have Guest invitation enabled, but that's for later.