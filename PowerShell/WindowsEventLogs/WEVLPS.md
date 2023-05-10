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
