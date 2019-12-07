---
layout: post
title: "Getting UserAccountControl flags in AD a bit faster"
tags: [Active Directory, UserAccountControl, PowerShell]
---

It's been a while since my last post on this blog, but this is a quick one that might make your scripts run a bit faster if you want to fetch the UserAccountControl flags for multiple users (especially if you have thousands of users in AD).

For those of you who don't know: [UserAccountControl](https://support.microsoft.com/en-us/help/305144/how-to-use-useraccountcontrol-to-manipulate-user-account-properties) is a value in AD that defines if the user don't requires a password or if password never expires for the user and a whole bunch of other stuff that's important to keep track of. It's a 32bit (4 byte) value that defines a bunch of options for a user account in AD depending on what bits that are set.

The function below is at least 1ms faster (0.3ms avg) than the other methods I've tried so far. 
That saves me around 25-30 seconds for an AD audit run and will come in handy when I convert it to a C# version for the REST API that my Graylog [Lookup Tables](https://docs.graylog.org/en/3.1/pages/lookuptables.html) use.

    # If Bit -eq <int> then UAC has the corresponding value
    # Set the hash outside of the function to save time
    $UserAccountControlHash = @{

        #Bit Description
        0 = 'SCRIPT'
        1 = 'ACCOUNTDISABLE'
        3 = 'HOMEDIR_REQUIRED'
        4 = 'LOCKOUT'
        5 = 'PASSWD_NOTREQD'
        6 = 'PASSWD_CANT_CHANGE'
        7 = 'ENCRYPTED_TEXT_PWD_ALLOWED'
        8 = 'TEMP_DUPLICATE_ACCOUNT'
        9 = 'NORMAL_ACCOUNT'
        11 = 'INTERDOMAIN_TRUST_ACCOUNT'
        12 = 'WORKSTATION_TRUST_ACCOUNT'
        13 = 'SERVER_TRUST_ACCOUNT'
        16 = 'DONT_EXPIRE_PASSWORD'
        17 = 'MNS_LOGON_ACCOUNT'
        18 = 'SMARTCARD_REQUIRED'
        19 = 'TRUSTED_FOR_DELEGATION'
        20 = 'NOT_DELEGATED'
        21 = 'USE_DES_KEY_ONLY'
        22 = 'DONT_REQ_PREAUTH'
        23 = 'PASSWORD_EXPIRED'
        24 = 'TRUSTED_TO_AUTH_FOR_DELEGATION'
        26 = 'PARTIAL_SECRETS_ACCOUNT'
    }

    $UACValue = 65244


    Function Get-UACProperties {
        param(
            [parameter(mandatory)]
            [uint32]$UACValue
        )

        # Create a BitArray from the bytes
        $BitArray = [System.Collections.BitArray]::new([System.BitConverter]::GetBytes($UACValue))

        # Get the value from the hash where BitArray -eq $True
        $UserAccountControlHash.GetEnumerator().Where({
            $BitArray[$_.Name]
        }).Value
    }

Example on how to use it:

    $User = Get-ADUser -Identity johndoe -Properties UserAccountControl
    Get-UACProperties $User.UserAccountControl

And to get average the execution time:

    $Rand = 1..867200 |  Get-Random -Count 100
    $Rand | Foreach {
        Measure-Command -Expression {
            Get-UACProperties -UACValue $_
        }
    } | Measure -Property TotalMilliseconds -Average | Select -ExpandProperty Average