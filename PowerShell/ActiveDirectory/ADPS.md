#

# Sync AD / Azure
```
$pdc = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().DomainControllers.where({$_.Roles -match 'PdcRole'}).Name
Invoke-Command -ComputerName $pdc -ScriptBlock { $(repadmin /syncall /Aed).split([System.Environment]::NewLine,[System.StringSplitOptions]::RemoveEmptyEntries) }
Invoke-Command -ComputerName $pdc ScriptBlock { $(repadmin /replsummary).split([System.Environment]::NewLine,[System.StringSplitOptions]::RemoveEmptyEntries) }
Invoke-Command -ComputerName <ADSync> -ScriptBlock { $null=Import-Module 'C:\Program Files\Microsoft Azure AD Sync\Bin\ADSync\ADSync.psd1'; $null=Start-ADSyncSyncCycle -PolicyType delta; start-sleep 45 }
```

# get-aduserlogondetails
```
function get-aduserlogondetails {
    param (
        $aduser,
        # list of DCs
        [string[]]$computer = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().DomainControllers.Name
    )
   
    $r = invoke-command -computer $computer -command { 
        param($aduser) 
        get-aduser $aduser -Properties lastlogon,lastLogonTimestamp,badPasswordTime | 
        % { 
            [pscustomobject]@{
                name = $_.name;
                samaccountname = $_.samaccountname;
                userprincipalname = $_.userprincipalname;
                badPasswordTime = [datetime]::fromfiletime($_.badPasswordTime);
                lastlogon = [datetime]::fromfiletime($_.lastlogon);
                lastlogontimestamp = [datetime]::fromfiletime($_.lastlogontimestamp)
            } 
        } 
    } -argumentlist $aduser;
   
    [pscustomobject]@{
        name = $r[0].name;
        samaccountname = $r[0].samaccountname;
        userprincipalname = $r[0].userprincipalname;
        badPasswordTime = $r.badPasswordTime | sort -descending | select -first 1;
        lastlogon = $r.lastlogon | sort -descending | select -first 1;
        lastlogontimestamp = $r.lastlogontimestamp | sort -descending | select -first 1;
        PSComputerName = $r.PSComputerName
    }
}
```
