hyper-v switch embedded teams (set)

VMQ is now enabled at the virtual machine settings
*IMPORTANT update network drivers was key to making this work*

**Performance and Hardware Offloads**
Because the teaming is integrated in the Hyper-V switch, features like Virtual Machine Multi-Queue (VMMQ) and virtual RSS can distribute VM traffic efficiently across host CPUs. 
Additionally, SET supports RDMA (Remote Direct Memory Access) on teamed adapters, allowing you to converge high-speed storage traffic (SMB Direct) with VM traffic without sacrificing RDMA’s low latency.

**Fault Tolerance and Resiliency**
SET provides NIC-level fault tolerance. If one physical NIC in the team fails or is disconnected, the Hyper-V virtual switch remains up on the remaining NIC(s) without interrupting the VMs’ connectivity.

**Simplified Management and Fewer Layers**
SET reduces complexity by removing one networking layer from the host
SET allows you to create one object – a Hyper-V vSwitch with embedded team members

```powershell

New-VMSwitch -Name "SETvSwitch" -NetAdapterName "Ethernet","Ethernet 2" -EnableEmbeddedTeaming $true -AllowManagementOS $false

Name       SwitchType NetAdapterInterfaceDescription
----       ---------- ------------------------------
SETvSwitch External   Teamed-Interface


Get-VMSwitchTeam -Name "SETvSwitch" | Select-Object LoadBalancingAlgorithm, TeamingMode

LoadBalancingAlgorithm       TeamingMode
----------------------       -----------
            HyperVPort SwitchIndependent
```

if you want to create seperate networkadapters (like team interfaces in NIC teaming) for specific VLANS you could use the following PowerShell cmdlets, for example if we want to use VLANID 101 as seperate NIC

```powershell
# example vlan2
Add-VMNetworkAdapter -ManagementOS:$false -Name "VLAN2" -SwitchName "SETvSwitch"
Set-VMNetworkAdapterVLAN -VMNetworkAdapterName "VLAN2" -vlanid 2 -Access
```

had problems removing set vswitch i want re-create

-In the PowerShell console, type in the command below and hit Enter to restart all network adapters and remove MUX objects.

netcfg -d

```powershell
Set-VMSwitchTeam -LoadBalancingAlgorithm HyperVPort

Get-WmiObject -Class Win32_Processor | select Name,SocketDesignation,NumberOfCores,NumberOfLogicalProcessors

Name                                      SocketDesignation NumberOfCores NumberOfLogicalProcessors
----                                      ----------------- ------------- -------------------------
Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz Proc 1                        6                        12
Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz Proc 2                        6                        12

(Get-VMHostNumaNode).count
1

get-NetAdapterVMQ

Name                           InterfaceDescription              Enabled BaseVmqProcessor MaxProcessors NumberOfReceive
                                                                                                        Queues
----                           --------------------              ------- ---------------- ------------- ---------------
Ethernet                       HP Ethernet 10Gb 2-port 557SFP... True    0:0                            31
Ethernet 2                     HP Ethernet 10Gb 2-port 557S...#2 True    0:0                            31


```

|            | p   | l   | p   | l   | p   | l   | p   | l   | p   | l   | p   | l   | p   | l   | p   | l   | p   | l   | p   |
| ---------- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| logical    | 0   | 1   | 2   | 3   | 4   | 5   | 6   | 7   | 8   | 9   | 10  | 11  | 12  | 13  | 14  | 15  | 16  |     |     |
| socket     | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 2   | 2   | 2   | 2   | 2   |     |     |
| numa       | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   | 1   |     |     |
| ethernet   |     |     | x   |     | x   |     | x   |     | x   |     | x   |     |     |     |     |     |     |     |     |
| ethernet 2 |     |     |     |     |     |     |     |     |     |     |     |     |     |     | x   |     | x   |     |     |

