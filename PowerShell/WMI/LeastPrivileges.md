# automate least privilege permissions for servers

https://www.logicmonitor.com/support/getting-started/advanced-logicmonitor-setup/windows-server-monitoring-and-principle-of-least-privilege

https://softcomet.freshdesk.com/support/solutions/articles/6000222183-using-wmi-without-having-full-administrator-permissions

## set least privileges

``` powershell
function SetLeastPrivileges {
Param (
        [switch]$add,
        [switch]$remove,
        [switch]$help,
        [Parameter(Mandatory=$False)][string] $UserName,
        [Parameter(Mandatory=$False)][string] $Password,
        [Parameter(Mandatory=$False)][string] $Path
)

## PowerShell Version Check
$version = $PSVersionTable.PSVersion.Major
if ([int]$version -lt 5) {
  Write-Output "PowerShell version 5 is the minimum mandatory requirement to run this script"
  exit
}

if ($help -eq $true) {
  Write-Output "Usage : [-help|-add|-remove][-UserName userName][-Password password][-Path path]
                  -help         Help Prompt     - Show this message.
                  -add|-remove  Operation Flag  - Operation Flag is required '-add' for adding and '-remove' for reversal.
                  -UserName     Non-Admin User  - Name of Non-Admin User under which want to move Collector services. Mandatory when not '-help'.
                  -Password     Password        - Password of the Non-Admin User.
                  -Path         Install path    - Installation path of the Collector (default: `"C:\Program Files\LogicMonitor`")

                  Example 0 : .\Windows_NonAdmin_Config.ps1 -help
                  Example 1 : .\Windows_NonAdmin_Config.ps1 -add -UserName LOGICMONITOR\MyUserName
                  Example 2 : .\Windows_NonAdmin_Config.ps1 -remove -UserName LOGICMONITOR\MyUserName -Password MySecurePassword -Path `"C:\Program Files\Path\LogicMonitor`"
                  "
   exit
}

