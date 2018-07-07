---
layout: post
title: When disaster strikes your AD
---

A while back we had a AD-RES session with microsoft, a session focused on developing a rescue routine for your AD. And here are some of the things i learned or got reminded about, not only related to DR:

### The computers remembers the password last two passwords
When rescuing AD, one of the most unsettling things is the thought of having to repair the trust-relationship for all computers that has changed it's password since the backup you are restoring from.

Well, it turns out that the machine stores 2 passwords: The one it uses and the one it had before, so restoring to a previous backup **should not** be a problem.

### Never trust just one platform
Having your domaincontrollers on more than one hardware platform (ie. VMWare and Bare Metal or VMWare and Hyper-V) migitiates the risk tremendously. Especially if VMWare auth is down because of that you can't authenticate to AD.

### Never trust one form of backup
Using both Veeam and Windows Server Backup for your DC's is a great idéa. Especially if the Veeam backup got hacked or is corrupt.
Also, if you are a premier support customer; Microsoft does only support Windows Server Backup.

### Keep your ADSM (active directory safe mode) passwords properly documented and stored!
This is an easy one to forget about, especially if you have inherited an environment. If it's not documented and locked into a safe; change the password and document it properly.

### Plan for that your DR scenarios might have to take place offline
In case of a security breach, the network might have to be taken offline. Plan for DR accordingly. And, DC's might have to be kept offline during recovery so that a DC with a larger RID-number on it's objects dosen't overwrite the data that you just restored.

### Most AD recovery isn't a DR scenario per say
But a mass deletion in AD is severe enough. Doublecheck that you have the recycle-bin enabled in your domain and develop scripts to quickly mass-restore objects. What we use:

```powershell
    # This restores the OU's first, and after that the objects in order. Else it will try to recreate the objects in an OU or object that dosen't exist and fail.
    # Replace the date with the date that that the mass-deletion took place
    $FromDate = Get-Date "2018-02-12 13:52:02"
    $Deleted = Get-ADObject -Filter {(isdeleted -eq $true) -and (WhenChanged -gt $FromDate)} -IncludeDeletedObjects -Properties * | sort lastknownparent -Descending
    $Deleted | ? {$_.objectclass -eq "organizationalUnit"} | Restore-ADObject
    $Deleted | ? {$_.objectclass -ne "organizationalUnit"} | Restore-ADObject
```

### Use the microsoft tiering model for securing important infrastructure
Read more about it here: https://docs.microsoft.com/en-us/windows-server/identity/securing-privileged-access/securing-privileged-access-reference-material

### Coffee and perhaps something to eat is the AD-admins best friend
Give AD time to replicate and go and grab a coffee. Being to much in a hurry WILL make things worse

### Document your AD in an easy way
Use the "Active directory topology diagrammer" to document your AD and keep it in the same binder as the DR documentation. This will save the one rescuing the AD a lot of headache and even for you since everybody reacts differently during a crisis.

### Emergency admin
You should have an emergency admin, and it should be monitored for logins and locked in a safe. Password should be changed regularly.

### Practice DR yearly
We all know this, but we don't do it because of time.
Create a recurring meeting, one or two days a year for practicing to force yourself to make time for it.

### AD is stable, and most DR scenarios isn't because of a failiure of AD
Most DR scenarios is because of a security breach. I yet again refer to: https://docs.microsoft.com/en-us/windows-server/identity/securing-privileged-access/securing-privileged-access-reference-material

### The DC that you recover to SHOULD be able to handle most of the load for a period of time
When recovering AD, at some time, only one DC will be available. And all machines will try to go towards it. When creating or buying a spare machine that AD will be restored to - add a lot of CPU.

### Write the DR documentation so that it's easy to follow.
You might not be around when it happens, you might have been hit by a buss. And the one the company decides to call in panic might not be the one best suited for the job.

### It's OK since Server 2008 to change IP and DNS of the domaincontrollers
This seems to be the biggest no-no in the AD community. But according to microsoft it's been supported for a while.
You want to flush/register DNS tho, and scan through your DNS-records. I've since done this in a test-forest, DMZ-forest and production and never had a problem.
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

### GPO's that are backed up with the powershell cmdlets dosen't remember the linked OU's
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
* That the server is in an auto patch group
* That the distribution of DC's in the auto patch groups are even, so that 50% of the dc's don't auto update at the same time.
* That it can reach other DC's

### Have your boss in on the DR plans, and agree that he will act as a gatekeeper during a DR scenario
Having someone holding the door and acting as a information channel during a DR scenario is important. Especially since one error might lead to you having to start over the DR routine from step 1 (An old DC writing over the recovered contents of a new DC for example). A room with a lockable door is preferred.



----
****