```powershell
Set-NetAdapterVMQ -Name Ethernet -BaseProcessorNumber 2
Set-NetAdapterVMQ -Name "Ethernet 2" -BaseProcessorNumber 14

PS C:\Windows\system32> get-NetAdapterVMQ

Name                           InterfaceDescription              Enabled BaseVmqProcessor MaxProcessors NumberOfReceiveQ
                                                                                                        ueues
----                           --------------------              ------- ---------------- ------------- ----------------
Ethernet                       HP Ethernet 10Gb 2-port 557SFP... True    0:2                            31
Ethernet 2                     HP Ethernet 10Gb 2-port 557S...#2 True    0:14                           31
```


![[Pasted image 20250917152004.png]]

![[Pasted image 20250917152118.png]]

https://172.16.2.159/
![[Pasted image 20250917152022.png]]


# Poor Network Performance on Windows Server 2019 Hyper-V
https://www.justinho.com/blog/2020/05/30/Windows-Server-2019-vRSS-Hyper-V.html
For now I dont recommend disabling vrss or RSC yet
Since we have vmq working as intended is disabling vrss or rsc recommended in this scenario .. probably not

```powershell
PS C:\Windows\system32> Get-NetTCPSetting | ft -AutoSize

SettingName      CongestionProvider MinRto(ms) InitialRto(ms) CwndRestart DelayedAckTimeout DelayedAckFrequency AutoTuningLevelEffective
-----------      ------------------ ---------- -------------- ----------- ----------------- ------------------- ------------------------
Automatic
InternetCustom   CUBIC                     300           3000 False                      40                   2 Local
DatacenterCustom CUBIC                      20           3000 False                      10                   2 Local
Compat           NewReno                   300           3000 False                     200                   2 Local
Datacenter       CUBIC                      20           3000 False                      10                   2 Local
Internet         CUBIC                     300           3000 False                      40                   2 Local

PS C:\Windows\system32> Set-NetTCPSetting -SettingName "InternetCustom" -CongestionProvider CTCP
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "InternetCustom" -DelayedAckTimeoutMs 50
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "InternetCustom" -ForceWS Disabled
PS C:\Windows\system32>
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "DatacenterCustom" -CongestionProvider DCTCP
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "DatacenterCustom" -CwndRestart True
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "DatacenterCustom" -ForceWS Disabled
PS C:\Windows\system32>
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "Compat" -ForceWS Disabled
PS C:\Windows\system32>
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "Datacenter" -CongestionProvider DCTCP
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "Datacenter" -CwndRestart True
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "Datacenter" -ForceWS Disabled
PS C:\Windows\system32>
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "Internet" -CongestionProvider CTCP
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "Internet" -DelayedAckTimeoutMs 50
PS C:\Windows\system32>  Set-NetTCPSetting -SettingName "Internet" -ForceWS Disabled
PS C:\Windows\system32> Get-NetTCPSetting | ft -AutoSize

SettingName      CongestionProvider MinRto(ms) InitialRto(ms) CwndRestart DelayedAckTimeout DelayedAckFrequency AutoTuningLevelEffective
-----------      ------------------ ---------- -------------- ----------- ----------------- ------------------- ------------------------
Automatic
InternetCustom   CTCP                      300           3000 False                      50                   2 Local
DatacenterCustom DCTCP                      20           3000 True                       10                   2 Local
Compat           NewReno                   300           3000 False                     200                   2 Local
Datacenter       DCTCP                      20           3000 True                       10                   2 Local
Internet         CTCP                      300           3000 False                      50                   2 Local
```

settings not enabled

``` powershell
 #RUN ONLY ON NATIVE2019, HV2019, VM2019
 #Disable RSS & RSC on the TCP-Stack 
 #netsh int tcp show global
 netsh int tcp set global RSS=Disabled
 netsh int tcp set global RSC=Disabled

 #RUN ONLY ON HV2019
 #Disable Software RSC on all vSwitches
 Get-VMSwitch | Set-VMSwitch -EnableSoftwareRsc $false
    
 #Disable vRSS on all VMs on the host
 Get-VM | Set-VMNetworkAdapter -VrssEnabled $FALSE
```
