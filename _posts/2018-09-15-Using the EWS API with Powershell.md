---
layout: post
title: Using powershell to utilize the Exchange Web Services API
tags: [Powershell, EWS, Exchange]
---
I wanted to write a new guide to show you the thought process that I have when I'm exploring .net classes in Powershell.
And something that most of you use is Exchange so I wrote a few lines on how to explore and use the EWS API.


## Downloading the EWS managed API
First download and install the EWS managed API from [here.](https://www.microsoft.com/en-us/download/details.aspx?id=42951)


## Connecting to Office 365 with EWS

I would suggest doing this simple for you in the future and make yourself a function:


{% highlight powershell %}
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

    # First import the DLL, else none of the EWS classes will be available

    Try{
        Import-Module -Name "C:\Program Files\Microsoft\Exchange\Web Services\2.2\Microsoft.Exchange.WebServices.dll"
    }
    Catch{
        Write-Error "Error importing C:\Program Files\Microsoft\Exchange\Web Services\2.2\Microsoft.Exchange.WebServices.dll"
        Write-Error $_.Exception
        Break
    }
    
    # Set the domain in networkcredential - don't know if this is the way to do it but it works.
    $Credential.GetNetworkCredential().Domain = $Domain

    # Create ExchangeCredential object from regular [PSCredential]
    $ExchangeCredential = New-Object Microsoft.Exchange.WebServices.Data.WebCredentials($Credential.Username, $Credential.GetNetworkCredential().Password, $Credential.GetNetworkCredential().Domain)
    
    # Create the ExchangeService object
    $ExchangeService = New-Object Microsoft.Exchange.WebServices.Data.ExchangeService 
    $ExchangeService.UseDefaultCredentials = $true 
    
    # Add the ExchangeCredential to the ExchangeService
    $ExchangeService.Credentials = $ExchangeCredential
    
    # {$true} is needed if it's office 365 with a redirect on the autodiscovery. I think there's a lot more secure ways to check if
    # a redirection is correct.
    $ExchangeService.AutodiscoverUrl($AutoDiscoverUrl, {$true})
    
    # Return the object
    return $ExchangeService
}
{% endhighlight %}
{% highlight powershell %}
{% endhighlight %}

Easy, right?
Now, use that shiny new function to connect to EWS

{% highlight powershell %}

$Credential = Get-Credential
$ExchangeService = Connect-EWS -Credential $Credential -Domain contoso.com -AutoDiscoverUrl $Credential.UserName


{% endhighlight %}

Now the EWS API has created a ExchangeService object for you in the **$ExchangeService** variable.
This is the variable that you use mainly to bind to stuff such like calendars, inboxes, public folders etc.


## Binding

If you want to get an inbox or calendar, you need to bind to it. And to do this you would want to use it in conjunction with a 'Well known name' so that exchange knows where to look.

Binding is one of the first things you will need to do if you want to fetch or manipulate items in exchange with EWS.
And the bind is usually, if not always to a folder. Since everything in exchange and outlook is based around a folder structure.


Lets fetch my inbox:

{% highlight powershell %}
# First, we need to specify that we want a 'Well known folder' called inbox by using the enum
# called 'Microsoft.Exchange.WebServices.Data.WellKnownFolderName':

$WellKnownFolderName = [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Inbox

# Next we will use  the class 'Microsoft.Exchange.WebServices.Data.Folder' to bind to my inbox 
# with the help of the folder name and the $ExchangeService object
$Inbox = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ExchangeService, $WellKnownFolderName)

{% endhighlight %}


Now we have our variable $Inbox with a folder object that is bound to our inbox.
But what can we do with it?

## Exploring EWS

{% highlight powershell %}

# Fetch all the methods
$Inbox | Get-Member -MemberType Method | select Name 

Name                        
----                        
Copy                        
Delete                      
Empty                       
Equals                      
FindFolders <---------
FindItems                   
GetHashCode                 
GetLoadedPropertyDefinitions
GetType                     
Load                        
MarkAllItemsAsRead          
MarkAllItemsAsUnread        
Move                        
RemoveExtendedProperty      
Save                        
SetExtendedProperty         
ToString                    
TryGetProperty              
Update                      

{% endhighlight %}


Found something familiar? I would guess that the method 'MarkAllItemsAsRead' is the same method that outlook calls in online mode.
Tho i wouldn't mess around to much with stuff like Delete or move on my own account.


