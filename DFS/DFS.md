#Add Add the DFS Namespace feature
Add-WindowsFeature FS-DFS-Namespace

#Add the key to enable Consolidation Readirection to the registry
New-Item -Type Registry  HKLM:SYSTEM\CurrentControlSet\Services\Dfs
New-Item -Type Registry  HKLM:SYSTEM\CurrentControlSet\Services\Dfs\Parameters
New-Item -Type Registry  HKLM:SYSTEM\CurrentControlSet\Services\Dfs\Parameters\Replicated
New-ItemProperty  HKLM:SYSTEM\CurrentControlSet\Services\Dfs\Parameters\Replicated ServerConsolidationRetry -Value 1

#Create a directory and share for the namespace
$oldSrv = "#<server>"
New-Item -Type Directory "C:\DFSRoots\$oldSrv"
# hide the shared folder, appedning '$' is not supported in dfsnroot
attrib +h +s "C:\DFSRoots\$oldSrv"
New-SmbShare -Name $oldSrv -Path "C:\DFSRoots\$oldSrv"

#Create the DFS namespace
New-DFSNroot -Type Standalone -TargetPath "\\localhost\$oldSrv" -path "\\localhost\$oldSrv"

#Add dfsnfolder and folder target; a target is a pointer to SMB file that host content
#when user bowse a folder with target the clients receives a referral to redirect the client machine
$oldShare = "G"
New-DfsnFolder -Path \\localhost\$oldSrv\$oldShare -TargetPath \\localhost\$oldShare
$oldShare = "MATERIALS"
New-DfsnFolder -Path \\localhost\$oldSrv\$oldShare -TargetPath \\localhost\$oldShare

### stage the files using robocopy ###
######################################

### rename oldserver, shutdown ###
##################################

#Add an alternative computername
netdom computername localhost /add fqdn
# OptionalNames for Server service [SMB]
# HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\lanmanserver\parameters

#Uppdate DNS
ipconfig /registerdns


<#
$altNames = @("computername")
$hostName = "hostname"
#New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\Lsa -Name DisableLoopbackCheck -PropertyType DWord -Value 1
#New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0 -Name BackConnectionHostNames -PropertyType MultiString -Value $altNames
New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\lanmanserver\parameters -Name OptionalNames -PropertyType MultiString -Value $altNames
New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\lanmanserver\parameters -Name DisableStrictNameChecking -PropertyType DWord -Value 1
New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\lanmanserver\parameters -Name DnsOnWire -PropertyType DWord -Value 1
#>

#Reboot to allow windows to get kerberos with all alternative names
shutdown -r -t 0

#test from a client

may need to revisit this:
recently resolved an odd issue
netsh int ip reset

the case for not using dfsn with root consolidation?
why? 
to new IT admins, there is overhead in learning how to setup, maintain and TSHOOT unknowns.
speed.. adding new folder with target is fast.
however when a client accesses the folder with target browsing to \\<server>\#<server>\<folder>
this can temporarily fail even after reboot and then in minutes it can start working.
I really wanted this to work however in a lab setting it just doesn't work as fast nor is as easily manageable.
If you don't have the resources or capacity to maintain and TSHOOT issues like above or unknowns.
I cannot recommend DFSN with root consolidation in a production environment.
If folder replication is needed than using DFS is probably the best option

Alternative?

### stage the files using robocopy ###
######################################

### rename oldserver, shutdown ###
##################################

#Add an alternative computername
netdom computername localhost /add fqdn
#Uppdate DNS
ipconfig /registerdns

#Reboot to allow windows to get kerberos with all alternative names
shutdown -r -t 0

# OptionalNames for Server service [SMB] -multi-string value
# HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\lanmanserver\parameters

# clients need dns suffix search order, can be set via GPO
wmic nicconfig call SetDNSSuffixSearchOrder (domain.com)
#windows dns short name resolution
#Computer > Templates > Network > DNS Client > DNS Suffix Search List
#https://www.serverbrain.org/networking-guide-2003/managing-dns-suffixes-problem.html
