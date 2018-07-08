---
layout: post
title: Some of my notes when it comes to Disaster Recovery of Active directory and security
---

A while back we had a couple of session with microsoft, one of those focused on DR of Active Directory and another one was on AD health. Here are some of my notes and things learned. Some of them are obvious but might need a reminder and other ones might not be well known:

### The computers remembers the password last two passwords
When rescuing AD, one of the most unsettling things is the thought of having to repair the trust-relationship for all computers that has changed it's password since the backup you are restoring from.

Well, it turns out that the machine stores 2 passwords: The one it uses and the one it had before, so restoring to a previous backup **should not** be a problem. Depending on the age of your backup.

### Never trust one platform
Having your domaincontrollers on more than one hardware platform (ie. VMWare and Bare Metal or VMWare and Hyper-V) migitiates the risk tremendously. Especially if VMWare auth is down because of that you can't authenticate to AD.

### Never trust one backup platform
Using both Veeam and Windows Server Backup for your DC's is a great idea. Especially if the Veeam backup got hacked or is corrupt, tapes are corrupt etc.
Also, if you are a premier support customer; Microsoft does only support Windows Server Backup.

### Keep your ADSM (active directory safe mode) passwords properly documented and stored!
This is an easy one to forget about, especially if you have inherited an environment. If it's not documented and locked into a safe; change the password and document it properly.

### Plan for that your DR scenarios might have to take place offline
In case of a security breach, the network might have to be taken offline. Plan for DR accordingly. And, DC's might have to be kept offline during recovery so that a DC with a larger RID-number on it's objects dosen't overwrite the data that you just restored.

### Most AD recovery isn't a DR scenario per say
But a mass deletion in AD is severe enough. Doublecheck that you have the recycle-bin enabled in your domain and develop scripts to quickly mass-restore objects. What we use:


    # This restores the OU's first, and after that the objects in order. Else it will try to recreate the objects in an OU or object that dosen't exist and fail.
    # Replace the date with the date that that the mass-deletion took place
    $FromDate = Get-Date "2018-03-30 13:02:02"
    $Deleted = Get-ADObject -Filter {(isdeleted -eq $true) -and (WhenChanged -gt $FromDate)} -IncludeDeletedObjects -Properties * | sort lastknownparent -Descending
    $Deleted | ? {$_.objectclass -eq "organizationalUnit"} | Restore-ADObject
    $Deleted | ? {$_.objectclass -ne "organizationalUnit"} | Restore-ADObject