Let's find out how to fetch some emails in our inbox, the **'FindItems'** method looks like a nice candidate:


{% highlight powershell %}

# Lets see what we need to supply the FindItems method
$Inbox.FindItems

# ( a lot of stuff comes up, lets make it more readable with some regex voodoo and numbered rows:)

$Inbox.FindItems.OverloadDefinitions -replace "(.+ )(.+\()(.+)",'$2$3' | foreach {"$n - $_";$n++}

1 - FindItems(Microsoft.Exchange.WebServices.Data.SearchFilter searchFilter, Microsoft.Exchange.WebServices.Data.ItemView view)
2 - FindItems(string queryString, Microsoft.Exchange.WebServices.Data.ItemView view)
3 - FindItems(Microsoft.Exchange.WebServices.Data.ItemView view) <----------
4 - FindItems(Microsoft.Exchange.WebServices.Data.SearchFilter searchFilter, Microsoft.Exchange.WebServices.Data.ItemView view, Microsoft.Exchange.WebServices.Data.Grouping groupBy)
5 - FindItems(string queryString, Microsoft.Exchange.WebServices.Data.ItemView view, Microsoft.Exchange.WebServices.Data.Grouping groupBy)
6 - FindItems(Microsoft.Exchange.WebServices.Data.ItemView view, Microsoft.Exchange.WebServices.Data.Grouping groupBy)

{% endhighlight %}


This tells us that we need to supply the **FindItems** method with one of the combinations above.

If we want to list everything in our inbox, the most likely candidate is row 3: 
**'FindItems(Microsoft.Exchange.WebServices.Data.ItemView view)'**


So we need to find out what **'Microsoft.Exchange.WebServices.Data.ItemView'** is.


{% highlight powershell %}

# Type this into an IDE and it will give you all of it's available class methods: '[Microsoft.Exchange.WebServices.Data.ItemView]::'
# The only thing we can use here is the New() class method

[Microsoft.Exchange.WebServices.Data.ItemView]::new

OverloadDefinitions
------------------------------------
Microsoft.Exchange.WebServices.Data.ItemView new(int pageSize)
Microsoft.Exchange.WebServices.Data.ItemView new(int pageSize, int offset)
Microsoft.Exchange.WebServices.Data.ItemView new(int pageSize, int offset, Microsoft.Exchange.WebServices.Data.OffsetBasePoint offsetBasePoint)


# So this dosen't look to complicated, lets create a variable from this class method and supply it with an int


$ItemView = [Microsoft.Exchange.WebServices.Data.ItemView]::new(99999)


{% endhighlight %}


## Fetching items from your inbox

With a quick recap:

1. We found out that we would try the **$Inbox.FindItems()** method
2. To fetch as many items as possible, the most viable option seemed to be the Overload that only needed an "**ItemView**" object in it
3. We used the class method "**New**" on the **[Microsoft.Exchange.WebServices.Data.ItemView]** class to create a variable named $ItemView


{% highlight powershell %}

$ItemView = [Microsoft.Exchange.WebServices.Data.ItemView]::new(99999)
$Inbox.FindItems($ItemView)

{% endhighlight %}

Now, if you're not the inbox-zero type like me, you would problaly want to press ctrl+c depending on how many emails you have.
Let's see what we can do with the items...


## Exploring items and forwarding email


{% highlight powershell %}

# Select an item
$Item = $Inbox.FindItems($ItemView) | select -first 1

# Let's see what properties that are available
($Item | Get-Member -MemberType Properties).Name
# (long list)

# Let's see what what methods that are available
($Item | Get-Member -MemberType Method).Name
Copy
CreateForward
CreateReply
Delete
Equals
Forward
GetHashCode
GetLoadedPropertyDefinitions
GetType
Load
Move
RemoveExtendedProperty
Reply
Save
Send
SendAndSaveCopy
SetExtendedProperty
SuppressReadReceipt
ToString
TryGetProperty
Update

{% endhighlight %}

Some interesting stuff here.
We can forward the email, Reply to it, move it, send it, delete it etc.

What if we wanted to forward it?

{% highlight powershell %}

$Item.Forward

OverloadDefinitions
------------------------------
void Forward(Microsoft.Exchange.WebServices.Data.MessageBody bodyPrefix, Params Microsoft.Exchange.WebServices.Data.EmailAddress[] toRecipients)
void Forward(Microsoft.Exchange.WebServices.Data.MessageBody bodyPrefix, System.Collections.Generic.IEnumerable[Microsoft.Exchange.WebServices.Data.EmailAddress] toRecipients)

