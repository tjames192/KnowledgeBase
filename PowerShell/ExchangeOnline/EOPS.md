# Migrate OnPrem DistributionGroup to ExchangeOnline
```
# distribution group
$dgfilter = Get-DistributionGroup -Identity <DistributionGroup> | ? { $_.RecipientType -ne "MailUniversalSecurityGroup" }

# snapshot distribution group properties to file 
$dgfilter | select * | Export-Clixml testdistributiongroup.xml

# snapshot all distribution group membership to file 
$dgfilter | % {$dg=$_; Get-DistributionGroupMember $dg.primarysmtpaddress | select @{l='DGName';e={$dg.name}},@{l='DGSMTPAddress';e={$dg.primarysmtpaddress}},* } | Export-Clixml testdistributiongroupmembership.xml 

# import distribution group details
$oldGroups = (Import-Clixml .\testdistributiongroup.xml)
$oldMembers = (Import-Clixml .\testdistributiongroupmembership.xml)

# Move OnPrem Object to a non ADSync OU
$ADGroup = Get-ADGroup $dgfilter.Identity -ErrorAction SilentlyContinue
$ADGroup | Get-ADObject | Move-ADObject -TargetPath "OU=Groups,OU=ToBeDeleted,DC=,DC="

# Sync AD
$pdc = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().DomainControllers.where({$_.Roles -match 'PdcRole'}).Name
Invoke-Command -ComputerName $pdc -ScriptBlock { $(repadmin /syncall /Aed).split([System.Environment]::NewLine,[System.StringSplitOptions]::RemoveEmptyEntries) }

# Sync Azure AD
Invoke-Command -ComputerName <ADSync> -ScriptBlock { $null=Import-Module 'C:\Program Files\Microsoft Azure AD Sync\Bin\ADSync\ADSync.psd1'; $null=Start-ADSyncSyncCycle -PolicyType delta; start-sleep 45 }

# Create ExchangeOnline Distribution Group
Connect-ExchangeOnline

write-verbose "Setting variables to create new group ..."
$group          = $oldGroups | ?{$_.name -eq $adgroup.name}
$members        = $oldMembers | ?{$_.dgname -like $adgroup.name}
$memberOf       = $oldMembers | ?{$_.name -eq $adgroup.name}
$managedby      = if($group.managedby -eq "Organization Management") {$null} else {@($group.managedby)}
$emailaddresses = @($group.emailaddresses)
$type           = if($group.recipienttype -eq "MailUniversalSecurityGroup") {"Security"} else { "Distribution" }

New-DistributionGroup -name $group.name -DisplayName $group.displayname -Alias $group.alias -RequireSenderAuthenticationEnabled $group.RequireSenderAuthenticationEnabled -ModerationEnabled $group.ModerationEnabled -BypassNestedModerationEnabled $group.BypassNestedModerationEnabled -SendModerationNotifications $group.SendModerationNotifications -MemberDepartRestriction $group.MemberDepartRestriction -Type $type

$members | % {add-distributiongroupmember $group.name -Member $_.Name}

Set-DistributionGroup -Identity $group.Identity -EmailAddresses $emailaddresses
```
