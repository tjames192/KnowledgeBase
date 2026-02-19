# PowerShell steps for PDQ or any..
```
# set windows to high performance
$HighPerformance = (powercfg /l).where({$_ -match 'High performance'})

if ($HighPerformance) {
 powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
 (powercfg /l).where({$_ -match 'High performance'})
} else {
 powercfg -duplicatescheme 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
 powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
 (powercfg /l).where({$_ -match 'High performance'})
}

```

```
# pdq file copy step copies to an intermediate directory (PDQDeployRunner)
# then copies to %temp%\win11

# this copies directly to temp
$src = "\\domain\repo\pdq\utilities\win11"
$dst = "$env:temp\win11"
$argList = "$src $dst *.* /S /E /MT:16 /DCOPY:D /COPY:DT /NP /XX /XO /XN /XC /R:0 /W:0"

$robocopy = start-process robocopy -argumentlist $argList -wait -NoNewWindow -PassThru

$robocopy

$ExitCode = $robocopy.ExitCode

# https://learn.microsoft.com/en-us/troubleshoot/windows-server/backup-and-storage/return-codes-used-robocopy-utility
if ($ExitCode -gt 1) {
# greater than 1 indicates a possible error
exit $ExitCode
}
else {
exit $ExitCode
}
```

```
$win11 = "$env:temp\win11\setup.exe"
Start-Process $win11 -ArgumentList '/quiet /noreboot /eula accept /migratedrivers all /Compat IgnoreWarning /Telemetry Disable /ImageIndex 1 /product server /BitLocker AlwaysSuspend /Compat IgnoreWarning /ShowOOBE none /DynamicUpdate NoDrivers /auto upgrade' -Wait
```

```
# reset performance to Balanced mode
$BalancePerformance = "381b4222-f694-41f0-9685-ff5bb260df2e" 
powercfg /setactive $BalancePerformance
(powercfg /l).where({$_ -like '*) *'})

```
