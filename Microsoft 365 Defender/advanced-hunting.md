# Advanced Hunting

## Web Content Filtering Searches

```
DeviceEvents
| where ActionType in ('SmartScreenUrlWarning','ExploitGuardNetworkProtectionBlocked')
| extend ParsedFields=parse_json(AdditionalFields)
| project DeviceName, ActionType, Timestamp, RemoteUrl, InitiatingProcessAccountName, ResponseCategory=tostring(ParsedFields.ResponseCategory),Experience=tostring(ParsedFields.Experience)
```

```
//SmartScreenBlocks or ExploitGuardNetworkProtectionBlocked which is 3rd party process or browser
DeviceEvents
| where ActionType in ('SmartScreenUrlWarning','ExploitGuardNetworkProtectionBlocked')
| extend ParsedFields=parse_json(AdditionalFields)
| project DeviceName, ActionType, Timestamp, RemoteUrl, InitiatingProcessAccountName, ResponseCategory=tostring(ParsedFields.ResponseCategory),Experience=tostring(ParsedFields.Experience)
| summarize Total = count() by RemoteUrl, ActionType, ResponseCategory
```
```
DeviceNetworkEvents
| where RemoteUrl has_any ("bankofamerica")
| extend ParsedFields=parse_json(AdditionalFields)
| order by Timestamp desc
| project Timestamp, ActionType, DeviceName, InitiatingProcessAccountName, InitiatingProcessAccountUpn, LocalIP, RemoteIP, LocalPort, RemotePort, RemoteUrl
```


## Additional Information
https://samilamppu.com/2021/11/02/microsoft-defender-for-endpoint-web-content-filtering-test-drive/

## fooUser@domain.onmicrosoft.com - InitiatingProcessAccountUpn
worth mentioning that the foouser is also present on devices that are Hybrid joined and then enrolled into intune via group policy (Even when set to use user credentials)
https://epmstuff.wordpress.com/