### Use the microsoft tiering model for securing important infrastructure
Read more about it [here](https://docs.microsoft.com/en-us/windows-server/identity/securing-privileged-access/securing-privileged-access-reference-material)

This will hopefully make it so that you don't have to rebuild the entire environment in case of a security breach.

### Coffee and perhaps something to eat the AD admins best friend.
Give AD time to replicate and go and grab a coffee. Being to much in a hurry WILL make things worse

### Document your AD in an easy way
Use the "Active directory topology diagrammer" to document your AD and keep it in the same binder as the DR documentation. This will save the one rescuing the AD a lot of headache and even for you since everybody reacts differently during a crisis.

### Emergency admin account
You should have an emergency admin account, and it should be monitored for logins and locked in a safe. Password should be changed regularly.

### Practice DR yearly
We all know this, but we don't do it because of time.
Create a recurring meeting, one or two days a year for practicing to force yourself to make time for it.

After practicing this the first time and documenting a routine, the worries AD breaking down is minimal. And the big black hole of worry when it comes to this shrinks.

### AD is stable, and most DR scenarios isn't because of a failiure of AD
Most DR scenarios is because of a security breach. I yet again refer to: [this](https://docs.microsoft.com/en-us/windows-server/identity/securing-privileged-access/securing-privileged-access-reference-material)

### The DC that you recover to SHOULD be able to handle most of the load for a period of time
When recovering AD, at some time, only one DC will be available. And all machines will try to go towards it. When creating or buying a spare machine that AD will be restored to - add a lot of CPU.

### Write the DR documentation so that it's easy to follow.
You might not be around when it happens, you might have been hit by a buss. And the one the company decides to call in panic might not be the one best suited for the job.

### It's OK since Server 2008 to change IP and DNS of the domaincontrollers
This seems to be the biggest no-no in the AD community. But according to microsoft it's been supported for a while and it seems to be an inherited belief in the sysadmin community.
Not to say it isn't risky, it is and some depending systems might not handle it.

You want to flush/register DNS tho, and scan through your DNS-records. I've since done this in a test-forest, DMZ-forest and couple of production-forests and never had a problem.
This comes in especially handy in environments where you haven't load balanced LDAP/DNS and need to keep the same names/ip of some DC's

How we did it:

1. Promote a new DC.
2. Demote old DC.
3. Change name of old DC.
4. Remove old DC from domain.
5. Change IP of old DC, turn it off.
6. Change name of new DC to old DC's name.
7. Change IP of new DC to the old DC's IP
8. ipconfig /flushdns
9. ipconfig /registerdns
10. Wait until "repadmin /showrepl" is OK, grab a coffee.
11. Change name of the new DC to the old DC's name.
12. ipconfig /flushdns
13. ipconfig /registerdns
14. Wait until "repadmin /showrepl" is OK, grab a coffee.

Out of hundreds of systems and thousands of computers and servers, only 3 systems choked when we did this on 5 DC's.

### GPO's that are backed up with the powershell cmdlets don't store the linked OU's
This might come as a nasty suprise for some. Use the Get-GPOReport and parse the XML for the links that you store in the same folder as the GPO backup.

### Write pester tests for testing baseline of your DC's
You might not remember to put all the roles and configs in, and you might want to test that the networking team has done their jobs.
So testing the baseline of your DC's is important. What we currently test with pester after installing a new DC:

* Can resolve towards our edge DNS servers
* That all roles and features needed are installed
* That the DFS namespace resolves properly
* That no replication errors are occuring
* Get-ADUser works aganist the server
* That the server can resolve DNS
* AV is installed and exclusions are made
* That firewall ports are opened/closed
* That the server is in an auto patch group
* That the distribution of DC's in the auto patch groups are even, so that 50% of the dc's don't auto update at the same time.
* That it can reach other DC's

etc.

### Have your boss in on the DR plans, and agree that he will act as a gatekeeper during a DR scenario
Having someone holding the door and acting as a information channel during a DR scenario is important. Especially since one error might lead to you having to start over the DR routine from step 1 (An old DC writing over the recovered contents of a new DC for example). A room with a lockable door is preferred.

### Load balancing the primary DNS and LDAP
This is a great idea. Especially when a lot of stuff is bound directly to the DC's.
This will make it easier to restart, replace and remove DC's. F5 for example handles this fine.

### Moving FSMO roles is easy

```powershell
    # If FSMO role holder is online:
    Move-ADDirectoryServerOperationMasterRole -Identity "Target-DC" -OperationMasterRole SchemaMaster,RIDMaster,InfrastructureMaster,DomainNamingMaster,PDCEmulator
    # If FSMO role holder is crashed and you need to sieze the roles
    Move-ADDirectoryServerOperationMasterRole -Identity "Target-DC" -OperationMasterRole SchemaMaster,RIDMaster,InfrastructureMaster,DomainNamingMaster,PDCEmulator -Force
```

### It's normal for a demote of a DC to leave some thrash DNS records
Scan your DNS records, either manually or with a script after leftover records from the old DC's and delete them. 

### Schema changes isn't final until next defragmantation of the JET database
This occurs once every 12h. even tho it works before that.

### If you're going to monitor one thing, monitor for JET database errors on the domaincontrollers
This is a sign of corruption in the AD database. [Here's the event ID's.](https://support.microsoft.com/en-in/help/4042791/jet-database-errors-and-recovery-steps)

### Monitor DFS-R for SYSVOL and Netlogon replication errors
A restore of those can be quite annoying, but not to hard. [Read about it here](https://support.microsoft.com/en-us/help/2958414/dfs-replication-how-to-troubleshoot-missing-sysvol-and-netlogon-shares)

Just be carefull so that you don't overwrite a good share that you were supposed to use.
And double check that GPO's are working after a restore, else restore GPO from last known good backup. Otherwise it might cause a mismatch between GPO version in AD and GPO version in SYSVOL.

### Domain isn't a security bondary, a forest is
 [I yet again, refer to the tiering model.](https://support.microsoft.com/en-us/help/2958414/dfs-replication-how-to-troubleshoot-missing-sysvol-and-netlogon-shares)

[This is a good read as well.](https://blogs.technet.microsoft.com/389thoughts/2017/06/19/ad-2016-pam-trust-how-it-works-and-safety-advisory/)

### Monitor for NTLMv1 usage and disable it
NTLMv1 is roughly 30 years old and an obselete authentication method.
What it does is that it from the beginning only supported 7 characters + 1 parity bit like this:

\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[*]

This is simple enought to crack, 7 chars is done in no time at all. According to what i found on the internet it's 57^7 combinations and takes around 10 minutes.
Now, afterwards they added support for 14 chars and that should take, but did they make it 14 whole bytes + a parity bit? NO...

If they made it like this:

\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[*\]

The password would in theory take 204 million years for a brute force attack to crack it. But how it works is that it splits the password in two like this:

\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[\*\] + \[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[ ]\[\*\]

So it takes in theory 20 minutes instead...

On top of that, if your password is lets say, 11 charachters it fills the remaining bytes with 0.

\[M]\[Y]\[P]\[A]\[S]\[S]\[W]\[\*\] + \[O]\[R]\[D]\[!]\[0]\[0]\[0]\[\*\]

Did you notice how it's all caps? That's because the NTLM password is converted to all caps before hashed into the database.
NTLMv1 is dumb and should be disabled.

If you installed your forest from scratch with with Server 2016, NTLMv1 is disabled by default.

### If you keep your systems patched, security breaches through software vunerabilities is rare
The most common point of entry is through identity theft.
This is why it's even more important to use the microsoft security model when designing security for your AD.

Because if the hacker has owned a computer by calling Debbie and asking nicley, and you have been logged on with an account that has Domain Admin rights on that machine; The hacker owns your network.

### When scanning software for missing patches use a software or script that uses the wsusscn2.cab
Use the WSUS offline catalog when scanning for missing patches! A lot of software just contacts your local WSUS and if WSUS dosen't have any patches to offer it assumes that it's good.
The truth is that there might be a lot of patches missing on your system, and scanning with the offline WSUS catalog will catch it.

### Upgrading the forest functional level is best done daytime
A lot of sneaky errors can occur when uppgrading the forest/domain functional level.

One of those are that the KRBTGT (Kerberos Ticket Granting Ticket) password is changed.
Windows systems tend to follow the change without any problems, but *nix systems talking kerberos might not and you might have to restart them.
So if your environment has a lot of important applications running in linux, especially if they are critical; do it during daytime and cooperate with your *nix team.

From my experience, this is best suited at 10AM. People have arrived to work, are awake and ain't hungry.

Also, do yourself a faviour and upgrade in a test forest with the most critical apps first.

----
****