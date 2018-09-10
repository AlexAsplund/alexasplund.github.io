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


```Powershell
<#
.Synopsis
   Script that imports a public folder calendar to a mailbox.
.DESCRIPTION
   Script that imports a public folder calendar to a mailbox.
   Need EWS api installed and Offce365 admin rights
.EXAMPLE
    This will give the $ExchangeCredential.Username account FullAccess permissions to meetingroom1.contoso.com
    After that it wil take the calendar from -PublicFolderPath and copy the items in it to meetingroom1@contoso.com calendar.
    The last two actions renames the public folder calendar to 'Not used - *DATE* - <DisplayName>'

   .\Import-CalendarFromPublicFolder.ps1 -ExchangeCredential $ExchangeCredential -Office365AdminCredential $Office365AdminCredential -PublicFolderPath 'HR\Rooms\Meeting room 1' -RoomMailAddress meetingroom1@contoso.com -ChangePermissions
#>
param(
    [parameter(Mandatory)]
    [pscredential]$ExchangeCredential,

    [parameter(Mandatory)]
    [pscredential]$Office365AdminCredential,

    [parameter(Mandatory)]
    [string]$PublicFolderCalendarPath,

    [parameter(Mandatory)]
    [string]$RoomMailAddress,

    [int]$ItemLimit = 999999,

    [switch]$ChangePermissions
)

Import-Module -Name "C:\Program Files\Microsoft\Exchange\Web Services\2.2\Microsoft.Exchange.WebServices.dll"

#########################################
# Functions
#########################################

function Connect-EWS {
    param(
        [parameter(Mandatory = $True)]
        [PSCredential]
        $Credential,
        [parameter(Mandatory = $True)]
        [string]
        $Domain,
        [parameter(Mandatory = $True)]
        [string]
        $AutoDiscoverUrl
        
    )

    # Set the domain in networkcredential - don't know if this is the way to do it but it works.
    $Credential.GetNetworkCredential().Domain = $Domain

    # Create ExchangeCredential from regular [PSCredential]
    $ExchangeCredential = New-Object Microsoft.Exchange.WebServices.Data.WebCredentials($Credential.Username, $Credential.GetNetworkCredential().Password, $Credential.GetNetworkCredential().Domain)
    
    # Create the exchange service object
    $ExchangeService = New-Object Microsoft.Exchange.WebServices.Data.ExchangeService 
    
    # Not sure if this is needed
    $ExchangeService.UseDefaultCredentials = $true 
    
    # Add the ExchangeCredential to the ExchangeService
    $ExchangeService.Credentials = $ExchangeCredential
    
    # {$true} is needed if it's office 365 with a redirect on the autodiscovery. I think there's a lot more secure ways to check if
    # a redirection is correct.
    $ExchangeService.AutodiscoverUrl($AutoDiscoverUrl, {$true})
    
    return $ExchangeService
}

#########################################
# Connect to Office365
#########################################
if($ChangePermissions){


    try{
        Write-Output "Connecting to Exchange Online"
        $Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri https://outlook.office365.com/powershell-liveid/ -Credential $Office365AdminCredential -Authentication Basic -AllowRedirection
        Import-PSSession $Session -DisableNameChecking

    }
    catch{
    
        Write-Error $_
        Write-Error "Error connecting to office 365"
        Break

    }

    Try{
        Write-Output "Fetching mailbox"
        $Mailbox = Get-Mailbox $RoomMailAddress
    }
    catch{
        Write-Error "Error fetching mailbox"
        Write-Error $_
        Break
    }

    Try {
        Write-Output "Adding permissions"
        $Mailbox | Add-MailboxPermission -User $ExchangeCredential.UserName -AccessRights FullAccess -InheritanceType All -AutoMapping $false
    }
    catch{
       Write-Error "Error setting permissions on mailbox"
       Write-Error $_
       Break
    }

}

#########################################
# Connect to EWS
#########################################

$ExchangeService = Connect-EWS -Credential $ExchangeCredential -Domain ($ExchangeCredential.UserName -replace "^(.+)(@.+)$",'$2') -AutoDiscoverUrl $ExchangeCredential.UserName

#########################################
# Resolve public folder calendar
#########################################

$WellKnownPublicFolderName = [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::PublicFoldersRoot
$PublicFolderRoot = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ExchangeService, $WellKnownPublicFolderName)
$PublicFolderView = [Microsoft.Exchange.WebServices.Data.FolderView]::new($ItemLimit)
$FolderPath = ($PublicFolderCalendarPath -split "\\") 
$Folder = $Null


$FolderPath | foreach {
    
    $FolderName = $_
    $ParentFolder = $Folder

    if($Folder -eq $Null){
        $Folder = $PublicFolderRoot.FindFolders($PublicFolderView) | ? {$_.DisplayName -eq $FolderName}
    }
    else{
        $Folder = $Folder.FindFolders($PublicFolderView) | ? {$_.DisplayName -eq $FolderName}
    }   

}

#########################################
# Get Room folder ID
#########################################

 
$WellKnownFolderCalendar = [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Calendar
$RoomFolder = [Microsoft.Exchange.WebServices.Data.FolderId]::new($WellKnownFolderCalendar, $RoomMailAddress)
$RoomCalendar = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ExchangeService, $RoomFolder)
$RoomCalendarId = $RoomCalendar.Id




#########################################
# Copy items to folder
#########################################

$Items = $Folder.FindItems([Microsoft.Exchange.WebServices.Data.ItemView]::new(1000000)) | ? {$_.Start -ge (Get-Date).AddDays(-30)}

foreach($Item in $Items){
    
    Try{
        $Item.Copy($RoomCalendarId)
        Write-Output "Copied|$($RoomCalendarId.UniqueId)|$($Path)|ID: $($Item.Id.UniqueId)"
    }
    catch{
        Write-Output "Error Copying Appointment|$($RoomCalendarId.UniqueId)|$($Path)|ID: $($Item.Id.UniqueId)"
    }

}

#######################################
# Rename old calendar
#######################################

$Date = Get-Date -Format "yyyy-MM-dd"
$Folder.DisplayName = ("Not used - $Date - "+$Folder.DisplayName)
try{
    $Folder.Update()
}
catch{
    Write-Output "Error Renaming Old Calendar|$($Folder.DisplayName)"
}




######################################
# Remove mailbox permission
######################################
if($ChangePermissions){
    $Mailbox | Remove-MailboxPermission -User $ExchangeCredential.UserName -AccessRights FullAccess -InheritanceType All
}
```



****
----