{% endhighlight %}

**'Microsoft.Exchange.WebServices.Data.MessageBody bodyPrefix'**:
This tells us that it problably want some kind of prefix, probably 'fw' or something right?

**'Microsoft.Exchange.WebServices.Data.EmailAddress[] toRecipients'** and **'System.Collections.Generic.IEnumerable[Microsoft.Exchange.WebServices.Data.EmailAddress] toRecipients'**:
This tells us that we will either need to supply one email address or a **collection** of email addresses.

Let's explore the **'Microsoft.Exchange.WebServices.Data.MessageBody'**:

{% highlight powershell %}
[Microsoft.Exchange.WebServices.Data.MessageBody]::new

Microsoft.Exchange.WebServices.Data.MessageBody new()
Microsoft.Exchange.WebServices.Data.MessageBody new(Microsoft.Exchange.WebServices.Data.BodyType bodyType, string text)
Microsoft.Exchange.WebServices.Data.MessageBody new(string text)

{% endhighlight %}

Looks like you can supply a **BodyType** here as well, if you want to send  the message with HTML or Text formating.
But let's skip that and just create a new MessageBody with the string 'CUSTOMFORWARD: <subject>'.

If my theory is right the email will arrive as "CUSTOMFORWARD: <Subject>" right?
And since you problably can and since it's easy: Let's supply the email as a regular string


{% highlight powershell %}

$MessageBody = [Microsoft.Exchange.WebServices.Data.MessageBody]::new("CUSTOMFORWARD: $($Item.Subject)")
$Item.Forward($MessageBody, '<your email>')


{% endhighlight %}

Look in your inbox and see the result.

## Moving an email

As we saw earlier, there was a 'Move()' method as well.

This need either a 'FolderId' if you want to move it to a self made folder, another mailbox or to public folders.
Or it wants a 'WellKnownFolderName' (as we used earlier).

Let's move it to Drafts:

{% highlight powershell %}

$WellKnownFolderName = [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Drafts
$Item.Move($WellKnownFolderName)

{% endhighlight %}

And now it shows up in the drafts folder.

## Exploring other items

In exchange there is notes, to do lists, calendars and so on. All of those are fetchable with about the same method.

### Fetching calendar items

{% highlight powershell %}

$WellKnownCalendarFolder = [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Calendar
$Calendarbind = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ExchangeService, $WellKnownCalendarFolder)
$ItemView = [Microsoft.Exchange.WebServices.Data.ItemView]::new(99999)

$Calendarbind.FindItems($ItemView)

# Or fetch with FindAppointments() with a date range

$CalendarView = [Microsoft.Exchange.WebServices.Data.CalendarView]::((Get-Date),(Get-Date).AddDays(30))
$Calendarbind.FindAppointments


{% endhighlight %}

### Fetching notes

{% highlight powershell %}
$ItemView = [Microsoft.Exchange.WebServices.Data.ItemView]::new(99999)
$WellKnownNoteFolder = [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Notes
$NoteBind = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ExchangeService, $WellKnownNoteFolder)

$NoteBind.FindItems($ItemView)

{% endhighlight %}

### Fetching todo's

{% highlight powershell %}

$ItemView = [Microsoft.Exchange.WebServices.Data.ItemView]::new(99999)
$WellKnownTodoFolder = [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Tasks
$TodoBind = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ExchangeService, $WellKnownTodoFolder)

$TodoBind.FindItems($ItemView)

{% endhighlight %}

### Fetching folders and items from public folders


{% highlight powershell %}

$FolderView = [Microsoft.Exchange.WebServices.Data.FolderView]::new(10000)
$WellKnownPublicFolder = [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::PublicFoldersRoot
$PublicFolderBind = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ExchangeService, $WellKnownPublicFolder)


# List all folders
$PublicFolderBind.FindFolders($FolderView)

# List all items 
$ItemView = [Microsoft.Exchange.WebServices.Data.ItemView]::new(99999)
$PublicFolderBind.FindItems($ItemView)

# Get a subfolder

$Folder = $PublicFolderBind.FindFolders($FolderView) | ? {$_.DisplayName -eq 'HR Calendar'}
$Folder.FindItems($ItemView)

{% endhighlight %}