if (($help -eq $false) -and ($UserName -eq "" -or $UserName -eq $null)) {
    Write-Output "UserName is mandatory argument, Kindly pass correct UserName. Check `".\Windows_NonAdmin_Config.ps1 -help`" for further help."
  exit
}

Add-Type -TypeDefinition @'
using System;
namespace PS_LSA
{
    using System.ComponentModel;
    using System.Runtime.InteropServices;
    using System.Security;
    using System.Security.Principal;
    using LSA_HANDLE = IntPtr;

    public enum Rights
    {
        SeTrustedCredManAccessPrivilege,             // Access Credential Manager as a trusted caller
        SeNetworkLogonRight,                         // Access this computer from the network
        SeTcbPrivilege,                              // Act as part of the operating system
        SeMachineAccountPrivilege,                   // Add workstations to domain
        SeIncreaseQuotaPrivilege,                    // Adjust memory quotas for a process
        SeInteractiveLogonRight,                     // Allow log on locally
        SeRemoteInteractiveLogonRight,               // Allow log on through Remote Desktop Services
        SeBackupPrivilege,                           // Back up files and directories
        SeChangeNotifyPrivilege,                     // Bypass traverse checking
        SeSystemtimePrivilege,                       // Change the system time
        SeTimeZonePrivilege,                         // Change the time zone
        SeCreatePagefilePrivilege,                   // Create a pagefile
        SeCreateTokenPrivilege,                      // Create a token object
        SeCreateGlobalPrivilege,                     // Create global objects
        SeCreatePermanentPrivilege,                  // Create permanent shared objects
        SeCreateSymbolicLinkPrivilege,               // Create symbolic links
        SeDebugPrivilege,                            // Debug programs
        SeDenyNetworkLogonRight,                     // Deny access this computer from the network
        SeDenyBatchLogonRight,                       // Deny log on as a batch job
        SeDenyServiceLogonRight,                     // Deny log on as a service
        SeDenyInteractiveLogonRight,                 // Deny log on locally
        SeDenyRemoteInteractiveLogonRight,           // Deny log on through Remote Desktop Services
        SeEnableDelegationPrivilege,                 // Enable computer and user accounts to be trusted for delegation
        SeRemoteShutdownPrivilege,                   // Force shutdown from a remote system
        SeAuditPrivilege,                            // Generate security audits
        SeImpersonatePrivilege,                      // Impersonate a client after authentication
        SeIncreaseWorkingSetPrivilege,               // Increase a process working set
        SeIncreaseBasePriorityPrivilege,             // Increase scheduling priority
        SeLoadDriverPrivilege,                       // Load and unload device drivers
        SeLockMemoryPrivilege,                       // Lock pages in memory
        SeBatchLogonRight,                           // Log on as a batch job
        SeServiceLogonRight,                         // Log on as a service
        SeSecurityPrivilege,                         // Manage auditing and security log
        SeRelabelPrivilege,                          // Modify an object label
        SeSystemEnvironmentPrivilege,                // Modify firmware environment values
        SeDelegateSessionUserImpersonatePrivilege,   // Obtain an impersonation token for another user in the same session
        SeManageVolumePrivilege,                     // Perform volume maintenance tasks
        SeProfileSingleProcessPrivilege,             // Profile single process
        SeSystemProfilePrivilege,                    // Profile system performance
        SeUnsolicitedInputPrivilege,                 // "Read unsolicited input from a terminal device"
        SeUndockPrivilege,                           // Remove computer from docking station
        SeAssignPrimaryTokenPrivilege,               // Replace a process level token
        SeRestorePrivilege,                          // Restore files and directories
        SeShutdownPrivilege,                         // Shut down the system
        SeSyncAgentPrivilege,                        // Synchronize directory service data
        SeTakeOwnershipPrivilege                     // Take ownership of files or other objects
    }

    [StructLayout(LayoutKind.Sequential)]
    struct LSA_OBJECT_ATTRIBUTES
    {
        internal int Length;
        internal IntPtr RootDirectory;
        internal IntPtr ObjectName;
        internal int Attributes;
        internal IntPtr SecurityDescriptor;
        internal IntPtr SecurityQualityOfService;
    }

    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    struct LSA_UNICODE_STRING
    {
        internal ushort Length;
        internal ushort MaximumLength;
        [MarshalAs(UnmanagedType.LPWStr)]
        internal string Buffer;
    }

    [StructLayout(LayoutKind.Sequential)]
    struct LSA_ENUMERATION_INFORMATION
    {
        internal IntPtr PSid;
    }

    internal sealed class Win32Sec
    {
        [DllImport("advapi32", CharSet = CharSet.Unicode, SetLastError = true)]
        internal static extern uint LsaOpenPolicy(
            LSA_UNICODE_STRING[] SystemName,
            ref LSA_OBJECT_ATTRIBUTES ObjectAttributes,
            int AccessMask,
            out IntPtr PolicyHandle
        );

        [DllImport("advapi32", CharSet = CharSet.Unicode, SetLastError = true)]
        internal static extern uint LsaAddAccountRights(
            LSA_HANDLE PolicyHandle,
            IntPtr pSID,
            LSA_UNICODE_STRING[] UserRights,
            int CountOfRights
        );

        [DllImport("advapi32", CharSet = CharSet.Unicode, SetLastError = true)]
        internal static extern uint LsaRemoveAccountRights(
            LSA_HANDLE PolicyHandle,
            IntPtr pSID,
            bool AllRights,
            LSA_UNICODE_STRING[] UserRights,
            int CountOfRights
        );

        [DllImport("advapi32", CharSet = CharSet.Unicode, SetLastError = true)]
        internal static extern uint LsaEnumerateAccountRights(
            LSA_HANDLE PolicyHandle,
            IntPtr pSID,
            out IntPtr /*LSA_UNICODE_STRING[]*/ UserRights,
            out ulong CountOfRights
        );

        [DllImport("advapi32", CharSet = CharSet.Unicode, SetLastError = true)]
        internal static extern uint LsaEnumerateAccountsWithUserRight(
            LSA_HANDLE PolicyHandle,
            LSA_UNICODE_STRING[] UserRights,
            out IntPtr EnumerationBuffer,
            out ulong CountReturned
        );

        [DllImport("advapi32")]
        internal static extern int LsaNtStatusToWinError(int NTSTATUS);

        [DllImport("advapi32")]
        internal static extern int LsaClose(IntPtr PolicyHandle);

        [DllImport("advapi32")]
        internal static extern int LsaFreeMemory(IntPtr Buffer);
    }

    internal sealed class Sid : IDisposable
    {
        public IntPtr pSid = IntPtr.Zero;
        public SecurityIdentifier sid = null;

        public Sid(string account)
        {
            try { sid = new SecurityIdentifier(account); }
            catch { sid = (SecurityIdentifier)(new NTAccount(account)).Translate(typeof(SecurityIdentifier)); }
            Byte[] buffer = new Byte[sid.BinaryLength];
            sid.GetBinaryForm(buffer, 0);

            pSid = Marshal.AllocHGlobal(sid.BinaryLength);
            Marshal.Copy(buffer, 0, pSid, sid.BinaryLength);
        }

        public void Dispose()
        {
            if (pSid != IntPtr.Zero)
            {
                Marshal.FreeHGlobal(pSid);
                pSid = IntPtr.Zero;
            }
            GC.SuppressFinalize(this);
        }
        ~Sid() { Dispose(); }
    }

    public sealed class LsaWrapper : IDisposable
    {
        enum Access : int
        {
            POLICY_READ = 0x20006,
            POLICY_ALL_ACCESS = 0x00F0FFF,
            POLICY_EXECUTE = 0X20801,
            POLICY_WRITE = 0X207F8
        }
        const uint STATUS_ACCESS_DENIED = 0xc0000022;
        const uint STATUS_INSUFFICIENT_RESOURCES = 0xc000009a;
        const uint STATUS_NO_MEMORY = 0xc0000017;
        const uint STATUS_OBJECT_NAME_NOT_FOUND = 0xc0000034;
        const uint STATUS_NO_MORE_ENTRIES = 0x8000001a;

        IntPtr lsaHandle;

        public LsaWrapper() : this(null) { } // local system if systemName is null
        public LsaWrapper(string systemName)
        {
            LSA_OBJECT_ATTRIBUTES lsaAttr;
            lsaAttr.RootDirectory = IntPtr.Zero;
            lsaAttr.ObjectName = IntPtr.Zero;
            lsaAttr.Attributes = 0;
            lsaAttr.SecurityDescriptor = IntPtr.Zero;
            lsaAttr.SecurityQualityOfService = IntPtr.Zero;
            lsaAttr.Length = Marshal.SizeOf(typeof(LSA_OBJECT_ATTRIBUTES));
            lsaHandle = IntPtr.Zero;
            LSA_UNICODE_STRING[] system = null;
            if (systemName != null)
            {
                system = new LSA_UNICODE_STRING[1];
                system[0] = InitLsaString(systemName);
            }

            uint ret = Win32Sec.LsaOpenPolicy(system, ref lsaAttr, (int)Access.POLICY_ALL_ACCESS, out lsaHandle);
            if (ret == 0) return;
            if (ret == STATUS_ACCESS_DENIED) throw new UnauthorizedAccessException();
            if ((ret == STATUS_INSUFFICIENT_RESOURCES) || (ret == STATUS_NO_MEMORY)) throw new OutOfMemoryException();
            throw new Win32Exception(Win32Sec.LsaNtStatusToWinError((int)ret));
        }

        public void AddPrivilege(string account, Rights privilege)
        {
            uint ret = 0;
            using (Sid sid = new Sid(account))
            {
                LSA_UNICODE_STRING[] privileges = new LSA_UNICODE_STRING[1];
                privileges[0] = InitLsaString(privilege.ToString());
                ret = Win32Sec.LsaAddAccountRights(lsaHandle, sid.pSid, privileges, 1);
            }
            if (ret == 0) return;
            if (ret == STATUS_ACCESS_DENIED) throw new UnauthorizedAccessException();
            if ((ret == STATUS_INSUFFICIENT_RESOURCES) || (ret == STATUS_NO_MEMORY)) throw new OutOfMemoryException();
            throw new Win32Exception(Win32Sec.LsaNtStatusToWinError((int)ret));
        }

        public void RemovePrivilege(string account, Rights privilege)
        {
            uint ret = 0;
            using (Sid sid = new Sid(account))
            {
                LSA_UNICODE_STRING[] privileges = new LSA_UNICODE_STRING[1];
                privileges[0] = InitLsaString(privilege.ToString());
                ret = Win32Sec.LsaRemoveAccountRights(lsaHandle, sid.pSid, false, privileges, 1);
            }
            if (ret == 0) return;
            if (ret == STATUS_ACCESS_DENIED) throw new UnauthorizedAccessException();
            if ((ret == STATUS_INSUFFICIENT_RESOURCES) || (ret == STATUS_NO_MEMORY)) throw new OutOfMemoryException();
            throw new Win32Exception(Win32Sec.LsaNtStatusToWinError((int)ret));
        }

        public Rights[] EnumerateAccountPrivileges(string account)
        {
            uint ret = 0;
            ulong count = 0;
            IntPtr privileges = IntPtr.Zero;
            Rights[] rights = null;

            using (Sid sid = new Sid(account))
            {
                ret = Win32Sec.LsaEnumerateAccountRights(lsaHandle, sid.pSid, out privileges, out count);
            }
            if (ret == 0)
            {
                rights = new Rights[count];
                for (int i = 0; i < (int)count; i++)
                {
                    LSA_UNICODE_STRING str = (LSA_UNICODE_STRING)Marshal.PtrToStructure(
                        IntPtr.Add(privileges, i * Marshal.SizeOf(typeof(LSA_UNICODE_STRING))),
                        typeof(LSA_UNICODE_STRING));
                    rights[i] = (Rights)Enum.Parse(typeof(Rights), str.Buffer);
                }
                Win32Sec.LsaFreeMemory(privileges);
                return rights;
            }
            if (ret == STATUS_OBJECT_NAME_NOT_FOUND) return null;  // No privileges assigned
            if (ret == STATUS_ACCESS_DENIED) throw new UnauthorizedAccessException();
            if ((ret == STATUS_INSUFFICIENT_RESOURCES) || (ret == STATUS_NO_MEMORY)) throw new OutOfMemoryException();
            throw new Win32Exception(Win32Sec.LsaNtStatusToWinError((int)ret));
        }

        public string[] EnumerateAccountsWithUserRight(Rights privilege, bool resolveSid = true)
        {
            uint ret = 0;
            ulong count = 0;
            LSA_UNICODE_STRING[] rights = new LSA_UNICODE_STRING[1];
            rights[0] = InitLsaString(privilege.ToString());
            IntPtr buffer = IntPtr.Zero;
            string[] accounts = null;

            ret = Win32Sec.LsaEnumerateAccountsWithUserRight(lsaHandle, rights, out buffer, out count);
            if (ret == 0)
            {
                accounts = new string[count];
                for (int i = 0; i < (int)count; i++)
                {
                    LSA_ENUMERATION_INFORMATION LsaInfo = (LSA_ENUMERATION_INFORMATION)Marshal.PtrToStructure(
                        IntPtr.Add(buffer, i * Marshal.SizeOf(typeof(LSA_ENUMERATION_INFORMATION))),
                        typeof(LSA_ENUMERATION_INFORMATION));

                        if (resolveSid) {
                            try {
                                accounts[i] = (new SecurityIdentifier(LsaInfo.PSid)).Translate(typeof(NTAccount)).ToString();
                            } catch (System.Security.Principal.IdentityNotMappedException) {
                                accounts[i] = (new SecurityIdentifier(LsaInfo.PSid)).ToString();
                            }
                        } else { accounts[i] = (new SecurityIdentifier(LsaInfo.PSid)).ToString(); }
                }
                Win32Sec.LsaFreeMemory(buffer);
                return accounts;
            }
            if (ret == STATUS_NO_MORE_ENTRIES) return null;  // No accounts assigned
            if (ret == STATUS_ACCESS_DENIED) throw new UnauthorizedAccessException();
            if ((ret == STATUS_INSUFFICIENT_RESOURCES) || (ret == STATUS_NO_MEMORY)) throw new OutOfMemoryException();
            throw new Win32Exception(Win32Sec.LsaNtStatusToWinError((int)ret));
        }

        public void Dispose()
        {
            if (lsaHandle != IntPtr.Zero)
            {
                Win32Sec.LsaClose(lsaHandle);
                lsaHandle = IntPtr.Zero;
            }
            GC.SuppressFinalize(this);
        }
        ~LsaWrapper() { Dispose(); }

        // helper functions:
        static LSA_UNICODE_STRING InitLsaString(string s)
        {
            // Unicode strings max. 32KB
            if (s.Length > 0x7ffe) throw new ArgumentException("String too long");
            LSA_UNICODE_STRING lus = new LSA_UNICODE_STRING();
            lus.Buffer = s;
            lus.Length = (ushort)(s.Length * sizeof(char));
            lus.MaximumLength = (ushort)(lus.Length + sizeof(char));
            return lus;
        }
    }

    public sealed class TokenManipulator
    {
        [StructLayout(LayoutKind.Sequential, Pack = 1)]
        internal struct TokPriv1Luid
        {
            public int Count;
            public long Luid;
            public int Attr;
        }

        internal const int SE_PRIVILEGE_DISABLED = 0x00000000;
        internal const int SE_PRIVILEGE_ENABLED = 0x00000002;
        internal const int TOKEN_QUERY = 0x00000008;
        internal const int TOKEN_ADJUST_PRIVILEGES = 0x00000020;

        internal sealed class Win32Token
        {
            [DllImport("advapi32.dll", ExactSpelling = true, SetLastError = true)]
            internal static extern bool AdjustTokenPrivileges(
                IntPtr htok,
                bool disall,
                ref TokPriv1Luid newst,
                int len,
                IntPtr prev,
                IntPtr relen
            );

            [DllImport("kernel32.dll", ExactSpelling = true)]
            internal static extern IntPtr GetCurrentProcess();

            [DllImport("advapi32.dll", ExactSpelling = true, SetLastError = true)]
            internal static extern bool OpenProcessToken(
                IntPtr h,
                int acc,
                ref IntPtr phtok
            );

            [DllImport("advapi32.dll", SetLastError = true)]
            internal static extern bool LookupPrivilegeValue(
                string host,
                string name,
                ref long pluid
            );

            [DllImport("kernel32.dll", ExactSpelling = true)]
            internal static extern bool CloseHandle(
                IntPtr phtok
            );
        }

        public static void AddPrivilege(Rights privilege)
        {
            bool retVal;
            int lasterror;
            TokPriv1Luid tp;
            IntPtr hproc = Win32Token.GetCurrentProcess();
            IntPtr htok = IntPtr.Zero;
            retVal = Win32Token.OpenProcessToken(hproc, TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, ref htok);
            tp.Count = 1;
            tp.Luid = 0;
            tp.Attr = SE_PRIVILEGE_ENABLED;
            retVal = Win32Token.LookupPrivilegeValue(null, privilege.ToString(), ref tp.Luid);
            retVal = Win32Token.AdjustTokenPrivileges(htok, false, ref tp, Marshal.SizeOf(tp), IntPtr.Zero, IntPtr.Zero);
            Win32Token.CloseHandle(htok);
            lasterror = Marshal.GetLastWin32Error();
            if (lasterror != 0) throw new Win32Exception();
        }

        public static void RemovePrivilege(Rights privilege)
        {
            bool retVal;
            int lasterror;
            TokPriv1Luid tp;
            IntPtr hproc = Win32Token.GetCurrentProcess();
            IntPtr htok = IntPtr.Zero;
            retVal = Win32Token.OpenProcessToken(hproc, TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, ref htok);
            tp.Count = 1;
            tp.Luid = 0;
            tp.Attr = SE_PRIVILEGE_DISABLED;
            retVal = Win32Token.LookupPrivilegeValue(null, privilege.ToString(), ref tp.Luid);
            retVal = Win32Token.AdjustTokenPrivileges(htok, false, ref tp, Marshal.SizeOf(tp), IntPtr.Zero, IntPtr.Zero);
            Win32Token.CloseHandle(htok);
            lasterror = Marshal.GetLastWin32Error();
            if (lasterror != 0) throw new Win32Exception();
        }
    }
}
'@ # This type (PS_LSA) is used by Grant-UserRight, Revoke-UserRight, Get-UserRightsGrantedToAccount, Get-AccountsWithUserRight, Grant-TokenPriviledge, Revoke-TokenPrivilege

#Variables used for Logging Purpose.
#PART 1
$global:nameSpace=0
#PART 2
$global:addUser=0
#PART 3
$global:secDesc=0
$global:skipService=0
$global:secDescScManager=0
$global:skipSecDescScManager=0
#PART 4
$agentServiceName = "logicmonitor-agent"
$watchdogServiceName = "logicmonitor-watchdog"

$Timestamp = (Get-Date).ToString("yyyy-MM-dd-HH-mm-ss")
$LogFileName = "WindowsNonAdminConfigLogs-$Timestamp.txt"

New-Item -ItemType dir -Path "$env:APPDATA\LogicMonitor\Logs" -ErrorAction SilentlyContinue
$logPath = "$env:APPDATA\LogicMonitor\Logs"

Function Log-Message ([String]$Message) {
    Add-Content -Path "$logPath\$LogFileName" $Message
}

$isDomainPresent=$true
$pos = $UserName.IndexOf("\")

if ($pos -eq -1) {
    $isDomainPresent=$false
}

if ($isDomainPresent) {
    $DomainName = $UserName.Substring(0, $pos)
    $UserName = $UserName.Substring($pos + 1)
}

$ComputerName = [System.Net.Dns]::GetHostByName($env:computerName).HostName

$TimeStampFormat = "yyyy-MM-dd HH:mm:ss.fff:zzz"
$Timestamp = (Get-Date).ToString($TimeStampFormat)
Log-Message "[$Timestamp] ComputerName = $ComputerName"
Log-Message "[$Timestamp] Selected Domain = $DomainName"
Log-Message "[$Timestamp] Selected UserName = $UserName"

$isCollectorHost=$false

$agentService = Get-Service -Name $agentServiceName -ErrorAction SilentlyContinue
$watchdogService = Get-Service -Name $watchdogServiceName -ErrorAction SilentlyContinue
if (($agentService.Length -gt 0) -and ($watchdogService.Length -gt 0)) {
    $isCollectorHost=$true
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] LogicMonitor Agent and Watchdog Services found on this host $ComputerName"
}

## FUNCTION FOR PART 1
## Function used to set the permissions for WMI-namespace-security
Function Set-WmiNamespaceSecurity {
    [cmdletBinding()]
    Param (
        [parameter(Mandatory=$true,Position=0)][string] $namespace,
        [parameter(Mandatory=$true,Position=1)][string] $operation,
        [parameter(Mandatory=$true,Position=2)][string] $account,
        [parameter(Position=3)][string[]] $permissions = $null,
        [bool] $allowInherit = $true,
        [bool] $deny = $false,
        [string] $computer = ".",
        [System.Management.Automation.PSCredential] $credential = $null
    )

    Process {
        $ErrorActionPreference = "Stop"
        $global:nameSpace=0

        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] Set-WmiNamespaceSecurity call with $operation operation"

        Function Get-AccessMaskFromPermission ($permissions) {

        #NameSpace Access Rights COnstants https://docs.microsoft.com/en-us/windows/win32/wmisdk/namespace-access-rights-constants
            $WBEM_ENABLE = 1
            $WBEM_METHOD_EXECUTE = 2
            $WBEM_FULL_WRITE_REP = 4
            $WBEM_PARTIAL_WRITE_REP = 8
            $WBEM_WRITE_PROVIDER = 0x10
            $WBEM_REMOTE_ACCESS = 0x20
            $WBEM_RIGHT_SUBSCRIBE = 0x40
            $WBEM_RIGHT_PUBLISH = 0x80
            $READ_CONTROL = 0x20000
            $WRITE_DAC = 0x40000

           #All possible permissions for Namespace Security.
            $WBEM_RIGHTS_FLAGS = $WBEM_ENABLE, $WBEM_METHOD_EXECUTE, $WBEM_FULL_WRITE_REP, $WBEM_PARTIAL_WRITE_REP,
                                 $WBEM_WRITE_PROVIDER, $WBEM_REMOTE_ACCESS, $READ_CONTROL, $WRITE_DAC
            $WBEM_RIGHTS_STRINGS = "Enable", "MethodExecute", "FullWrite", "PartialWrite", "ProviderWrite", "RemoteAccess", "ReadSecurity", "WriteSecurity"

            $permissionTable = @{}

            for ($i = 0; $i -lt $WBEM_RIGHTS_FLAGS.Length; $i++) {
                $permissionTable.Add($WBEM_RIGHTS_STRINGS[$i].ToLower(), $WBEM_RIGHTS_FLAGS[$i])
            }

            $accessMask = 0

            foreach ($permission in $permissions) {
                if (-not $permissionTable.ContainsKey($permission.ToLower())) {
                    throw "Unknown permission: $permission</code>nValid permissions: $($permissionTable.Keys)"
                }
                $accessMask += $permissionTable[$permission.ToLower()]
            }
            $accessMask
        }

        $invokeparams = @{Namespace=$namespace;Path="__systemsecurity=@"}

        try {
            #Get Security Descriptor
            $output = Invoke-WmiMethod @invokeparams -Name GetSecurityDescriptor
            if ($output -eq $null) {
                throw "Failed to retrieve security descriptor. Output is null."
            }
        } catch {
            $errorMessage = "Error occurred: $_"
            Write-Output $errorMessage
            Write-Output "Error occurred in GetSecurityDescriptor step. Kindly retry or do the setup manually."
            $Timestamp = (Get-Date).ToString($TimeStampFormat)
            Log-Message "[$Timestamp] $errorMessage"
            Log-Message "[$Timestamp] Error occurred in GetSecurityDescriptor step. Kindly retry or do the setup manually."
            exit 1
        }

        #Descriptor and flags for inheritance.
        #NameSpace Ace Flags-Constans https://docs.microsoft.com/en-us/windows/win32/wmisdk/namespace-ace-flag-constants
        $acl = $output.Descriptor
        $OBJECT_INHERIT_ACE_FLAG = 0x1
        $CONTAINER_INHERIT_ACE_FLAG = 0x2

        # Create a NTAccount object
        $ntAccount = New-Object System.Security.Principal.NTAccount($account)

        # Get the SID of the account
        $sid = $ntAccount.Translate([System.Security.Principal.SecurityIdentifier]).Value

        if ($sid -eq $null) {
            throw "Empty SID, Account $account was not found. Kindly check and try again."
        }

        try {
            if ($operation -eq "add") {
                if ($permissions -eq $null) {
                    throw "-Permissions must be specified for an $operation operation"
                }

                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] Try setting WMI Namespace security for computer = $computer"

                #Get all the permissions that need to added.
                $accessMask = Get-AccessMaskFromPermission($permissions)

                $ace = (New-Object System.Management.ManagementClass("win32_Ace")).CreateInstance()
                $ace.AccessMask = $accessMask
                if ($allowInherit) {
                    $ace.AceFlags =  $CONTAINER_INHERIT_ACE_FLAG
                } else {
                    $ace.AceFlags = 0
                }

                $trustee = (New-Object System.Management.ManagementClass("win32_Trustee")).CreateInstance()
                $trustee.SidString = $sid
                $ace.Trustee = $trustee

               # Namespace ACE-TYPE-CONSTANTS. https://docs.microsoft.com/en-us/windows/win32/wmisdk/namespace-ace-type-constants
                $ACCESS_ALLOWED_ACE_TYPE = 0x0
                $ACCESS_DENIED_ACE_TYPE = 0x1

                if ($deny) {
                    $ace.AceType = $ACCESS_DENIED_ACE_TYPE
                } else {
                # Allow This Namespace and subnamespace rights.
                    $ace.AceType = $ACCESS_ALLOWED_ACE_TYPE
                }

                $acl.DACL += $ace.psobject.immediateBaseObject
            } else {
                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] Try unsetting WMI Namespace security for computer = $computer"

                [System.Management.ManagementBaseObject[]]$newDACL = @()
                foreach ($ace in $acl.DACL) {
                    if ($ace.Trustee.SidString -ne $sid) {
                        $newDACL += $ace.psobject.immediateBaseObject
                    }
                }

                $acl.DACL = $newDACL.psobject.immediateBaseObject
            }
        } catch {
            if ($operation -eq "add") {
                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] Error in Set-WmiNamespaceSecurity call for computer = $Computer."
                Log-Message "[$Timestamp] Error Message : $PSItem.Exception.Messages"
                Log-Message "[$Timestamp] Error in Line Number: $PSItem.InvocationInfo.ScriptLineNumber"
                Log-Message "[$Timestamp] Script Stack Trace : "
                Log-Message $_.ScriptStackTrace
            } else {
                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] Error in Set-WmiNamespaceSecurity call for computer = $Computer."
                Log-Message "[$Timestamp] Error Message : $PSItem.Exception.Messages"
                Log-Message "[$Timestamp] Error in Line Number: $PSItem.InvocationInfo.ScriptLineNumber"
                Log-Message "[$Timestamp] Script Stack Trace : "
                Log-Message $_.ScriptStackTrace
            }
        }

        $setparams = @{Name="SetSecurityDescriptor";ArgumentList=$acl.psobject.immediateBaseObject} + $invokeParams

        try {
            $output = Invoke-WmiMethod @setparams
            if ($output -eq $null) {
                throw "Failed to set security descriptor. Output is null."
            }
        } catch {
            $errorMessage = "Error occurred: $_"
            Write-Output $errorMessage
            Write-Output "Error occurred in SetSecurityDescriptor step. Kindly retry or do the setup manually."
            $Timestamp = (Get-Date).ToString($TimeStampFormat)
            Log-Message "[$Timestamp] $errorMessage"
            Log-Message "[$Timestamp] Error occurred in SetSecurityDescriptor step. Kindly retry or do the setup manually."
            exit 1
        }

        $global:nameSpace = 1
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] WMI Namespace Security with $operation operation was successful for computer = $computer"
    }
}

## FUNCTION FOR PART 2
## Function adds user to required User Groups
Function Set-UserLocalGroup {
    [cmdletBinding()]
    Param(
        [Parameter(Mandatory=$True,Position=0)][string] $user,
        [Parameter(Mandatory=$True,Position=1)][string] $operation
    )

    ## Groups to which the user is being added
    $LocalGroups = "Performance Monitor Users", "Event Log Readers", "Remote Management Users", "Distributed COM Users"

    ForEach ($group in $LocalGroups) {
        net localgroup $group $user /$operation 2>&1 | Out-Null
        if ($LASTEXITCODE -eq 0) {
            $global:addUser = $global:addUser + 1
            Log-Message "User successfully processed for the $group, Operation = $operation"
        }
        else {
            Log-Message "WARNING - User not found in the $group"
        }
    }
}

## FUNCTION FOR PART 3
## Function used to change Security Descriptor of SCManager and its Services
Function ChangeSecurityDescriptor {
    [cmdletBinding()]
    Param (
        [parameter(Mandatory=$true,Position=0)][string] $computer,
        [parameter(Mandatory=$true,Position=1)][string] $operation
    )

    #Getting SID of user so that we can add them in the security Descriptor.
    if ($isDomainPresent) {
        $objuser = New-Object System.Security.Principal.NTAccount($DomainName, $UserName)
    } else {
        $objuser = New-Object System.Security.Principal.NTAccount($ComputerName, $UserName)
    }
    $SecurityIdentifierObject = $objuser.Translate([System.Security.Principal.SecurityIdentifier])
    $UserSID = [string]$SecurityIdentifierObject.Value

    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] SID of the user selected is $UserSID"

    $SDDLToAppend = "(A;;LCRPRCLOCC;;;$UserSID)".ToString()

    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] SDDL String is $SDDLToAppend"

    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Trying to change sddl of ScManager and Win32 Services of Computer = $computer with sddl = $SDDLToAppend"
    #Get SDDL of SCManager
    $scManagerSSDL = sc.exe sdshow SCMANAGER
    $scManagerSSDL = [string]$scManagerSSDL
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Sc-Manager SDDL String before edit = $scManagerSSDL for Computer = $computer"
    $posSC = $scManagerSSDL.IndexOf([string]$SDDLToAppend)

    #Checking if the SDDL generated is already present in the SDDL of SCMAnager (i.e the user already has the permissions defined for it)
    if ($posSC -eq -1) {
        $positionSC = $scManagerSSDL.IndexOf("S:")
        $scDACL = $scManagerSSDL.Substring(0, $positionSC)
        $scSACL = $scManagerSSDL.Substring($positionSC)

        $newScSDDL = [string]$scDACL + [string]$SDDLToAppend + [string]$scSACL
        $newScSDDL = [string]$newScSDDL
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] new SDDL for ScManager for computer = $computer after appending = $newScSDDL "

        #Change SDDL of SCMANAGER
        $resultScManager = sc.exe \\$computer sdset SCMANAGER $newScSDDL
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] $resultScManager"
        if ($resultScManager -match "SUCCESS") {
            $global:secDescScManager = $global:secDescScManager + 1
        }
    } elseif (($posSC -ige 1) -and ($operation -eq "remove")) {
        $newScSDDL = $scManagerSSDL -replace $SDDLToAppend -replace ""

        #Change SDDL of SCMANAGER
        $resultScManager = sc.exe \\$computer sdset SCMANAGER $newScSDDL
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] $resultScManager"
        if ($resultScManager -match "SUCCESS") {
            $global:secDescScManager = $global:secDescScManager + 1
        }
    } else {
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] SCManager ssdl already present"
        $global:skipSecDescScManager = $global:skipSecDescScManager + 1
    }

    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] SDDL of SCManager Changed"

    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Start processing for win32_services for computer = $computer"

    # Get all Win32_Services.
    $Services = Get-Service | Select-Object Name

    if ($isCollectorHost) {
        $agentService = Get-Service -Name $agentServiceName | Select-Object Name
        $watchdogService = Get-Service -Name $watchdogServiceName | Select-Object Name

        $Services += $agentService
        $Services += $watchdogService
    }

    #Change SDDL of Services and give its access to the user. We will use the $SDDLToAppend built above.
    for ($i=0; $i -lt $Services.Length; $i++) {
        $value = [string]$Services[$i].Name
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] Trying to change security descriptor of Win32_Service = $value"

        $existingSDDL = sc.exe \\$computer sdshow $value
        $existingSDDL = [string]$existingSDDL
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] SDDL of Service before appending = $value = $existingSDDL"
        $pos = $existingSDDL.IndexOf([string]$SDDLToAppend)
        #Check if access is already present for that user.
        if($pos -eq -1) {
            $position = $existingSDDL.IndexOf("S:")
            if ($position -ge 0) {
                $DACL = $existingSDDL.Substring(0, $position)
                $SACL = $existingSDDL.Substring($position)

                $newSDDL = [string]$DACL + [string]$SDDLToAppend +[string]$SACL
                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] New SDDL String for win32_service = $value for computer=$computer is $newSDDL"
                $resultService = sc.exe \\$computer sdset $value $newSDDL
                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] $resultService"
                if ($resultService -match "SUCCESS") {
                    $global:secDesc = $global:secDesc + 1
                }
            } else {
                $newSDDL = [string]$existingSDDL + [string]$SDDLToAppend
                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] New SDDL String for win32_service = $value for computer=$computer is $newSDDL"
                $resultService = sc.exe \\$computer sdset $value $newSDDL
                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] $resultService"
                if ($resultService -match "SUCCESS") {
                    $global:secDesc = $global:secDesc + 1
                }
            }
        } elseif (($pos -ige 1) -and ($operation -eq "remove")) {
            $newSDDL = $existingSDDL -replace $SDDLToAppend -replace ""

                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] New SDDL String for win32_service = $value for computer=$computer is $newSDDL"
                $resultService = sc.exe \\$computer sdset $value $newSDDL
                $Timestamp = (Get-Date).ToString($TimeStampFormat)
                Log-Message "[$Timestamp] $resultService"
                if ($resultService -match "SUCCESS") {
                    $global:secDesc = $global:secDesc + 1
                }
        } else {
            $Timestamp = (Get-Date).ToString($TimeStampFormat)
            Log-Message "[$Timestamp] SSDL Already present for service = $value and Computer=$computer"
            $global:skipService = $global:skipService + 1
        }
    }
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Processing finished for win32_services for computer = $computer"
    $Timestamp = (Get-Date).ToString($TimeStampFormat)

    #Restart the winMgmt service.
    Log-Message "[$Timestamp] Trying to restart Wmimgmt for computer= $computer"
    Restart-Service winmgmt -force
}

# FUNCTION FOR PART 4.1
# Assigns user rights to accounts
function Grant-UserRight {
    [CmdletBinding(SupportsShouldProcess=$true)]
    param (
        [Parameter(Mandatory=$true, Position=0, ValueFromPipelineByPropertyName=$true, ValueFromPipeline=$true)][String] $UserName,
        [Parameter(Mandatory=$true, Position=1, ValueFromPipelineByPropertyName=$true)][PS_LSA.Rights[]] $Rights,
        [Parameter(ValueFromPipelineByPropertyName=$true, HelpMessage="Computer name")][String] $Computer
    )
    process {
        $lsa = New-Object PS_LSA.LsaWrapper($Computer)
        foreach ($Right in $Rights) {
          if ($PSCmdlet.ShouldProcess($UserName, "Grant $Right right")) {
            $lsa.AddPrivilege($UserName, $Right)
          }
        }
    }
}

# FUNCTION FOR PART 4.2
# Changes the User/Creds for the service passed to it
Function Set-ServiceAcctCreds() {
    [cmdletBinding()]
    Param (
        [parameter(Mandatory=$true, Position=0, ValueFromPipelineByPropertyName=$true)][string] $strServiceName,
        [parameter(Mandatory=$true, Position=1, ValueFromPipelineByPropertyName=$true)][string] $newUserName,
        [parameter(Mandatory=$true, Position=1, ValueFromPipelineByPropertyName=$true)][string] $newPassword
    )
  $filter = 'Name=' + "'" + $strServiceName + "'" + ''
  $strCompName = "localhost"
  $service = Get-WMIObject -ComputerName $strCompName -namespace "root\cimv2" -class Win32_Service -Filter $filter
  $service.Change($null, $null, $null, $null, $null, $null, $newUserName, $newPassword)
  $service.StopService()
  while ($service.Started) {
    sleep 2
    $service = Get-WMIObject -ComputerName $strCompName -namespace "root\cimv2" -class Win32_Service -Filter $filter
  }
}

## FUNCTION FOR PART 4 ## PARENT FUNCTION
Function DoCollectorHostConfig {
    [cmdletBinding()]
    Param (
        [parameter(Mandatory=$true, Position=0, ValueFromPipelineByPropertyName=$true)][string] $UserName
    )

    # Fetch Password from the user if not provided through the shell.
    if ($Password -eq $null -or $Password -eq "") {
        $Credential = Get-Credential -UserName $UserName -Message "Enter the password for $UserName"
        $Password = $Credential.GetNetworkCredential().Password
    }

    # Test UserName and Password for the user
    Add-Type -AssemblyName System.DirectoryServices.AccountManagement
    $type = [DirectoryServices.AccountManagement.ContextType]::Machine
    $PrincipalContext = [DirectoryServices.AccountManagement.PrincipalContext]::new($type)
    $verifiedCred = $PrincipalContext.ValidateCredentials($UserName, $Password)

    if (!$verifiedCred) {
        Write-Output "Incorrect credentials, Collector services are still running with Original User. Please re-run the script with correct credentials."
        exit
    }

    $agentPath = if ($Path -eq $null -or $Path -eq "") {Read-Host "Please enter collector installed location. (Press enter for default : C:\Program Files\LogicMonitor) "}
                    else { $Path }

    if ($null -eq $agentPath -or $agentPath -eq '') {
        $agentPath = "C:\Program Files\LogicMonitor"
    }

    $Acl = Get-Acl $agentPath

    if ($null -eq $Acl) {
        Write-Output "The agentPath is not $agentPath or is not provided. Please provide correct path where collector is installed and run the script again."
        Write-Output "Collector services will continue running under Administrator privileges."
        exit
    }
    if ($UserName.length -gt 63) {
        Write-Output "The length of UserName is not meeting the requirements."
    }

    try {
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] Stopping LogicMonitor Agent and Watchdog Services."

        sc.exe stop $agentServiceName
        sc.exe stop $watchdogServiceName
    } catch {
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Write-Output "Error in stopping LogicMonitor Agent and Watchdog Services."
        Log-Message "[$Timestamp] Error in stopping LogicMonitor Agent and Watchdog Services."
        Log-Message "[$Timestamp] Error - $PSItem.Exception.Messages"
    }

    ## PART 4.1
    try {
        $Rights = "SeServiceLogonRight","SeChangeNotifyPrivilege","SeAuditPrivilege","SeIncreaseQuotaPrivilege","SeAssignPrimaryTokenPrivilege","SeImpersonatePrivilege","SeCreateGlobalPrivilege","SeProfileSingleProcessPrivilege","SeDebugPrivilege"
        Grant-UserRight -UserName $UserName -Rights $Rights
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] Granted $Rights rights to $UserName"
    } catch {
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] Error in Part 4.1 Grant-UserRight call - $PSItem.Exception.Messages"
        Log-Message "[$Timestamp] $PSItem.Exception.StackTrace"
    }

    ## PART 4.2
    try {
        Set-ServiceAcctCreds -strServiceName $agentServiceName -newUserName $UserName -newPassword $Password
        Set-ServiceAcctCreds -strServiceName $watchdogServiceName -newUserName $UserName -newPassword $Password
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] Updated credentials for agent and watchdog services"
    } catch {
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] Error in Part 4.2 Set-ServiceAcctCreds call - $PSItem.Exception.Messages"
        Log-Message "[$Timestamp] $PSItem.Exception.StackTrace"
    }

    $Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("$UserName", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
    $Acl.SetAccessRule($Ar)
    Set-Acl $agentPath $Acl

    #Restart Watchdog
    $serviceStatus = (Get-Service "$watchdogServiceName").Status;
    if ($serviceStatus -eq "Stopped") {
        sc.exe start $watchdogServiceName
    }
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] The collector services are now switched to run under user $UserName."
}

$ResultString = New-Object -TypeName "System.Text.StringBuilder";
$LogSeparatorString = "#################################################################################"

if ($add) {
    ##PART 1
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 1 tasks start for Computer = $ComputerName"
    if ($isDomainPresent) {
        Set-WMINamespaceSecurity root add "$DomainName\$UserName" Enable,ReadSecurity,RemoteAccess -computer $ComputerName
    } else {
        Set-WMINamespaceSecurity root add "$UserName" Enable,ReadSecurity,RemoteAccess -computer $ComputerName
    }
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 1 tasks finish for Computer = $ComputerName"
    Write-Host "Part 1 Task - WMI Namespace Security Changes Completed"
    $Result = "[$Timestamp] $ComputerName = WmiNameSpaceSecuritySet=$global:nameSpace `n"
    [void]$ResultString.Append($Result)
    Log-Message $LogSeparatorString

    ##PART 2
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 2 tasks start for Computer = $ComputerName"
    Set-UserLocalGroup $UserName "add"
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 2 tasks finish for Computer = $ComputerName"
    Write-Host "Part 2 Task - Adding user to required groups Completed"
    $Result = "[$Timestamp] $ComputerName = UserAddedToGroupsCount=$global:addUser `n"
    [void]$ResultString.Append($Result)
    Log-Message $LogSeparatorString

    ##PART 3
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 3 SDDL Changes started - ADD"
    ChangeSecurityDescriptor "$ComputerName" "Add"
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    $Result = "[$Timestamp] $ComputerName = securityDescriptorChangedSuccessfully=$global:secDesc, ServicesSkipped=$global:skipService, ScManagerDescChanged=$global:secDescScManager `n"
    [void]$ResultString.Append($Result)
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 3 SDDL Changes Completed - ADD"
    Write-Host "Part 3 Task - SDDL Changes Completed"
    Log-Message $LogSeparatorString

    ##PART 4
    if ($isCollectorHost -eq $true) {
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] Part 4 Task - Collector Host Specific Config started."
        if ($isDomainPresent) {
            DoCollectorHostConfig -UserName "$DomainName\$UserName"
        } else {
            DoCollectorHostConfig -UserName "$ComputerName\$UserName"
        }
        $Timestamp = (Get-Date).ToString($TimeStampFormat)
        Log-Message "[$Timestamp] Part 4 Task - Collector Host Specific Config completed."
        Write-Host "Part 4 Task - Collector Host Specific Config completed."
        Log-Message $LogSeparatorString
    }
} else {
    ##Reversal Logic
    ## PART 1
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Tasks start for Computer = $ComputerName"
    if ($isDomainPresent) {
        Set-WMINamespaceSecurity root delete "$DomainName\$UserName" -computer $ComputerName
    } else {
        Set-WMINamespaceSecurity root delete "$UserName" -computer $ComputerName
    }
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Tasks finish for Computer = $ComputerName"
    Write-Host "Part 1 Task - WMI Namespace Security Changes Completed"
    $Result = "[$Timestamp] $ComputerName = WmiNameSpaceSecurityUnset=$global:nameSpace `n"
    [void]$ResultString.Append($Result)
    Log-Message $LogSeparatorString

    ##PART 2
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 2 reversal tasks start for Computer = $ComputerName"
    Set-UserLocalGroup $UserName "delete"
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 2 reversal tasks finish for Computer = $ComputerName"
    Write-Host "Part 2 Task - Removing user from required groups Completed"
    $Result = "[$Timestamp] $ComputerName = UserRemovedFromGroupCount=$global:addUser `n"
    [void]$ResultString.Append($Result)
    Log-Message $LogSeparatorString

    ##PART 3
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 3 SDDL Changes started - REMOVE"
    ChangeSecurityDescriptor "$ComputerName" "Remove"
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    $Result = "[$Timestamp] $ComputerName = securityDescriptorChangedSuccessfully=$global:secDesc, ServicesSkipped=$global:skipService, ScManagerDescChanged=$global:secDescScManager `n"
    [void]$ResultString.Append($Result)
    $Timestamp = (Get-Date).ToString($TimeStampFormat)
    Log-Message "[$Timestamp] Part 3 SDDL Changes Completed - REMOVE"
    Write-Host "Part 3 Task - SDDL Changes Completed"
    Log-Message $LogSeparatorString
}

Log-Message "RESULTS == "
Log-Message "$ResultString"
}

```

``` powershell
SetLeastPrivileges -add -username "domain\wmiuser"
```
