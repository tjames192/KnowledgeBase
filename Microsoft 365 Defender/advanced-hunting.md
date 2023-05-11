# Advanced Hunting

## Web Content Filtering Searches

```
DeviceEvents
| where ActionType in ('SmartScreenUrlWarning','ExploitGuardNetworkProtectionBlocked')
| extend ParsedFields=parse_json(AdditionalFields)
| project DeviceName, ActionType, Timestamp, RemoteUrl, InitiatingProcessAccountName, ResponseCategory=tostring(ParsedFields.ResponseCategory),Experience=tostring(ParsedFields.Experience)
```

## Additional Information
https://samilamppu.com/2021/11/02/microsoft-defender-for-endpoint-web-content-filtering-test-drive/
