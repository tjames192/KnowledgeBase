# Audit SMB1
```
$servers = Get-ADComputer -Filter * -Properties OperatingSystem,OperatingSystemVersion,name,dnshostname | select name,dnshostname,OperatingSystem,OperatingSystemVersion,DistinguishedName | ? {$_.OperatingSystem -like "*server*"}

$smb = foreach ($computer in $servers.dnshostname) { 
    $res = Invoke-Command -ComputerName $computer -Command {
        $WinEvent = Get-WinEvent -ListLog * | ? {$_.logname -eq "Microsoft-Windows-SMBServer/Audit"}
        $SmbServer = get-SmbServerConfiguration | select AuditSmb1Access
        [pscustomobject] @{
            AuditSmb1Access = $SmbServer.AuditSmb1Access
            LogName = $WinEvent.LogName
            RecordCount = $WinEvent.RecordCount
        }
    } -ErrorAction SilentlyContinue | select PSComputerName, AuditSmb1Access, LogName, RecordCount
    if ($res) {
        $res
    }
    else {
        [pscustomobject]@{
            PSComputerName = $computer
            AuditSmb1Access = $null
            LogName = $null
            RecordCount = $null
        }
    }
}
$AuditServers = $smb.where({$_.recordcount -gt 0})
$AuditEvents = foreach ($computer in $AuditServers.PSComputerName) {
    $res = Invoke-Command -ComputerName $computer -Command {
        $WinEvents = Get-WinEvent -LogName "Microsoft-Windows-SMBServer/Audit"
        $SMB1Events = $WinEvents.where({$_.id -eq 3000})
        
        ForEach ($Event in $SMB1Events) {
            $eventXML = [xml]$Event.ToXml()
            
            [pscustomobject]@{
                ClientName = $eventXML.Event.EventData.Data.'#text'
                Server = $Event.MachineName
                TimeCreated = $Event.TimeCreated
            }
        }
    } -ErrorAction SilentlyContinue | select PSComputerName, ClientName, Server, TimeCreated
    
    if ($res) { $res }
}
```

# Azure FileSync
```
$WinEvents = Get-WinEvent -LogName "Microsoft-FileSync-Agent/RecallResults" -ComputerName <FileSyncAgent>
ForEach ($Event in $WinEvents) {
    $eventXML = [xml]$Event.ToXml()
    $hash = @{}
    $baseObjs = $eventXML.Event.EventData.data.PSObject.BaseObject
    ForEach ($obj in $baseObjs) {
        try {
             $hash[$obj.Name] = $obj.'#text'
             $hash['TimeCreated'] = $Event.TimeCreated
        }
        catch {}
    }
    try {
        New-Object PSObject -property $hash
    } 
    catch {}
}
```
